# 基础组件：NetVConnection

在ATS中VConnection抽象很有趣，几乎跟OSI的模型对应了起来：

||OSI Layer||VC Class||Comments||
||6||SSLNetVConnection||提供了SSL会话协议支持||
||4||UnixNetVConnection||提供了成员保存socket fd，建立了socket fd与VIO之间的数据流逻辑||
||3||NetVConnection||有了IP信息，但是没有提供socket fd||
||2||VConnection||基类||

## 定义

NetVConnection继承自VConnection，对其进行了扩展，下面的分析只对扩展的部分做描述，VConnection的部分，请参考VConnection部分的介绍。

```
class NetVConnection : public VConnection
{
public:
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf) = 0;
  virtual VIO *do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner = false) = 0;
  virtual void do_io_close(int lerrno = -1) = 0;
  virtual void do_io_shutdown(ShutdownHowTo_t howto) = 0;

  // 发送带外数据
  // 将长度为len的缓冲区buf内的数据以带外方式发送，
  // 发送成功，回调cont时传递VC_EVENT_OOB_COMPLETE事件
  // 对方关闭，回调cont时传递VC_EVENT_EOS事件
  // cont回调必须是可重入的
  // 每一个VC同一时间只能有一个send_OOB操作
  virtual Action *send_OOB(Continuation *cont, char *buf, int len);

  // 取消带外数据的发送
  // 取消之前调用send_OOB安排的带外数据发送操作，但是可能会有部分数据已经发送出去。
  // 执行之后，不会再有任何对cont的回调，而且也不能再访问之前send_OOB返回的Action对象。
  // 当之前的send_OOB操作没有完成，而又需要进行一个新的send_OOB操作时，必须要执行cancel_OOB取消之前的操作。
  virtual void cancel_OOB();

  ////////////////////////////////////////////////////////////
  // 连接的超时设置
  //         active_timeout 用于连接的整个生存周期
  //     inactivity_timeout 用于最近一次读写操作之后的生存周期
  // 可以重复调用这些方法，以复位超时统计时间。
  // 由于这些方法不是线程安全的，所以你只能在 “被回调” 时调用这些函数。
  
  /*
    关于 “超时” 的定义:

    如果超时发生，与NetVC读取端关联的状态机(SM)首先被回调，以确保NetVC上已经读取到了数据，并且读端没有被关闭。
    如果两个条件都不满足，那么NetProcessor尝试回调写入端关联的状态机(SM)。
    PS：读端被关闭（读半关闭，读到0字节），如果没有数据要写入，通常意味着连接被对端关闭了。

    回调读端状态机，
        如果返回了EVENT_DONE：
            那么就不会再回调写端状态机；
        如果返回的不是EVENT_DONE，并且写端状态机与读端不是同一个状态机（比较回调函数的指针）：
            NetProcessor将尝试回调写端状态机。

    回调写端之前，要确保数据已经被写入，并且写端没有关闭（写半关闭）。

    状态机收到TIMEOUT事件，只是一个通知，表示计时时间已到，但是NetVC仍然可用。

    除非通过set_active_timeout() 或 set_inactivity_timeout()重置计时器，否则后续的超时信号将不再被触发。
    
    在ATS中通过InactivityCop状态机来遍历Keep alive队列和Active队列，实现对超时计时器的检测并回调状态机。
    关于 InactivityCop 状态机更详细的历史追述，可参阅：https://issues.apache.org/jira/browse/TS-1405
  */
  
  /**
    在指定时间之后通知状态机(SM)

    设置一个时间量（纳秒/毫微秒，十亿分之一秒为单位），在此之后回调与此VC关联的状态机(SM)，并传递VC_EVENT_ACTIVE_TIMEOUT事件。
    
    The timeout is value is ignored if neither the read side nor the write side of the connection is currently active. 
    
    如果当前连接没有处于active状态（没有读、写的操作），那么就会忽略这个值。
    PS：事实上，是优先判断Inactivity Timeout，当出现Inactivity Timeout时就不会在回调Active Timeout事件。
    
    每次调用此函数，都会重新对超时时间进行计时。
    通过这个方法，可以让状态机(SM)处理那些长时间不关闭（但是有通信）的链接。
  */
  virtual void set_active_timeout(ink_hrtime timeout_in) = 0;

  /**
    如果发起的IO操作没有在指定时间完成，就通知状态机(SM)
    
    如果在最近一次状态机操作NetVC之后，达到设置的时间量（纳秒/毫微秒，十亿分之一秒为单位），NetVC在读端和写端仍然都处于空闲状态，将会回调与此NetVC关联的状态机(SM)，并传递VC_EVENT_INACTIVITY_TIMEOUT事件。
    
    计时器将被重置：
        发生读操作
        发生写操作
        调用此函数：set_inactivity_timeout()
    
    The timeout is value is ignored if neither the read side nor the write side of the connection is currently active. See section on timeout semantics above.
   */
  virtual void set_inactivity_timeout(ink_hrtime timeout_in) = 0;

  /**
    清除Active Timeout
    
    在调用set_active_timeout()重新设置之前，不会再有Active Timeout事件发送。
  */
  virtual void cancel_active_timeout() = 0;

  /**
    清除Inactivity Timeout
    
    在调用set_inactivity_timeout()重新设置之前，不会再有Active Timeout事件发送。
  */
  virtual void cancel_inactivity_timeout() = 0;

  // 添加NetVC到keep alive队列
  virtual void add_to_keep_alive_queue() = 0;
  // 从keep alive队列移除NetVC
  virtual void remove_from_keep_alive_queue() = 0;
  // 添加NetVC到active队列
  virtual bool add_to_active_queue() = 0;
  // 没有从active队列移除NetVC的方法？？

  // 返回当前active_timeout的值（纳秒）
  virtual ink_hrtime get_active_timeout() = 0;

  // 返回当前inactivity_timeout的值（纳秒）
  virtual ink_hrtime get_inactivity_timeout() = 0;

  // 特殊机制：如果写操作导致缓冲区为空（MIOBuffer内没有数据可写了），就使用指定的事件回调状态机
  /** Force an @a event if a write operation empties the write buffer.

     此机制简称为：WBE
     在IOCoreNet处理中，如果我们设置了这个特殊的机制，那么：
         在下一次从MIOBuffer向socket fd写数据的过程中，
         如果MIOBuffer中当前可用于写操作的数据都被写完：
             将使用指定的event回调状态机

     如果传入的event为0，表示取消这个操作。
      
     同其它IO事件一样，回调时传递的是VIO数据类型。
      
     这个操作是单次触发，如果需要重复触发，就需要再次设置。

     PS：在NetHandler回调的write_to_net_io()方法中的回调逻辑如下：
             如果MIOBuffer有剩余空间，则首先发送WRITE READY回调状态机填充MIOBuffer
             然后开始将MIOBuffer里的数据写入socket fd
             保存WBE到wbe_event，wbe_event = WBE;
             如果MIOBuffer被写空
                 清除WBE状态，WBE = 0;
             如果完成了VIO，发送WRITE COMPLETE回调状态机，结束写操作
             如果已经发送WRITE READY并且设置了WBE
                 发送web_event回调状态机，如果回调返回值不是EVENT_CONT，结束写操作
             如果之前没有发送WRITE READY回调过状态机
                 发送WRITE READY回调状态机，如果返回值不是EVENT_CONT，结束写操作
             如果MIOBuffer被写空，禁止VC上的后续写事件，结束写操作
             重新调度读、写操作
             
      The event is sent only if otherwise no event would be generated.
   */
  virtual void trapWriteBufferEmpty(int event = VC_EVENT_WRITE_READY);

  // 返回Local sockaddr 对象的指针，支持IPv4和IPv6
  sockaddr const *get_local_addr();

  // 返回Local ip，只支持IPv4，不建议使用的老旧方法，即将废弃
  in_addr_t get_local_ip();
  
  // 返回 Local port
  uint16_t get_local_port();

  // 返回Remote sockaddr 对象的指针，支持IPv4和IPv6
  sockaddr const *get_remote_addr();

  // 返回 remote ip
  in_addr_t get_remote_ip();

  // 返回 remote port
  uint16_t get_remote_port();

  // 保存了用户设置的结构体
  // 在ATS中将对主体对象的设置信息独立出来做法，在多个地方都有应用，
  // 这样做可以在创建一组相同设置的对象时共用同一个设置实例，以减少对内存的占用。
  NetVCOptions options;

  /** Attempt to push any changed options down */
  virtual void apply_options() = 0;

  //
  // Private
  // 以下方法不建议直接调用，没有声明为private类型是为了顺利通过代码的编译
  //

  // 当SocksProxy的transprency打开时，用来获取主机地址
  SocksAddrType socks_addr;

  // 指定NetVC的属性
  // 用的比较多的就是HttpProxyPort::TRANSPORT_BLIND_TUNNEL
  unsigned int attributes;
  
  // 指向持有此NetVC的EThread
  EThread *thread;

  /// PRIVATE: The public interface is VIO::reenable()
  virtual void reenable(VIO *vio) = 0;

  /// PRIVATE: The public interface is VIO::reenable()
  virtual void reenable_re(VIO *vio) = 0;

  /// PRIVATE
  virtual ~NetVConnection() {}

  /**
    PRIVATE: instances of NetVConnection cannot be created directly
    by the state machines. The objects are created by NetProcessor
    calls like accept connect_re() etc. The constructor is public
    just to avoid compile errors.

  */
  NetVConnection();

  virtual SOCKET get_socket() = 0;

  /** Set the TCP initial congestion window */
  virtual int set_tcp_init_cwnd(int init_cwnd) = 0;

  /** Set local sock addr struct. */
  virtual void set_local_addr() = 0;

  /** Set remote sock addr struct. */
  virtual void set_remote_addr() = 0;

  // for InkAPI
  bool
  get_is_internal_request() const
  {
    return is_internal_request;
  }

  void
  set_is_internal_request(bool val = false)
  {
    is_internal_request = val;
  }

  // 透明代理(TPROXY)支持
  // 获取 transparency 状态
  bool
  get_is_transparent() const
  {
    return is_transparent;
  }
  // 设置 transparency 状态
  void
  set_is_transparent(bool state = true)
  {
    is_transparent = state;
  }

private:
  NetVConnection(const NetVConnection &);
  NetVConnection &operator=(const NetVConnection &);

protected:
  IpEndpoint local_addr;
  IpEndpoint remote_addr;

  bool got_local_addr;
  bool got_remote_addr;

  bool is_internal_request;
  /// Set if this connection is transparent.
  bool is_transparent;
  /// Set if the next write IO that empties the write buffer should generate an event.
  int write_buffer_empty_event;
};

inline NetVConnection::NetVConnection()
  : VConnection(NULL), attributes(0), thread(NULL), got_local_addr(0), got_remote_addr(0), is_internal_request(false),
    is_transparent(false), write_buffer_empty_event(0)
{
  ink_zero(local_addr);
  ink_zero(remote_addr);
}

inline void
NetVConnection::trapWriteBufferEmpty(int event)
{
  write_buffer_empty_event = event;
}
```

