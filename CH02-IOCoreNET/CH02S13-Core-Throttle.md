# 核心部件：Throttle

Throttle 并不是一个类，而是几个全局变量、统计值和一组函数。

通过在下面的方法中调用这些相关的函数，实现了Throttle

  - NetAccept::do_blocking_accept
  - NetAccept::acceptFastEvent
  - UnixNetVConnection::connectUp

ATS 的 Throttle 的设计分为：

  - Net Throttle，这包括：
    - ACCEPT 方向
    - CONNECT 方向
  - Emergency Throttle，这包括：
    - 最大文件句柄控制部分
    - 保留文件句柄用于 backdoor 的部分

## 定义

由于 Throttle 的实现并不是一个类，因此分散在多个源代码文件中。下面的介绍中，为了方便，对顺序作了调整。

### 常量和变量

首先是几个常量宏定义和枚举值：

```
source: P_UnixNet.h
// 警告信息输出时，需要间隔 24小时 及以上，避免重复发送大量警告信息
#define TRANSIENT_ACCEPT_ERROR_MESSAGE_EVERY HRTIME_HOURS(24)
// 警告信息输出时，需要间隔 10秒 及以上，避免重复发送大量警告信息
#define NET_THROTTLE_MESSAGE_EVERY HRTIME_MINUTES(10)

// 文件句柄保留数量，保留给Cache子系统。
#define THROTTLE_FD_HEADROOM (128 + 64) // CACHE_DB_FDS + 64

// 为何 ACCEPT 要预留 10% ？？？
#define NET_THROTTLE_ACCEPT_HEADROOM 1.1  // 10%
#define NET_THROTTLE_CONNECT_HEADROOM 1.0 // 0%

// 在ACCEPT限定值发生时，连续放弃接下来的 5 个新连接，并向其发送连接达到限定值的信息
// 这只有在开启“向客户端发送连接达到限定值信息”的功能时，才会生效，默认此功能是关闭的
// also the 'throttle connect headroom'
#define THROTTLE_AT_ONCE 5

// 剩余文件描述符限定值
#define EMERGENCY_THROTTLE 16
// 剩余文件描述符紧急限定值
#define HYPER_EMERGENCY_THROTTLE 6

// 定义 Net Throttle 的类型：ACCEPT 和 CONNECT
enum ThrottleType {
  ACCEPT,
  CONNECT,
};
```

然后是几个相关的全局变量，根据初始化的顺序进行解释：

```
source: UnixNet.cc
// 操作系统对于文件打开数的限定(ATS对于文件打开总数的硬限定)
//     其值受到配置项：proxy.config.system.file_max_pct 的影响，默认值为 90%
//     为系统允许的最大文件打开数 * file_max_pct
// 首先，调用 Main.cc:1535 init_system() 完成初始化：
//     fds_limit = ink_max_out_rlimit(RLIMIT_NOFILE, true, false);
// 然后再调用 Main.cc:1538 adjust_sys_settings()：
//     首先，根据 proxy.config.system.file_max_pct 进行调整，
//     然后，根据 proxy.config.net.connections_throttle 进行再调整
//     如 connections_throttle 超过 fds_limit - THROTTLE_FD_HEADROOM 值时会：
//       尝试重设操作系统的文件打开数限定值为 connections_throttle，并更新 fds_limit 
// 最后再调用 Main.cc:1664 check_fd_limit()：
//     如果 fds_limit - THROTTLE_FD_HEADROOM 小于 1 时会抱错并停止ATS进程
// 总之，fds_limit 为ATS能够打开的文件句柄数的最大值。
int fds_limit = 8000;

// 当前允许的最大 Socket 句柄数（不包含backdoor，cache 文件句柄等）
//     其值来自配置项：proxy.config.net.connections_throttle
// 首先，调用 Main.cc:1711 ink_net_init()，由其再调用 Net.cc:43 configure_net()
//     从配置文件中读取该值，
//     并注册处理函数change_net_connections_throttle()，在此配置项变更时重设该值。
// 然后再调用 Main.cc:1771 netProcessor.start()，由其调用 UnixNetProcessor.cc:405 change_net_connections_throttle()
//     change_net_connections_throttle() 函数的实现可能有bug，
//       它忽略了来自配置文件中的值
//       而直接通过 fds_limit 来计算该值，因此 proxy.config.net.connections_throttle 不能热更新
int fds_throttle;

// 当前允许的最大连接数，其值受到配置项：proxy.config.net.connections_throttle 的影响
// 当配置项为 -1 时，会使用 fds_limit - THROTTLE_FD_HEADROOM 的值进行默认设置
// 当配置项 >= 0 时，会使用 配置项 的值进行设置，但是其值仍然不能超过 fds_throttle 的值
int net_connections_throttle;

// 用来记录上一次警告信息输出的时间，用来限制信息输出的频率
ink_hrtime last_throttle_warning;
ink_hrtime last_shedding_warning;
ink_hrtime emergency_throttle_time;
ink_hrtime last_transient_accept_error;
```

