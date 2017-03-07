# 核心组件：UnixNetVConnection

实际上IOCoreNet子系统，与EventSystem是一样的，也有Thread，Processor和Event，只是名字不一样了：

|  EventSystem   |    epoll   |  Polling SubSystem  |        NetSubSystem       |
|:--------------:|:----------:|:-------------------:|:-------------------------:|
|      Event     |  socket fd |       EventIO       |     UnixNetVConnection    |
|     EThread    | epoll_wait |      PollCont       | NetHandler，InactivityCop |
| EventProcessor |  epoll_ctl |       EventIO       |        NetProcessor       |

- 像 Event 一样，UnixNetVConnection 也提供了面向上层状态机的方法
  - do_io_* 系列
  - (set|cancel)_*_timeout 系列
  - (add|remove)_*_queue 系列
  - 还有一部分面向上层状态机的方法在NetProcessor中定义
- UnixNetVConnection 也提供了面向底层状态机的方法
  - 通常由NetHandler来调用
  - 可以把这些方法视作NetHandler状态机的专用回调函数
  - 我个人觉得，应该把所有跟socket打交道的函数都放在NetHandler里面
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

## 定义 & 方法

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
  // 每次读、写操作都会重置IDLE Timeout的计时器
  ink_hrtime inactivity_timeout_in;
  // 定义active timeout，我理解为NetVC LifeCycle Timeout，就是一个NetVC可以生存多久
  // 重设可以延长NetVC的寿命
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
  // 对端（peer side）的地址
  //     当ATS接受一个客户端的连接时，把客户端的IP地址填入这个变量。
  //     当ATS连接一个服务端的时候，把服务端IP地址填入这个变量。
  //     因此这相当于get_remote_addr()的结果，不要被server_addr的字面意思弄晕了。
  // AMC大神在这里做了一个注释，觉得这儿重复定义了，其实可以使用remote_addr或con.addr代替
  // 我提交了一个patch，使用get_remote_addr()来代替server_addr这个变量
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

## NetHandler的延伸：从Socket到MIOBuffer

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
      // 因此不能再继续操作该VIO了，只能重新调度vc，把vc放回到read_ready_list等待下一次NetHandler的调用
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

