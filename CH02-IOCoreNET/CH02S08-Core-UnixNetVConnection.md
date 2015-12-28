# 核心组件：UnixNetVConnection

实际上IOCoreNet子系统，与EventSystem是一样的，也有Thread，Processor和Event，只是名字不一样了：

|  EventSystem   |        NetSubSystem       |
|:--------------:|:-------------------------:|
|      Event     |     UnixNetVConnection    |
|     EThread    | NetHandler，InactivityCop |
| EventProcessor |        NetProcessor       |

- 像 Event 一样，UnixNetVConnection 也提供了面向上层状态机的方法
  - do_io_* 系列
  - (set|cancel)_*_timeout 系列
  - (add|remove)_*_queue 系列
- UnixNetVConnection 也提供了面向底层状态机的方法
  - 大多数由NetHandler来调用
- UnixNetVConnection 也是状态机
  - 因此它也有自己的handler（回调函数）
    - NetAccept调用acceptEvent
    - InactivityCop调用mainEvent
    - 构造函数会初始化为startEvent，用于调用connectUp()，这是面向NetProcessor的
  - 大致有以下三条调用路径：
    - EThread  －－－  NetAccept  －－－ UnixNetVConnection
    - EThread  －－－  NetHandler  －－－  UnixNetVConnection
    - EThread  －－－  InactivityCop  －－－  UnixNetVConnection

由于它既是Event，又是SM，所以从形态上来看，UnixNetVConnection 要比 Event 复杂的多。

由于 SSLNetVConnection 继承自 UnixNetVConnection，因此在 UnixNetVConnection 的定义中也为支持 SSL 预留了部分内容。

## 定义

以下内容为了描述方便，对代码行的顺序做了部分调整。

