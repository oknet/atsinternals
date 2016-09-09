# 核心组件：HttpClientSession

HttpClientSession 是 ProxyClientSession 的继承类。

由于 ProxyClientSession 继承自 VConnection，所以 HttpClientSession 从本质上来说也是一个 VConnection。

既然是 VConnection，那么它就像是一个文件句柄一样：

  - 提供 do_io_*()
  - 提供 reenable() 方法

在 HttpClientSession::new_connection() 方法中，总是要求传入一个 NetVConnection 类型
  - 传入的 NetVConnection 对象会保存到 HttpClientSession::client_vc 成员中
  - NetVConnection 的基类同样也是 VConnection 类型

对比一下 NetVConnection 和 HttpClientSession :

|     Type    |   NetVConnection   |    HttpClientSession   |
|:-----------:|:------------------:|:----------------------:|
|     资源    |       con.fd       |        client_vc       |
|    读回调   |     read._cont     | client_vc->read._cont  |
|    写回调   |     write._cont    | client_vc->write._cont |
|     驱动    |     NetHandler     |           none         |

很有趣的现象：

  - HttpClientSession 把 NetVConnection 作为资源来访问
    - 看上去就像是 NetVConnection 对 Socket FD 做了封装，
    - 而 HttpClientSession 又对 NetVConnection 做了一层封装
    - 所以 XxxSM 也会把 HttpClientSession 当作资源来访问
  - HttpClientSession 直接把 do_io 操作透传给了 NetVConnection

前面的章节讲过 NetVConnection 自身也是一个状态机，有三个状态及对应的回调函数：

  - startEvent 处理connect
  - acceptEvent 处理accept
  - mainEvent 处理timeout

HttpClientSession 自身也是一个状态机，也有多个状态及对应的回调函数：

  - state_wait_for_close
  - state_keep_alive
  - state_slave_keep_alive

具体的状态和功能，接下来进行分析。

## 定义