## NetHandler的延伸：从MIOBuffer到Socket

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
  // 如果 towrite < ntodo : 那么本次一定不会回调WRITE_COMPLETE到上层状态机
  // 并且 缓冲区中还有剩余空间，可以填充更多数据时
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
  // wattempted 表示尝试写的数据量，total_written 表示实际写成功的数据量
  int64_t total_written = 0, wattempted = 0;
  // 使用 needs 表示在执行完本次写操作之后，通过 或运算 设置对应的标志位，标记是否需要重新调度读、写操作：
  //     用来实现SSLNetVC在握手过程中需要的多次读写过程，
  //     对于UnixNetVC，当VIO未完成并且源MIOBuffer有数据可读时，总是设置EVENTIO_WRITE
  int needs = 0;
  // r 值为最后一次写操作的返回值，大于0表示实际写入的字节，小于等于0表示错误码
  int64_t r = vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);

  // if we have already moved some bytes successfully, summarize in r
  // 如果部分写成功，调整 r 值，用于表示实际发送成功的数据量。
  if (total_written != wattempted) {
    if (r <= 0)
      r = total_written - wattempted;
    else
      r = total_written - wattempted + r;
  }
  // check for errors
  // 当 r<=0 表示没有任何数据发送，因为没有一次成功的写操作
  if (r <= 0) { // if the socket was not ready,add to WaitList
    // 判断EAGAIN的情况，表示写缓冲满了
    if (r == -EAGAIN || r == -ENOTCONN) {
      NET_INCREMENT_DYN_STAT(net_calls_to_write_nodata_stat);
      // 从write_ready_list中删除，等下一次epoll_wait事件再继续写
      if ((needs & EVENTIO_WRITE) == EVENTIO_WRITE) {
        vc->write.triggered = 0;
        nh->write_ready_list.remove(vc);
        write_reschedule(nh, vc);
      }
      // 对于UnixNetVConnection，needs只在load_buffer_and_write中设置EVENTIO_WRITE
      // 下面的判断用于SSLNetVConnection
      if ((needs & EVENTIO_READ) == EVENTIO_READ) {
        vc->read.triggered = 0;
        nh->read_ready_list.remove(vc);
        read_reschedule(nh, vc);
      }
      return;
    }
    
    // 如果 r==0 或者 连接关闭，回调上层状态机传递EOS事件
    if (!r || r == -ECONNRESET) {
      vc->write.triggered = 0;
      write_signal_done(VC_EVENT_EOS, nh, vc);
      return;
    }
    
    // 其它情况，回调上层状态机传递ERROR事件，并将 -r 作为data传递
    vc->write.triggered = 0;
    write_signal_error(nh, vc, (int)-r);
    return;
  } else {
    // 当 r >= 0 时，表示数据发送成功了，至少发送出去一部分数据
    //
    // WBE = Write Buffer Empty
    // 这是一个特殊的机制，当写操作把数据源的MIOBuffer完全消费掉了，会再次回调上层状态机，并传递wbe_event事件
    // 这个机制是一次性有效，每次触发后，都要在上层状态机重新设置，否则，下一次写操作完成之后就不会触发这个机制了。
    // 但是如果遇到 VIO 同时完成了，那么就不会回调上层状态机
    int wbe_event = vc->write_buffer_empty_event; // save so we can clear if needed.

    NET_SUM_DYN_STAT(net_write_bytes_stat, r);

    // Remove data from the buffer and signal continuation.
    // 根据实际发送的数据，对MIOBuffer中的数据进行消费
    ink_assert(buf.reader()->read_avail() >= r);
    buf.reader()->consume(r);
    // 同时更新完成发送的数据量到VIO计数器中
    ink_assert(buf.reader()->read_avail() >= 0);
    s->vio.ndone += r;

    // If the empty write buffer trap is set, clear it.
    // 如果数据源的MIOBuffer完全被消费掉了，那么重置wbe_event为0
    if (!(buf.reader()->is_read_avail_more_than(0)))
      vc->write_buffer_empty_event = 0;

    // 刷新超时计时器
    net_activity(vc, thread);
    
    // If there are no more bytes to write, signal write complete,
    // 此处：由于 ntodo=s->vio.ntodo() 是在发送数据之前做的赋值，
    // 所以，ntodo表示VIO内剩余需要发送的总数据量，运行到此处时应该是大于 0 的
    // 但是，s->vio.ntodo() 则是实时获取发送之后的值，若为 0 则表示完成了 VIO
    ink_assert(ntodo >= 0);
    if (s->vio.ntodo() <= 0) {
      // 如果 VIO 的要求全部完成了，那么回调WRITE_COMPLETE给上层状态机，并返回
      write_signal_done(VC_EVENT_WRITE_COMPLETE, nh, vc);
      return;
    // 如果 VIO 的要求还没有完成：
    } else if (signalled && (wbe_event != vc->write_buffer_empty_event)) {
      // @a signalled means we won't send an event, and the event values differing means we
      // had a write buffer trap and cleared it, so we need to send it now. 
      // ！！！此处的注释信息与代码不一致，应该是注释写错了！！！
      // 按照代码含义解释为：
      //     如果之前回调过上层状态机，而且设置了wbe_event，那么则回调wbe_event给上层状态机
      // 如果返回值不为EVENT_CONT，那么就表示上层状态机关闭了vc，就不用再继续处理了。
      if (write_signal_and_update(wbe_event, vc) != EVENT_CONT)
        return;
      // ！！！此处存在问题，上层状态机可能会修改mutex，如果此处继续运行可能会出现问题！！！
    } else if (!signalled) {
      // 如果之前没有回调上层状态机，那么回调WRITE_READY给上层状态机
      // 如果返回值不为EVENT_CONT，那么就表示上层状态机关闭了vc，就不用再继续处理了。
      if (write_signal_and_update(VC_EVENT_WRITE_READY, vc) != EVENT_CONT) {
        return;
      }
      
      // change of lock... don't look at shared variables!
      // 如果上层状态机把VIO的mutex更换了，那么我们之前的锁定已经失效
      // 因此不能再继续操作该VIO了，只能重新调度vc，把vc放回到write_ready_list等待下一次NetHandler的调用
      if (lock.get_mutex() != s->vio.mutex.m_ptr) {
        write_reschedule(nh, vc);
        return;
      }
    }
    
    // 当前 VIO 的要求还没有完成，如果：
    //     写操作之前回调过上层状态机，但是没有设置wbe_event，写操作之后并未回调过上层状态机
    // 或者：
    //     写操作之后回调了上层状态机，但是上层状态机没有关闭VC，也没有修改Mutex
    // 那么，如果：
    //     源MIOBuffer内的数据已经全部发送了：buf.reader()->read_avail() == 0
    if (!buf.reader()->read_avail()) {
      // 那么就关闭此VC的写操作，并返回
      write_disable(nh, vc);
      return;
    }

    // 源MIOBuffer内的数据没有发送完成时，根据needs值，重新调度读、写操作
    // 对于UnixNetVC，此时needs总是设置EVENTIO_WRITE
    if ((needs & EVENTIO_WRITE) == EVENTIO_WRITE) {
      write_reschedule(nh, vc);
    }
    // 对于SSLNetVC，可能会同时设置EVENTIO_READ
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
// 通过MIOBufferAccessor buf消费MIOBuffer内指定长度为towrite的数据，并发送出去
// 以下参数，调用前必须初始化为0：
//     wattempted 为一个地址，在返回时设置为最后一次调用write/writev时，尝试发送的数据长度
//  total_written 为一个地址，在返回时设置为尝试发送的数据长度，调用者需要根据返回值来计算发送成功的数据长度。
//                    r  < 0 : total_written - wattempted;
//                    r >= 0 : total_written - wattempted + r;
//          needs 为一个地址，在返回时设置为接下来需要进行测操作，通过 或操作 可分别标记读（EVENTIO_READ）、写（EVENTIO_WRITE）
// 返回值：
//     最后一次调用write/writev的返回值。
//
// 在 SSLNetVConnection 通过重载此方法，完成SSL数据的加密发送
int64_t
UnixNetVConnection::load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                          int &needs)
{
  int64_t r = 0;

  // XXX Rather than dealing with the block directly, we should use the IOBufferReader API.
  // 获取数据在MIOBuffer的起始位置及第一个数据块
  int64_t offset = buf.reader()->start_offset;
  IOBufferBlock *b = buf.reader()->block;

  // 通过一个 do-while 循环，至少运行一次write/writev操作
  // 写操作成功则继续循环，在遇到任何一次错误时停止循环
  do {
    // 为了支持多个IOBufferBlock，通过writev来完成写操作，需要构建IOVec向量数组
    IOVec tiovec[NET_MAX_IOV];
    int niov = 0;
    // total_written_last 为上一次write/writev调用完成后的total_written值
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
    // 注意，在IOVec向量数组构建完成后total_written已经被累加了即将发送的数据长度
    // 设置wattempted为 本轮循环 / 本次调用write/writev 时，尝试发送的数据长度
    wattempted = total_written - total_written_last;
    // 如果只有一个数据块，就使用write，否则使用writev
    if (niov == 1)
      r = socketManager.write(con.fd, tiovec[0].iov_base, tiovec[0].iov_len);
    else
      r = socketManager.writev(con.fd, &tiovec[0], niov);

    // 调试数据发送
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

    // 更新写操作次数计数器
    ProxyMutex *mutex = thread->mutex;
    NET_INCREMENT_DYN_STAT(net_calls_to_write_stat);
    // 如果数据读取成功(r == rattempted)，并且发送任务没有完成(total_written < towrite)，则继续do-while循环读取数据
  } while (r == wattempted && total_written < towrite);
  // 如果读取出现了失败(r != rattempted)：
  //     则跳出do-while循环，停止读取操作，此时可能完成了部分数据的发送
  // 也可能是完成了发送任务(total_written == towrite)

  // 设置后续需要写操作的标志
  needs |= EVENTIO_WRITE;

  // 返回最后一次write/writev的返回值
  // 调用者需要根据 r 值，来修正 total_written 值
  return (r);
}
```

## NetHandler的延伸：对上层状态机的回调处理

如果设置了读、写VIO，那么在读、写操作中，都会产生对上层状态机的回调：

  - 传递COMPLETE事件：read_signal_done，write_signal_done
  - 传递ERROR事件：read_signal_error，write_signal_error
  - 传递READY事件：read_signal_and_update，write_signal_and_update

而 signal_error 和 signal_done 又都是调用 signal_and_update 作为底层实现，因此接下来以read_signal_and_update来说明（write_signal_and_update的流程跟它完全一致）。

```
static inline int
read_signal_and_update(int event, UnixNetVConnection *vc)
{
  // 增加 vc 重入计数器
  vc->recursion++;
  if (vc->read.vio._cont) {
    // 如果上层状态机设置了VIO，那么回调VIO关联的上层状态机
    vc->read.vio._cont->handleEvent(event, &vc->read.vio);
  } else {
    // 如果上层状态机没有设置VIO，那么执行缺省处理
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // 出现EOS，错误，超时，设置vc关闭
      // 其中：EOS和ERROR为IOCoreNET产生，Timeout由InactivityCop产生
      Debug("inactivity_cop", "event %d: null read.vio cont, closing vc %p", event, vc);
      vc->closed = 1;
      break;
    default:
      Error("Unexpected event %d for vc %p", event, vc);
      ink_release_assert(0);
      break;
    }
  }
  // 只有重入计数器为1，并且vc设置为关闭时，才执行关闭vc的操作
  if (!--vc->recursion && vc->closed) {
    /* BZ  31932 */
    ink_assert(vc->thread == this_ethread());
    close_UnixNetVConnection(vc, vc->thread);
    return EVENT_DONE;
  } else {
    return EVENT_CONT;
  }
}