### 内部方法

```
source: P_UnixNet.h
// PRIVATE: 这是一个 Throttle 内部方法
// 根据统计值，计算出当前已经打开的连接数。
// 但是对于 Net Throttle 的类型，会有一个保留值(惩罚值)：
//   对于 ACCEPT 返回值会比实际打开的连接数多10%
//     这里我理解为一种保险措施：
//       只要控制住 ACCEPT 进来的连接数，那么就不会有过多的 CONNECT 连接出去，
//       这样就可以在接近连接耗尽（10%余量的情况下）时，优先暂缓 ACCEPT 动作，
//       由于ATS是一个PROXY，没有进来的连接，那么就几乎不会有 CONNECT 连接出去。
//   对于 CONNECT 返回值为实际打开的连接数
TS_INLINE int
net_connections_to_throttle(ThrottleType t)
{
  double headroom = t == ACCEPT ? NET_THROTTLE_ACCEPT_HEADROOM : NET_THROTTLE_CONNECT_HEADROOM;
  int64_t sval    = 0;

  NET_READ_GLOBAL_DYN_SUM(net_connections_currently_open_stat, sval);
  int currently_open = (int)sval;
  // deal with race if we got to multiple net threads
  if (currently_open < 0)
    currently_open = 0;
  return (int)(currently_open * headroom);
}

// PRIVATE: 这是一个 Throttle 内部方法
//   目前仅由 check_net_throttle() 调用
// 用来判定当前是否处于正在限制连接的时刻
// 返回值：
//   true: 当前正在限制新建连接中
//  false: 当前未对新建连接进行限制
TS_INLINE bool
emergency_throttle(ink_hrtime now)
{
  return (bool)(emergency_throttle_time > now);
}
```

### 外部方法