```
class HttpClientSession : public ProxyClientSession
{
public:
  typedef ProxyClientSession super; ///< Parent type.
  HttpClientSession();

  // Implement ProxyClientSession interface.
  // 首先，定义 3 个在 ProxyClientSession 声明为纯虚函数的方法
  virtual void destroy();

  virtual void
  start()
  {
    new_transaction();
  }

  void new_connection(NetVConnection *new_vc, MIOBuffer *iobuf, IOBufferReader *reader, bool backdoor);

  // Implement VConnection interface.
  // 定义 NetVConnection 要求的 do_io_* 和 reenable 方法
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0);
  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false);

  virtual void do_io_close(int lerrno = -1);
  virtual void do_io_shutdown(ShutdownHowTo_t howto);
  virtual void reenable(VIO *vio);

  // HttpClientSession 定义的方法
  // 用来创建一个新的事务
  void new_transaction();

  // 用来管理 半关闭 状态的三个方法
  void
  set_half_close_flag()
  {
    half_close = true;
  };
  void
  clear_half_close_flag()
  {
    half_close = false;
  };
  bool
  get_half_close_flag() const
  {
    return half_close;
  };
  // 由 HttpSM 调用，分离 HttpSM 与 HttpClientSession 的关联
  // 执行此操作之后，HttpSM 不会再被 Client 端的 NetVConnection 上的读写事件触发
  virtual void release(IOBufferReader *r);

  // 接着，定义 2 个在 ProxyClientSession 声明为纯虚函数的方法
  virtual NetVConnection *
  get_netvc() const
  {
    return client_vc;
  };
  // release_netvc() 只是分离 NetVConnection 与 HttpClientSession 的关联，并不会关闭 NetVConnection
  virtual void
  release_netvc()
  {
    client_vc = NULL;
  }

  // 关联一个 HttpServerSession 到当前的 HttpClientSession
  // 当 HttpSM 处理完一个 HTTP请求 后，就会：
  //     通过 attach_server_session 把 Server NetVConnection 的回调交给 HttpClientSession，
  //     然后释放 HttpSM 自身。
  // 如果该请求命中了 Cache，不需要通过 OServer 获取，那么就会提前把 Server NetVConnection 关联到 HttpClientSession，
  //     此时，由于 HttpSM 并未完成一个完整的请求处理过程，会设置 transaction_done = false;
  // 如果传入的 HttpServerSession 为 NULL：
  //     则表示取消当前 HttpClientSession 上的关联。
  // 当下一个 HTTP请求 进入后，再由 HttpClientSession 创建一个新的 HttpSM 来处理，当 HttpSM 需要与 OServer 建立连接时，
  //     会向 HttpClientSession 查找是否有之前使用过的 HttpServerSession，
  // 此时则可以把之前关联的 HttpServerSession 通过 get_server_session 取回。
  virtual void attach_server_session(HttpServerSession *ssession, bool transaction_done = true);
  // 返回已经关联到 HttpClientSession 的 HttpServerSession
  HttpServerSession *
  get_server_session() const
  {
    return bound_ss;
  };

  // Used for the cache authenticated HTTP content feature
  HttpServerSession *get_bound_ss();

  // Functions for manipulating api hooks
  // 实现与Hook相关的添加操作，同时会设置 HttpSM(current_reader) 的成员 hooks_set = 1
  void ssn_hook_append(TSHttpHookID id, INKContInternal *cont);
  void ssn_hook_prepend(TSHttpHookID id, INKContInternal *cont);

  // 由于 Http 协议存在 Keep Alive 功能，此计数器用来记录一个连接上处理了多少个 Http 请求
  int
  get_transact_count() const
  {
    return transact_count;
  }

  // 对 HTTP/2 的支持
  void
  set_h2c_upgrade_flag()
  {
    upgrade_to_h2c = true;
  }

private:
  HttpClientSession(HttpClientSession &);

  // 三个状态处理函数
  int state_keep_alive(int event, void *data);
  int state_slave_keep_alive(int event, void *data);
  int state_wait_for_close(int event, void *data);
  // 设置 TCP INIT CWND
  void set_tcp_init_cwnd();

  // 用来表示 HttpClientSession 的状态
  enum C_Read_State {
    HCS_INIT,
    HCS_ACTIVE_READER,
    HCS_KEEP_ALIVE,
    HCS_HALF_CLOSED,
    HCS_CLOSED,
  };

  // 是一个自增长ID，用来表示当前 HttpClientSession 的编号，会打印到 Debug 日志中，便于追踪
  int64_t con_id;
  // 保存传入的 NetVConnection 资源
  NetVConnection *client_vc;
  // 用来进行内存分配调试的 magic 值
  int magic;
  // 用来记录一个连接上处理了多少个 Http 请求
  int transact_count;
  // 开启 TCP INIT CWND
  bool tcp_init_cwnd_set;
  // 半关闭状态
  bool half_close;
  // 用来帮助统计当前ATS处理的并发连接数：http_current_client_connections_stat
  // 当前Session计入并发连接后，该值为 true
  // 当前Session从并发连接减除后，该值为 false
  // 但是在 do_io_close 中执行了减除操作后，为什么还要在 release 方法中再次判断？？？
  bool conn_decrease;
  // HTTP/2 支持
  bool upgrade_to_h2c; // Switching to HTTP/2 with upgrade mechanism

  // 指向关联的 HttpServerSession，这样在Keep Alive支持时，直接可以获得上一次请求中使用的 HttpServerSession
  // 可以保证同一个连接上的请求，可以发送到对应的 HttpServerSession
  HttpServerSession *bound_ss;

  // 对应 ka_vio 的 MIOBuffer
  MIOBuffer *read_buffer;
  // 与 read_buffer 对应的 IOBufferReader
  IOBufferReader *sm_reader;
  // 指向 HttpSM 状态机
  HttpSM *current_reader;
  // 描述 HttpClientSession 的当前状态
  C_Read_State read_state;

  // 处于 Keep Alive 支持时，在两个请求中间的状态时使用的VIO
  VIO *ka_vio;
  VIO *slave_ka_vio;

  Link<HttpClientSession> debug_link;

public:
  /// Local address for outbound connection.
  IpAddr outbound_ip4;
  /// Local address for outbound connection.
  IpAddr outbound_ip6;
  /// Local port for outbound connection.
  uint16_t outbound_port;
  /// Set outbound connection to transparent.
  bool f_outbound_transparent;
  /// Transparently pass-through non-HTTP traffic.
  bool f_transparent_passthrough;
  /// DNS resolution preferences.
  HostResStyle host_res_style;
  /// acl record - cache IpAllow::match() call
  const AclRecord *acl_record;

  // for DI. An active connection is one that a request has
  // been successfully parsed (PARSE_DONE) and it remains to
  // be active until the transaction goes through or the client
  // aborts.
  // 为 true ，表示当前 HttpClientSession 上的请求已经被 HttpSM 成功解析，接下来需要等待 ATS 向客户端发送数据
  // 此时，这个 NetVC 被认为是 active connection，需要等待该事务处理完成，或者连接异常中断。
  bool m_active;
};
```

## 状态（State）

