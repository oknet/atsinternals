# 核心组件：SocksEntry

SocksEntry 是ATS作为Socks Client向Socks代理服务器发起Socks请求的状态机。

当 上层SM 调用 netProcessor.connect_re() 的时候，根据 socks 的配置判断出是否需要通过 Socks代理服务器 发起到目标IP和端口的连接，但是这一切对 上层SM 都是透明的。

## 定义

```
// 与SSL协议一样，Socks协议可以理解为一种传输层的协议
//   区别是Socks只有握手过程，但是没有通信加解密功能
// 因此，这里也像SSLNetVConnection一样定义了SocksNetVC
// 不过由于Socks协议不需要提供通信加解密功能，
// 所以，它可以与UnixNetVConnection共享net_read_io和write_to_net_io
typedef UnixNetVConnection SocksNetVC;

struct SocksEntry : public Continuation {
  // 用来保存发送给 Socks代理服务器 的Socks请求
  // 同时用来接收自 Socks代理服务器 返回的Socks响应
  MIOBuffer *buf;
  IOBufferReader *reader;

  // 由 connect_re() 创建的用来与 Socks代理服务器连接的 NetVC
  SocksNetVC *netVConnection;

  // Changed from @a ip and @a port.
  // 保存目标的IP地址和端口，将通过Socks代理服务器来访问该目标
  IpEndpoint target_addr; ///< Original target address.
  // Changed from @a server_ip, @a server_port.
  // 保存Socks代理服务器的IP地址和端口（下面的英文注释是错误的）
  IpEndpoint server_addr; ///< Origin server address.

  // 用来保存当前尝试与 Socks代理服务器 连接的次数
  int nattempts;

  // 在Socks握手完成后，会回调该Action连接的状态机
  Action action_;
  // 在与 Socks代理服务器 建立连接的所有尝试失败后，将向上述状态回调 NET_EVENT_OPEN_FAILED，并传递 -lerrno 值
  int lerrno;
  // 通过 schedule_*() 方法设置超时时间，保存其返回值
  // 如遇到 Socks 超时，SocksEntry 会收到 EVENT_INTERVAL 事件
  //   注意：如遇到 NetVConnection 超时，仍然会收到 EVENT_*_TIMEOUT 事件
  Event *timeout;
  // 与 Socks代理服务器 通信时采用的协议版本
  unsigned char version;

  // 表示ATS与Socks代理服务器正处于握手过程中，
  // true: ATS已经向Socks代理服务器发出了请求
  bool write_done;

  // 用于实现Socks5认证的函数指针，Socks4协议时为NULL
  SocksAuthHandler auth_handler;
  // 用于表示发起的Socks请求中的Socks命令字
  // 如果其值为 NORMAL_SOCKS，则表示 CONNECT 命令，也是SocksEntry能够支持的
  // 如果其值不为 NORMAL_SOCKS，ATS 则会在发送了Socks请求之后与目标服务器建立 Blind Tunnel
  unsigned char socks_cmd;

  // socks server selection:
  // 如果有多个 Socks代理服务器可以使用时，将逐个尝试
  ParentConfigParams *server_params;
  HttpRequestData req_data; // We dont use any http specific fields.
  ParentResult server_result;

  // 初始化回调函数
  int startEvent(int event, void *data);
  // 主处理回调函数
  int mainEvent(int event, void *data);
  // 当存在多个可用的 Socks代理服务器 时，通过findServer选择下一个
  void findServer();
  // 初始化 SocksEntry 实例
  void init(ProxyMutex *m, SocksNetVC *netvc, unsigned char socks_support, unsigned char ver);
  // 回调上层状态机 连接成功 或 失败，然后释放SocksEntry
  void free();

  // 构造函数
  SocksEntry()
    : Continuation(NULL),
      netVConnection(0),
      nattempts(0),
      lerrno(0),
      timeout(0),
      version(5),
      write_done(false),
      auth_handler(NULL),
      socks_cmd(NORMAL_SOCKS)
  {
    memset(&target_addr, 0, sizeof(target_addr));
    memset(&server_addr, 0, sizeof(server_addr));
  }
};

typedef int (SocksEntry::*SocksEntryHandler)(int, void *);

extern ClassAllocator<SocksEntry> socksAllocator;
```

## 方法

SocksEntry::init(mutex, vc, socks_support, ver)

- 用于初始化 SocksEntry 对象
  - 初始化用于建立Socks会话的交互缓冲区
  - 设置Socks协议版本
  - 初始回调函数为 startEvent
  - 获得目标IP和端口
  - 设置一个定时回调，实现对 Socks代理服务器 连接建立超时的管理
