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

## 参考资料

- [HttpClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.h)
- [HttpClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpClientSession.cc)