```
source: P_UnixNet.h
// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 用于检测当前的 Throttle 状态
//   首先判定已经打开的网络连接是否超过限制（根据ACCEPT／CONNECT类型分别加权判定）
//   同时还要检测当前是否处于 Throttle 状态。
// 返回值：
//   true: 已经达到 Throttle 状态
//  false: 未达到 Throttle 状态
TS_INLINE bool
check_net_throttle(ThrottleType t, ink_hrtime now)
{
  int connections = net_connections_to_throttle(t);

  if (connections >= net_connections_throttle)
    return true;

  if (emergency_throttle(now))
    return true;

  return false;
}

//
// Emergency throttle when we are close to exhausting file descriptors.
// Block all accepts or connects for N seconds where N
// is the amount into the emergency fd stack squared
// (e.g. on the last file descriptor we have 14 * 14 = 196 seconds
// of emergency throttle).
//
// Hyper Emergency throttle when we are very close to exhausting file
// descriptors.  Close the connection immediately, the upper levels
// will recover.
//
// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 用于检测传入的文件句柄是否已经接近耗尽值
// 在 ATS 中主要由两类文件句柄的消耗：
//   网络连接（包含Client与ATS之间的连接 和 ATS与Origin Server之间的连接）
//   磁盘访问（这包含 CACHE_DB_FDS 128个 文件句柄，还有 64个 保留文件句柄）
// 而对于文件句柄的限制，也分为两个层级：
//   第一层级，可用文件句柄数量小于 16，为 EMERGENCY_THROTTLE
//     触发该等级时，会设置 emergency_throttle_time，这样整个ATS会暂停创建新连接一段时间
//   第二层级，可用文件句柄数量小于  6，为 HYPER_EMERGENCY_THROTTLE
//     在执行第一层级动作的基础上，还会直接关闭这个文件句柄
// 该函数使用文件句柄是一个 int 类型的字增长变量的特性实现对剩余可用文件句柄的判定
// 返回值：
//   true：达到文件句柄耗尽限制（文件句柄可能被关闭）
//  false：未达限制
// 暂停时间：
//   首先，计算出当剩余可用文件句柄低于 EMERGENCY_THROTTLE（16） 值多少，保存到 over 变量。
//   例如，当前可用文件句柄为10，这样 over 值为 16 - 10 ＝ 6
//   则暂停时间为 over * over = 6 * 6 = 36 秒
//   将当前时间加上36秒之后，保存到 emergency_throttle_time，在到达该时间之前都不会创建新连接。
TS_INLINE bool
check_emergency_throttle(Connection &con)
{
  int fd        = con.fd;
  int emergency = fds_limit - EMERGENCY_THROTTLE;
  if (fd > emergency) {
    int over                = fd - emergency;
    emergency_throttle_time = Thread::get_hrtime() + (over * over) * HRTIME_SECOND;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "too many open file descriptors, emergency throttling");
    int hyper_emergency = fds_limit - HYPER_EMERGENCY_THROTTLE;
    if (fd > hyper_emergency)
      con.close();
    return true;
  }
  return false;
}
```

### 加载配置项

```
// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 用来更新 net_connections_throttle 值
// 在配置文件更新之后，重设 net_connections_throttle 的值
// 但是这里好像有 bug，这里并未接受传入的值，而是根据 fds_limit 值进行了计算。
TS_INLINE int
change_net_connections_throttle(const char *token, RecDataT data_type, RecData value, void *data)
{
  (void)token;
  (void)data_type;
  (void)value;
  (void)data;
  int throttle = fds_limit - THROTTLE_FD_HEADROOM;
  if (fds_throttle < 0) // ??? bug ??? fds_throttle should be @value ?
    net_connections_throttle = throttle;
  else {
    net_connections_throttle = fds_throttle;  // ??? bug ??? fds_throttle should be @value ?
    if (net_connections_throttle > throttle)
      net_connections_throttle = throttle;
  }
  return 0;
}
```

### 错误判定与警告信息输出