在 HttpClientSession 的定义中通过一个枚举类型定义了五个状态值，用来指示 HttpClientSession 的状态：

  - HCS_INIT
    - 构造函数设置的初始值，没有什么用处
  - HCS_ACTIVE_READER
    - 表示当前正在由 HttpSM 来接管来自 NetVC 的数据流
    - 这里把 HttpSM 称之为 reader，因为 HttpClientSession 本身也是一个 VConnection
    - VConnection 一端连接了可以读写的资源，一端连接了状态机。
  - HCS_KEEP_ALIVE
    - 表示当前 HttpSM 已经与 HttpClientSession 分离
    - 当前由 HttpClientSession::state_keep_alive 接管 NetVConnection 的读事件
    - ka_vio 成员保存了 do_io_read 返回的VIO
    - read_buffer 成员作为接收数据的缓冲区
    - Inactivity Timeout 为 ka_in
    - 取消了 Active Timeout
    - 当Client与ATS之间的连接支持Keep Alive时，在第一个请求处理完成之后，就会进入这个状态，直到下一个请求的到来
  - HCS_HALF_CLOSED
    - 由于某种原因，HttpSM 发送完响应之后，就不会再接收来自 Client 的请求了，此时会设置半关闭
    - 此时应该由 Client 主动断开连接，当 HttpClientSession 收到 EVENT_EOS 时再完全关闭
    - 当前由 HttpClientSession::state_wait_for_close 接管 NetVConnection 上的读事件
    - ka_vio 成员保存了 do_io_read 返回的VIO
    - read_buffer 成员作为接收数据的缓冲区
    - 但是后续接收到的信息，都会被直接丢弃
  - HCS_CLOSED
    - 表示当前 HttpClientSession 进入关闭状态
    - 接下来会触发 TS_HTTP_SSN_CLOSE_HOOK，然后就会关闭 NetVConnection

在上面的介绍中，同时介绍了两个回调函数

  - state_keep_alive
  - state_wait_for_close

还有一个回调函数

  - state_slave_keep_alive

在 ATS 中，HttpServerSession 被设计为 HttpClientSession 的 slave，对应的 HttpClientSession 就是 master，在 master 上收到一个请求就会创建一个 HttpSM 来解析，然后会把 slave 关联到 master 上，当处理完成一个 HTTP 请求之后，就会释放 HttpSM 对象。

但是 master 与 slave 的关系仍然需要保留，这样才能在 Http Keep alive 支持中，继续把来自 Client 的请求仍然发送到上一次使用的那个TCP连接。

但是在两个 HTTP 请求中存在一个间隙，这个间隙里没有 HttpSM 对象来接管来自 Client NetVConnection 和 Server NetVConnection 的读事件，于是这个任务就安排给了 HttpClientSession，所以：

  - ka_vio 用来保存 Client NetVConnection 上的 do_io_read 返回的 VIO
  - read_buffer 是用来接收数据的缓冲区
  - ka_slave 用来保存 Server NetVConnection 上的 do_io_read 返回的 VIO
  - ssession 指向与 HttpClientSession 关联的 HttpServerSession 对象
  - ssession->read_buffer 是用来接收数据的缓冲区

## 方法

下面按照运行时，各个方法被调用的顺序进行分析，首先是被HttpSessionAccept::accept()方法调用的: new_connection()