```

## NetHandler的延伸：关于 VC 回调的重入

在Proxy的实现里，由ClientSide的ServerVC，和ServerSide的ClientVC组成一个Tunnel：

  - ServerVC接受来自Client的数据，然后通过ClientVC发送给Origin Server
  - 此时ServerVC是主动的，而ClientVC是被动的
  - ClientVC接受来自Origin Server的数据，然后通过ServerVC发送给Client
  - 此时ClientVC是主动的，而ServerVC是被动的

考虑一个可能的场景：

  - 如果ServerVC由ET_NET 0管理，ClientVC由ET_NET 1管理
  - ServerVC接收到了来自Client的数据，正在写入ClientVC
  - 而此时ClientVC也接受到了来自Origin Server的数据，正在读取数据
  - 当然这两个操作是有两个不同的状态机负责，但是VC却都是同一个：ClientVC

PS：这个场景是我的想象，我还没有去代码中寻找回调重入的场景，如果你能提供ATS中与此相关的代码，请提交pull request，谢谢！

那么对于ClientVC，就出现了并发的读、写操作，如果其中一方认为该VC需要关闭：

  - 就必须等待所有使用此VC的状态机都结束
  - 此时，就遇到了VC的回调重入问题

ATS通过给VC设置一个重入计数器来解决这个问题。

问题: 为什么回调状态机时，不对vc的mutex上锁？

  - 使用mutex应该遵循一个原则：访问一个对象的内部资源时，才对此对象的mutex上锁，因此：
    - 在nethandler执行读、写操作之前，对vio.mutex进行了上锁，因为它要操作vio内的MIOBuffer
    - 在nethandler在执行读、写操作之前或之后，回调上层状态机时，也是同归vio._cont对上层状态机进行回调
  - 在回调状态机时，没有访问任何的vc对象的内部资源，因此不需要对vc的mutex进行上锁

## NetHandler的延伸：小节

在 NetHandler::mainNetEvent 中，通过以下嵌套调用：

  - UnixNetVConnection::net_read_io(this, trigger_event->ethread)
    - read_from_net(nh, this, lthread)
  - write_to_net(nh, vc, thread)
    - write_to_net_io(nh, vc, thread)
      - load_buffer_and_write

实现了数据流在socket与buffer之间的传递，因此我更觉的read_from_net，write_to_net和write_to_net_io是NetHandler的一部分，而只有 net_read_io，load_buffer_and_write 才算是 UnixNetVConnection的方法，因为继承自它的SSLNetVConnection需要重载这两个方法来实现数据的加解密。

## 超时控制和管理

对于Active Timeout 和 Inactivity Timeout，ATS分别提供了下面的方法：

  - get_(inactivity|active)_timeout
  - set_(inactivity|active)_timeout
  - cancel_(inactivity|active)_timeout

```
// get方法比较简单，直接返回成员的值
// 这里的后缀 _in 跟 schedule_in 的后缀是一个意思，是相对当前时间的，相对超时时间
TS_INLINE ink_hrtime
UnixNetVConnection::get_active_timeout()
{
  return active_timeout_in;
}

TS_INLINE ink_hrtime
UnixNetVConnection::get_inactivity_timeout()
{
  return inactivity_timeout_in;
}

// set方法需要根据超时机制的方式，由宏定义确定
// 但是无论哪种方式都需要设置 inactivity_timeout_in 的值
TS_INLINE void
UnixNetVConnection::set_inactivity_timeout(ink_hrtime timeout)
{
  Debug("socket", "Set inactive timeout=%" PRId64 ", for NetVC=%p", timeout, this);
  inactivity_timeout_in = timeout;
#ifdef INACTIVITY_TIMEOUT
  // 如果之前设置了inactivity_timeout，就要先取消之前设置的
  if (inactivity_timeout)
    inactivity_timeout->cancel_action(this);

  // 然后再判断是否要设置新的超时控制，inactivity_timeout_in == 0 表示取消超时控制
  if (inactivity_timeout_in) {
    if (read.enabled) {
      // 读操作激活，设置读操作的超时事件
      ink_assert(read.vio.mutex->thread_holding == this_ethread() && thread);
      // 分为同步调度和异步调度，设定在 inactivity_timeout_in 时间之后回调VC状态机的mainEvent
      if (read.vio.mutex->thread_holding == thread)
        inactivity_timeout = thread->schedule_in_local(this, inactivity_timeout_in);
      else
        inactivity_timeout = thread->schedule_in(this, inactivity_timeout_in);
    } else if (write.enabled) {
      // 在读操作没有设置的情况下，判断
      // 写操作激活，设置写操作的超时事件
      ink_assert(write.vio.mutex->thread_holding == this_ethread() && thread);
      // 分为同步调度和异步调度，设定在 inactivity_timeout_in 时间之后回调VC状态机的mainEvent
      if (write.vio.mutex->thread_holding == thread)
        inactivity_timeout = thread->schedule_in_local(this, inactivity_timeout_in);
      else
        inactivity_timeout = thread->schedule_in(this, inactivity_timeout_in);
    } else {
      // 读写操作都未设置，超时设置不生效
      // 清除指向event的指针
      inactivity_timeout = 0;
    }
  } else {
    // 清除指向event的指针
    inactivity_timeout = 0;
  }
#else
  // 设置 inactivity_timeout 对应的超时时间的绝对值
  next_inactivity_timeout_at = Thread::get_hrtime() + timeout;
#endif
}

// set方法需要根据超时机制的方式，由宏定义确定
// 但是无论哪种方式都需要设置 active_timeout_in 的值
// 流程上与 set_inactivity_timeout 完全一致
TS_INLINE void
UnixNetVConnection::set_active_timeout(ink_hrtime timeout)
{
  Debug("socket", "Set active timeout=%" PRId64 ", NetVC=%p", timeout, this);
  active_timeout_in = timeout;
#ifdef INACTIVITY_TIMEOUT
  if (active_timeout)
    active_timeout->cancel_action(this);
  if (active_timeout_in) {
    if (read.enabled) {
      ink_assert(read.vio.mutex->thread_holding == this_ethread() && thread);
      if (read.vio.mutex->thread_holding == thread)
        active_timeout = thread->schedule_in_local(this, active_timeout_in);
      else
        active_timeout = thread->schedule_in(this, active_timeout_in);
    } else if (write.enabled) {
      ink_assert(write.vio.mutex->thread_holding == this_ethread() && thread);
      if (write.vio.mutex->thread_holding == thread)
        active_timeout = thread->schedule_in_local(this, active_timeout_in);
      else
        active_timeout = thread->schedule_in(this, active_timeout_in);
    } else
      active_timeout = 0;
  } else
    active_timeout = 0;
#else
  next_activity_timeout_at = Thread::get_hrtime() + timeout;
#endif
}

// 取消超时设置
// cancel方法也需要根据超时机制的方式，由宏定义确定
// 但是无论哪种方式都需要对 inactivity_timeout_in 的值清零
TS_INLINE void
UnixNetVConnection::cancel_inactivity_timeout()
{
  Debug("socket", "Cancel inactive timeout for NetVC=%p", this);
  inactivity_timeout_in = 0;
#ifdef INACTIVITY_TIMEOUT
  // 取消事件
  if (inactivity_timeout) {
    Debug("socket", "Cancel inactive timeout for NetVC=%p", this);
    inactivity_timeout->cancel_action(this);
    inactivity_timeout = NULL;
  }
#else
  // 设置 inactivity_timeout 对应的超时时间的绝对值为 0
  next_inactivity_timeout_at = 0;
#endif
}