- 参数 mutex 为发起该连接的状态机的 mutex
  - 也就是 netProcessor.connect_re() 调用者的 mutex
- 参数 vc 为已经设置了目标IP和端口，即将发起 connect() 调用的 SocksNetVC
  - 该方法从vc中获得目标IP和端口并保存在 target_addr 成员
  - 然后将选中的 Socks 代理服务器的IP和端口保存在 server_addr 成员
- 参数 socks_support 默认为 NORMAL_SOCKS
  - 该参数主要用于重新建立到 Socks 代理服务器的连接，例如：在连接超时，多个Socks代理服务器可用时
  - 设置 options.socks_support 为 NO_SOCKS 后，调用 connect_re() 方法就不会再进入到 SocksEntry
- 参数 ver 表示 Socks 协议的版本
  - 默认为 (unisgned char) 5，表示采用 Socks 5 协议与 Socks代理服务器 建立通信
- 作为 init() 方法的调用者，将会使用 server_addr 成员替代 vc 的目标IP和端口
  - 这样在稍后发起的 connect() 调用，就会与 Socks 代理服务器建立 TCP 连接

```
void
SocksEntry::init(ProxyMutex *m, SocksNetVC *vc, unsigned char socks_support, unsigned char ver)
{
  // 初始化SocksEntry的mutex，与调用者共享同一个 mutex
  mutex = m;
  // 创建会话交互缓冲区
  buf = new_MIOBuffer();
  reader = buf->alloc_reader();

  // 设置 socks_cmd 默认值，通常为 NORMAL_SOCKS(0)
  socks_cmd = socks_support;

  // 设置所采用的 Socks 协议的版本
  if (ver == SOCKS_DEFAULT_VERSION)
    version = netProcessor.socks_conf_stuff->default_version;
  else
    version = ver;

  // 设置事件回调函数为 startEvent
  SET_HANDLER(&SocksEntry::startEvent);

  // 这里在 6.0.x 分支上存在bug，应该为 vc->get_remote_addr()
  // 从 vc 上获得目标IP和端口，保存在 target_addr 成员
  // 将来构造 Socks 请求时，通过 target_addr 成员构造目标IP和端口
  ats_ip_copy(&target_addr, vc->get_local_addr());

// 该宏在 I_Socks.h 中定义
#ifdef SOCKS_WITH_TS
  // 初始化 req_data，在调用 findParent() 方法查找 socks.config 规则时使用
  req_data.hdr = 0;
  req_data.hostname_str = 0;
  req_data.api_info = 0;
  req_data.xact_start = time(0);

  // 这里表示仅支持 IPv4 作为目标地址
  assert(ats_is_ip4(&target_addr));
  // 保存目标IP和端口到 req_data.dest_ip
  ats_ip_copy(&req_data.dest_ip, &target_addr);

  // we dont have information about the source. set to destination's
  // 使用目标IP和端口初始化 req_data.src_ip 只是为了保证此处有一个合法的信息
  ats_ip_copy(&req_data.src_ip, &target_addr);

  // 通过Socks配置文件，加载可用的 Socks 代理服务器
  server_params = SocksServerConfig::acquire();
#endif

  // 重置尝试次数，然后开始查找可用的 Socks 代理服务器
  nattempts = 0;
  findServer();

  // 创建一个定时回调，在 n 秒之后回调 SocksEntry::startEvent 判定超时情况
  timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->server_connect_timeout));
  write_done = false;
}
```

SocksEntry::findServer()

- 用来找到可以个可用的 Socks代理服务器
- 在 socks.config 中可以为特定的目标指定 Socks代理服务器，这样可以根据不同的用途使用不同的 Socks代理服务器
- 如果在 socks.config 中没有找到，还可以从 records.config 中查找 proxy.config.socks.default_servers
- 在与 Socks代理服务器 尝试建立TCP连接时，
  - 可以设置最大的重试次数（per_server_connection_attempts）
  - 还可以设置总的重试次数（connection_attempts）
    - 避免可用 Socks代理服务器 较多时，反复重试导致连接超时
    - 或者将此值设置为: 可用Socks代理服务器数量 * per_server_connection_attempts，以保证尝试所有的 Socks代理服务器