## 参考资料
- [I_NetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/I_NetVConnection.h)
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)

# 基础组件：VConnection

VConnection是所有提供IO功能的Connection类的基类，自身无法实现功能，必须通过派生类实现具体功能。

VConnection类是由Processor返回的一种单向或双向数据管道的抽象描述。

在某种意义上来说，与文件描述符的功能相似。

根本上来说，VConnection是定义Stream IO处理方法的基类。

它也是一个被Processor回调的Continuation。

VConnection定义了一个可以串联的虚拟管道，可以让ATS在网络、状态机、磁盘间形成流式数据通信，是上层通信机制的很关键一环。

```
                                          +------ UserSM ------+
      UpperSM Area                       /       (HTTPSM)       \
                                        /                        \
                                       /                          \
*** CS_FD ********************** CS_VIO ************************** SS_VIO ********************** SS_FD ***
          \                     /                                        \                     /
           +------ CS_VC ------+          EventSystem & IO Core           +------ SS_VC ------+

 CS_FD = Client Side File Descriptor
 SS_FD = Server Side File Descriptor
 CS_VC = Client Side VConnection
 SS_VC = Server Side VConnection
CS_VIO = Client Side VIO
SS_VIO = Server Side VIO
UserSM = User Defined SM ( HTTPSM )
****** = Separator Line
```