```
void
HttpClientSession::new_connection(NetVConnection *new_vc, MIOBuffer *iobuf, IOBufferReader *reader, bool backdoor)
{
  ink_assert(new_vc != NULL);
  ink_assert(client_vc == NULL);
  // 保存 new_vc 到 HttpClientSession
  // 相当于把 NetVC 绑定到了 HttpClientSession，解除绑定时使用 release_netvc() 方法
  client_vc = new_vc;
  // 设置 magic 值，用于debug内存越界等问题
  magic = HTTP_CS_MAGIC_ALIVE;
  // HttpClientSession 共享 NetVC 的 mutex
  mutex = new_vc->mutex;
  // 尝试上锁
  // 由于与 NetVC 共享 mutex，因此这里应该、必须上锁成功
  MUTEX_TRY_LOCK(lock, mutex, this_ethread());
  // 这里的 assert 是一定不会触发的，如果触发，就一定是出现了问题
  ink_assert(lock.is_locked());

  // Disable hooks for backdoor connections.
  // 不允许 backdoor 被 hook
  this->hooks_on = !backdoor;

  // Unique client session identifier.
  // 给 HttpClientSession 生成一个唯一ID，在 debug 日志里会打印出这个 ID，方便追踪和分析
  con_id = ProxyClientSession::next_connection_id();

  // 对 HTTP 当前并发客户端事务计数器做递增
  HTTP_INCREMENT_DYN_STAT(http_current_client_connections_stat);
  conn_decrease = true;
  // 对 HTTP 客户端总连接计数器做递增
  HTTP_INCREMENT_DYN_STAT(http_total_client_connections_stat);
  // 如果是HTTPS请求，还需要对 HTTPS 客户端总连接计数器做递增
  if (static_cast<HttpProxyPort::TransportType>(new_vc->attributes) == HttpProxyPort::TRANSPORT_SSL) {
    HTTP_INCREMENT_DYN_STAT(https_total_client_connections_stat);
  }

  /* inbound requests stat should be incremented here, not after the
   * header has been read */
  // 用来统计HTTP进来的总连接数，跟 http_total_client_connections_stat 的区别是...？？？
  HTTP_INCREMENT_DYN_STAT(http_total_incoming_connections_stat);

  // check what type of socket address we just accepted
  // by looking at the address family value of sockaddr_storage
  // and logging to stat system
  // 分别统计 ipv4 和 ipv6 的HTTP客户端连接数
  switch (new_vc->get_remote_addr()->sa_family) {
  case AF_INET:
    HTTP_INCREMENT_DYN_STAT(http_total_client_connections_ipv4_stat);
    break;
  case AF_INET6:
    HTTP_INCREMENT_DYN_STAT(http_total_client_connections_ipv6_stat);
    break;
  default:
    // don't do anything if the address family is not ipv4 or ipv6
    // (there are many other address families in <sys/socket.h>
    // but we don't have a need to report on all the others today)
    break;
  }

#ifdef USE_HTTP_DEBUG_LISTS
  ink_mutex_acquire(&debug_cs_list_mutex);
  debug_cs_list.push(this, this->debug_link);
  ink_mutex_release(&debug_cs_list_mutex);
#endif

  DebugHttpSsn("[%" PRId64 "] session born, netvc %p", con_id, new_vc);

  // 用来接管 SSLNetVC 中剩余未读取的数据内容
  if (!iobuf) {
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(new_vc);
    if (ssl_vc) {
      iobuf = ssl_vc->get_ssl_iobuf();
      sm_reader = ssl_vc->get_ssl_reader();
    }
  }

  // 如果 SSLNetVC 中没有剩余数据，就创建一个新的 MIOBuffer 准备接收来自客户端的请求
  read_buffer = iobuf ? iobuf : new_MIOBuffer(HTTP_HEADER_BUFFER_SIZE_INDEX);
  // 创建对应的 IOBufferReader
  // 这里存在与 SSLNetVC 部分不同步的情况，可能导致 bug，在最新版本中，已经取消了 SSLNetVC 中的 iobuf
  sm_reader = reader ? reader : read_buffer->alloc_reader();

  // INKqa11186: Use a local pointer to the mutex as
  // when we return from do_api_callout, the ClientSession may
  // have already been deallocated.
  // 由于从 do_api_callout 返回后，可能 HttpClientSession 已经被释放了，
  // 因此这里创建了一个本地变量来保存 mutex，保证从 do_api_callout 返回后的正常解锁。
  EThread *ethis = this_ethread();
  Ptr<ProxyMutex> lmutex = this->mutex;
  MUTEX_TAKE_LOCK(lmutex, ethis);
  // 由于在 HttpClientSession 中没有定义 do_api_callout 方法，
  // 因此这里是直接调用基类 ProxyClientSession 的 do_api_callout 方法。
  do_api_callout(TS_HTTP_SSN_START_HOOK);
  MUTEX_UNTAKE_LOCK(lmutex, ethis);
  // lmutex 是自动指针，这里是不是可以不写这句？
  lmutex.clear();
}
```

为了便于分析，我们按照没有任何 Plugin Hook 在 SSN START 这里来分析，

  - 那么 do_api_callout 会直接调用 handle_api_return(TS_EVENT_HTTP_CONTINUE)
  - 然后 handle_api_return 会调用 start() 方法
  - HttpClientSession::start() 直接调用了 new_transaction() 方法

```
void
HttpClientSession::new_transaction()
{
  ink_assert(current_reader == NULL);
  PluginIdentity *pi = dynamic_cast<PluginIdentity *>(client_vc);

  // 用来判断 client_vc 是否是一个 plugin 创建的 vc
  // 如果不是的话，则要把 client_vc 添加到 NetHandler 的 active queue 中
  // ATS 通过active queue来限定最大并发连接数，
  // 如果添加失败，则说明已经达到最大并发连接数，则直接调用 do_io_close() 关闭 HttpClientSession
  if (!pi && client_vc->add_to_active_queue() == false) {
    // no room in the active queue close the connection
    this->do_io_close();
    return;
  }

  // Defensive programming, make sure nothing persists across
  // connection re-use
  // 预防性设计，确保不会出现连接重用
  half_close = false;

  // 设置 HttpClientSession 的状态为 HttpSM 接管
  read_state = HCS_ACTIVE_READER;
  // 创建 HttpSM 对象
  current_reader = HttpSM::allocate();
  // 初始化 HttpSM 对象
  current_reader->init();
  // Http事务计数器累加
  transact_count++;
  DebugHttpSsn("[%" PRId64 "] Starting transaction %d using sm [%" PRId64 "]", con_id, transact_count, current_reader->sm_id);

  // 绑定 HttpClientSession 与 HttpSM
  // 此时 HttpSM 通过调用 HttpClientSession 的 do_io 方法，已经把 client_vc 的 I/O 事件回调指向了 HttpSM 状态机。
  current_reader->attach_client_session(this, sm_reader);
  // 对 Plugin 创建的 VC 额外处理，此处跳过，暂不做分析
  if (pi) {
    // it's a plugin VC of some sort with identify information.
    // copy it to the SM.
    current_reader->plugin_tag = pi->getPluginTag();
    current_reader->plugin_id = pi->getPluginId();
  }
}
```