```
source: P_UnixNet.h
// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 针对 accept() 的错误状态进行判定
// 参数：
//   res 在 accept() 返回值小于 0 时，取 errno 值。
// 返回值：
//   1: 继续重试
//   0: 输出警告，可以重试
//  -1: 严重错误
// 1  - transient
// 0  - report as warning
// -1 - fatal
TS_INLINE int
accept_error_seriousness(int res)
{
  switch (res) {
  case -EAGAIN:
  case -ECONNABORTED:
  case -ECONNRESET: // for Linux
  case -EPIPE:      // also for Linux
    return 1;
  case -EMFILE:
  case -ENOMEM:
#if defined(ENOSR) && !defined(freebsd)
  case -ENOSR:
#endif
    ink_assert(!"throttling misconfigured: set too high");
#ifdef ENOBUFS
  case -ENOBUFS:
#endif
#ifdef ENFILE
  case -ENFILE:
#endif
    return 0;
  case -EINTR:
    ink_assert(!"should be handled at a lower level");
    return 0;
#if defined(EPROTO) && !defined(freebsd)
  case -EPROTO:
#endif
  case -EOPNOTSUPP:
  case -ENOTSOCK:
  case -ENODEV:
  case -EBADF:
  default:
    return -1;
  }
}

// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 输出 accept() 时遇到的错误信息，并限制错误信息的输出频率（24小时）
TS_INLINE void
check_transient_accept_error(int res)
{
  ink_hrtime t = Thread::get_hrtime();
  if (!last_transient_accept_error || t - last_transient_accept_error > TRANSIENT_ACCEPT_ERROR_MESSAGE_EVERY) {
    last_transient_accept_error = t;
    Warning("accept thread received transient error: errno = %d", -res);
#if defined(linux)
    if (res == -ENOBUFS || res == -ENFILE)
      Warning("errno : %d consider a memory upgrade", -res);
#endif
  }
}

// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 避免输出大量重复的“连接已达到溢出限定值”的警告信息
// * 这个方法没有任何地方使用
// 限制输出“number of connections reaching shedding limit”警告信息的时间间隔为 10 秒及以上
TS_INLINE void
check_shedding_warning()
{
  ink_hrtime t = Thread::get_hrtime();
  if (t - last_shedding_warning > NET_THROTTLE_MESSAGE_EVERY) {
    last_shedding_warning = t;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "number of connections reaching shedding limit");
  }
}

// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 避免输出大量重复的“连接已达限定值”的警告信息
// 限制输出“too many connections, throttling”警告信息的时间间隔为 10 秒及以上
TS_INLINE void
check_throttle_warning()
{
  ink_hrtime t = Thread::get_hrtime();
  if (t - last_throttle_warning > NET_THROTTLE_MESSAGE_EVERY) {
    last_throttle_warning = t;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "too many connections, throttling");
  }
}
```

### 输出警告信息到客户端

```
source: UnixNetAccept.cc
// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 当开启了 throttle_error_message 功能（开启此功能的代码已经不见了）后，遇到 Throttle 状态时：
//   ATS会连续 accept() 最多 THROTTLE_AT_ONCE 个连接
//   主动向这些连接发送 throttle_error_message 内的信息
//   最后关闭这些连接
//   同时阻塞 accept 操作 proxy.config.net.throttle_delay 毫秒，默认值为：50毫秒
// 如果没有开启（当前默认值）
//   则不会调用此方法，而只是简单的阻塞 accept 操作
//
// Send the throttling message to up to THROTTLE_AT_ONCE connections,
// delaying to let some of the current connections complete
//
static int 
send_throttle_message(NetAccept *na)
{
  struct pollfd afd;
  Connection con[100];
  char dummy_read_request[4096];

  afd.fd     = na->server.fd;
  afd.events = POLLIN;

  int n = 0;
  while (check_net_throttle(ACCEPT, Thread::get_hrtime()) && n < THROTTLE_AT_ONCE - 1 && (socketManager.poll(&afd, 1, 0) > 0)) {
    int res = 0;
    if ((res = na->server.accept(&con[n])) < 0)
      return res;
    n++;
  }
  safe_delay(net_throttle_delay / 2); 
  int i = 0;
  for (i = 0; i < n; i++) {
    socketManager.read(con[i].fd, dummy_read_request, 4096);
    socketManager.write(con[i].fd, unix_netProcessor.throttle_error_message, strlen(unix_netProcessor.throttle_error_message));
  }
  safe_delay(net_throttle_delay / 2); 
  for (i = 0; i < n; i++)
    con[i].close();
  return 0;
}
```

## Throttle 的实现

由于在 缓存子系统 中总是使用固定数量的文件句柄，而 ATS 又为其它系统（如日志）保留了 64 个文件句柄（这64个保留的文件句柄的用途还需要再确认），这样可能使用到大量文件句柄的就只有 网络子系统 了。