首先看一下 [NetState](https://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetState.h)，

```
struct NetState {
  volatile int enabled;
  VIO vio;
  Link<UnixNetVConnection> ready_link;
  SLink<UnixNetVConnection> enable_link;
  int in_enabled_list;
  int triggered;

  NetState() : enabled(0), vio(VIO::NONE), in_enabled_list(0), triggered(0) {}
};

```

然后是 UnixNetVConnection，

```
// 计算VIO成员在NetState实例中的内存偏移量
#define STATE_VIO_OFFSET ((uintptr_t) & ((NetState *)0)->vio)
// 返回指向包含指定VIO的NetState实例的指针
#define STATE_FROM_VIO(_x) ((NetState *)(((char *)(_x)) - STATE_VIO_OFFSET))

// 关闭指定VC的读写操作，实际是修改指定VC的NetState实例read或write里的enabled成员为0
#define disable_read(_vc) (_vc)->read.enabled = 0
#define disable_write(_vc) (_vc)->write.enabled = 0
// 打开指定VC的读写操作，实际是修改指定VC的NetState实例read或write里的enabled成员为1
#define enable_read(_vc) (_vc)->read.enabled = 1
#define enable_write(_vc) (_vc)->write.enabled = 1
```

上面几个宏定义都是用于操作NetState结构的，下面会使用到。

```
class UnixNetVConnection : public NetVConnection
{
public:
  // 面向上层状态机的方法
  // 设置读、写、关闭、半关闭操作
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf);
  virtual VIO *do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner = false);
  virtual void do_io_close(int lerrno = -1);
  virtual void do_io_shutdown(ShutdownHowTo_t howto);

  // 专门给 TS API 提供的方法
  // 由TSVConnReadVIOGet, TSVConnWriteVIOGet, TSVConnClosedGet, TSTransformOutputVConnGet调用
  virtual bool get_data(int id, void *data);

  // 发送带外数据的子状态机及对应的cancel方法
  virtual Action *send_OOB(Continuation *cont, char *buf, int len);
  virtual void cancel_OOB();

  // 支持 SSL 功能的方法
  virtual void
  setSSLHandshakeWantsRead(bool /* flag */)
  { return; }
  virtual bool
  getSSLHandshakeWantsRead()
  { return false; }
  virtual void
  setSSLHandshakeWantsWrite(bool /* flag */)
  { return; }
  virtual bool
  getSSLHandshakeWantsWrite()
  { return false; }


  ////////////////////////////////////////////////////////////
  // Set the timeouts associated with this connection.      //
  // active_timeout is for the total elasped time of        //
  // the connection.                                        //
  // inactivity_timeout is the elapsed time from the time   //
  // a read or a write was scheduled during which the       //
  // connection  was unable to sink/provide data.           //
  // calling these functions repeatedly resets the timeout. //
  // These functions are NOT THREAD-SAFE, and may only be   //
  // called when handing an  event from this NetVConnection,//
  // or the NetVConnection creation callback.               //
  ////////////////////////////////////////////////////////////
  // 面向上层状态机提供的超时管理
  virtual void set_active_timeout(ink_hrtime timeout_in);
  virtual void set_inactivity_timeout(ink_hrtime timeout_in);
  virtual void cancel_active_timeout();
  virtual void cancel_inactivity_timeout();
  // 面向上层状态机提供的连接池管理
  virtual void add_to_keep_alive_queue();
  virtual void remove_from_keep_alive_queue();
  virtual bool add_to_active_queue();
  virtual void remove_from_active_queue();

  // 当上层状态机调用VIO::reenable()时，由其发起调用，不应该由上层状态机调用
  // 用于激活读、或写操作的VIO
  // 实际上每一个VC内部都有两个VIO，特别对应读、写操作
  // The public interface is VIO::reenable()
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  virtual SOCKET get_socket();

  virtual ~UnixNetVConnection();

  /////////////////////////////////////////////////////////////////
  // instances of UnixNetVConnection should be allocated         //
  // only from the free list using UnixNetVConnection::alloc().  //
  // The constructor is public just to avoid compile errors.     //
  /////////////////////////////////////////////////////////////////
  // UnixNetVConnection 实例只能通过alloc()方法分配
  // 实际上是通过ProxyAllocator或者全局空间分配器完成分配
  // 在全局空间分配器内有一个UnixNetVConnection的实例，它会运行这个构造函数完成初始化
  // 通过 ProxyAllocator 或者 全局空间分配器 分配的实例都是直接copy内部实例的内存空间完成初始化
  UnixNetVConnection();

private:
  UnixNetVConnection(const NetVConnection &);
  UnixNetVConnection &operator=(const NetVConnection &);

public:
  /////////////////////////
  // UNIX implementation //
  /////////////////////////
  
  // 设置包含该VIO实例的NetState实例的enabled成员为1
  // 并且同时重新设置inactivity timeout的计时器
  void set_enabled(VIO *vio);

  // 已经废弃不用了
  // 看函数名字，猜测是获取本地Socket Address的结构
  void get_local_sa();

  // these are not part of the pure virtual interface.  They were
  // added to reduce the amount of duplicate code in classes inherited
  // from NetVConnection (SSL).
  virtual int
  sslStartHandShake(int event, int &err)
  {
    (void)event;
    (void)err;
    return EVENT_ERROR;
  }
  virtual bool
  getSSLHandShakeComplete()
  {
    return (true);
  }
  virtual bool
  getSSLClientConnection()
  {
    return (false);
  }
  virtual void
  setSSLClientConnection(bool state)
  {
    (void)state;
  }
  
  // 进行读操作，从socket上读取数据，生产到Read VIO内的MIOBuffer，直接调用read_from_net()实现
  virtual void net_read_io(NetHandler *nh, EThread *lthread);
  // 进行写操作，由write_to_net_io调用，从Write VIO内的MIOBufferAccessor消费数据，然后写入socket，返回实际消费的字节数
  virtual int64_t load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                        int &needs);

  // 关闭当前VC的Read VIO，直接调用read_disable()方法实现
  //     取消超时控制，从NetHandler的read_ready_list中删除，从epoll中删除
  void readDisable(NetHandler *nh);
  // 注意，在ATS中没有 writeDisable !!!

  // 当VIO的读操作遇到错误时，此方法被调用，上层状态机会收到 ERROR 事件
  //     设置vc->lerrno=err后，直接调用read_signal_done(VC_EVENT_ERROR)方法实现
  void readSignalError(NetHandler *nh, int err);
  // 注意，UnixNetVConnection类中，没有 writeSignalError 方法，但是存在write_signal_error()函数

  // 当VIO的读操作完成时，此方法被调用，上层状态机会收到 READ_COMPLETE 或 EOS 事件
  //     直接调用read_signal_done(event)方法实现
  //     如果上层状态机在回调中没有关闭VC，
  //         那么会通过 read_reschedule 从NetHandler的read_ready_list移除对应的VC
  //     如果该VC被关闭，则由close_UnixNetVConnection负责从NetHandler的read和write队列移除VC
  int readSignalDone(int event, NetHandler *nh);
  // 注意，UnixNetVConnection类中，没有 writeSignalDone 方法，但是存在write_signal_done()函数

  // 当完成一个VIO的读操作时，此方法被调用，上层状态机会收到 READ_READY 事件
  //     直接调用read_signal_and_update(event)方法实现
  int readSignalAndUpdate(int event);
  // 注意，UnixNetVConnection类中，没有 writeSignalAndUpdate 方法，但是存在write_signal_and_update()函数

  // 重新调度当前VC的读操作
  //     直接调用read_reschedule()方法实现
  //     如果 read.triggered 和 read.enabled 都为 true 就会重新添加到NetHandler的read_ready_list队列
  //     否则，从NetHandler的read_ready_list队列移除对应的vc
  void readReschedule(NetHandler *nh);
  // 重新调度当前VC的写操作
  //     直接调用write_reschedule()方法实现
  //     如果 write.triggered 和 write.enabled 都为 true 就会重新添加到NetHandler的write_ready_list队列
  //     否则，从NetHandler的write_ready_list队列移除对应的vc
  void writeReschedule(NetHandler *nh);
  
  // 更新inactivity timeout计时器，每次执行读写操作后，都要调用此方法来更新计时器
  //     直接调用net_activity()方法实现
  void netActivity(EThread *lthread);
  
  /**
   * If the current object's thread does not match the t argument, create a new
   * NetVC in the thread t context based on the socket and ssl information in the
   * current NetVC and mark the current NetVC to be closed.
   */
  // 将VC从原来的EThread迁移到当前的EThread。
  // 这是一个2015年10月才新增加的一个方法，由Yahoo团队实现，参考：
  //     官方JIRA：TS-3797
  //     官方Github：https://github.com/apache/trafficserver/commit/c181e7eea93592fa496247118f72e8323846fd5a
  //     官方Wiki：https://cwiki.apache.org/confluence/display/TS/Threading+Issues+And+NetVC+Migration
  UnixNetVConnection *migrateToCurrentThread(Continuation *c, EThread *t);

  // 指向上层状态机
  Action action_;
  // 指示当前VC是否已经关闭
  volatile int closed;
  // 读写两个状态，其内各包含一个VIO实例，对应读写操作
  NetState read;
  NetState write;

  // 定义四种link的类型，但是此处并未定义实例，这些link的实例在NetHandler中声明
  LINKM(UnixNetVConnection, read, ready_link)
  SLINKM(UnixNetVConnection, read, enable_link)
  LINKM(UnixNetVConnection, write, ready_link)
  SLINKM(UnixNetVConnection, write, enable_link)

  // 定义用于InactivityCop内部的链表节点
  LINK(UnixNetVConnection, cop_link);
  // 定义用于实现keepalive连接池管理的链表节点
  LINK(UnixNetVConnection, keep_alive_queue_link);
  // 定义用于实现timeout操作的链表节点
  LINK(UnixNetVConnection, active_queue_link);

  // 定义inactivity timeout，我理解为IDLE Timeout，就是这个连接上没有任何数据传输的时候，最长等待时间
  ink_hrtime inactivity_timeout_in;
  // 定义active timeout，我理解为OPERATION Timeout，就是当进行一个异步读、写操作时，最长等待时间
  ink_hrtime active_timeout_in;
#ifdef INACTIVITY_TIMEOUT
  Event *inactivity_timeout;
  Event *activity_timeout;
#else
  ink_hrtime next_inactivity_timeout_at;
  ink_hrtime next_activity_timeout_at;
#endif

  // 用于操作epoll的描述符
  //     在reenable, reenable_re, acceptEvent, connectUp中使用到
  EventIO ep;
  // 指向管理此连接的NetHandler
  NetHandler *nh;
  // 此连接的唯一ID，自增长值，通过UnixNetProcessor.cc::net_next_connection_number()分配
  unsigned int id;
  // 服务器端的地址（感觉此处仅仅是为了兼容性）
  //     在NetAccept中，在connect_re_internal中，都使用ats_ip_copy方法，从con.addr直接内存复制过来
  //     在Connection::connect中调用setRemote方法，也使用ats_ip_copy方法，从target直接内存复制到con.addr
  // AMC大神在这里做了一个注释，觉得这儿重复定义了，其实可以使用con.addr
  // amc - what is this for? Why not use remote_addr or con.addr?
  IpEndpoint server_addr; /// Server address and port.

  // 可以是一个flags，或者用来标记半关闭的状态
  union {
    unsigned int flags;
#define NET_VC_SHUTDOWN_READ 1
#define NET_VC_SHUTDOWN_WRITE 2
    struct {
      unsigned int got_local_addr : 1;
      unsigned int shutdown : 2;
    } f;
  };

  // 与当前VC实例对应的底层socket描述结构，其内包含文件描述符
  Connection con;
  
  // 统计UnixNetVConnection作为状态机时的重入情况
  int recursion;
  // 记录当前实例的创建时间
  ink_hrtime submit_time;
  // 带外数据的回调方法指针
  OOB_callback *oob_ptr;
  
  // true＝当前VC实例通过DEDICATED EThread模式的NetAccept创建
  // 表示该VC的内存由全局空间分配，在释放时，需要直接调用全局空间释放
  bool from_accept_thread;

  // 用来调试和追踪数据传输的
  // es - origin_trace associated connections
  bool origin_trace;
  const sockaddr *origin_trace_addr;
  int origin_trace_port;

  // 作为状态机时的回调方法
  // startEvent通常由connectUp设置
  int startEvent(int event, Event *e);
  // acceptEvent通常由NetAccept在创建VC后设置
  int acceptEvent(int event, Event *e);
  // mainEvent通常由InactivityCop调用，用来实现超时控制
  // startEvent和acceptEvent在执行完成之后，都会转入mainEvent
  int mainEvent(int event, Event *e);
  
  // 作为Client发起一个socket连接，状态机回调函数设置为startEvent
  virtual int connectUp(EThread *t, int fd);
  
  /**
   * Populate the current object based on the socket information in in the
   * con parameter.
   * This is logic is invoked when the NetVC object is created in a new thread context
   */
  // 这是一个2015年10月才新增加的一个方法，由Yahoo团队实现，参见上面的：migrateToCurrentThread
  virtual int populate(Connection &con, Continuation *c, void *arg);
  
  // 释放该VC，并不会释放内存，只是把内存归还到freelist内存池或者全局内存池
  //     同时会释放该vc使用的mutex，上层状态机使用的mutex，读、写VIO使用的mutex
  virtual void free(EThread *t);

  // 返回成员：inactivity_timeout_in
  virtual ink_hrtime get_inactivity_timeout();
  // 返回成员：active_timeout_in
  virtual ink_hrtime get_active_timeout();

  // 设置本地地址，local_addr.sa
  virtual void set_local_addr();
  // 设置远端地址，remote_addr
  virtual void set_remote_addr();
  
  // 设置 TCP 的 INIT CWND 参数
  virtual int set_tcp_init_cwnd(int init_cwnd);
  
  // 设置 TCP 的一些参数
  //     直接调用 con.apply_options(options) 实现
  virtual void apply_options();

  // 进行写操作，为了与SSL部分兼容，实际是调用load_buffer_and_write来完成socket的写操作
  friend void write_to_net_io(NetHandler *, UnixNetVConnection *, EThread *);

  void
  setOriginTrace(bool t)
  {
    origin_trace = t;
  }

  void
  setOriginTraceAddr(const sockaddr *addr)
  {
    origin_trace_addr = addr;
  }

  void
  setOriginTracePort(int port)
  {
    origin_trace_port = port;
  }
};

extern ClassAllocator<UnixNetVConnection> netVCAllocator;

typedef int (UnixNetVConnection::*NetVConnHandler)(int, void *);

```

## 从Socket到MIOBuffer

数据是如何从Socket读取到MIOBuffer的？

前面在介绍NetAccept的时候，我们知道：

  - NetAccept负责接受一个新连接，并创建一个VC
  - 然后向上层状态机的Acceptor传递EVENT_NET_ACCEPT事件
  - 由上层状态机的Acceptor来创建对应的上层状态机
  - 然后就由NetHandler来根据上层状态机注册的读、写VIO 和 收到的读、写等事件来回调上层状态机。

因此，对于此问题的答案，要从NetHandler找起。

回顾 NetHandler::mainNetEvent 的分析，我们知道：

  - NetHandler 通过 epoll_wait 发现底层 socket 是否有数据可读取
  - 然后把有数据可读取的vc放入read_ready_list
  - 然后遍历read_ready_list
  - 对于Read VIO处于激活状态的vc，调用 vc->net_read_io(this, trigger_event->ethread) 实现数据的接收
  - 然后回调上层状态机
  - 对于接收的结果：成功，失败，是否完成VIO，向上层状态机传递不同的事件，同时传递VC或VIO

下面详细分析 net_read_io 函数：

```
// net_read_io 是一个封装函数，实际上调用了read_from_net()
// 在ssl的实现里，会重载这个方法，调用ssl_read_from_net()
void
UnixNetVConnection::net_read_io(NetHandler *nh, EThread *lthread)
{
  read_from_net(nh, this, lthread);
}

// Read the data for a UnixNetVConnection.
// Rescheduling the UnixNetVConnection by moving the VC
// onto or off of the ready_list.
// Had to wrap this function with net_read_io for SSL.
// 这才是真正负责读取socket并且把读取到的数据放入MIOBuffer的方法
static void
read_from_net(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  NetState *s = &vc->read;
  ProxyMutex *mutex = thread->mutex;
  int64_t r = 0;

  // 尝试获取对此VC的VIO的Mutex锁
  MUTEX_TRY_LOCK_FOR(lock, s->vio.mutex, thread, s->vio._cont);
  if (!lock.is_locked()) {
    // 如果没有拿到锁，就重新调度，等下一次再读取
    // read_reschedule 会把此vc放回到read_ready_link的队列尾部
    // 由于NetHandler的处理方式，必然会在本次EventSystem回调NetHandler中完成所有的读操作
    //     不会延迟到下一次EventSystem对NetHandler的回调
    read_reschedule(nh, vc);
    return;
  }

  // It is possible that the closed flag got set from HttpSessionManager in the
  // global session pool case.  If so, the closed flag should be stable once we get the
  // s->vio.mutex (the global session pool mutex).
  // 拿到锁之后，首先判断该vc是否被异步关闭了，因为 HttpSessionManager 会管理一个全局的session池。
  if (vc->closed) {
    close_UnixNetVConnection(vc, thread);
    return;
  }
  // if it is not enabled.
  // 再判断，该vc的读操作是否被异步禁止了
  if (!s->enabled || s->vio.op != VIO::READ) {
    read_disable(nh, vc);
    return;
  }

  // 下面才开始进入读操作前的准备工作
  // 获取缓冲区，准备写入数据
  MIOBufferAccessor &buf = s->vio.buffer;
  // 缓冲区如果不存在则报异常并crash
  ink_assert(buf.writer());

  // if there is nothing to do, disable connection
  // 在VIO中定义了总共需要读取的数据的总长度，还有已经完成的数据读取长度
  // ntodo 是剩余的，还需要读取的数据长度
  int64_t ntodo = s->vio.ntodo();
  if (ntodo <= 0) {
    // ntodo为0，表示VIO定义的读操作在之前就已经全部完成了
    // 因此直接禁止此VC的读操作
    read_disable(nh, vc);
    return;
  }
  
  // 查看MIOBuffer缓冲区剩余的可写空间，作为本次即将读取的数据长度
  int64_t toread = buf.writer()->write_avail();
  // 如果 toread 大于 ntodo，就调整一下 toread 的值
  // 需要注意：toread 可能会等于 0
  if (toread > ntodo)
    toread = ntodo;

  // read data
  // 读取数据
  // 因为MIOBuffer的底层可能会有多个IOBufferBlock，因此会使用readv系统调用，注册多个数据块到IOVec向量完成读取操作。
  // total_read 表示实际读取到的数据长度
  int64_t rattempted = 0, total_read = 0;
  int niov = 0;
  IOVec tiovec[NET_MAX_IOV];
  // 只有 toread 大于 0 的时候才进行真正的读操作
  if (toread) {
    IOBufferBlock *b = buf.writer()->first_write_block();
    do {
      // niov 值被设置为本次IOVec向量的数量，表示使用了几个内存块来保存数据
      niov = 0;
      // rattempted 值被设置为本次计划读取的数据长度
      rattempted = 0;
      
      // 根据底层的IOBufferBlock来构建IOVec向量数组
      // 但是每次构建的IOVec向量数量最大值为 NET_MAX_IOV
      // 如果要读的数据比较多，就分成多次调用readv
      while (b && niov < NET_MAX_IOV) {
        int64_t a = b->write_avail();
        if (a > 0) {
          tiovec[niov].iov_base = b->_end;
          int64_t togo = toread - total_read - rattempted;
          if (a > togo)
            a = togo;
          tiovec[niov].iov_len = a;
          rattempted += a;
          niov++;
          if (a >= togo)
            break;
        }
        b = b->next;
      }

      // 如果只有一个数据块，就调用read方法，否则调用readv方法
      if (niov == 1) {
        r = socketManager.read(vc->con.fd, tiovec[0].iov_base, tiovec[0].iov_len);
      } else {
        r = socketManager.readv(vc->con.fd, &tiovec[0], niov);
      }
      // 更新读操作次数计数器
      NET_INCREMENT_DYN_STAT(net_calls_to_read_stat);

      // 数据读取的调试追踪
      if (vc->origin_trace) {
        char origin_trace_ip[INET6_ADDRSTRLEN];

        ats_ip_ntop(vc->origin_trace_addr, origin_trace_ip, sizeof(origin_trace_ip));

        if (r > 0) {
          TraceIn((vc->origin_trace), vc->get_remote_addr(), vc->get_remote_port(), "CLIENT %s:%d\tbytes=%d\n%.*s", origin_trace_ip,
                  vc->origin_trace_port, (int)r, (int)r, (char *)tiovec[0].iov_base);

        } else if (r == 0) {
          TraceIn((vc->origin_trace), vc->get_remote_addr(), vc->get_remote_port(), "CLIENT %s:%d closed connection",
                  origin_trace_ip, vc->origin_trace_port);
        } else {
          TraceIn((vc->origin_trace), vc->get_remote_addr(), vc->get_remote_port(), "CLIENT %s:%d error=%s", origin_trace_ip,
                  vc->origin_trace_port, strerror(errno));
        }
      }

      // 更新 total_read 计数器
      total_read += rattempted;
    // 如果数据读取成功(r == rattempted)，并且VIO的任务没有完成(total_read < toread)，则继续do-while循环读取数据
    } while (rattempted && r == rattempted && total_read < toread);
    // 如果读取出现了失败(r != rattempted)，则跳出do-while循环，停止读取操作，此时可能完成了部分数据的读取

    // if we have already moved some bytes successfully, summarize in r
    // 根据 r 的值对total_read计数器进行调整
    if (total_read != rattempted) {
      if (r <= 0)
        r = total_read - rattempted;
      else
        r = total_read - rattempted + r;
    }
    // check for errors
    // 当 r<=0 表示发生了错误
    if (r <= 0) {
      // EAGAIN 表示缓冲区读空了，没有数据可度，那么就准备下一次epoll_wait触发时再读取
      if (r == -EAGAIN || r == -ENOTCONN) {
        NET_INCREMENT_DYN_STAT(net_calls_to_read_nodata_stat);
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        return;
      }

      // 如果连接被关闭了，就触发EOS通知上层状态机
      if (!r || r == -ECONNRESET) {
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        read_signal_done(VC_EVENT_EOS, nh, vc);
        return;
      }
      
      // 其它情况，触发ERROR通知上层状态机
      vc->read.triggered = 0;
      read_signal_error(nh, vc, (int)-r);
      return;
    }
    
    // 读取成功，更新数据读取计数器
    NET_SUM_DYN_STAT(net_read_bytes_stat, r);

    // Add data to buffer and signal continuation.
    // 更新MIOBuffer的计数器，增加可供消费的数据量
    buf.writer()->fill(r);
#ifdef DEBUG
    if (buf.writer()->write_avail() <= 0)
      Debug("iocore_net", "read_from_net, read buffer full");
#endif
    // 更新VIO的计数器，增加已经完成的数据量
    s->vio.ndone += r;
    // 刷新超时计时器
    net_activity(vc, thread);
  } else
    r = 0;
  // 如果缓冲区写满了，则没有发生真正的读操作，设置 r 为 0

  // Signal read ready, check if user is not done
  // 当 r>0 时，表示本次读取到了数据
  //     那么要回调上层状态机，但是需要判断应该传递什么样的事件
  if (r) {
    // If there are no more bytes to read, signal read complete
    ink_assert(ntodo >= 0);
    if (s->vio.ntodo() <= 0) {
      // 如果 VIO 的要求全部完成了，那么回调READ_COMPLETE给上层状态机
      read_signal_done(VC_EVENT_READ_COMPLETE, nh, vc);
      Debug("iocore_net", "read_from_net, read finished - signal done");
      return;
    } else {
      // 如果 VIO 的要求没有完成，那么回调READ_READY给上层状态机
      // 如果返回值不为EVENT_CONT，那么就表示上层状态机关闭了vc，就不用再继续处理了。
      if (read_signal_and_update(VC_EVENT_READ_READY, vc) != EVENT_CONT)
        return;

      // change of lock... don't look at shared variables!
      // 如果上层状态机把VIO的mutex更换了，那么我们之前的锁定已经失效
      // 因此不能再继续操作该VIO了，只能重新调度vc，把vc放回到read_ready_list等待下一次NetHandler的下一次调用
      if (lock.get_mutex() != s->vio.mutex.m_ptr) {
        read_reschedule(nh, vc);
        return;
      }
    }
  }
  
  // If here are is no more room, or nothing to do, disable the connection
  // 如果：
  //     MIOBuffer没有空间可写（没有发生真正的读操作）
  // 或者：
  //     读取了数据，但是没有完成全部的VIO操作，而且上层状态机没有关闭VC，也没有修改Mutex
  // 那么，如果：
  //     上层状态机可能取消了VIO的操作：s->vio.ntodo() <= 0
  //     上层状态机可能禁止了读VIO：s->enabled == 0
  //     上层状态机可能把MIOBuffer用光了：buf.writer()->write_avail() == 0
  if (s->vio.ntodo() <= 0 || !s->enabled || !buf.writer()->write_avail()) {
  // 那么就关闭此VC的读操作。
    read_disable(nh, vc);
    return;
  }

  // 其它情况：重新调度vc的读操作，此时，以下条件必然满足：
  //     s->vio.ntodo() > 0
  //     s->enabled == 1
  //     buf.writer()->write_avail() > 0
  read_reschedule(nh, vc);
}
```

## 创建与Origin Server的连接

如何创建与源服务器的连接？

## 从MIOBuffer到Socket

数据是如何从MIOBuffer写入Socket的？

回顾 NetHandler::mainNetEvent 的分析，我们知道：

  - NetHandler 通过 epoll_wait 发现底层 socket 是否可写
  - 然后把有数据可读取的vc放入write_ready_list
  - 然后遍历write_ready_list
  - 对于Write VIO处于激活状态的vc，调用 write_to_net(this, vc, trigger_event->ethread) 实现数据的发送
  - 然后回调上层状态机
  - 对于发送的结果：成功，失败，是否完成VIO，向上层状态机传递不同的事件，同时传递VC或VIO

下面详细分析 write_to_net 函数：

```
// write_to_net 是一个封装函数，实际上调用了write_to_net_io()
// 需要注意的是：write_to_net() 和 write_to_net_io() 都不是 UnixNetVConnection的内部方法
// 对于SSLVC来说，同样也会调用这两个方法
void
write_to_net(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  // 这行可以删除，废弃的代码
  ProxyMutex *mutex = thread->mutex;

  NET_INCREMENT_DYN_STAT(net_calls_to_writetonet_stat);
  NET_INCREMENT_DYN_STAT(net_calls_to_writetonet_afterpoll_stat);


  write_to_net_io(nh, vc, thread);
}

//
// Write the data for a UnixNetVConnection.
// Rescheduling the UnixNetVConnection when necessary.
//
// write_to_net_io 并不负责发送数据，它需要调用 load_buffer_and_write() 完成数据的发送
// 但是它负责在数据发送前和发送之后，回调上层状态机
// 对于 SSLNetVConnection，通过重载 load_buffer_and_write 实现数据的加密发送
void
write_to_net_io(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  NetState *s = &vc->write;
  ProxyMutex *mutex = thread->mutex;

  // 尝试获取对此VC的VIO的Mutex锁
  MUTEX_TRY_LOCK_FOR(lock, s->vio.mutex, thread, s->vio._cont);
  if (!lock.is_locked() || lock.get_mutex() != s->vio.mutex.m_ptr) {
    // 如果没有拿到锁，
    //     或者拿到锁之后，该VIO的Mutex被异步更改了。
    //     (这是比read_net_io多出来的一个判断，感觉好神奇，不太懂怎么会出现这种情况)
    // 就重新调度，等下一次再读取
    // write_reschedule 会把此vc放回到write_ready_link的队列尾部
    // 由于NetHandler的处理方式，必然会在本次EventSystem回调NetHandler中完成所有的写操作，直到内核的写缓冲区都满了
    //     不会延迟到下一次EventSystem对NetHandler的回调
    write_reschedule(nh, vc);
    return;
  }

  // This function will always return true unless
  // vc is an SSLNetVConnection.
  // 判断当前vc是否是一个SSLVC，此处暂时跳过。
  if (!vc->getSSLHandShakeComplete()) {
    // 此处略去了对SSLVC处理的代码
    // ...
  }
  
  // 此处没有判断VC被异步关闭的情况？？？vc->closed ？？？
  
  // If it is not enabled,add to WaitList.
  // 如果，该vc的写操作被异步禁止了
  // 或者，VIO的操作方法不是写操作（此处是比Read部分多出来的判断条件）
  if (!s->enabled || s->vio.op != VIO::WRITE) {
    write_disable(nh, vc);
    return;
  }
  
  // If there is nothing to do, disable
  // 在VIO中定义了总共需要发送的数据的总长度，还有已经完成的数据发送长度
  // ntodo 是剩余的，还需要发送的数据长度
  int64_t ntodo = s->vio.ntodo();
  if (ntodo <= 0) {
    // ntodo为0，表示VIO定义的写操作在之前就已经全部完成了
    // 因此直接禁止此VC的写操作
    write_disable(nh, vc);
    return;
  }

  // 下面才开始进入写操作前的准备工作
  // 获取缓冲区，准备获取数据进行发送
  MIOBufferAccessor &buf = s->vio.buffer;
  // 缓冲区如果不存在则报异常并crash
  ink_assert(buf.writer());
  
  // 对比read部分，是先判断缓冲区是否存在，再判断ntodo
  // 因此，对于Write VIO处于禁止状态的时候，Write VIO MIOBuffer是可能为空的。
  // 官方JIRA：https://issues.apache.org/jira/browse/TS-4055 描述的bug，就是由于没有判断Write VIO的MIOBuffer为空导致的

  // Calculate amount to write
  // 查看MIOBuffer缓冲区中包含的可供读取的数据，作为本次即将发送的数据长度
  int64_t towrite = buf.reader()->read_avail();
  // 如果 towrite 大于 ntodo，就调整一下 towrite 的值
  // 需要注意：towrite 可能会等于 0
  if (towrite > ntodo)
    towrite = ntodo;

  // 标记在本次调用中，是否已经回调过上层状态机
  //     由于在写操作之前，可能会先回调一次上层状态机
  //     在写操作之后，是否还要再次回调上层状态机，需要查看之前是否已经回调过
  //     所以，通过一个临时变量记录一下。
  int signalled = 0;

  // signal write ready to allow user to fill the buffer
  // 当 towrite < ntodo 并且 缓冲区中还有剩余空间，可以填充更多数据时
  if (towrite != ntodo && buf.writer()->write_avail()) {
    // 回调上层状态机，传递WRITE_READY事件
    // 让上层状态机在写操作之前填充一次缓冲区
    if (write_signal_and_update(VC_EVENT_WRITE_READY, vc) != EVENT_CONT) {
      // 如果在上层状态机中发生了错误，就直接返回
      return;
    }
    // 重新计算VIO的ntodo的值，因为在上层状态机中可能会重新设置VIO
    ntodo = s->vio.ntodo();
    // 如果VIO被设置为已完成，就禁止vc的Write VIO，直接返回
    if (ntodo <= 0) {
      write_disable(nh, vc);
      return;
    }
    
    // 标记上层状态机已经被回调过了
    signalled = 1;
    
    // 重新计算towrite
    // Recalculate amount to write
    towrite = buf.reader()->read_avail();
    if (towrite > ntodo)
      towrite = ntodo;
  }
  
  // if there is nothing to do, disable
  ink_assert(towrite >= 0);
  // 此时towrite仍然可能为0，因此如果towrite为0，就禁止vc的Write VIO，直接返回
  if (towrite <= 0) {
    write_disable(nh, vc);
    return;
  }

  // 调用 vc->load_buffer_and_write 完成实际的写操作
  int64_t total_written = 0, wattempted = 0;
  int needs = 0;
  int64_t r = vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);

  // if we have already moved some bytes successfully, summarize in r
  if (total_written != wattempted) {
    if (r <= 0)
      r = total_written - wattempted;
    else
      r = total_written - wattempted + r;
  }
  // check for errors
  if (r <= 0) { // if the socket was not ready,add to WaitList
    if (r == -EAGAIN || r == -ENOTCONN) {
      NET_INCREMENT_DYN_STAT(net_calls_to_write_nodata_stat);
      if ((needs & EVENTIO_WRITE) == EVENTIO_WRITE) {
        vc->write.triggered = 0;
        nh->write_ready_list.remove(vc);
        write_reschedule(nh, vc);
      }
      if ((needs & EVENTIO_READ) == EVENTIO_READ) {
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        read_reschedule(nh, vc);
      }
      return;
    }
    if (!r || r == -ECONNRESET) {
      vc->write.triggered = 0;
      write_signal_done(VC_EVENT_EOS, nh, vc);
      return;
    }
    vc->write.triggered = 0;
    write_signal_error(nh, vc, (int)-r);
    return;
  } else {
    int wbe_event = vc->write_buffer_empty_event; // save so we can clear if needed.

    NET_SUM_DYN_STAT(net_write_bytes_stat, r);

    // Remove data from the buffer and signal continuation.
    ink_assert(buf.reader()->read_avail() >= r);
    buf.reader()->consume(r);
    ink_assert(buf.reader()->read_avail() >= 0);
    s->vio.ndone += r;

    // If the empty write buffer trap is set, clear it.
    if (!(buf.reader()->is_read_avail_more_than(0)))
      vc->write_buffer_empty_event = 0;

    net_activity(vc, thread);
    // If there are no more bytes to write, signal write complete,
    ink_assert(ntodo >= 0);
    if (s->vio.ntodo() <= 0) {
      write_signal_done(VC_EVENT_WRITE_COMPLETE, nh, vc);
      return;
    } else if (signalled && (wbe_event != vc->write_buffer_empty_event)) {
      // @a signalled means we won't send an event, and the event values differing means we
      // had a write buffer trap and cleared it, so we need to send it now.
      if (write_signal_and_update(wbe_event, vc) != EVENT_CONT)
        return;
    } else if (!signalled) {
      if (write_signal_and_update(VC_EVENT_WRITE_READY, vc) != EVENT_CONT) {
        return;
      }
      // change of lock... don't look at shared variables!
      if (lock.get_mutex() != s->vio.mutex.m_ptr) {
        write_reschedule(nh, vc);
        return;
      }
    }
    if (!buf.reader()->read_avail()) {
      write_disable(nh, vc);
      return;
    }

    if ((needs & EVENTIO_WRITE) == EVENTIO_WRITE) {
      write_reschedule(nh, vc);
    }
    if ((needs & EVENTIO_READ) == EVENTIO_READ) {
      read_reschedule(nh, vc);
    }
    return;
  }
}

// This code was pulled out of write_to_net so
// I could overwrite it for the SSL implementation
// (SSL read does not support overlapped i/o)
// without duplicating all the code in write_to_net.
// 在 SSLNetVConnection 通过重载此方法，完成SSL数据的加密发送
int64_t
UnixNetVConnection::load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                          int &needs)
{
  int64_t r = 0;

  // XXX Rather than dealing with the block directly, we should use the IOBufferReader API.
  int64_t offset = buf.reader()->start_offset;
  IOBufferBlock *b = buf.reader()->block;

  do {
    IOVec tiovec[NET_MAX_IOV];
    int niov = 0;
    int64_t total_written_last = total_written;
    while (b && niov < NET_MAX_IOV) {
      // check if we have done this block
      int64_t l = b->read_avail();
      l -= offset;
      if (l <= 0) {
        offset = -l;
        b = b->next;
        continue;
      }
      // check if to amount to write exceeds that in this buffer
      int64_t wavail = towrite - total_written;
      if (l > wavail)
        l = wavail;
      if (!l)
        break;
      total_written += l;
      // build an iov entry
      tiovec[niov].iov_len = l;
      tiovec[niov].iov_base = b->start() + offset;
      niov++;
      // on to the next block
      offset = 0;
      b = b->next;
    }
    wattempted = total_written - total_written_last;
    if (niov == 1)
      r = socketManager.write(con.fd, tiovec[0].iov_base, tiovec[0].iov_len);
    else
      r = socketManager.writev(con.fd, &tiovec[0], niov);

    if (origin_trace) {
      char origin_trace_ip[INET6_ADDRSTRLEN];
      ats_ip_ntop(origin_trace_addr, origin_trace_ip, sizeof(origin_trace_ip));

      if (r > 0) {
        TraceOut(origin_trace, get_remote_addr(), get_remote_port(), "CLIENT %s:%d\tbytes=%d\n%.*s", origin_trace_ip,
                 origin_trace_port, (int)r, (int)r, (char *)tiovec[0].iov_base);

      } else if (r == 0) {
        TraceOut(origin_trace, get_remote_addr(), get_remote_port(), "CLIENT %s:%d closed connection", origin_trace_ip,
                 origin_trace_port);
      } else {
        TraceOut(origin_trace, get_remote_addr(), get_remote_port(), "CLIENT %s:%d error=%s", origin_trace_ip, origin_trace_port,
                 strerror(errno));
      }
    }

    ProxyMutex *mutex = thread->mutex;
    NET_INCREMENT_DYN_STAT(net_calls_to_write_stat);
  } while (r == wattempted && total_written < towrite);

  needs |= EVENTIO_WRITE;

  return (r);
}
```

## Inactivity Timeout 的实现

## InactivityCop 状态机

## 参考资料

- [P_UnixNetState.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetState.h)
- [P_UnixNet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [P_UnixNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetVConnection.h)
- [UnixNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