```
void
SocksEntry::findServer()
{
  nattempts++;

#ifdef SOCKS_WITH_TS
  if (nattempts == 1) {
    ink_assert(server_result.r == PARENT_UNDEFINED);
    // 寻找一个可用的 Socks代理服务器，
    server_params->findParent(&req_data, &server_result);
  } else {
    socks_conf_struct *conf = netProcessor.socks_conf_stuff;
    // 如果不是第一次尝试，那么要判断是否达到了重试次数上限
    //     如果未达上限，直接返回，这样会再次尝试与该 Socks代理服务器 建立连接
    // 参考: proxy.config.socks.per_server_connection_attempts
    if ((nattempts - 1) % conf->per_server_connection_attempts)
      return; // attempt again

    // 达到了重试次数上限
    // 设置该 Socks代理服务器 为down的状态
    server_params->markParentDown(&server_result);

    // 参考: proxy.config.socks.connection_attempts
    if (nattempts > conf->connection_attempts)
      server_result.r = PARENT_FAIL;
    else
      server_params->nextParent(&req_data, &server_result);
  }

  // 对于 ParentSelection 相关的分析这里暂时不展开介绍。
  // server_result.r == PARENT_DIRECT: 表示未匹配中规则，同时 default_servers 没有设置
  // server_result.r == PARENT_SPECIFIED: 表示找到了一个可用的 Socks代理服务器
  // server_result.r == PARENT_FAIL: 表示虽然匹配了一条 socks.config 中的规则，但是该规则没有配置有效的 Socks代理服务器
  switch (server_result.r) {
  case PARENT_SPECIFIED:
    // Original was inet_addr, but should hostnames work?
    // ats_ip_pton only supports numeric (because other clients
    // explicitly want to avoid hostname lookups).
    // 在 socks.config 中配置 Socks代理服务器 的规则时，如果填写了域名，而不是IP的话，
    //     在 setup_socks_servers() 加载规则的过程中，会把域名解析为IP地址，
    //     解析失败时，会把 hostname 设置为 255.255.255.255 这个地址
    // 因此这里使用 pton 方法把 hostname 直接写入到 server_addr，
    //     由于存在上述加载配置时的错误处理策略，因此 pton 方法应该不会失败
    // 注1：在 ats_ip_pton 中，对于IPv4的地址通过 inet_aton 转换为 struct in_addr，
    //      inet_aton 认为 255.255.255.255 是一个有效的 IPv4 地址
    // 注2：在 setup_socks_servers() 中是接受 Socks代理服务器 的IP地址为 IPv6 的情况，
    //      但是SocksEntry被设计为只能代理目标IP地址为 IPv4 的请求。
    if (0 == ats_ip_pton(server_result.hostname, &server_addr)) {
      ats_ip_port_cast(&server_addr) = htons(server_result.port);
    } else {
      Debug("SocksParent", "Invalid parent server specified %s", server_result.hostname);
    }
    break;

  default:
    ink_assert(!"Unexpected event");
  case PARENT_DIRECT:
  case PARENT_FAIL:
    // 在没有找到可用的 Socks代理服务器 时，将 server_addr 清零，此时 ats_is_ip() 返回false
    memset(&server_addr, 0, sizeof(server_addr));
  }
#else
  if (nattempts > netProcessor.socks_conf_stuff->connection_attempts)
    memset(&server_addr, 0, sizeof(server_addr));
  else
    ats_ip_copy(&server_addr, &g_socks_conf_stuff->server_addr);
#endif // SOCKS_WITH_TS

  char buff[INET6_ADDRSTRLEN];
  Debug("SocksParents", "findServer result: %s:%d", ats_ip_ntop(&server_addr.sa, buff, sizeof(buff)),
        ats_ip_port_host_order(&server_addr));
}
```

SocksEntry::free()

- 与 Socks代理服务器 建立 Socks 通信之后 或 遇到任何错误 通过调用 free() 方法通知状态机，然后释放 SocksEntry 占用的内存。
- 需要取消 超时事件
- 需要在出错时关闭 netVC
- 需要考虑状态机已经关闭的情况（如状态机已经关闭会通过Action对象调用cancel方法）
- 需要重置 do_io 操作，然后才能释放 SocksEntry 对象

