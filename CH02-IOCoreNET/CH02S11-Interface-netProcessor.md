# 接口界面：netProcessor

在 ATS 中声明了一个 NetProcessor 类型的全局唯一实例：netProcessor，当我们需要使用网络I/O操作的时候，可以通过该实例进行操作，例如：

  - netProcessor.allocate_vc
    - 创建一个新的 NetVC
  - netProcessor.accept
    - 接收一个来自客户端的新连接
  - netProcessor.main_accept
    - 与 accept 功能基本相同，只是它要求调用者传入一个已经完成 bind 操作的 listen fd
  - netProcessor.connect_re
    - 发起一个到服务器的新连接，发出TCP SYN之后就立即回调状态机（此时连接可能未能成功建立）
  - netProcessor.connect_s
    - 发起一个到服务器的新连接，但是只在成功建立连接后才回调状态机（通过CheckConnect子状态机实现）

netProcessor 还负责在 ET_NET 线程组，为每一个线程创建 NetHandler 等状态机来处理 NetVC 的读写。

现在我们已经基本对网络子系统的各个部分进行了介绍，下面我们将 Net Sub-System 与 EventSystem 进行对比，可以发现：

- 把 NetHandler 看作是 EventSystem 的 EThread
  - NetHandler 管理多个保存 NetVC 的队列
  - EThread 管理多个保存 Event 的队列
- 把 NetVConnection 看作是 EventSystem 的 Event
  - NetHandler 遍历 NetVC 的队列，回调与 NetVC 关联状态机
  - EThread 遍历 Event 的队列，回调与 Event 关联的状态机
- 把 NetProcessor 看作是 EventSystem 的 EventProcessor
  - NetProcessor 提供了创建 NetVC 对象的功能（EventProcessor 提供了创建 Event 对象的功能）
  - NetProcessor 提供了创建并初始化 NetHandler 对象的功能（EventProcessor 提供了创建 EThread 对象的功能）
- 对于调度功能，与 EventSystem 的 schedule_\*() 方法的对比
  - NetProcessor 提供了 accept 和 connect 就是把新的 NetVC 放入 NetHandler 的队列
  - NetVConnection 提供了 reenable(), do_io() 等方法把 NetVC 放入 NetHandler 的队列
  - NetHandler 提供了 ::read_disable(), ::write_disable(), class EventIO 等方法和类，实现对其队列的操作

## 基类 NetProcessor

NetProcessor 继承自基类 Processor，是 IOCoreNet 对外提供的一个 API 集合。

基类 NetProcessor 只是定义 API，具体实现是通过 UnixNetProcessor 继承类来完成。

### 定义

