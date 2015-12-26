# 核心组件：UnixNetVConnection

实际上IOCoreNet子系统，与EventSystem是一样的，也有Thread，Processor和Event，只是名字不一样了：

| EventSystem  |   NetSubSystem   |
|:------------:|:----------------:|
|     Event    |UnixNetVConnection|
|    EThread   |    NetHandler    |
|EventProcessor|   NetProcessor   |

但是，从形态上来看，UnixNetVConnection 要比 Event 复杂的多。

由于 SSLNetVConnection 继承自 UnixNetVConnection，因此在 UnixNetVConnection 的定义中也为支持 SSL 预留了部分内容。

## 定义

以下内容为了描述方便，对代码行的顺序做了部分调整。

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
  // The constructor is public just to avoid compile errors.      //
  /////////////////////////////////////////////////////////////////
  UnixNetVConnection();

private:
  UnixNetVConnection(const NetVConnection &);
  UnixNetVConnection &operator=(const NetVConnection &);

public:
  /////////////////////////
  // UNIX implementation //
  /////////////////////////
  void set_enabled(VIO *vio);

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
  virtual void net_read_io(NetHandler *nh, EThread *lthread);
  virtual int64_t load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                        int &needs);
  void readDisable(NetHandler *nh);
  void readSignalError(NetHandler *nh, int err);
  int readSignalDone(int event, NetHandler *nh);
  int readSignalAndUpdate(int event);
  void readReschedule(NetHandler *nh);
  void writeReschedule(NetHandler *nh);
  void netActivity(EThread *lthread);
  /**
   * If the current object's thread does not match the t argument, create a new
   * NetVC in the thread t context based on the socket and ssl information in the
   * current NetVC and mark the current NetVC to be closed.
   */
  UnixNetVConnection *migrateToCurrentThread(Continuation *c, EThread *t);

  Action action_;
  volatile int closed;
  NetState read;
  NetState write;

  LINK(UnixNetVConnection, cop_link);
  LINKM(UnixNetVConnection, read, ready_link)
  SLINKM(UnixNetVConnection, read, enable_link)
  LINKM(UnixNetVConnection, write, ready_link)
  SLINKM(UnixNetVConnection, write, enable_link)
  LINK(UnixNetVConnection, keep_alive_queue_link);
  LINK(UnixNetVConnection, active_queue_link);

  ink_hrtime inactivity_timeout_in;
  ink_hrtime active_timeout_in;
#ifdef INACTIVITY_TIMEOUT
  Event *inactivity_timeout;
  Event *activity_timeout;
#else
  ink_hrtime next_inactivity_timeout_at;
  ink_hrtime next_activity_timeout_at;
#endif

  EventIO ep;
  NetHandler *nh;
  unsigned int id;
  // amc - what is this for? Why not use remote_addr or con.addr?
  IpEndpoint server_addr; /// Server address and port.

  union {
    unsigned int flags;
#define NET_VC_SHUTDOWN_READ 1
#define NET_VC_SHUTDOWN_WRITE 2
    struct {
      unsigned int got_local_addr : 1;
      unsigned int shutdown : 2;
    } f;
  };

  Connection con;
  int recursion;
  ink_hrtime submit_time;
  OOB_callback *oob_ptr;
  bool from_accept_thread;

  // es - origin_trace associated connections
  bool origin_trace;
  const sockaddr *origin_trace_addr;
  int origin_trace_port;

  int startEvent(int event, Event *e);
  int acceptEvent(int event, Event *e);
  int mainEvent(int event, Event *e);
  virtual int connectUp(EThread *t, int fd);
  /**
   * Populate the current object based on the socket information in in the
   * con parameter.
   * This is logic is invoked when the NetVC object is created in a new thread context
   */
  virtual int populate(Connection &con, Continuation *c, void *arg);
  virtual void free(EThread *t);

  virtual ink_hrtime get_inactivity_timeout();
  virtual ink_hrtime get_active_timeout();

  virtual void set_local_addr();
  virtual void set_remote_addr();
  virtual int set_tcp_init_cwnd(int init_cwnd);
  virtual void apply_options();

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

## 方法



## 参考资料

- [P_UnixNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetVConnection.h)