接下来会转入 HttpSM 的处理流程，

  - 在 HttpSM 需要建立与 OS 的连接时，如果没有开启 HttpServerSession 的共享功能
    - 会在 HttpSM::do_http_server_open() 方法中调用 httpSessionManager.acquire_session()
    - 在该方法中会通过 get_bound_ss() 获得已经绑定的 HttpServerSession，保存到 existing_ss
    - 然后调用 HttpClientSession::attach_server_session(NULL) 清除绑定关系
    - 然后再验证之前获得的 HttpServerSession，通过验证后调用 HttpSM::attach_server_session(existing_ss) 将其作为 HttpSM 的 ServerSession
    - 未通过验证，调用 HttpClientSession::attach_server_session(NULL) 清除绑定关系

然后 HttpSM 会创建 HttpTunnel，在 HttpTunnel 传递完所有数据之后：

  - 在 HttpSM::tunnel_handler_server() 中：
    - 判断 OS 是否支持 Keep alive，
    - 判断 没有遇到服务端的 EOS，TIMEOUT 事件，
    - 判断 客户端的数据流没有出现异常中止的情况，
    - 如果开启了 attach_server_session_to_client 功能，会调用 HttpClientSession::attach_server_session(server_session)
    - 如果没有开启该功能，则调用 server_session->release() 把 HttpServerSession 放回到连接池
  - 在 HttpSM::release_server_session() 中：
    - 如果这是由 Cache 提供内容响应
    - 则通过 HttpClientSession::attach_server_session(server_session, false) 绑定

因此，HttpClientSession::attach_server_session 的功能是：

  - 在 HttpSM 处理完一个事务之后，
  - 如果需要当前的 HttpServerSession 继续处理下一个来自 HttpClientSession 的事务
  - 那么就把当前的 HttpServerSession 绑定到 HttpClientSession 上
  - 需要注意与 HttpSM::attach_server_session 的区别。

```
void
HttpClientSession::attach_server_session(HttpServerSession *ssession, bool transaction_done)
{
  if (ssession) {
    // 传入的 ssession 非空，表示要把 ssession 绑定到当前的 HttpClientSession
    ink_assert(bound_ss == NULL);
    // 设置 HttpServerSession 的状态为 已经绑定到 HttpClientSession
    ssession->state = HSS_KA_CLIENT_SLAVE;
    // 把 HttpServerSession 保存到 bound_ss 成员
    bound_ss = ssession;
    DebugHttpSsn("[%" PRId64 "] attaching server session [%" PRId64 "] as slave", con_id, ssession->con_id);
    ink_assert(ssession->get_reader()->read_avail() == 0);
    ink_assert(ssession->get_netvc() != client_vc);

    // handling potential keep-alive here
    // 这里应该仅针对来自 HttpSM::tunnel_handler_server() 的调用生效
    // 对于来自 httpSessionManager.acquire_session() 的调用，感觉此处应该存在bug？
    if (m_active) {
      m_active = false;
      HTTP_DECREMENT_DYN_STAT(http_current_active_client_connections_stat);
    }
    // Since this our slave, issue an IO to detect a close and
    //  have it call the client session back.  This IO also prevent
    //  the server net conneciton from calling back a dead sm
    // 能够被关联 HttpServerSession，说明支持 Keep alive 特性，因此直接设置由 state_keep_alive 接管 HttpServerSession
    SET_HANDLER(&HttpClientSession::state_keep_alive);
    slave_ka_vio = ssession->do_io_read(this, INT64_MAX, ssession->read_buffer);
    // 这里有点奇怪，为啥 HttpServerSession 的状态机要指向到 HttpClientSession？
    ink_assert(slave_ka_vio != ka_vio);

    // Transfer control of the write side as well
    // 关闭 HttpServerSession 的数据发送，感觉这里应该是 do_io_write(NULL, 0, NULL)
    ssession->do_io_write(this, 0, NULL);

    if (transaction_done) {
      // 默认为 true，表示调用该方法时已经处理完一个事务
      // 接下来将要等待客户端发起下一个事务
      // 所以设置 inactivity timeout 为 OS 端 keep alive 超时
      ssession->get_netvc()->set_inactivity_timeout(
        HRTIME_SECONDS(current_reader->t_state.txn_conf->keep_alive_no_activity_timeout_out));
      // 同时取消 active timeout
      ssession->get_netvc()->cancel_active_timeout();
    } else {
      // we are serving from the cache - this could take a while.
      // 只有从 Cache 中返回需要通过用户认证才能访问的内容时
      // 由于是从 Cache 中返回内容，需要的时间不可控，所以暂时取消 HttpServerSession 上 NetVC 的超时检测
      // 由于 ATS 具有缓存机制，在 keep alive 连接上，可能有的事务（请求）可以命中 Cache，有的无法命中，
      // 因此，当命中 Cache 时，由 Cache 提供内容期间，需要保持 HttpServerSession，必须临时关闭超时控制。
      ssession->get_netvc()->cancel_inactivity_timeout();
      ssession->get_netvc()->cancel_active_timeout();
    }
  } else {
    // 传入的 ssession 为空，表示要把绑定到当前 HttpClientSession 的信息清除
    ink_assert(bound_ss != NULL);
    bound_ss = NULL;
    slave_ka_vio = NULL;
  }
}
```