```
void
SocksEntry::free()
{
  MUTEX_TRY_LOCK(lock, action_.mutex, this_ethread());
  // 在 6.0.x 的版本里 lock 前面少了一个感叹号。
  // 这个bug是在 53e56ffc7499648bc096d63f9c0f9da0ea9212ba 修复 TS-3156 时产生的
  if (!lock.is_locked()) {
    // 这里认为调用 free 方法之前必然已经持有 action_.mutex 的锁了，
    //   如果不能立即上锁成功，则是出现了异常
    // 我认为这里应该是早期的代码存在其它的bug导致问题，在这里加锁以避免问题产生
    //   所以我觉得这里应该完全没有必要上锁，只需要判断 mutex->thread_holding == this_ethread()
    // Socks continuation share the user's lock
    // so acquiring a lock shouldn't fail
    ink_assert(0);
    return;
  }

  // 取消之前设置的超时
  // 这里应该在 cancel 之后，将 timeout 赋值为 NULL
  if (timeout)
    timeout->cancel(this);

#ifdef SOCKS_WITH_TS
  // 如果没有错误，要把这个 Socks代理服务器的状态设置为可用
  if (!lerrno && netVConnection && server_result.retry)
    server_params->recordRetrySuccess(&server_result);
#endif

  // 下面的逻辑有些混乱，应该可以做一些优化。
  // 如果被cancel或者出错时，关闭 netVC
  //   这里应该在关闭netVC后设置 netVC 为 NULL
  if ((action_.cancelled || lerrno) && netVConnection)
    netVConnection->do_io_close();

  // 如果没有被cancel，那么需要回调状态机
  if (!action_.cancelled) {
    // 如果上面把 netVC 设置为 NULL，这里就完全可以根据 netVC 是否为 NULL 对状态机回调不同的事件。
    if (lerrno || !netVConnection) {
      // 如果出错 或 netVC 为 NULL，则回调 OPEN_FAILED 并传递错误信息
      Debug("Socks", "retryevent: Sent errno %d to HTTP", lerrno);
      NET_INCREMENT_DYN_STAT(socks_connections_unsuccessful_stat);
      action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)(intptr_t)(-lerrno));
    } else {
      // 否则要回调 OPEN 并且传递 netVC 给状态机
      // 说明成功建立了 Socks 通信
      // 这里的 do_io 重设存在问题，不应该使用 this，应该使用 NULL
      netVConnection->do_io_read(this, 0, 0);
      netVConnection->do_io_write(this, 0, 0);
      netVConnection->action_ = action_; // assign the original continuation
      // 这里猜测是想把 target_addr 复制回 netVC->server_addr 以实现完全透明的通过 SocksEntry 建立连接
      // 但是这儿写成了 server_addr
      ats_ip_copy(&netVConnection->server_addr, &server_addr);
      Debug("Socks", "Sent success to HTTP");
      NET_INCREMENT_DYN_STAT(socks_connections_successful_stat);
      action_.continuation->handleEvent(NET_EVENT_OPEN, netVConnection);
    } 
  }
  // 回收对资源的引用
#ifdef SOCKS_WITH_TS
  SocksServerConfig::release(server_params);
#endif

  // 释放资源
  free_MIOBuffer(buf);
  // 回收 action_->mutex 的引用计数
  action_ = NULL;
  // 回收 this->mutex 的引用计数
  mutex = NULL;
  // 回收 SocksEntry 对象
  socksAllocator.free(this);
}
```

SocksEntry::startEvent(event, data)
- 用来完成与 Socks代理服务器 之间的TCP连接
- 连接建立后切换到 mainEvent
- 如果遇到超时或连接失败，调用findServer()判断：
  - 对当前 Socks代理服务器 尝试重连
  - 或 选择下一个可用 Socks代理服务器
- 重连之前会对上一个 netVC 执行 do_io_close()

