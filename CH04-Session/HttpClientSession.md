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
|      读     |     read._cont     | client_vc->read._cont  |
|      写     |     write._cont    | client_vc->write._cont |
|     驱动    |     NetHandler     |           none         |

很有趣的现象：

  - HttpClientSession 把 NetVConnection 作为资源来访问
    - 看上去就像是 NetVConnection 对 Socket FD 做了封装，
    - 而 HttpClientSession 又对 NetVConnection 做了一层封装
    - 所以 XxxSM 也会把 HttpClientSession 当作资源来访问
  - HttpClientSession 直接把 do_io 操作透传给了 NetVConnection
  - 前面讲过 NetVConnection 自身也是一个状态机，有三个状态及对应的事件回调函数
    - startEvent 处理connect
    - acceptEvent 处理accept
    - mainEvent 处理timeout
  - HttpClientSession 自身也是一个状态机，也有对应的状态回调函数
    - state_wait_for_close
    - state_keep_alive
    - state_slave_keep_alive
    - 具体的功能，接下来进行分析


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
  // ???
  bool conn_decrease;
  // HTTP/2 支持
  bool upgrade_to_h2c; // Switching to HTTP/2 with upgrade mechanism

  // 指向关联的 HttpServerSession，这样在Keep Alive支持时，直接可以获得上一次请求中使用的 HttpServerSession
  // 可以保证同一个连接上的请求，可以发送到对应的 HttpServerSession
  HttpServerSession *bound_ss;

  // ????
  MIOBuffer *read_buffer;
  // ????
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
  bool m_active;
};
```

## 状态（State）

## 理解 ClientSession 与 NetVConnection


## 参考资料

- [HttpClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.h)
- [HttpClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.cc)
