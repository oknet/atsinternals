# 核心部件：Throttle

Throttle 并不是一个类，而是几个全局变量、统计值和一组函数。

通过在下面的方法中调用这些相关的函数，实现了Throttle

  - NetAccept::do_blocking_accept
  - UnixNetAccept::connectUp
  - UnixNetProcessor::connect_re_internal

ATS 的 Throttle 的设计分为：

  - Net Throttle，这包括：
    - ACCEPT 部分
    - CONNECT 部分
  - Emergency Throttle，这包括：
    - 最大文件句柄控制部分
    - 保留文件句柄用于 backdoor 的部分

# 定义

首先是几个常量宏定义和枚举值：

```
source: P_UnixNet.h
#define THROTTLE_FD_HEADROOM (128 + 64) // CACHE_DB_FDS + 64

#define TRANSIENT_ACCEPT_ERROR_MESSAGE_EVERY HRTIME_HOURS(24)

// also the 'throttle connect headroom'
#define THROTTLE_AT_ONCE 5
#define EMERGENCY_THROTTLE 16
#define HYPER_EMERGENCY_THROTTLE 6

#define NET_THROTTLE_ACCEPT_HEADROOM 1.1  // 10%
#define NET_THROTTLE_CONNECT_HEADROOM 1.0 // 0%
#define NET_THROTTLE_MESSAGE_EVERY HRTIME_MINUTES(10)

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

ink_hrtime last_throttle_warning;
ink_hrtime last_shedding_warning;
ink_hrtime emergency_throttle_time;
ink_hrtime last_transient_accept_error;
```

最后是一组函数

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

TS_INLINE void
check_shedding_warning()
{
  ink_hrtime t = Thread::get_hrtime();
  if (t - last_shedding_warning > NET_THROTTLE_MESSAGE_EVERY) {
    last_shedding_warning = t;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "number of connections reaching shedding limit");
  }
}

TS_INLINE bool
emergency_throttle(ink_hrtime now)
{
  return (bool)(emergency_throttle_time > now);
}

// PUBLIC: 这是提供给 IOCore Net Subsystem 的 Interface
// 用于检测当前的 Throttle 状态
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

TS_INLINE void
check_throttle_warning()
{
  ink_hrtime t = Thread::get_hrtime();
  if (t - last_throttle_warning > NET_THROTTLE_MESSAGE_EVERY) {
    last_throttle_warning = t;
    RecSignalWarning(REC_SIGNAL_SYSTEM_ERROR, "too many connections, throttling");
  }
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

TS_INLINE int
change_net_connections_throttle(const char *token, RecDataT data_type, RecData value, void *data)
{
  (void)token;
  (void)data_type;
  (void)value;
  (void)data;
  int throttle = fds_limit - THROTTLE_FD_HEADROOM;
  if (fds_throttle < 0)
    net_connections_throttle = throttle;
  else {
    net_connections_throttle = fds_throttle;
    if (net_connections_throttle > throttle)
      net_connections_throttle = throttle;
  }
  return 0;
}

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
```