当一个事务（请求）结束时，HttpSM 会根据情况调用 HttpClientSession::release() 或者 HttpClientSession::do_io_close()

  - 如果客户端支持 keep alive，那么就会调用 release()
  - 否则会调用 do_io_close() 
  - 可以参考 HttpSM::tunnel_handler_ua() 的代码进行分析

```
// release 方法只是把 HttpClientSession 与 HttpSM 之间的关联断开
// 但是 HttpClientSession 与 HttpServerSession 之间的关系，仍然不变
// 而且不会主动关闭会话，但是由于 keep alive 连接队列满了，可能会导致被迫关闭的情况
void
HttpClientSession::release(IOBufferReader *r)
{
  // release 调用必须由上层状态机完成一个事务处理后发起
  // 因此在调用 release 时，HttpClientSession 的状态必须为 HCS_ACTIVE_READER
  ink_assert(read_state == HCS_ACTIVE_READER);
  ink_assert(current_reader != NULL);
  // 从 HttpSM 获得客户端的 keep alive 的超时时间设置
  MgmtInt ka_in = current_reader->t_state.txn_conf->keep_alive_no_activity_timeout_in;

  DebugHttpSsn("[%" PRId64 "] session released by sm [%" PRId64 "]", con_id, current_reader->sm_id);
  // 解除 HttpClientSession 与 HttpSM 的关联
  current_reader = NULL;

  // handling potential keep-alive here
  // 如果该连接为 active connection，则转为 inactive
  // 同时对 HTTP 当前活动客户端连接计数器做递减
  if (m_active) {
    m_active = false;
    HTTP_DECREMENT_DYN_STAT(http_current_active_client_connections_stat);
  }
  // Make sure that the state machine is returning
  //  correct buffer reader
  // 容错性检查，要求必须传入正确的 IOBufferReader
  ink_assert(r == sm_reader);
  if (r != sm_reader) {
    // 否则通过立即调用 do_io_close() 关闭会话，来减小此异常/问题产生的影响
    this->do_io_close();
    return;
  }

  // 对 HTTP 当前并发客户端事务计数器做递减
  HTTP_DECREMENT_DYN_STAT(http_current_client_transactions_stat);

  // Clean up the write VIO in case of inactivity timeout
  // 关闭 client_vc 上的写操作
  this->do_io_write(NULL, 0, NULL);

  // Check to see there is remaining data in the
  //  buffer.  If there is, spin up a new state
  //  machine to process it.  Otherwise, issue an
  //  IO to wait for new data
  if (sm_reader->read_avail() > 0) {
    // 查看在读取缓冲区内是否有从 client_vc 接收到的请求数据，
    //   例如在 pipeline 模式下，可能已经有下一个请求在读取缓冲区中了
    // 直接调用 new_transaction() 开始下一个请求的处理。
    DebugHttpSsn("[%" PRId64 "] data already in buffer, starting new transaction", con_id);
    new_transaction();
  } else {
    // 如果没有任何数据
    DebugHttpSsn("[%" PRId64 "] initiating io for next header", con_id);
    // HttpClientSession 进入 HCS_KEEP_ALIVE 状态
    read_state = HCS_KEEP_ALIVE;
    // 由 state_keep_alive 来接管 client_vc 上读取到的数据
    SET_HANDLER(&HttpClientSession::state_keep_alive);
    ka_vio = this->do_io_read(this, INT64_MAX, read_buffer);
    ink_assert(slave_ka_vio != ka_vio);
    // 设置等待超时时间（由 HttpSM 获取）
    client_vc->set_inactivity_timeout(HRTIME_SECONDS(ka_in));
    // 取消 active timeout
    client_vc->cancel_active_timeout();
    // 将 client_vc 放入 NetHandler 的 keep_alive 连接队列
    client_vc->add_to_keep_alive_queue();
    // 如果连接队列满了，会立即关闭 client_vc，state_keep_alive()会收到 TIMEOUT 事件，然后调用 do_io_close()
  }
}
```