Throttle 在网络子系统中分成两个部分来实现：

  - NetAccept 接受新连接之后，对 Throttle 进行判断
  - connectUp 在创建一个文件描述符用于connect()之前，对 Throttle 进行判断

这部分的实现，在 6.0.x 分支上有一个bug：TS-4879，由于 check_emergency_throttle() 在极限情况下会直接关闭文件句柄，但是在代码中没有处理这个状况，而且继续使用已经被关闭的文件句柄创建了NetVC，由此导致这个NetVC只能被缺省的超时控制来管理，导致NetVC在较长的时间里不关闭。

下面采用修复后的代码进行分析：

## 在 NetAccept 中的实现

NetAccept 有多种运行模式，首先分析在 Dedicated EThread 中以 blocking accept 方式运行的：

```
int
NetAccept::do_blocking_accept(EThread *t) 
{
  int res                = 0;
  int loop               = accept_till_done;
  UnixNetVConnection *vc = NULL;
  Connection con;

  // do-while for accepting all the connections
  // added by YTS Team, yamsat
  do {
    ink_hrtime now = Thread::get_hrtime();

    // Throttle accepts
    // 对于 backdoor 服务不进行限制，
    //   由于 backdoor 用于实现 traffic_server 与 traffic_manager 之间的心跳，因此必须保证其服务的可用性。
    // 然后判断 ACCEPT 方向是否已经达到连接限制，或者处于文件描述符限制的状态
    while (!backdoor && check_net_throttle(ACCEPT, now)) {
      // 已经达到限制
      // 输出警告信息
      check_throttle_warning();
      // 未设置 throttle_error_message （默认）
      if (!unix_netProcessor.throttle_error_message) {
        // 阻塞 net_throttle_delay 毫秒
        // 由于 do_blocking_accept() 运行在 Dedicated EThread 中，因此不会阻塞
        safe_delay(net_throttle_delay);
      } else if (send_throttle_message(this) < 0) {
        // 否则，发送 throttle_error_message
        goto Lerror;
      }
      // 更新 now 值用于 check_net_throttle
      now = Thread::get_hrtime();
    }   

    if ((res = server.accept(&con)) < 0) {
    Lerror:
      // 接受新连接失败时，限制错误信息的输出频率
      int seriousness = accept_error_seriousness(res);
      // 遇到不严重的错误，如EAGAIN等
      if (seriousness >= 0) { // not so bad
        // 如果需要发送警告信息
        if (!seriousness)     // bad enough to warn about
          check_transient_accept_error(res);
        // 阻塞 net_throttle_delay 毫秒，以节省CPU占用
        safe_delay(net_throttle_delay);
        return 0;
      }   
      if (!action_->cancelled) {
        SCOPED_MUTEX_LOCK(lock, action_->mutex, t);
        action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        Warning("accept thread received fatal error: errno = %d", errno);
      }
      return -1;
    }

    // 由于是阻塞方式 accept() 在 accept() 之前判断的连接限定值可能已经是很久之前的事情了
    // 所以当我们拿到一个合理连接时仍然需要根据 FD 的值判定它是否超出了文件描述符限制
    // The con.fd may exceed the limitation of check_net_throttle() because we do blocking accept here.
    if (check_emergency_throttle(con)) {
      // 如果超出了限制，还需要检查这个 FD 是否已经被关闭了
      // The `con' could be closed if there is hyper emergency
      if (con.fd == NO_FD) {
        return 0;
      }
    }

    // Use 'NULL' to Bypass thread allocator
    vc = (UnixNetVConnection *)this->getNetProcessor()->allocate_vc(NULL);