// cancel方法也需要根据超时机制的方式，由宏定义确定
// 但是无论哪种方式都需要对 active_timeout_in 的值清零
// 流程上与 cancel_inactivity_timeout 完全一致
TS_INLINE void
UnixNetVConnection::cancel_active_timeout()
{
  Debug("socket", "Cancel active timeout for NetVC=%p", this);
  active_timeout_in = 0;
#ifdef INACTIVITY_TIMEOUT
  if (active_timeout) {
    Debug("socket", "Cancel active timeout for NetVC=%p", this);
    active_timeout->cancel_action(this);
    active_timeout = NULL;
  }
#else
  next_activity_timeout_at = 0;
#endif
}
```

不管是 Inactivity Timeout 还是 Active Timeout 都要求激活了读、写操作才能设置，那么这两种Timeout到底有什么区别？

在 iocore/net/NetVCTest.cc 文件中：

```
void
NetVCTest::start_test()
{
  test_vc->set_inactivity_timeout(HRTIME_SECONDS(timeout));
  test_vc->set_active_timeout(HRTIME_SECONDS(timeout + 5));

  read_buffer = new_MIOBuffer();
  write_buffer = new_MIOBuffer();

  reader_for_rbuf = read_buffer->alloc_reader();
  reader_for_wbuf = write_buffer->alloc_reader();

  if (nbytes_read > 0) {
    read_vio = test_vc->do_io_read(this, nbytes_read, read_buffer);
  } else {
    read_done = true;
  }

  if (nbytes_write > 0) {
    write_vio = test_vc->do_io_write(this, nbytes_write, reader_for_wbuf);
  } else {
    write_done = true;
  }
}
```

设置的 active_timeout 要多出5秒，由此可以猜测，active_timeout 要比 inactivity_timeout 的时间长。

在 iocore/net/I_NetVConnection.h 中的注释，说明了差别(配上简单翻译)：

```
  /**
    在状态机使用此NetVC一定时间之后，会收到VC_EVENT_ACTIVE_TIMEOUT事件。
    如果读、写都未在此NetVC激活，那么此值会被忽略。
    这个功能防止状态机较长时间的保持连接的打开状态。
    
    Sets time after which SM should be notified.
    Sets the amount of time (in nanoseconds) after which the state
    machine using the NetVConnection should receive a
    VC_EVENT_ACTIVE_TIMEOUT event. The timeout is value is ignored
    if neither the read side nor the write side of the connection
    is currently active. The timer is reset if the function is
    called repeatedly This call can be used by SMs to make sure
    that it does not keep any connections open for a really long
    time.
    ...
   */
  virtual void set_active_timeout(ink_hrtime timeout_in) = 0;
    
  /**
    当状态机请求在此NetVC执行的IO操作没有完成时，
    在读写都处于IDLE状态一定时间之后，状态机会收到VC_EVENT_INACTIVITY_TIMEOUT事件
    任何读写操作的发生，都会导致计时器被重置。
    如果读、写都未在此NetVC激活，那么此值会被忽略。
    
    Sets time after which SM should be notified if the requested
    IO could not be performed. Sets the amount of time (in nanoseconds),
    if the NetVConnection is idle on both the read or write side,
    after which the state machine using the NetVConnection should
    receive a VC_EVENT_INACTIVITY_TIMEOUT event. Either read or
    write traffic will cause timer to be reset. Calling this function
    again also resets the timer. The timeout is value is ignored
    if neither the read side nor the write side of the connection
    is currently active. See section on timeout semantics above.
   */
  virtual void set_inactivity_timeout(ink_hrtime timeout_in) = 0;
```

所以，

 - Active Timeout，是设置一个NetVC的最大生存时间
 - Inactivity Timeout，是设置一个最长的IDLE时间

上面介绍了获取超时设置，设置超时时间，取消超时控制的方法，那么在ATS中是由谁具体实现了超时的功能呢？

请继续阅读下面mainEvent回调函数的分析。

## UnixNetVConnection 状态机的回调函数

UnixNetVConnection 具有多态性，它除了具有与Event类型相似的传递事件的功能，同时也是一个状态机，有它自己的回调函数。

### 接受新链接（acceptEvent）

回调函数 acceptEvent 是 NetAccept 在 NetVConnection 里的延伸。

当ATS设置了独立的 accept 线程，一个新的 socket 连接建立后，

  - 就新建一个 vc 与此 socket 关联，
  - 并设置此 vc 的回调函数为acceptEvent，
  - 然后根据轮训算法从对应的线程组中找到一个线程，
  - 把此 vc 交给该线程管理，
  - 在第一次回调时，会调用acceptEvent

下面来看看acceptEvent的流程分析：

```
int
UnixNetVConnection::acceptEvent(int event, Event *e)
{
  // 设置 vc 的thread，将有该thread管理此 vc，直到 vc 关闭
  thread = e->ethread;

  // 尝试对NetHandler上锁
  MUTEX_TRY_LOCK(lock, get_NetHandler(thread)->mutex, e->ethread);
  // 如果上锁失败
  if (!lock.is_locked()) {
    // 在 NetAccept::acceptEvent()中会调用 net_accept 方法，在 net_accept 方法中会同步回调 vc 状态机
    // 当event == EVENT_NONE则表示这是来自 net_accept 方法的同步回调
    // 通常只有 Management，Cluster 部分才会使用 NetAccept::init_accept 创建使用 net_accept 方法的 NetAccept 实例
    if (event == EVENT_NONE) {
      // 这里使用ethread调度函数来重新调度事件，因为这可能是一个跨线程调用，也就是：e->ethread != this_ethread()
      // 因此要使用支持原子操作的ethread里提供的调度方法，此时把此 vc 放入了 EThread 的外部队列
      thread->schedule_in(this, HRTIME_MSECONDS(net_retry_delay));
      // 由于是同步回调，要告知调用者此次调用已经完成，因为已经把该事件放入到管理此 vc 的线程中，由该线程接管后续工作了
      return EVENT_DONE;
    } else {
      // 如果是来自eventsystem的回调，则传递的应该是 EVENT_IMMEDIATE 事件
      // 因为 vc 在创建之后第一次放入线程时，是通过 schedule_imm 方法
      // 由于是来自eventsystem，也就是：e->ethread == this_ethread()
      // 只需要通过传递进来的 event 重新调度该事件在 net_retry_delay 时间之后再次回调即可
      // 此时是将此 vc 放入了 EThread 外部队列的本地队列
      e->schedule_in(HRTIME_MSECONDS(net_retry_delay));
      return EVENT_CONT;
    }
  }

  // 如果成功上锁
  
  // 判断是否被取消
  if (action_.cancelled) {
    free(thread);
    return EVENT_DONE;
  }

  // 设置回调函数为 mainEvent，之后IOCoreNet对此 vc 的回调都是mainEvent，直到 vc 关闭
  SET_HANDLER((NetVConnHandler)&UnixNetVConnection::mainEvent);

  // 通过PollDescriptor和EventIO把vc放入epoll_wait的监控中
  nh = get_NetHandler(thread);
  PollDescriptor *pd = get_PollDescriptor(thread);
  if (ep.start(pd, this, EVENTIO_READ | EVENTIO_WRITE) < 0) {
    // 错误处理
    Debug("iocore_net", "acceptEvent : failed EventIO::start\n");
    close_UnixNetVConnection(this, e->ethread);
    return EVENT_DONE;
  }

  // 加入到NetHandler管理的open_list，所有打开的vc都会放入open_list，接受超时管理
  nh->open_list.enqueue(this);

  // 如果采用EDGE模式，当然，我们使用epoll就是这种模式
#ifdef USE_EDGE_TRIGGER
  // Set the vc as triggered and place it in the read ready queue in case there is already data on the socket.
  Debug("iocore_net", "acceptEvent : Setting triggered and adding to the read ready queue");
  // 因为我们之前可能对socket设置了TCP_DEFER_ACCEPT，所以这儿可能已经有数据可读了
  // 直接模拟被epoll_wait激活时的操作，设置triggered，并将vc放入ready_list
  read.triggered = 1;
  nh->read_ready_list.enqueue(this);
#endif

  // 设置inactivity timeout
  if (inactivity_timeout_in) {
    UnixNetVConnection::set_inactivity_timeout(inactivity_timeout_in);
  }

  // 设置active timeout
  if (active_timeout_in) {
    UnixNetVConnection::set_active_timeout(active_timeout_in);
  }

  // 回调上层状态低，发送 NET_EVENT_ACCEPT 事件
  action_.continuation->handleEvent(NET_EVENT_ACCEPT, this);
  return EVENT_DONE;
}
```

### 发起新链接（startEvent）

当需要向外部发起一个连接时，首先创建一个 vc，然后设置好相关参数，就可以把 vc 放入eventsystem里了。

当startEvent被回掉的时候，就会调用connectUp，由connectUp完成后续的操作，在后面的部分会介绍connectUp方法。

```
int
UnixNetVConnection::startEvent(int /* event ATS_UNUSED */, Event *e)
{
  // 尝试对NetHandler上锁
  MUTEX_TRY_LOCK(lock, get_NetHandler(e->ethread)->mutex, e->ethread);
  if (!lock.is_locked()) {
    // 上锁失败则重新调度
    e->schedule_in(HRTIME_MSECONDS(net_retry_delay));
    return EVENT_CONT;
  }
  // 没有被取消时，调用connectUp
  if (!action_.cancelled)
    connectUp(e->ethread, NO_FD);
  else
    free(e->ethread);
  return EVENT_DONE;
}
```

### 主处理函数（mainEvent）

mainEvent 主要用来实现超时控制，Inactivity Timeout 和 Active Timeout的控制都在这里。

在ATS中，对TIMEOUT的处理分为两种模式：

  - 通过 active_timeout 和 inactivity_timeout 两个 Event 来完成
    - 就是把每一个 vc 的超时控制封装成一个定时执行的 Event 放入 EventSystem 来处理
    - 这会导致一个 vc 生成两个 Event，对EventSystem的处理压力非常的大，因此ATS设计了InactivityCop
    - ATS 5.3.x 及之前版本采用这种方式
  - 通过InactivityCop来完成
    - 这是一个专门管理连接超时的状态机，后面会专门介绍这个状态机
    - ATS 6.0.0 开始采用这种方式

在6.0.0之前，在[P_UnixNet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)中，定义了

```
#define INACTIVITY_TIMEOUT
```

这是采用基于EventSystem定时Event的机制实现的超时控制，在UnixNetVConnection中也定义了两个成员来配合实现：

```
#ifdef INACTIVITY_TIMEOUT
  Event *inactivity_timeout;
  Event *activity_timeout;