```
// do_io_close 首先将 HttpClientSession 与 HttpServerSession 之间的关联断开
// 然后把 HttpServerSession 放回连接池，然后执行会话的关闭过程
void
HttpClientSession::do_io_close(int alerrno)
{
  // 如果是由 HttpSM 处理完一个 HTTP 请求后调用，状态值应该为 HCS_ACTIVE_READER
  // 但是 do_io_close 调用可能来自 HttpClientSession 内部，例如：
  //   在 HCS_KEEP_ALIVE 状态时收到了 EOS 或者 TIMEOUT 事件
  if (read_state == HCS_ACTIVE_READER) {
    // 对 HTTP 当前并发客户端事务计数器做递减
    HTTP_DECREMENT_DYN_STAT(http_current_client_transactions_stat);
    // 如果该连接为 active connection，则转为 inactive
    // 同时对 HTTP 当前活动客户端连接计数器做递减
    if (m_active) {
      m_active = false;
      HTTP_DECREMENT_DYN_STAT(http_current_active_client_connections_stat);
    }
  }

  // Prevent double closing
  // 防止多重关闭的 assert
  ink_release_assert(read_state != HCS_CLOSED);

  // If we have an attached server session, release
  //   it back to our shared pool
  // 回收 HttpServerSession 到 Session Pool
  if (bound_ss) {
    bound_ss->release();
    bound_ss = NULL;
    slave_ka_vio = NULL;
  }

  if (half_close && this->current_reader) {
    // 判断执行半关闭操作
    read_state = HCS_HALF_CLOSED;
    // 由 state_wait_for_close 处理半关闭
    SET_HANDLER(&HttpClientSession::state_wait_for_close);
    DebugHttpSsn("[%" PRId64 "] session half close", con_id);

    // We want the client to know that that we're finished
    //  writing.  The write shutdown accomplishes this.  Unfortuantely,
    //  the IO Core semantics don't stop us from getting events
    //  on the write side of the connection like timeouts so we
    //  need to zero out the write of the continuation with
    //  the do_io_write() call (INKqa05309)
    // 向客户端发送半关闭，客户端可以通过 read() 返回值为0，判断关闭事件
    client_vc->do_io_shutdown(IO_SHUTDOWN_WRITE);

    // 关注后续 client_vc 上的数据，但是这些数据在 state_wait_for_close 直接被丢弃
    ka_vio = client_vc->do_io_read(this, INT64_MAX, read_buffer);
    ink_assert(slave_ka_vio != ka_vio);

    // [bug 2610799] Drain any data read.
    // If the buffer is full and the client writes again, we will not receive a
    // READ_READY event.
    // 直接丢弃来自 client_vc 剩余未处理的数据
    sm_reader->consume(sm_reader->read_avail());

    // Set the active timeout to the same as the inactive time so
    //   that this connection does not hang around forever if
    //   the ua hasn't closed
    // 设置一个超时时间，防止客户端长时间不关闭
    client_vc->set_active_timeout(HRTIME_SECONDS(current_reader->t_state.txn_conf->keep_alive_no_activity_timeout_out));
  } else {
    // 否则执行完全关闭操作 
    read_state = HCS_CLOSED;
    // clean up ssl's first byte iobuf
    // 清理 SSLNetVC 的 iobuf
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(client_vc);
    if (ssl_vc) {
      ssl_vc->set_ssl_iobuf(NULL);
    }
    // 切换到 HTTP/2，暂不做分析
    if (upgrade_to_h2c) {
      Http2ClientSession *h2_session = http2ClientSessionAllocator.alloc();

      h2_session->set_upgrade_context(&current_reader->t_state.hdr_info.client_request);
      h2_session->new_connection(client_vc, NULL, NULL, false /* backdoor */);
      // Handed over control of the VC to the new H2 session, don't clean it up
      this->release_netvc();
      // TODO Consider about handling HTTP/1 hooks and stats
    } else {
      DebugHttpSsn("[%" PRId64 "] session closed", con_id);
    }
    // 统计 平均每个客户端连接上的 HTTP 事务数
    HTTP_SUM_DYN_STAT(http_transactions_per_client_con, transact_count);
    // 对 HTTP 当前并发客户端连接计数器做递减
    HTTP_DECREMENT_DYN_STAT(http_current_client_connections_stat);
    conn_decrease = false;
    // 触发 SSN CLOSE HOOK
    do_api_callout(TS_HTTP_SSN_CLOSE_HOOK);
  }
}
```

为了便于分析，我们按照没有任何 Plugin Hook 在 SSN CLOSE 这里来分析：

  - 执行 client_vc->do_io_close() 关闭 NetVC
  - 执行 release_netvc() 解除 HttpClientSession 与 client_vc 的关联
  - 调用 HttpClientSession::destroy() 释放对象

```
void
HttpClientSession::destroy()
{
  DebugHttpSsn("[%" PRId64 "] session destroy", con_id);

  ink_release_assert(upgrade_to_h2c || !client_vc);
  ink_release_assert(bound_ss == NULL);
  ink_assert(read_buffer);

  // 设置 magic 值，表示当前对象占用的内存空间已经不再使用
  magic = HTTP_CS_MAGIC_DEAD;
  // 如果 NetVC 还没有被 HttpSM 接管，就提前导致关闭时，read_buffer 需要在这里释放
  if (read_buffer) {
    free_MIOBuffer(read_buffer);
    read_buffer = NULL;
  }

#ifdef USE_HTTP_DEBUG_LISTS
  ink_mutex_acquire(&debug_cs_list_mutex);
  debug_cs_list.remove(this, this->debug_link);
  ink_mutex_release(&debug_cs_list_mutex);
#endif

  // 感觉这里应该不会被运行 ...
  // 在调用 release 之前必然会调用 do_io_close，在 do_io_close 中已经进行了处理。
  if (conn_decrease) {
    HTTP_DECREMENT_DYN_STAT(http_current_client_connections_stat);
    conn_decrease = false;
  }

  // 调用 ProxyClientSession::destroy()
  super::destroy();
  // 释放对象所占用的内存空间
  THREAD_FREE(this, httpClientSessionAllocator, this_thread());
}
```