...
  } while (loop);

  return 1;
}
```

然后再分析在 Regular EThread 中以non-blocking accept模式运行：

```
// 通过周期性Event，acceptFastEvent 被周期性回调
int   
NetAccept::acceptFastEvent(int event, void *ep)
{   
  Event *e = (Event *)ep;
  (void)event;
  (void)e;
  int bufsz, res = 0;
  Connection con;
        
  PollDescriptor *pd     = get_PollDescriptor(e->ethread);
  UnixNetVConnection *vc = NULL;
  int loop               = accept_till_done;
    
  do {
    // 对于 backdoor 服务不进行限制，
    //   由于 backdoor 用于实现 traffic_server 与 traffic_manager 之间的心跳，因此必须保证其服务的可用性。
    // 然后判断 ACCEPT 方向是否已经达到连接限制
    // 注意，这里对文件描述符限制状态的判定可能不够准确
    if (!backdoor && check_net_throttle(ACCEPT, Thread::get_hrtime())) {
      // ifd 这个成员好像没用了
      ifd = NO_FD;
      // 连接被限定时直接返回到 EventSystem
      return EVENT_CONT;
    }

    socklen_t sz = sizeof(con.addr);
    int fd       = socketManager.accept4(server.fd, &con.addr.sa, &sz, SOCK_NONBLOCK | SOCK_CLOEXEC);
    con.fd       = fd;
...
    // check return value from accept()
    if (res < 0) {
      res = -errno;
      if (res == -EAGAIN || res == -ECONNABORTED
#if defined(linux)
          || res == -EPIPE
#endif
          ) {
        goto Ldone;
      } else if (accept_error_seriousness(res) >= 0) {
        // 接受新连接失败时，限制错误信息的输出频率
        check_transient_accept_error(res);
        // 这里没有阻塞，跳转到 Ldone 后直接返回到 EventSystem 等待下一次回调
        // 每次被回调时，首先会检查是否处于连接限定的状态
        goto Ldone;
      }
      if (!action_->cancelled)
        action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
      goto Lerror;
    }
    // 在创建VC之前，没有再次通过 check_emergency_throttle 来判断状态 FD，
    // 这是因为：
    //   我们在非阻塞accept()调用，从判定连接限制到accept()调用之间只有非常短的时间，没有很大的必要性
    //   虽然在多线程的环境下，但是对同一端口进行accept()调用的所有 NetAccept 都共享同一个 mutex
    //   每接受一个新连接，就会检查一次连接限定
    // 但是这里也存在隐患：同时会有其它端口同时进行 accept() 调用。
    // 由于没有调用 check_emergency_throttle：
    //   在接受新连接之后，并未对 FD 进行检查，也就无法对 emergency_throttle_time 进行更新
    //   因此，emergency_throttle_time 的更新仅仅在 connectUp 中进行更新
    // 由于 check_net_throttle 对于 ACCEPT 方向有 10% 的预留，因此：
    //   在 accept() 返回一个 FD 时，距离文件描述符的限定还有 10% 的余量
    vc = (UnixNetVConnection *)this->getNetProcessor()->allocate_vc(e->ethread);
...
  } while (loop);

Ldone:
  return EVENT_CONT;

Lerror:
  server.close();
  e->cancel();
  if (vc)
    vc->free(e->ethread);
  NET_DECREMENT_DYN_STAT(net_accepts_currently_open_stat);
  delete this;
  return EVENT_DONE;
}
```

### 在 connectUp 中的实现

```
int
UnixNetVConnection::connectUp(EThread *t, int fd)
{
  int res; 

  thread = t;
  // 首先判断 CONNECT 方向是否已经达到连接限制，或者处于文件描述符限制的状态
  if (check_net_throttle(CONNECT, submit_time)) {
    // 已经达到限制
    // 输出警告信息
    check_throttle_warning();
    // 回掉状态机 NET_EVENT_OPEN_FAILED 事件
    action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)-ENET_THROTTLING);
    free(t);
    return CONNECT_FAILURE;
  }