当FD处于可读或者可写的状态时，回调VC，把FD上的读取到的数据生产到到VIO里，或者把VIO里的数据拿出来消费掉写入到FD里。

当VIO里的数据产生变化时，回调UserSM，并告知变化的原因。

所以，可以将VC理解为负责把数据在FD和VIO之间移动的状态机。

## 定义

VConnection继承自Continuation，因此VConnection首先是一个延续，它有Cont Handler和Mutex。

```

class VConnection : public Continuation
{
public:
  virtual ~VConnection();

  virtual VIO *do_io_read(Continuation *c = NULL, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0) = 0;

  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false) = 0;

  virtual void do_io_close(int lerrno = -1) = 0;

  virtual void do_io_shutdown(ShutdownHowTo_t howto) = 0;

  VConnection(ProxyMutex *aMutex);

// Private
  // Set continuation on a given vio. The public interface
  // is through VIO::set_continuation()
  virtual void set_continuation(VIO *vio, Continuation *cont);

  // Reenable a given vio.  The public interface is through VIO::reenable
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  virtual bool
  get_data(int id, void *data)

  virtual bool
  set_data(int id, void *data)

public:
  int lerrno;
};

```

### 通用Event Code

VC_EVENT_INACTIVITY_TIMEOUT

  - 表示在一定时间内: 没有从连接读取到数据 / 没有向连接写入数据
  - 对于读操作可能是buffer为空，对于写操作可能是连接的缓冲区满了