```
int
SocksEntry::startEvent(int event, void *data)
{
  if (event == NET_EVENT_OPEN) {
    // 连接建立成功
    netVConnection = (SocksNetVC *)data;

    // 如果选择采用 Socks v5 协议建立连接，需要处理认证消息
    if (version == SOCKS5_VERSION)
      auth_handler = &socks5BasicAuthHandler;

    // 切换到 mainEvent
    SET_HANDLER((SocksEntryHandler)&SocksEntry::mainEvent);
    // 透传 NET_EVENT_OPEN 到 mainEvent
    // 我觉得把 NET_EVENT_OPEN 的处理放在 startEvent 里会比较好，mainEvent 太长了
    mainEvent(NET_EVENT_OPEN, data);
  } else {
    // 连接建立失败 或 超时
    if (timeout) {
      timeout->cancel(this);
      timeout = NULL;
    }

    char buff[INET6_ADDRPORTSTRLEN];
    Debug("Socks", "Failed to connect to %s", ats_ip_nptop(&server_addr.sa, buff, sizeof(buff)));

    // 查找下一个可用的 Socks服务器
    // 或 对当前的 Socks服务器 重新连接
    findServer();

    // 如果下一个 Socks服务器 的IP地址不合法，
    //   视为连接错误，通过 free() 回调上层状态机，
    if (!ats_is_ip(&server_addr)) {
      Debug("Socks", "Unable to open connection to the SOCKS server");
      lerrno = ESOCK_NO_SOCK_SERVER_CONN;
      free();
      return EVENT_CONT;
    }

    // 多余代码
    if (timeout) {
      timeout->cancel(this);
      timeout = 0;
    }

    // 关闭连接失败的 netVC，为重新连接做准备
    if (netVConnection) {
      netVConnection->do_io_close();
      netVConnection = 0;
    }

    // 设置重新建立连接的超时时间
    timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->server_connect_timeout));

    write_done = false;

    NetVCOptions options;
    // NO_SOCKS 通知 connect_re 直接与目标IP和端口建立连接
    // 如果不设置 NO_SOCKS，connect_re 会使用 Socks代理服务器 与目标IP和端口建立连接
    options.socks_support = NO_SOCKS;
    netProcessor.connect_re(this, &server_addr.sa, &options);
  }

  return EVENT_CONT;
}
```

SocksEntry::mainEvent
- 如果采用Socks v5协议，基于RFC1928实现
  - 发送认证协商头
  - 接收Socks服务器返回的认证方式
  - 如果采用用户名密码方式认证，则发送用户名及密码完成认证，基于RFC1929实现
  - 认证成功后进入 Socks 命令发送阶段
  ```
  +-----+-----+-------+------+----------+----------+
  | VER | CMD |  RSV  | ATYP | DST.ADDR | DST.PORT |
  +-----+-----+-------+------+----------+----------+
  |  1  |  1  | X'00' |  1   | Variable |    2     |
  +-----+-----+-------+------+----------+----------+
  ```

- 如果采用Socks v4协议
  - 无认证过程，直接进入 Socks 命令发送阶段
  ```
  +----+----+----+----+----+----+----+----+----+----+....+----+
  | VN | CD | DSTPORT |      DSTIP        | USERID       |NULL|
  +----+----+----+----+----+----+----+----+----+----+....+----+
  |  1 |  1 |    2    |         4         | variable|    |  1 |
  +----+----+----+----+----+----+----+----+----+----+....+----+
  ```

- Socks 命令各个字段解释如下：
  - VER / VN: Socks 协议版本号，0x04 或 0x05
  - CMD / CD: Socks 命令字
    - 0x01：CONNECT
    - 0x02：BIND
    - 0x03：UDP Associate
  - RSV：保留字节
  - ATYP：目标IP类型
    - 0x01：IPv4
    - 0x03：Domain Name
    - 0x04：IPv6
  - DST.ADDR: 根据 ATYP 的类型，可能为 IPv4，IPv6地址 或 [len]"domain name"
  - DST.PORT / DSTPORT: 目标端口(网络字节序)
  - DSTIP: IPv4 地址
  - USERID：用户名，长度为0字节表示没有用户名
  - NULL：其值为0x00，用来表示用户名的结束