```
/**
  This is the heart of the Net system. Provides common network APIs,
  like accept, connect etc. It performs network I/O on behalf of a
  state machine.
*/
class NetProcessor : public Processor
{
public:
  /** Options for @c accept.
    为 accept 方法定义 Options
   */
  struct AcceptOptions {
    typedef AcceptOptions self; ///< Self reference type.

    /// Port on which to listen.
    /// 0 => don't care, which is useful if the socket is already bound.
    int local_port;
    /// Local address to bind for accept.
    /// If not set -> any address.
    IpAddr local_ip;
    /// IP address family.
    /// @note Ignored if an explicit incoming address is set in the
    /// the configuration (@c local_ip). If neither is set IPv4 is used.
    int ip_family;
    /// Should we use accept threads? If so, how many?
    int accept_threads;
    /// Event type to generate on accept.
    EventType etype;
    /** If @c true, the continuation is called back with
        @c NET_EVENT_ACCEPT_SUCCEED
        or @c NET_EVENT_ACCEPT_FAILED on success and failure resp.
    */
    bool f_callback_on_open;
    /** Accept only on the loopback address.
        Default: @c false.
     */
    bool localhost_only;
    /// Are frequent accepts expected?
    /// Default: @c false.
    bool frequent_accept;
    bool backdoor;

    /// Socket receive buffer size.
    /// 0 => OS default.
    int recv_bufsize;
    /// Socket transmit buffer size.
    /// 0 => OS default.
    int send_bufsize;
    /// Socket options for @c sockopt.
    /// 0 => do not set options.
    uint32_t sockopt_flags;
    uint32_t packet_mark;
    uint32_t packet_tos;

    /** Transparency on client (user agent) connection.
        @internal This is irrelevant at a socket level (since inbound
        transparency must be set up when the listen socket is created)
        but it's critical that the connection handling logic knows
        whether the inbound (client / user agent) connection is
        transparent.
    */
    bool f_inbound_transparent;

    /// Default constructor.
    /// Instance is constructed with default values.
    AcceptOptions() { this->reset(); }
    /// Reset all values to defaults.
    self &reset();
  };

  /**
    Accept connections on a port.
    Callbacks:
      - cont->handleEvent( NET_EVENT_ACCEPT, NetVConnection *) is
        called for each new connection
      - cont->handleEvent(EVENT_ERROR,-errno) on a bad error
    Re-entrant callbacks (based on callback_on_open flag):
      - cont->handleEvent(NET_EVENT_ACCEPT_SUCCEED, 0) on successful
        accept init
      - cont->handleEvent(NET_EVENT_ACCEPT_FAILED, 0) on accept
        init failure
    @param cont Continuation to be called back with events this
      continuation is not locked on callbacks and so the handler must
      be re-entrant.
    @param opt Accept options.
    @return Action, that can be cancelled to cancel the accept. The
      port becomes free immediately.
   */
  inkcoreapi virtual Action *accept(Continuation *cont, AcceptOptions const &opt = DEFAULT_ACCEPT_OPTIONS);

  /**
    Accepts incoming connections on port. Accept connections on port.
    Accept is done on all net threads and throttle limit is imposed
    if frequent_accept flag is true. This is similar to the accept
    method described above. The only difference is that the list
    of parameter that is takes is limited.
    Callbacks:
      - cont->handleEvent( NET_EVENT_ACCEPT, NetVConnection *) is called for each new connection
      - cont->handleEvent(EVENT_ERROR,-errno) on a bad error
    Re-entrant callbacks (based on callback_on_open flag):
      - cont->handleEvent(NET_EVENT_ACCEPT_SUCCEED, 0) on successful accept init
      - cont->handleEvent(NET_EVENT_ACCEPT_FAILED, 0) on accept init failure
    @param cont Continuation to be called back with events this
      continuation is not locked on callbacks and so the handler must
      be re-entrant.
    @param listen_socket_in if passed, used for listening.
    @param opt Accept options.
    @return Action, that can be cancelled to cancel the accept. The
      port becomes free immediately.
  */
  virtual Action *main_accept(Continuation *cont, SOCKET listen_socket_in, AcceptOptions const &opt = DEFAULT_ACCEPT_OPTIONS);

  // Connect 方法使用的 Options 在 I_NetVConnection.h 中定义了
  /**
    Open a NetVConnection for connection oriented I/O. Connects
    through sockserver if netprocessor is configured to use socks
    or is socks parameters to the call are set.
    Re-entrant callbacks:
      - On success calls: c->handleEvent(NET_EVENT_OPEN, NetVConnection *)
      - On failure calls: c->handleEvent(NET_EVENT_OPEN_FAILED, -errno)
    @note Connection may not have been established when cont is
      call back with success. If this behaviour is desired use
      synchronous connect connet_s method.
    @see connect_s()
    @param cont Continuation to be called back with events.
    @param addr target address and port to connect to.
    @param options @see NetVCOptions.
  */

  inkcoreapi Action *connect_re(Continuation *cont, sockaddr const *addr, NetVCOptions *options = NULL);

  /**
    Open a NetVConnection for connection oriented I/O. This call
    is simliar to connect method except that the cont is called
    back only after the connections has been established. In the
    case of connect the cont could be called back with NET_EVENT_OPEN
    event and OS could still be in the process of establishing the
    connection. Re-entrant Callbacks: same as connect. If unix
    asynchronous type connect is desired use connect_re().
    @param cont Continuation to be called back with events.
    @param addr Address to which to connect (includes port).
    @param timeout for connect, the cont will get NET_EVENT_OPEN_FAILED
      if connection could not be established for timeout msecs. The
      default is 30 secs.
    @param options @see NetVCOptions.
    @see connect_re()
  */
  Action *connect_s(Continuation *cont, sockaddr const *addr, int timeout = NET_CONNECT_TIMEOUT, NetVCOptions *opts = NULL);

  /**
    在 EventSystem 启动后，由 main() 函数调用 netProcessor.start() 来为每一个 EThread 安装 NetHandler 等状态机。
    之后才可以使用 netProcessor 提供的其它 API 实现具体的网络I/O操作。
    Starts the Netprocessor. This has to be called before doing any
    other net call.
    @param number_of_net_threads is not used. The net processor
      uses the Event Processor threads for its activity.
      该参数用来指定ET_NET的数量，但是ET_NET的数量总是与EThread的数量相同。
  */
  virtual int start(int number_of_net_threads, size_t stacksize) = 0;

  inkcoreapi virtual NetVConnection *allocate_vc(EThread *) = 0;

  /** Private constructor. */
  NetProcessor(){};

  /** Private destructor. */
  virtual ~NetProcessor(){};

  /** This is MSS for connections we accept (client connections). */
  static int accept_mss;

  //
  // The following are required by the SOCKS protocol:
  //
  // Either the configuration variables will give your a regular
  // expression for all the names that are to go through the SOCKS
  // server, or will give you a list of domain names which should *not* go
  // through SOCKS. If the SOCKS option is set to false then, these
  // variables (regular expression or list) should be set
  // appropriately. If it is set to TRUE then, in addition to supplying
  // the regular expression or the list, the user should also give the
  // the ip address and port number for the SOCKS server (use
  // appropriate defaults)

  /* shared by regular netprocessor and ssl netprocessor */
  // connect方法支持通过socks服务器发起到目的IP的连接
  // 该静态变量用来保存全局使用的Socks配置信息
  static socks_conf_struct *socks_conf_stuff;

  /// Default options instance.
  // 该静态变量用来保存缺省的 AcceptOptions
  static AcceptOptions const DEFAULT_ACCEPT_OPTIONS;

private:
  /** @note Not implemented. */
  virtual int
  stop()
  {
    ink_release_assert(!"NetProcessor::stop not implemented");
    return 1;
  }

  NetProcessor(const NetProcessor &);
  NetProcessor &operator=(const NetProcessor &);
};


/**
  Global NetProcessor singleton object for making net calls. All
  net processor calls like connect, accept, etc are made using this
  object.
  @code
    netProcesors.accept(my_cont, ...);
    netProcessor.connect_re(my_cont, ...);
  @endcode
*/
extern inkcoreapi NetProcessor &netProcessor;
```