#else
  ink_hrtime next_inactivity_timeout_at;
  ink_hrtime next_activity_timeout_at;
#endif
```

但是在6.0.0开始，这一行被注释掉了，则使用一个独立的状态机InactivityCop来实现超时控制。

本章节中以介绍早期的超时处理机制为主，在后面的章节中专门介绍InactivityCop状态机。


```
//
// The main event for UnixNetVConnections.
// This is called by the Event subsystem to initialize the UnixNetVConnection
// and for active and inactivity timeouts.
//
int
UnixNetVConnection::mainEvent(int event, Event *e)
{
  // assert 判断
  //     EVENT_IMMEDIATE 来自 InactivityCop 对Inactivity Timeout的回调
  //     EVENT_INTERVAL 来自 eventsystem 对Active Timeout的回调
  ink_assert(event == EVENT_IMMEDIATE || event == EVENT_INTERVAL);
  // 只有当前ethread就是管理该 vc 的thread，才可以回调此方法
  ink_assert(thread == this_ethread());

  // 由于是超时控制，因此要尝试加一堆的锁，要把所有使用到vc的各个方面都锁定
  // 这些关联的各个部分包括：NetHandler，Read VIO，Write VIO
  //     通常VIO的mutex会引用注册VIO的上层状态机的mutex
  MUTEX_TRY_LOCK(hlock, get_NetHandler(thread)->mutex, e->ethread);
  MUTEX_TRY_LOCK(rlock, read.vio.mutex ? (ProxyMutex *)read.vio.mutex : (ProxyMutex *)e->ethread->mutex, e->ethread);
  MUTEX_TRY_LOCK(wlock, write.vio.mutex ? (ProxyMutex *)write.vio.mutex : (ProxyMutex *)e->ethread->mutex, e->ethread);
  // 判断上面三个部分是否都上锁成功了
  if (!hlock.is_locked() || !rlock.is_locked() || !wlock.is_locked() ||
  // 判断Read VIO没有被更改
      (read.vio.mutex.m_ptr && rlock.get_mutex() != read.vio.mutex.m_ptr) ||
  // 判断Write VIO没有被更改
      (write.vio.mutex.m_ptr && wlock.get_mutex() != write.vio.mutex.m_ptr)) {
    // 上述判断若有一个失败，则需要重新调度
#ifdef INACTIVITY_TIMEOUT
    // 但是只有由active_timeout事件回调才重新调度
    // 为什么inactivity_timeout事件的回调在上锁失败后就直接返回了，而不进行重新调度呢？？？
    //     因为出现上锁失败的时候，那就意味着有一个能够重置inactivity_timeout计时器的操作正在进行，所以就没有必要再重新调度了
    if (e == active_timeout)
#endif
      e->schedule_in(HRTIME_MSECONDS(net_retry_delay));
    return EVENT_CONT;
  }

  // 全部上锁成功，并且VIO没有被更改
  // 判断 Event 是否被取消
  if (e->cancelled) {
    return EVENT_DONE;
  }

  // 接下来要判断是Active Timeout还是Inactivity Timeout
  int signal_event;
  Continuation *reader_cont = NULL;
  Continuation *writer_cont = NULL;
  ink_hrtime *signal_timeout_at = NULL;
  Event *t = NULL;
  // signal_timeout 是用来支持老的超时控制模式的，在新模式下固定的指向 t
  Event **signal_timeout;
  signal_timeout = &t;

#ifdef INACTIVITY_TIMEOUT
  // 采用EventSystem实现超时控制时：
  //     传入的 e 应该是inactivity_timeout或者active_timeout
  // 根据 e 的值判断超时发生的类型，设置参数
  if (e == inactivity_timeout) {
    signal_event = VC_EVENT_INACTIVITY_TIMEOUT;
    signal_timeout = &inactivity_timeout;
  } else {
    ink_assert(e == active_timeout);
    signal_event = VC_EVENT_ACTIVE_TIMEOUT;
    signal_timeout = &active_timeout;
  }
#else
  // 采用InactivityCop实现超时控制时：
  if (event == EVENT_IMMEDIATE) {
    // 如果是来自InactivityCop的回调，则传入的event固定为EVENT_IMMEDIATE
    // 这表示发生了 Inactivity Timeout
    // Inactivity Timeout
    /* BZ 49408 */
    // ink_assert(inactivity_timeout_in);
    // ink_assert(next_inactivity_timeout_at < ink_get_hrtime());
    // 如果：
    //     没有设置Inactivity Timeout (inactivity_timeout_in==0)
    //     没有出现Inactivity Timeout (next_inactivity_timeout_at > Thread::get_hrtime())
    // 直接返回 EVENT_CONT
    if (!inactivity_timeout_in || next_inactivity_timeout_at > Thread::get_hrtime())
      return EVENT_CONT;
    // 出现了Inactivity Timeout，因此设置回调上层状态机的 event 类型为 INACTIVITY_TIMEOUT
    signal_event = VC_EVENT_INACTIVITY_TIMEOUT;
    // 指针指向超时时间，这样后面就不用再根据超时类型判断应该取那个超时时间了
    signal_timeout_at = &next_inactivity_timeout_at;
  } else {
    // 当传入的event为EVENT_INTERVAL
    // 这表示发生了Active Timeout
    // 通常只有EventSystem对于定时、周期性执行的事件的回调才会传入EVENT_INTERVAL
    //     但是没有看到有任何地方做schedule，如果由EventSystem回调，那么传入的event是在哪里创建的？？？
    // event == EVENT_INTERVAL
    // Active Timeout
    // 出现了Active Timeout，因此设置回调上层状态机的 event 类型为 ACTIVE_TIMEOUT
    signal_event = VC_EVENT_ACTIVE_TIMEOUT;
    // 指针指向超时时间
    signal_timeout_at = &next_activity_timeout_at;
  }
#endif

  // 重置超时值
  *signal_timeout = 0;
  // 重置超时时间的绝对值
  *signal_timeout_at = 0;
  // 记录 writer_cont
  writer_cont = write.vio._cont;

  // 判断vc已经关闭的话，直接调用close_UnixNetVConnection关闭该vc
  if (closed) {
    close_UnixNetVConnection(this, thread);
    return EVENT_DONE;
  }

  // 从此处开始，下面的代码逻辑是按照 iocore/net/I_NetVConnection.h 的Line: 380 ~ 393
  //     在 Timeout symantics 一节中的描述来实现。

  // 如果：
  //     设置了Read VIO，op值可能为：VIO::READ 或者 VIO::NONE
  //     并且没有对读取端做半关闭，就是VC对应的Socket处于可读状态
  if (read.vio.op == VIO::READ && !(f.shutdown & NET_VC_SHUTDOWN_READ)) {
    // 记录 reader_cont
    reader_cont = read.vio._cont;
    // 向上层状态机回调超时事件，此时reader_cont已经被回调过了
    if (read_signal_and_update(signal_event, this) == EVENT_DONE)
      return EVENT_DONE;
  }

  // 前三个条件主要判断在刚才reader_cont回调时，是否又重新设置了超时控制，以及可能关闭vc（这个看上去多余？）：
  //     第一个条件判断超时是否被设置：!*signal_timeout
  //     第二个条件判断超时是否被设置：!*signal_timeout_at
  //     第三个条件判断vc是否被关闭：!closed，之前判断过，回调reader_cont时如果关闭了VC，那么会在上面返回EVENT_DONE
  // 如果：
  //     设置了Write VIO，op值可能为：VIO::WRITE 或者 VIO::NONE
  //     并且没有对发送端做半关闭，就是VC对应的Socket处于可写状态
  //     并且Write VIO对应的状态机不是刚刚回调过的reader_cont
  //     并且当前的Write VIO仍然是之前记录的writer_cont，因为前面回调reader_cont的时候，Write VIO状态机可能会被重设
  if (!*signal_timeout && !*signal_timeout_at && !closed && write.vio.op == VIO::WRITE && !(f.shutdown & NET_VC_SHUTDOWN_WRITE) &&
      reader_cont != write.vio._cont && writer_cont == write.vio._cont)
    // 向上层状态机回调超时事件
    if (write_signal_and_update(signal_event, this) == EVENT_DONE)
      return EVENT_DONE;

  // 最后总是返回 EVENT_DONE
  return EVENT_DONE;
}
```

## 方法

### 创建与Origin Server的连接（connectUp）

如何创建与源服务器的连接？

```
int
UnixNetVConnection::connectUp(EThread *t, int fd)
{
  int res;

  // 设置新vc的管理线程为 t
  thread = t;
  
  // 如果超出了允许的最大连接数，则立即宣告失败
  // 回调创建此vc的状态机，状态机需要关闭此vc
  if (check_net_throttle(CONNECT, submit_time)) {
    check_throttle_warning();
    action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)-ENET_THROTTLING);
    free(t);
    return CONNECT_FAILURE;
  }

  // Force family to agree with remote (server) address.
  options.ip_family = server_addr.sa.sa_family;

  //
  // Initialize this UnixNetVConnection
  //
  // 打印调试信息
  if (is_debug_tag_set("iocore_net")) {
    char addrbuf[INET6_ADDRSTRLEN];
    Debug("iocore_net", "connectUp:: local_addr=%s:%d [%s]\n",
          options.local_ip.isValid() ? options.local_ip.toString(addrbuf, sizeof(addrbuf)) : "*", options.local_port,
          NetVCOptions::toString(options.addr_binding));
  }

  // If this is getting called from the TS API, then we are wiring up a file descriptor
  // provided by the caller. In that case, we know that the socket is already connected.
  // 如果没有创建底层的socket fd，那么就创建一个
  if (fd == NO_FD) {
    res = con.open(options);
    if (res != 0) {
      goto fail;
    }
  } else {
    // 通常，来自TS API的调用，底层的socket fd已经创建好了
    int len = sizeof(con.sock_type);

    // This call will fail if fd is not a socket (e.g. it is a
    // eventfd or a regular file fd.  That is ok, because sock_type
    // is only used when setting up the socket.
    // 只需要设置一下 con 对象的成员
    safe_getsockopt(fd, SOL_SOCKET, SO_TYPE, (char *)&con.sock_type, &len);
    safe_nonblocking(fd);
    con.fd = fd;
    con.is_connected = true;
    con.is_bound = true;
  }

  // Must connect after EventIO::Start() to avoid a race condition
  // when edge triggering is used.
  // 通过 EventIO::start 把 vc 放入 epoll_wait 的监控下，READ & WRITE
  //     这里的注释说，当采用EPOLL ET模式时，必须先执行start再执行connect，来避免一个竞争条件
  //     但是我没看懂，到底是什么竞争条件？？？
  if (ep.start(get_PollDescriptor(t), this, EVENTIO_READ | EVENTIO_WRITE) < 0) {
    lerrno = errno;
    Debug("iocore_net", "connectUp : Failed to add to epoll list\n");
    action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)0); // 0 == res
    // con.close() 由 Connection 类的析构函数调用来关闭 fd
    free(t);
    return CONNECT_FAILURE;
  }

  // 对于底层socket fd没有创建的情况，才需要调用connect来发起连接
  // 注意，这里是非阻塞IO，所以后面要在状态机里判断socket fd可写才算是连接建立成功
  //     这里再去看上面提到的竞争条件，如果socket fd已经创建时，那么就出现了connect在EventIO::start之前的情况
  //     那么此处的竞争条件到底是什么？？？
  if (fd == NO_FD) {
    res = con.connect(&server_addr.sa, options);
    if (res != 0) {
      // con.close() 由 Connection 类的析构函数调用来关闭 fd
      goto fail;
    }
  }

  check_emergency_throttle(con);

  // start up next round immediately

  // 切换vc的handler为mainEvent
  SET_HANDLER(&UnixNetVConnection::mainEvent);

  // 放入 open_list 队列
  nh = get_NetHandler(t);
  nh->open_list.enqueue(this);

  ink_assert(!inactivity_timeout_in);
  ink_assert(!active_timeout_in);
  this->set_local_addr();
  // 回调创建此vc的状态机，NET_EVENT_OPEN，创建此vc的状态机会继续回调上层状态机
  // 在上层状态机，可以设置VIO，之后上层状态机就可以收到READ|WRITE_READY的事件了
  // 注意，这里并未判断非阻塞的connect方法是否成功打开了连接
  action_.continuation->handleEvent(NET_EVENT_OPEN, this);
  return CONNECT_SUCCESS;