VC_EVENT_ACTIVITY_TIMEOUT

  - 读/写操作超时

VC_EVENT_ERROR

  - 在读取/写入过程中发生了错误


## 方法

由于VConnection只是一个基类，因此下面的方法都需要在其继承类中重新进行定义。

但是为了保证定义的统一，在VConnection的继承类中必须要要按照以下基本的约定来实现这些方法。

### do_io_read

准备接收来自 VConnection 数据

  - 状态机（SM）调用它来从 VConnection 读取数据。
  - 处理机（Processor）实现这个读取功能时，首先获取锁，然后将新的数据放入buf，回调Continuation，然后释放锁。
  - 例如：让状态机处理以特殊字符标记事务结束数据传输协议。

可能的Event Code（在状态机回调 Continuation 时，VConn可能会使用这些值作为Event Code）

  - VC_EVENT_READ_READY
    - 数据已经添加到buffer，或者buffer满了
  - VC_EVENT_READ_COMPLETE
    - nbytes字节的数据已经被读取到buffer了
  - VC_EVENT_EOS
    - Stream的读取端关闭了连接

参数(Continuation *c, int64_t nbytes, MIOBuffer *buf)

  - c
    - 在读操作完成后，将要回调的Continuation，同时会传递Event Code过去
  - nbytes
    - 想要读取的字节数，如果不确定要读取多少，必须设定为INT64_MAX
  - buf
    - 读取到的数据会放入这里

返回值

  - 代表调度IO操作的VIO类型

注意

  - 当c＝NULL，nbytes＝0，buf＝NULL时表示取消之前设置的do_io_read操作。
  - 当没有数据可读，或者buf写满，会导致VIO被disable，需要调用reenable才会继续读取数据。

### do_io_write

准备向 VConnection 写入数据

  - 状态机（SM）调用它来向 VConnection 写入数据。
  - 处理机（Processor）实现这个读取功能时，首先获取锁，然后将来自buf的数据写入，回调Continuation，然后释放锁。