### 参考资料

- [I_NetProcessor.h](https://github.com/apache/trafficserver/tree/master/iocore/net/I_NetProcessor.h)

## 继承类 UnixNetProcessor

UnixNetProcessor 是 NetProcessor 在 类Unix 系统上的具体实现，而 NetProcessor 只是向外部用户提供接口的定义。

### 定义

```
struct UnixNetProcessor : public NetProcessor {
public:
  // 内部方法用来实现 accept 方法
  virtual Action *accept_internal(Continuation *cont, int fd, AcceptOptions const &opt);

  // 内部方法：用来实现 connect_re 方法
  Action *connect_re_internal(Continuation *cont, sockaddr const *target, NetVCOptions *options = NULL);
  // 内部方法：用来发起一个连接
  Action *connect(Continuation *cont, UnixNetVConnection **vc, sockaddr const *target, NetVCOptions *opt = NULL);

  // Virtual function allows etype to be upgraded to ET_SSL for SSLNetProcessor.  Does
  // nothing for NetProcessor
  // 内部方法：这个是用来提供 SSL 支持的，在SSLNetProcessor里会重写这个方法
  virtual void upgradeEtype(EventType & /* etype ATS_UNUSED */){};

  // 内部方法：创建 NetAccept 对象
  // 实际上应该是创建 UnixNetAccept 对象，然后返回基类 NetAccept 类型
  // 由于对Windows版本的支持被砍掉了，所以不存在 NTNetAccept 对象
  virtual NetAccept *createNetAccept();
  // 创建 UnixNetVConnection 对象，但是返回时采用基类的类型
  virtual NetVConnection *allocate_vc(EThread *t);

  // 创建 ET_NET 线程组，并对每一个 ET_NET 线程组
  // 为所有的EThread创建一个NetHandler状态机以及与NetHandler配合的周边组件
  virtual int start(int number_of_net_threads, size_t stacksize);

  // 此成员应该在某处被填充，当达到连接限制时，会向客户端发送填充的信息
  // 但是目前的代码里看不到有填充信息的操作，猜测应该是在 start() 里读取某个配置项。
  char *throttle_error_message;
  // 目前这个成员也没有在使用了
  // 在 branch 2.0.x 的代码里，UnixNetProcessor 还有一个 NetAccept 的原子队列：
  //     ASLL(NetAccept, link) accepts_on_thread
  // 感觉会在一个 NetAccept 运行时，处理这个队列上的所有 NetAccept 对象
  // 猜测：在较早的版本里，存在ET_ACCEPT，有兴趣的可以看一下 2.0.x 的代码
  // 这个应该就是保存了专门用来处理 NetAccept 队列的那个超级 NetAccept 状态机的 Event。
  Event *accept_thread_event;

  // offsets for per thread data structures
  // 记录netHandler和pollCont对象在线程堆栈里的偏移量
  off_t netHandler_offset;
  off_t pollCont_offset;

  // we probably wont need these members
  // 下面这两个变量只在 start() 函数中使用，实际上可以从类成员里去除了。
  // 当前 ET_NET 线程组里线程的数量
  int n_netthreads;
  // 指向 eventProcessor 里 ET_NET 线程组对象的指针数组
  EThread **netthreads;
};
```

### 参考资料

- [P_UnixNetProcessor.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNetProcessor.h)
- [UnixNetProcessor.cc](https://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetProcessor.cc)
