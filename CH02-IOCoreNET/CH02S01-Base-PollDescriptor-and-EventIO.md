# 基础组件：PollDescriptor 和 EventIO

PollDescriptor 是对多种平台上的I/O Poll操作的封装，目前ATS支持epoll，kequeu，port三种。

我们可以把 PollDescriptor 看成是一个队列：

- 把 Socket FD 放入 poll fd / PollDescriptor 队列
- 然后通过 Polling 操作，从 poll fd / PollDescriptor 队列中批量取出一部分 Socket FD
- 而且这是一个原子队列

EventIO 是 PollDescriptor 队列的成员，也是 PollDescriptor 对外提供的接口：

- 为每一个 Socket FD 提供一个 EventIO，把 Socket FD 封装到 EventIO 内
- 提供把 EventIO 自身放入 PollDescriptor 队列的接口

以下内容仅仅分析对epoll ET模式支持的实现。

## 定义

重温一下linux pollfd的结构，下面会用到

```
source: man 2 poll
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```

重温一下linux epoll_event的结构，下面会用到

```
source: man 2 epoll_wait
typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;    /* Epoll events */
    epoll_data_t data;      /* User data variable */
};
```

首先是PollDescriptor，它包含 Poll 句柄和保存结果集的数组。

```
source: iocore/net/P_UnixPollDescriptor.h
// 定义一个PollDescriptor可以处理的最大描述符的数量。
// 这里定义为32K＝32768 = 0x10000，如果要修改这个值，也建议采用16的整数倍，
//     以保证在存储这些数据时实现内存边界的对齐。
#define POLL_DESCRIPTOR_SIZE 32768

typedef struct pollfd Pollfd;

struct PollDescriptor {
  int result; // result of poll
#if TS_USE_EPOLL
  // 用于保存epoll_create创建的epoll fd
  int epoll_fd;

  // nfds 和 pfd在TCP的部分没有用到，目前应该只有UDP里才会使用。
  // 记录有多少fd被添加到了epoll fd中
  int nfds; // actual number
  // 每一个fd被添加到epoll fd之前会保存到pfd这个数组里
  Pollfd pfd[POLL_DESCRIPTOR_SIZE];

  // 用来保存epoll_wait返回的fd状态集，result中保存了实际的fd数量
  // 为什么在epoll_wait处使用函数内部变量，而要固定分配一个内存区域呢？
  //     由于epoll_wait需要非常高频率的运行，因此这个内存区域需要非常高频次的分配和释放，
  //     即使在函数内定义为内部数组变量，那么也会频繁的从栈里分配空间，
  //     所以不如分配一个固定的内存区域
  struct epoll_event ePoll_Triggered_Events[POLL_DESCRIPTOR_SIZE];
#endif
#if TS_USE_EPOLL
  // 下面四个宏定义是一组通用方法，对于kqueue等其它系统的I/O poll操作也都是抽象为这四个方法
  // 获取I/O poll描述符，对于epoll就是epoll fd
#define get_ev_port(a) ((a)->epoll_fd)
  // 获取指定fd当前的事件状态，在epoll_wait之后通过这个方法获取返回的fd的事件状态
#define get_ev_events(a, x) ((a)->ePoll_Triggered_Events[(x)].events)
  // 获取指定fd绑定的数据指针，对于epoll就是epoll_event结构体的data.ptr
#define get_ev_data(a, x) ((a)->ePoll_Triggered_Events[(x)].data.ptr)
  // 准备获取下一个fd，对于epoll来说这里没啥对应的操作
#define ev_next_event(a, x)
#endif

  // 从pfd中分配一个地址空间
  // 仅用于epoll对UDP的支持，如使用kqueue等其它系统的I/O poll操作，则直接返回NULL
  Pollfd *
  alloc()
  {
#if TS_USE_EPOLL
    // XXX : We need restrict max size based on definition.
    if (nfds >= POLL_DESCRIPTOR_SIZE) {
      nfds = 0;
    }
    return &pfd[nfds++];
#else
    return 0;
#endif
  }
private:
  // 初始化 epoll fd 及事件数组
  // 返回当前PollDescriptor
  PollDescriptor *
  init()
  {
    result = 0;
#if TS_USE_EPOLL
    nfds = 0;
    epoll_fd = epoll_create(POLL_DESCRIPTOR_SIZE);
    memset(ePoll_Triggered_Events, 0, sizeof(ePoll_Triggered_Events));
    memset(pfd, 0, sizeof(pfd));
#endif
    return this;
  }

public:
  // 构造函数，调用init()完成初始化
  PollDescriptor() { init(); }
};
```

然后是EventIO，向poll fd添加文件句柄，以及修改fd事件状态

```
source:iocore/net/P_UnixNet.h
#define EVENTIO_READ (EPOLLIN | EPOLLET)
#define EVENTIO_WRITE (EPOLLOUT | EPOLLET)
#define EVENTIO_ERROR (EPOLLERR | EPOLLPRI | EPOLLHUP)  

struct PollDescriptor;
typedef PollDescriptor *EventLoop;

struct EventIO {
  int fd;
  EventLoop event_loop;
#define EVENTIO_NETACCEPT 1
#define EVENTIO_READWRITE_VC 2
#define EVENTIO_DNS_CONNECTION 3
#define EVENTIO_UDP_CONNECTION 4
#define EVENTIO_ASYNC_SIGNAL 5
  int type;
  union {
    Continuation *c; 
    UnixNetVConnection *vc;
    DNSConnection *dnscon;
    NetAccept *na;
    UnixUDPConnection *uc;
  } data;
  int start(EventLoop l, DNSConnection *vc, int events);
  int start(EventLoop l, NetAccept *vc, int events);
  int start(EventLoop l, UnixNetVConnection *vc, int events);
  int start(EventLoop l, UnixUDPConnection *vc, int events);

  // 底层定义，不应该直接调用
  int start(EventLoop l, int fd, Continuation *c, int events);

  // Change the existing events by adding modify(EVENTIO_READ)
  // or removing modify(-EVENTIO_READ), for level triggered I/O
  int modify(int events);

  // Refresh the existing events (i.e. KQUEUE EV_CLEAR), for edge triggered I/O
  int refresh(int events);

  int stop();
  int close();
  EventIO()
  {
    type = 0;
    data.c = 0;
  }
};
```