可能的Event Code（在状态机回调 Continuation 时，VConn可能会使用这些值作为Event Code）

  - VC_EVENT_WRITE_READY
    - 来自buffer的数据已经写入，或者buffer为空
  - VC_EVENT_WRITE_COMPLETE
    - 来自buffer的nbytes字节的数据已经被写入

参数(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner)

  - c
    - 在读操作完成后，将要回调的Continuation，同时会传递Event Code过去
  - nbytes
    - 想要写入的字节数，如果不确定要写入多少，必须设定为INT64_MAX
  - buf
    - 数据源，将从这里读取数据，然后写入
  - owner
    - 未使用，默认false，如果设置为true，可能会引发assert ??

返回值

  - 代表调度IO操作的VIO类型

注意

  - 当c＝NULL，nbytes＝0，buf＝NULL时表示取消之前设置的do_io_write操作。
  - 当写数据失败，或者buf空了，会导致VIO被disable，需要调用reenable才会继续写数据。

### set_continuation

为指定VIO设置Continuation，通常应该由VIO::set_continuation()来调用此方法。

参数(VIO *vio, Continuation *cont)

  - vio
    - 将要设置Continuation的VIO
  - cont
    - 即将要关联到VIO的Continuation

### reenable & reenable_re

激活指定的VIO，通常应该由VIO::reenable()来调用此方法。

  - 让之前设置的VIO恢复运行，EventSystem将会呼叫处理机（Processor）。

参数(VIO *vio)

  - vio
    - 准备激活的VIO

### do_io_close

表示不再需要这个 VConnection 了 

  - 当状态机使用完一个VConnection，必须调用这个函数，表示可以进行回收。
  - 在调用了close之后
    - VConnection和底层的Processor都不能再发送任何与此VConnection相关联的Event给状态机。
    - 状态机也不能够访问该VConnection和从其获得/返回的VIO。

参数(int lerrno = -1)

  - lerrno
    - 用于表示这是一个正常的关闭还是一个异常的关闭
    - 两种关闭的区别由底层的VConnection类型来决定
    - 正常关闭，lerrno = -1，设置vc->closed = 1
    - 异常关闭，lerrno != -1，设置vc->.closed = -1, vc->lerrno = lerrno
    - 大多数情况下，ATS内部并未区别处理正常关闭与异常关闭，好像只有ClusterVC中做了区别对待。

### do_io_shutdown

终结VConnection的单向或者双向传输

  - 调用此方法后，与之相关方向的I/O操作都将被禁止
  - Processor 不可以发送任何与之相关的EVENT（甚至是Timeout Event）给状态机
  - 状态机也不可以使用来自被Shutdown方向的VIO
  - 即使双向传输都被Shutdown，当状态机需要回收该VConnection时，仍然必须通过调用do_io_close来完成回收操作。

参数(ShutdownHowTo_t howto)

  - howto 取值及含义
    - IO_SHUTDOWN_READ ＝ 0
      - 关闭读取端，表示该VConn不应该再产生Read Event
    - IO_SHUTDOWN_WRITE ＝ 1
      - 关闭写入端，表示该VConn不应该再产生Write Event
    - IO_SHUTDOWN_READWRITE = 2
      - 双向关闭，表示该VConn不应该再产生Read和Write Event

### get_data & set_data

这个功能被用于状态机，从/向一个VConnection传送信息，而不破坏VConnection的抽象

  - 其行为取决于哪种VConnection的类型被使用
  - 在派生类中存取自定义的数据结构

参数(int id, void *data)

  - id
    - 为枚举类型TSApiDataType
  - data
    - 对应的数据

返回值

- 成功返回True

```
  /** Used in VConnection::get_data(). */
  enum TSApiDataType {
    TS_API_DATA_READ_VIO = VCONNECTION_API_DATA_BASE,
    TS_API_DATA_WRITE_VIO,
    TS_API_DATA_OUTPUT_VC,
    TS_API_DATA_CLOSED,
    TS_API_DATA_LAST ///< Used by other classes to extend the enum values.
  };
```

## 参考资料
- [I_VConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VConnection.h)
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