```
int
SocksEntry::mainEvent(int event, void *data)
{
  int ret = EVENT_DONE;
  int n_bytes = 0;
  unsigned char *p;

  switch (event) {
  case NET_EVENT_OPEN:
    buf->reset();
    unsigned short ts;
    p = (unsigned char *)buf->start();
    ink_assert(netVConnection);

    if (auth_handler) {
      // socks5BasicAuthHandler 负责填充认证头部
      // 无认证: 0x05 0x01 0x00
      //   0x05: Version
      //   0x01: 1 个 Methrods
      //   0x00: Method＝0 无认证
      // 有认证: 0x05 0x02 0x00 0x02
      //   0x05: Version
      //   0x02: 2 个 Methods
      //   0x00: Method＝0 无认证
      //   0x02: Method＝2 用户名密码认证
      // 发送认证头部之后，从 Socks 服务器读取两个字节
      // 无认证: 0x05 0x00
      //   0x05: Version
      //   0x00: 无认证
      //   然后auth_handler被设置为 NULL，进入下面的 Socks 命令处理阶段
      // 有认证: 0x05 0x02
      //   0x05: Version
      //   0x02: 用户名密码认证，如果第二个字节是 0xff 表示认证方式协商失败
      //   然后切换到 socks5PasswdAuthHandler 负责填充用户名和密码:
      //     0x01 [ulen] "username" [plen] "password"
      //   0x01: Version，这里不是 Socks 协议的版本，而是用户名密码传输协议的版本
      //   ulen: 用户名长度（1 ~ 255)
      //   username: 用户名
      //   plen: 密码长度（1 ~ 255)
      //   password: 密码
      //   然后 socks5PasswdAuthHandler 负责验证认证结果:
      //     0x01 0x00
      //   0x01: Version，这里不是 Socks 协议的版本，而是用户名密码传输协议的版本
      //   0x00: 认证成功，其它值认证失败，必须关闭TCP连接
      //   认证成功后，auth_handler被设置为 NULL，进入下面的 Socks 命令处理阶段
      n_bytes = invokeSocksAuthHandler(auth_handler, SOCKS_AUTH_OPEN, p);
    } else {
      // Socks 命令构造阶段
      // Debug("Socks", " Got NET_EVENT_OPEN to SOCKS server\n");

      p[n_bytes++] = version;
      p[n_bytes++] = (socks_cmd == NORMAL_SOCKS) ? SOCKS_CONNECT : socks_cmd;
      // BUG1: 由于端口要按照网络字节序填写，因此不应该调用ntohs()
      // BUG2: 这里应该从 target_addr 获取目标端口信息来填写，在 ATS 支持IPv6的一个提交导致此问题产生
      ts = ntohs(ats_ip_port_cast(&server_addr));

      if (version == SOCKS5_VERSION) {
        // Socks5 协议IP地址在先
        p[n_bytes++] = 0; // Reserved
        if (ats_is_ip4(&server_addr)) {
          p[n_bytes++] = 1; // IPv4 addr
          memcpy(p + n_bytes, &server_addr.sin.sin_addr, 4);
          n_bytes += 4;
        } else if (ats_is_ip6(&server_addr)) {
          // 在 init() 里，有一处assert，要求目标IP必须是IPv4
          // 显然这里已经在Socks v5协议下支持IPv6
          p[n_bytes++] = 4; // IPv6 addr
          memcpy(p + n_bytes, &server_addr.sin6.sin6_addr, TS_IP6_SIZE);
          n_bytes += TS_IP6_SIZE;
        } else {
          Debug("Socks", "SOCKS supports only IP addresses.");
        }
      }

      // 填写端口
      memcpy(p + n_bytes, &ts, 2);
      n_bytes += 2;

      if (version == SOCKS4_VERSION) {
        // Socks4 协议IP地址在后
        if (ats_is_ip4(&server_addr)) {
          // for socks4, ip addr is after the port
          memcpy(p + n_bytes, &server_addr.sin.sin_addr, 4);
          n_bytes += 4;

          p[n_bytes++] = 0; // NULL
        } else {
          Debug("Socks", "SOCKS v4 supports only IPv4 addresses.");
        }
      }
      // BUG2: 结束，可以从 6.2.x 分之获得修复后的版本
    }

    buf->fill(n_bytes);

    // BUG3: 之前设置的是连接超时，现在设置的应该是 socks 通信超时，应该取消之前的超时重新设置新的超时时间
    if (!timeout) {
      /* timeout would be already set when we come here from StartEvent() */
      timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->socks_timeout));
    }

    // 发送 认证协商、认证信息、CONNECT命令
    netVConnection->do_io_write(this, n_bytes, reader, 0);
    // Debug("Socks", "Sent the request to the SOCKS server\n");

    ret = EVENT_CONT;
    break;

  case VC_EVENT_WRITE_READY:

    ret = EVENT_CONT;
    break;

  case VC_EVENT_WRITE_COMPLETE:
    // 这里的处理有些模糊
    // WRITE_COMPLETE 事件可能表示：
    //   1. 发送认证协商请求(Socks v5)
    //   2. 发送用户名及密码(Socks v5)
    //   3. 发送 Socks 命令
    // 第一个写操作，可能是1、3中的一个，第一次取消的是Socks connect timeout
    // 因此，1、3中的任何一个发送完成就认为Socks连接已经建立了。
    if (timeout) {
      timeout->cancel(this);
      timeout = NULL;
      write_done = true;
    }

    // 重用buffer
    buf->reset(); // Use the same buffer for a read now

    // auth_handler 不为 NULL 表示处于认证流程内，接下来要接收 Socks Server 返回的认证信息
    // 否则表示处于 Socks 命令流程内，接下来要接收 Socks Server 返回的命令执行情况
    if (auth_handler)
      n_bytes = invokeSocksAuthHandler(auth_handler, SOCKS_AUTH_WRITE_COMPLETE, NULL);
    // 如果是 CONNECT 命令
    else if (socks_cmd == NORMAL_SOCKS)
      // 根据 Socks 协议版本，计算出须要从 Socks Server 接收的信息长度
      //   SOCKS4_REP_LEN 为 8 字节，固定长度
      //   SOCKS5_REP_LEN 为 262 字节，是 Socks v5 协议下的最大信息长度
      n_bytes = (version == SOCKS5_VERSION) ? SOCKS5_REP_LEN : SOCKS4_REP_LEN;
    // 如果不是 CONNECT 命令
    else {
      // 通过 Tunnel 方式透传该连接
      Debug("Socks", "Tunnelling the connection");
      // let the client handle the response
      // 通过 free() 回调状态机 NET_EVENT_OPEN 并传递 VC
      // 这里的状态机应该是 SocksProxy，处理函数是 SocksProxy::mainEvent
      // SocksProxy 在调用 connect_re() 之前会设置其状态 state = SERVER_TUNNEL
      free();
      // 这里写成 return 会比较好
      break;
    }

    // 这里设置的是 Socks 通信超时时间，例如：
    //   1. 等待 Socks Server 返回认证协商结果
    //   2. 等待 Socks Server 返回用户名及密码的认证结果
    //   3. 等待 Socks Server 返回Socks命令的执行结果
    timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->socks_timeout));

    // 准备接收 Socks Server 返回的信息
    netVConnection->do_io_read(this, n_bytes, buf);

    ret = EVENT_DONE;
    break;

  case VC_EVENT_READ_READY:
    ret = EVENT_CONT;

    // 如果正在读取 Socks v5 的命令执行结果
    // auth_handler 为 NULL 表示已经完成了认证
    if (version == SOCKS5_VERSION && auth_handler == NULL) {
      // 由于 Socks v5 的执行结果分为 IPv4、IPv6 和 FQHN 三种情况，
      // 三种情况下，返回结果的数据长度是不同的，
      // 根据前 5 个字节的内容来判断并修正最终要读取的数据总长度。
      VIO *vio = (VIO *)data;
      p = (unsigned char *)buf->start();

      if (vio->ndone >= 5) {
        int reply_len;

        switch (p[3]) {
        case SOCKS_ATYPE_IPV4:
          reply_len = 10;
          break;
        case SOCKS_ATYPE_FQHN:
          reply_len = 7 + p[4];
          break;
        case SOCKS_ATYPE_IPV6:
          Debug("Socks", "Who is using IPv6 Addr?");
          reply_len = 22;
          break;
        default:
          reply_len = INT_MAX;
          Debug("Socks", "Illegal address type(%d) in Socks server", (int)p[3]);
        }

        // 当前已经读取到的数据已经达到了需要的长度
        if (vio->ndone >= reply_len) {
          vio->nbytes = vio->ndone;
          ret = EVENT_DONE;
        }
      }
    }

    // 如果是:
    //   1. Socks v4
    //   2. Socks v5 并且处于接收认证信息的过程
    //   3. Socks v5 等待命令执行结果，但是未完成数据的读取
    // 则返回，继续接收数据
    if (ret == EVENT_CONT)
      break;
  // Fall Through
  case VC_EVENT_READ_COMPLETE:
    // 取消之前设置的超时
    if (timeout) {
      timeout->cancel(this);
      timeout = NULL;
    }
    // Debug("Socks", "Successfully read the reply from the SOCKS server\n");
    p = (unsigned char *)buf->start();

    if (auth_handler) {
      // 处理认证逻辑
      SocksAuthHandler temp = auth_handler;

      if (invokeSocksAuthHandler(auth_handler, SOCKS_AUTH_READ_COMPLETE, p) < 0) {
        lerrno = ESOCK_DENIED;
        free();
      } else if (auth_handler != temp) {
        // here either authorization is done or there is another
        // stage left.
        mainEvent(NET_EVENT_OPEN, NULL);
      }

    } else {
      // 处理 Socks命令 逻辑
      bool success;
      if (version == SOCKS5_VERSION) {
        success = (p[0] == SOCKS5_VERSION && p[1] == SOCKS5_REQ_GRANTED);
        Debug("Socks", "received reply of length %" PRId64 " addr type %d", ((VIO *)data)->ndone, (int)p[3]);
      } else
        success = (p[0] == 0 && p[1] == SOCKS4_REQ_GRANTED);

      // ink_assert(*(p) == 0);
      // 根据 Socks Server 返回的结果判断命令是否成功执行
      // 失败的话设置 lerrno
      if (!success) { // SOCKS request failed
        Debug("Socks", "Socks request denied %d", (int)*(p + 1));
        lerrno = ESOCK_DENIED;
      } else {
        Debug("Socks", "Socks request successful %d", (int)*(p + 1));
        lerrno = 0;
      }
      // 根据 lerrno 的值通过 free() 回调状态机
      free();
    }

    break;

  // 以下超时和错误处理，请查看下面两个段落的分析
  case EVENT_INTERVAL:
    timeout = NULL;
    if (write_done) {
      lerrno = ESOCK_TIMEOUT;
      free();
      break;
    }
  /* else
     This is server_connect_timeout. So we treat this as server being
     down.
     Should cancel any pending connect() action. Important on windows

     fall through
   */
  case VC_EVENT_ERROR:
    /*This is mostly ECONNREFUSED on Unix */
    SET_HANDLER(&SocksEntry::startEvent);
    startEvent(NET_EVENT_OPEN_FAILED, NULL);
    break;

  // 对于 EOS / INACTIVITY_TIMEOUT / ACTIVE_TIMEOUT 则回调状态机 OPEN_FAILED
  // 因此 Inactivity Timeout 的值会影响到整个 Socks 连接的建立
  case VC_EVENT_EOS:
  case VC_EVENT_INACTIVITY_TIMEOUT:
  case VC_EVENT_ACTIVE_TIMEOUT:
    Debug("Socks", "VC_EVENT error: %s", get_vc_event_name(event));
    lerrno = ESOCK_NO_SOCK_SERVER_CONN;
    free();
    break;
  default:
    // BUGBUG:: could be active/inactivity timeout ...
    ink_assert(!"bad case value");
    Debug("Socks", "Bad Case/Net Error Event");
    lerrno = ESOCK_NO_SOCK_SERVER_CONN;
    free();
    break;
  }

  return ret;
}
```