fail:
  // 失败处理，保存errno的值
  lerrno = errno;
  // 回调创建此vc的状态机，状态机需要关闭此vc
  action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)(intptr_t)res);
  free(t);
  return CONNECT_FAILURE;
}
```

### 激活 VIO & VC（reenable & reenable_re）

在UnixNetVConnection中提供了 reenable 和 reenable_re 两个方法，分别对应了在[P_VIO.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_VIO.h)中定义的VIO的 reenable 和 reenable_re 两个方法。

```
TS_INLINE void
VIO::reenable()
{
  if (vc_server)
    vc_server->reenable(this);
}
TS_INLINE void
VIO::reenable_re()
{
  if (vc_server)
    vc_server->reenable_re(this);
}
```

我们知道VIO的一端是VC，另一端是MIOBuffer，而VC总是主动端，由IOCore驱动，因此对VIO的reenable就是VC的reenable。

那么 reenable 和 reenable_re 有什么样的区别？分别在什么场景使用？带着问题，先来注释和分析这两个方法。

```
//
// Function used to reenable the VC for reading or
// writing.
//
void
UnixNetVConnection::reenable(VIO *vio)
{
  // 获取包含此VIO的NetState对象
  // 对应NetState对象的enabled为true，表示已经处于激活状态了
  if (STATE_FROM_VIO(vio)->enabled)
    return;
  // 设置enabled为true，同时重置inactivity timeout
  set_enabled(vio);
  // thread成员在基类NetVConnection中定义，用来指向管理该vc的EThread
  // 在NetAccept或acceptEvent中初始化thread为e->ethread
  // 如果该vc还没有交付给一个EThread来管理，说明还没有被NetHandler管理，因此无法激活。
  if (!thread)
    return;
  
  // 执行reenable之前必须对vio上锁（对该vio上锁的线程必须是调用此方法的线程）
  EThread *t = vio->mutex->thread_holding;
  ink_assert(t == this_ethread());
  // 不能对已经关闭的 vc 执行 reenable 操作
  ink_assert(!closed);
  // 如果nethandler已经被当前线程上锁
  if (nh->mutex->thread_holding == t) {
    // 判断vio是read vio还是writevio
    // 根据triggered，放入ready_list中
    if (vio == &read.vio) {
      ep.modify(EVENTIO_READ);
      ep.refresh(EVENTIO_READ);
      if (read.triggered)
        nh->read_ready_list.in_or_enqueue(this);
      else
        nh->read_ready_list.remove(this);
    } else {
      ep.modify(EVENTIO_WRITE);
      ep.refresh(EVENTIO_WRITE);
      if (write.triggered)
        nh->write_ready_list.in_or_enqueue(this);
      else
        nh->write_ready_list.remove(this);
    }
  } else {
    // 可能没上锁，可能被其它线程上锁
    // 因此，尝试在当前线程中对nethandler上锁
    MUTEX_TRY_LOCK(lock, nh->mutex, t);
    if (!lock.is_locked()) {
      // 上锁失败，放入enable_list等待下一次nethandler被eventsystem回调时处理
      if (vio == &read.vio) {
        if (!read.in_enabled_list) {
          read.in_enabled_list = 1;
          nh->read_enable_list.push(this);
        }
      } else {
        if (!write.in_enabled_list) {
          write.in_enabled_list = 1;
          nh->write_enable_list.push(this);
        }
      }
      // 如果存在需要尽快处理的Signal Event，就调用signal_hook通知到对应的线程
      if (nh->trigger_event && nh->trigger_event->ethread->signal_hook)
        nh->trigger_event->ethread->signal_hook(nh->trigger_event->ethread);
    } else {
      // 上锁成功
      // 根据triggered，放入ready_list
      if (vio == &read.vio) {
        ep.modify(EVENTIO_READ);
        ep.refresh(EVENTIO_READ);
        if (read.triggered)
          nh->read_ready_list.in_or_enqueue(this);
        else
          nh->read_ready_list.remove(this);
      } else {
        ep.modify(EVENTIO_WRITE);
        ep.refresh(EVENTIO_WRITE);
        if (write.triggered)
          nh->write_ready_list.in_or_enqueue(this);
        else
          nh->write_ready_list.remove(this);
      }
    }
  }
}

