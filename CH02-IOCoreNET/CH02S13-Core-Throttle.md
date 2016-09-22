# 核心部件：Throttle

Throttle 并不是一个类，而是几个全局变量、统计值和一组函数：

  - 
  
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

enum ThrottleType {
  ACCEPT,
  CONNECT,
};
```

然后是几个相关的全局变量

```
source: UnixNet.cc
ink_hrtime last_throttle_warning;
ink_hrtime last_shedding_warning;
ink_hrtime emergency_throttle_time;
int net_connections_throttle;
int fds_throttle;
int fds_limit = 8000;
ink_hrtime last_transient_accept_error;
```

最后是一组函数

```
source: P_UnixNet.h
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


```