### SocksEntry::mainEvent() 中的超时管理

与超时处理相关的事件：

- NET_EVENT_OPEN
  - 执行 do_io_write() 同时设置超时时间为 socks_timeout
  - 此时设置的是发送信息到 Socks Server 的超时时间。

- WRITE_COMPLETE
  - 执行 do_io_read() 同时设置超时时间为 socks_timeout
  - 此时设置的是等待 Socks Server 做出响应的超时时间。

- READ_COMPLETE
  - 如果发现需要继续发送数据到 Socks Server，会递归调用 mainEvent 进入 NET_EVENT_OPEN 的处理流程。

- 在设置新的超时之前需要取消之前设置的超时
  - 在 READ_COMPLETE 和 WRITE_COMPLETE 中都会取消之前的超时设置
  - 由于 NET_EVENT_OPEN 中只会在未设置超时的情况才会设置新的超时时间，因此不需要先取消之前的超时设置。

通过上面的分析，可以清晰的看到 socks_timeout 是每一次发送 Socks请求 和接收 Socks响应 的超时时间。

### 与 Socks Server 连接建立超后的重试

在 WRITE_COMPLETE 的处理中，
  - 会将 write_done 设置为 true
  - 用来表示已经完成了 Socks请求 的发送，正在等待接收 Socks响应。
在 EVENT_INTERVAL 的处理中，
  - 如发现 write_done 为 true，则通常表示 socks_timeout，则回调状态机 OPEN_FAILED
  - 如 write_done 为 false，则通常表示 server_connect_timeout，会重新调用 startEvent() 重新进行 TCP 连接的建立。

也就是说，在 SocksEntry 中，

- 如果连接 Socks 服务器超时，会进行重试
- 如果在 Socks 通信过程中超时，会直接回调状态机 OPEN_FAILED


## 参考资料

- [P_Socks.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_Socks.h)
- [Socks.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/Socks.cc)
- [ParentSelection.cc](http://github.com/apache/trafficserver/tree/master/proxy/ParentSelection.cc)
- SOCKS on Wikipedia
  - [中文](https://zh.wikipedia.org/wiki/SOCKS)
  - [English](https://en.wikipedia.org/wiki/SOCKS)
- RFCs
  - [Socks v4](https://www.openssh.com/txt/socks4.protocol)
  - [Socks v5](https://tools.ietf.org/html/rfc1928)
  - [Socks v5 username/password authentication](https://tools.ietf.org/html/rfc1929)
- [Sample code](http://howtofix.pro/c-establish-connections-through-socks45-using-winsock2/)