...
  // 接下来判断文件描述符是否已经打开了
  // 如果是从TS API发起的请求，应该传入一个已经打开的文件描述符
  // If this is getting called from the TS API, then we are wiring up a file descriptor
  // provided by the caller. In that case, we know that the socket is already connected.
  if (fd == NO_FD) {
    // 如果没有打开，则立即创建一个
    // 由于是多线程的环境，因此即使在前面判断过，在打开一个文件描述符之后，仍然可能出现超出限定的问题
    // Due to multi-threads system, the fd returned from con.open() may exceed the limitation of check_net_throttle().
    res = con.open(options);
    if (res != 0) {
      goto fail;
    }
  } else {
    int len = sizeof(con.sock_type);

    // This call will fail if fd is not a socket (e.g. it is a
    // eventfd or a regular file fd.  That is ok, because sock_type
    // is only used when setting up the socket.
    safe_getsockopt(fd, SOL_SOCKET, SO_TYPE, (char *)&con.sock_type, &len);
    safe_nonblocking(fd);
    con.fd           = fd;
    con.is_connected = true;
    con.is_bound     = true;
  }

  // 因此，需要再次调用 check_emergency_throttle 来验证这个文件描述符是否超出了限定
  if (check_emergency_throttle(con)) {
    // 如果返回 true，则表示超出限定值，同时设置了惩罚时间，此时 accept 会被暂停一段时间
    //   我们需要进一步判定该 FD 是否被关闭
    // The `con' could be closed if there is hyper emergency
    if (con.fd == NO_FD) {
      // We need to decrement the stat because close_UnixNetVConnection only decrements with a valid connection descriptor.
      NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, -1);
      // Set errno force to EMFILE (reached limit for open file descriptors)
      errno = EMFILE;
      res   = -errno;
      goto fail;
    }
    // 如果已经达到限定值，但是该连接并未被关闭，仍然可以继续使用该连接
  }

  // Must connect after EventIO::Start() to avoid a race condition
  // when edge triggering is used.
  if (ep.start(get_PollDescriptor(t), this, EVENTIO_READ | EVENTIO_WRITE) < 0) {
    res = -errno;
    Debug("iocore_net", "connectUp : Failed to add to epoll list : %s", strerror(errno));
    goto fail;
  }

  if (fd == NO_FD) {
    res = con.connect(NULL, options);
    if (res != 0) {
      goto fail;
    }
  }
...
}
```

可以看到在 CONNECT 方向上没有执行 safe_delay() 的操作，这是因为：

  - safe_delay() 是一个阻塞操作
  - 当状态机遇到 NET_EVENT_OPEN_FAILED 时，错误代码为 -ENET_THROTTLING 时应该执行 reschedule 操作

### THROTTLE_FD_HEADROOM 限定

这是 ATS 为了非网络通信而保留的文件描述符，注释中解释了：

  - 128 来自 CACHE_DB_FDS
  - 64 没有解释，这里姑且认为是 Other

我查找了 CACHE_DB_FDS 的定义：

```
cache/I_CacheDefs.h:#define CACHE_DB_FDS 128
```

但是却没有找到任何地方使用了这个宏，与社区中其它开发者进行讨论之后：

  - 每个硬盘 8 个文件描述符
    - 随源代码发布的缺省配置文件records.config里面 proxy.config.cache.threads_per_disk INT 8
    - 但是源代码里缺省为 12，如果在records.config里面没有定义该配置项则为 12
  - 早期的硬件环境是 SCSI 硬盘
    - 每个SCSI通道最多可以连接 16 块磁盘

因此，用于CACHE_DB的文件描述符数量为：8 * 16 ＝ 128。

对于另外 64 个文件描述符的用途，目前还不清楚，猜测可能与DNS、日志系统有关系。

## 参考资料

- [P_UnixNet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [UnixNet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)
- [Net.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/Net.cc)
- [Main.cc](http://github.com/apache/trafficserver/tree/master/proxy/Main.cc)
- [UnixNetAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetAccept.cc)
- [UnixNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