void
UnixNetVConnection::reenable_re(VIO *vio)
{
  // 与 reenable 相比，少了对 enabled 为 true 的判断
  // thread成员在基类NetVConnection中定义，用来指向管理该vc的EThread
  // 在NetAccept或acceptEvent中初始化thread为e->ethread
  // 如果该vc还没有交付给一个EThread来管理，说明还没有被NetHandler管理，因此无法激活。
  if (!thread)
    return;

  // 执行reenable_re之前必须对vio上锁（对该vio上锁的线程必须是调用此方法的线程）
  EThread *t = vio->mutex->thread_holding;
  ink_assert(t == this_ethread());
  // 与 reenable 相比，没有判断vc->closed
  // 如果nethandler已经被当前线程上锁
  if (nh->mutex->thread_holding == t) {
    // 设置enabled为true，同时重置inactivity timeout
    set_enabled(vio);
    // 判断vio是read vio还是writevio
    // 如果 triggered 为 true，则直接触发读、写操作
    // 否则从 ready_list 中删除
    if (vio == &read.vio) {
      ep.modify(EVENTIO_READ);
      ep.refresh(EVENTIO_READ);
      if (read.triggered)
        net_read_io(nh, t);
      else
        nh->read_ready_list.remove(this);
    } else {
      ep.modify(EVENTIO_WRITE);
      ep.refresh(EVENTIO_WRITE);
      if (write.triggered)
        write_to_net(nh, this, t);
      else
        nh->write_ready_list.remove(this);
    }
  } else {
    // 可能没上锁，可能被其它线程上锁
    // 调用reenable来完成
    reenable(vio);
  }
}
```

对比 reenable 和 reenable_re：

  - reenable_re 可以立即触发vio的读写操作，可实现同步操作，而且可以在enabled状态重复触发读写操作
  - reenable 则只是把vc放入enable_list或者ready_list，需要等NetHandler在下一次遍历中处理

所以当需要立即触发读、写操作时，调用reenable_re

  - 但是仍然可能会出现NetHandler已经被其它线程上锁而出现的异步操作。
  - 调用reenable_re之后，可能会出现上层状态机的重入（我想这才是re后缀的真正含义）。

使用场景：

  - 当我们希望一个VIO操作立即完成时
  - 在 READY 事件中，调用reenable_re，可以让操作继续立即执行
  - 直到状态机收到 COMPLETE 事件，然后逐层返回
  - 但是这中间也会遇到异步的情况，reenable_re只是保证尽可能的同步执行

由于此方法会导致潜在的阻塞问题，在ATS中只有很少的地方使用到了reenable_re这个方法。

关于上锁的总结：

  - 调用reenable之前要首先对VIO上锁，通常VIO的锁就是上层状态机的锁，基本都会上锁
  - 在reenable执行时会对NetHandler上锁，这是因为：
    - 在A线程中可能会reenable一个在B线程中管理的vc
    - 而此时，NetHandler可能正在调用processor_enabled_list来批量处理B线程中管理的所有 vc
    - 这样就会出现资源访问的竞争条件

### 关闭和释放（close & free）

所有的NetVC及其继承类，都会调用close_UnixNetVConnection这个函数，它不是某个NetVC类的成员函数。

因此，对于SSLNetVConnection的关闭也会调用此函数，但是由于free方法是成员函数，所以```vc->free(t)```执行了对应NetVConnection继承类型的释放操作。

```
//
// Function used to close a UnixNetVConnection and free the vc
//
// 传入两个变量，一个是想要关闭的vc，另一个是当前EThread
void
close_UnixNetVConnection(UnixNetVConnection *vc, EThread *t)
{
  NetHandler *nh = vc->nh;
  
  // 取消带外数据的发送状态机
  vc->cancel_OOB();
  // 让epoll_wait停止监控此vc的socket fd
  vc->ep.stop();
  // 关闭socket fd
  vc->con.close();

  // 这里没有异步处理策略，必须保证是由管理此vc的线程发起调用
  // 事实上这里 t == this_ethread()，可以查阅ATS代码中调用close_UnixNetVConnection的地方来确认
  ink_release_assert(vc->thread == t);

  // 清理超时计数器
  vc->next_inactivity_timeout_at = 0;
  vc->next_activity_timeout_at = 0;
  vc->inactivity_timeout_in = 0;
  vc->active_timeout_in = 0;

  // 之前的版本没有判断nh是否为空，存在bug
  // 当nh不为空，就要将vc从几个队列中删除
  if (nh) {
    // 四个本地队列
    nh->open_list.remove(vc);
    nh->cop_list.remove(vc);
    nh->read_ready_list.remove(vc);
    nh->write_ready_list.remove(vc);
    
    // 两个enable_list是原子队列
    // 但是 in_enable_list 的操作却无法与原子操作同步，此处我觉得存在问题！！！
    if (vc->read.in_enabled_list) {
      nh->read_enable_list.remove(vc);
      vc->read.in_enabled_list = 0;
    }
    if (vc->write.in_enabled_list) {
      nh->write_enable_list.remove(vc);
      vc->write.in_enabled_list = 0;
    }
    
    // 两个仅用于InactivityCop本地队列
    vc->remove_from_keep_alive_queue();
    vc->remove_from_active_queue();
  }
  
  // 最后调用vc的free方法
  vc->free(t);
}
```

在vc关闭后，就需要对资源进行回收，由于vc的内存资源是通过allocate_vc函数分配的，因此在回收时也要将内存归还到内存池。

每一种类型的NetVC的继承类，都会有匹配的NetProcessor继承类提供allocate_vc函数和其自身提供的free函数。

```
void
UnixNetVConnection::free(EThread *t)
{
  // 只有当前线程可以释放vc资源
  // 这里 t == thread，与close_UnixNetVConnection是一样的。
  ink_release_assert(t == this_ethread());
  NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, -1);
  
  // clear variables for reuse
  // 释放vc的mutex
  this->mutex.clear();
  // 释放上层状态机的mutex
  action_.mutex.clear();
  got_remote_addr = 0;
  got_local_addr = 0;
  attributes = 0;
  // 释放vio的mutex，可能等于上层状态机的mutex，有判断，不会重复释放内存
  read.vio.mutex.clear();
  write.vio.mutex.clear();
  // 重置半关闭状态
  flags = 0;
  // 重置回调函数
  SET_CONTINUATION_HANDLER(this, (NetVConnHandler)&UnixNetVConnection::startEvent);
  // NetHandler 为空
  nh = NULL;
  // 重置 NetState 
  read.triggered = 0;
  write.triggered = 0;
  read.enabled = 0;
  write.enabled = 0;
  // 重置VIO
  read.vio._cont = NULL;
  write.vio._cont = NULL;
  read.vio.vc_server = NULL;
  write.vio.vc_server = NULL;
  // 重置options
  options.reset();
  // 重置 close 状态
  closed = 0;
  // 必须不在ready_link和enable_link中，在close_UnixNetVConnection中已经移除
  ink_assert(!read.ready_link.prev && !read.ready_link.next);
  ink_assert(!read.enable_link.next);
  ink_assert(!write.ready_link.prev && !write.ready_link.next);
  ink_assert(!write.enable_link.next);
  ink_assert(!link.next && !link.prev);
  // socket fd 已经关闭，在close_UnixNetVConnection中调用con.close()完成了关闭
  ink_assert(con.fd == NO_FD);
  ink_assert(t == this_ethread());

  // 判断allocate_vc方法为vc分配内存时，是由全局内存分配还是线程本地分配
  // 选择对应的方法归还内存资源
  if (from_accept_thread) {
    netVCAllocator.free(this);
  } else {
    THREAD_FREE(this, netVCAllocator, t);
  }
}
```

由于close_UnixNetVConnection和free都是不可重入的，所以在很多地方，我们都看到使用vc的重入计数器对重入进行判断。

例如，在do_io_close中：

```
void
UnixNetVConnection::do_io_close(int alerrno /* = -1 */)
{
  disable_read(this);
  disable_write(this);
  read.vio.buffer.clear();
  read.vio.nbytes = 0;
  read.vio.op = VIO::NONE;
  read.vio._cont = NULL;
  write.vio.buffer.clear();
  write.vio.nbytes = 0;
  write.vio.op = VIO::NONE;
  write.vio._cont = NULL;

  EThread *t = this_ethread();
  // 通过重入计数器判断是否可以直接调用close_UnixNetVConnection
  bool close_inline = !recursion && (!nh || nh->mutex->thread_holding == t);

  INK_WRITE_MEMORY_BARRIER;
  if (alerrno && alerrno != -1)
    this->lerrno = alerrno;
  if (alerrno == -1)
    closed = 1;
  else
    closed = -1;

  if (close_inline)
    close_UnixNetVConnection(this, t);
  // 此处对于不能立即关闭vc的情况没有任何操作了，也没有重新调度来延迟关闭vc
  // 那么什么时候调用close_UnixNetVConnection来关闭vc呢？答案是：在InactivityCop中会做出处理
}
```

### 关于 in_enable_list 操作的原子性问题的备忘录

在close_UnixNetVConnection中直接对in_enable_list进行判断和置0的操作

  - 如果没有对NetHandler上锁，那么就无法与原子操作同步
  - 那么到底NetHandler有没有上锁呢？答案是上锁了！
  - 但是在哪儿上锁的呢？

提示：NetAccept的mutex在init_accept_per_thread里面为何被设置为```get_NetHandler(t)->mutex``` ?

## 参考资料

- [P_UnixNetState.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetState.h)
- [P_UnixNet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [P_UnixNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetVConnection.h)
- [UnixNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