## 方法

### start

在需要将一个文件描述符以及文件描述符的延伸（NetVC，UDPVC，DNSConn，NetAccept）添加到PollDescriptor时，可以通过EventIO提供的start方法：

- start(EventLoop l, DNSConnection *vc, int events)
   - type = EVENTIO_DNS_CONNECTION
- start(EventLoop l, NetAccept *vc, int events)
   - type = EVENTIO_NETACCEPT
- start(EventLoop l, UnixNetVConnection *vc, events)
   - type = EVENTIO_READWRITE_VC
- start(EventLoop l, UnixUDPVConnection *vc, events)
   - type = EVENTIO_UDP_CONNECTION

上面这四个方法，都会设置成员type的值，最终都是调用基础方法：

- start(EventLoop l, int afd, Continuation *vc, events)
- 这个基础方法是支持多种IO Poll平台的，如：epoll，kqueue，port
- 这个方法通常不应该被直接调用，因为这个方法没有对type进行设置
- 参数说明：
   - EventLoop实际是poll fd的封装PollDescriptor
   - Continuation则用于回调
   - events表示所关注的事件，通常应该使用：EVENTIO_READ，EVENTIO_WRITE，EVENTIO_ERROR三者之一或者他们的“或”值

```
TS_INLINE int
EventIO::start(EventLoop l, int afd, Continuation *c, int e)
{
  data.c = c;
  fd = afd;
  event_loop = l;
#if TS_USE_EPOLL
  struct epoll_event ev;
  memset(&ev, 0, sizeof(ev));
  ev.events = e;
  ev.data.ptr = this;
#ifndef USE_EDGE_TRIGGER
  events = e;
#endif
  // 最后调用epoll_ctl把ev添加到epoll fd的事件池中，然后可以通过epoll_wait获取结果
  // 在ev中包含了指向该EventIO实例的指针，在每一个UnixNetVConnection里都有一个EventIO类型的成员。
  return epoll_ctl(event_loop->epoll_fd, EPOLL_CTL_ADD, fd, &ev);
#endif
}
```

### stop

对应start方法的则是stop方法，stop方法只有epoll和port的实现，没有kqueue的处理（我对kqueue不熟悉，不知道为何这里没有针对kqueue的处理）：

```
TS_INLINE int
EventIO::stop()
{
  if (event_loop) {
    int retval = 0;
#if TS_USE_EPOLL
    struct epoll_event ev;
    memset(&ev, 0, sizeof(struct epoll_event));
    ev.events = EPOLLIN | EPOLLOUT | EPOLLET;
    // 通过epoll_ctl从epoll fd中删除fd
    retval = epoll_ctl(event_loop->epoll_fd, EPOLL_CTL_DEL, fd, &ev);
#endif
    // 设置event_loop为NULL，因为所有的操作都要通过epoll fd进行，这里把event_loop设置为NULL，就相当于没有了epoll fd
    event_loop = NULL;
    return retval;
  }
  return 0;
}
```

### modify & refresh

start和stop分别对应添加、删除，然后还定义了修改和刷新两个方法，但是对于epoll ET模式，这两个方法都是空方法：

- modify
   - 修改，对epoll LT，kqueue LT，port有相关代码定义，这里就不做分析了
- refresh
   - 刷新，对kqueue LT，port有相关代码定义，这里就不做分析了

### close

最后还有一个close方法，用于关闭fd，首先调用stop方法停止polling，然后根据type的类型，调用该类型的close方法完成fd的关闭。

但是close方法只负责关闭DNS，NetAccept，VC三种类型，对于UDP，则不可以调用close方法 // 这是为什么？？？

```
TS_INLINE int
EventIO::close()
{
  stop();
  switch (type) {
  default:
    ink_assert(!"case");
  case EVENTIO_DNS_CONNECTION:
    return data.dnscon->close();
    break;
  case EVENTIO_NETACCEPT:
    return data.na->server.close();
    break;
  case EVENTIO_READWRITE_VC:
    return data.vc->con.close();
    break;
  }
  return -1;
}
```

close 方法好像只有dns系统调用，其它系统没有看到有任何调用此方法的操作：

- 在close_UnixNetVConnection中则是先调用stop方法，然后再自己调用了vc->con.close()
  - 其实是可以把调用stop方法修改为调用close方法的
- 在NetAccept的处理中，因为只有在遇到错误时才需要把listen fd从epoll fd中删除，因此这块很少用到
  - 在acceptFastEvent的Lerror标签处直接调用了server.close()，没有把listen fd从epoll fd中删除，貌似是一个小bug???
  - 在cancel中也是直接调用了server.close()，没有把listen fd从epoll fd中删除

## 参考资料

- man 2 poll
- man 2 epoll_wait
- [P_UnixPollDescriptor.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixPollDescriptor.h)
- [P_UnixNet.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