## 理解 HttpClientSession 与 NetVConnection

HttpClientSession 和 NetVConnection 同样都是继承自 VConnection 基类，而且 NetVConnection 作为 HttpClientSession 的成员。

可以看到 HttpClientSession 把 do_io 操作基本上透传给了 NetVConnection 成员。

如果 HttpClientSession 是继承自 NetVConnection，在 NetAccept 接受一个新的 socket 连接时，直接创建 HttpClientSession，是不是也是可以的？

  - 如果按照这样的设计，ATS就需要给每一种协议提供一种 NetAccept 的继承类，那么就变成了：
    - HttpNetAccept，HttpClientSession，HttpSM
  - 而目前的设计是：
    - NetAccept, ProtocolSessionProbeAccept, ProtocolProbeTrampoline, HttpClientSession，HttpSM
  - 可以看到，现有的设计可以更灵活的支持多种协议
  - 同时支持 NetVConnection 的多操作系统支持（UnixNetVConnection）

所以 HttpClientSession 理论上就是一种更具体的，与协议相关的 NetVConnection，方便与 HttpSM 的协同。

## TCP_INIT_CWND

CWND 是 Congestion Window 的缩写，INIT CWND 表示初始拥塞窗口。

在 Linux 3.0 以前，内核默认的 initcwnd 比较小，是根据 RFC3390 定义的方式，通过MSS的值计算得出：

  - initcwnd = 4
    - MSS <= 1095
  - initcwnd = 3
    - 1095 < MSS <= 2190
  - initcwnd = 2
    - MSS > 2190

通常，我们的 MSS 为 1460，因此初始的拥塞控制窗口为 3。

```
/* Convert RFC3390 larger initial window into an equivalent number of packets. 
 * This is based on the numbers specified in RFC 6861, 3.1. 
 */  
static inline u32 rfc3390_bytes_to_packets(const u32 smss)  
{  
    return smss <= 1095 ? 4 : (smss > 2190 ? 2 : 3);  
}  
```

Linux 3.0 以后，采取了Google的建议，把初始拥塞控制窗口调到了 10。

Google's advice：[An Argument for Increasing TCP's Initial Congestion Window](http://code.google.com/speed/articles/tcp_initcwnd_paper.pdf)

  - The recommended value of initcwnd is 10*MSS.

最初于 2010年10月 提出了一个 [IETF DRAFT](http://tools.ietf.org/html/draft-ietf-tcpm-initcwnd-00)，经过近10个版本的修订，于2013年4月成为 [RFC 6928](https://tools.ietf.org/html/rfc6928)。

但是作为一个TCP Server，要面对不同的延迟和带宽的用户，将 initcwnd 统一的设置为10并不可取，因为对于很多通过手机上网的客户端，由于带宽较窄，反而会导致严重的网络数据拥塞。

因此，ATS 创建了 [TS-940](https://issues.apache.org/jira/browse/TS-940) 来实现一个可配置的参数：

  - proxy.config.http.server_tcp_init_cwnd

同时为了支持基于条件，为不同的客户端设定不同的 initcwnd 的值，还支持通过 remap 来为每一个连接动态修改。

TS-940 的代码提交被分为两个部分：

  - https://github.com/apache/trafficserver/commit/0e4814487769a203ea3aa0522374798ec60dfe4c
    - 在 UnixNetProcessor::accept_internal 中对 listen fd 设置 initcwnd 的值
    - 该设置应该只对 Solaris 10+ 的版本支持
    - 这个设置对于所有accept的连接都是有效的
  - https://github.com/apache/trafficserver/commit/341a2c5aeac4a3073c5c84ea410455194cc909ab
    - 这个是在 HttpClientSession::do_io_write 中对首次执行写操作之前对当前 socket 连接设置 initcwnd 的值
    - 在进行首次 do_io_write 调用之前，肯定是已经从客户端读取到了 HTTP 请求，因此 remap 已经被执行过了
    - 通过 remap 插件 conf_remap 可以对当前连接的 initcwnd 进行动态修改
    - 相关的 InkAPI：
      - TSHttpTxnConfigIntSet(TSHttpTxn txnp, TS_CONFIG_HTTP_SERVER_TCP_INIT_CWND, value)


## 参考资料

- [HttpClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.h)
- [HttpClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.cc)
- [TCP_INIT_CWND相关](http://kb.cnblogs.com/page/209101/)
