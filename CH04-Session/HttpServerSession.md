# 核心组件：HttpServerSession

HttpServerSession 与 HttpClientSession 相似，其内也包含了 NetVConnection 的成员，是对 NetVConnection 的扩展。

HttpServerSession 直接继承自 VConnection 基类，在 ATS 中没有设计 ProxyServerSession 这样一个中间层的继承类。

与所有的 VConnection 继承类一样：

  - 提供 do_io_*()
  - 提供 reenable() 方法

在 HttpServerSession::new_connection 方法中要求传入一个 NetVConnection，相对于 HttpClientSession::new_connection 传入的参数要少

  - 传入的 NetVConnection 对象会保存到 HttpServerSession::server_vc 成员中

比较特别的地方是，HttpServerSession 不是一个状态机，它没有定义任何事件回调函数

  - 但是 HttpClientSession::state_state_keep_alive 是专门为 HttpServerSession 服务的

因此，HttpServerSession 是从属于 HttpClientSession 的

  - 第一个HTTP请求由 HttpSM 解析完成后，如果需要连接 OServer 时
    - 则会在 NetVConnection 连接成功后创建 HttpServerSession 对象
  - 此时 HttpServerSession 上的事件都由 HttpSM 处理
  - 把第一个HTTP请求对应的响应发送给客户端之后，HttpSM 把 HttpServerSession 交给 HttpClientSession 来管理，HttpSM 则会释放自身，此时
    - Client NetVConnection 上的事件由 HttpClientSession 处理
    - OServer NetVConnection 上的事件也由 HttpClientSession 处理
    - 此时处于一个 TCP 连接上，第一个HTTP请求结束后，第二个HTTP请求还未到来的“间隙”
  - 第二个HTTP请求由 HttpSM 解析完成后，如果需要连接 OServer 时
    - 则会向 HttpClientSession 请求之前使用过的 HttpServerSession，如果存在就会继续复用
  - 然后新的 HttpSM 会接管 HttpServerSession 上的事件
  - 如此重复执行...

如果在“间隙”时遇到 HttpClientSession 被动关闭的情况，或者 HttpServerSession 出现了 Keep alive 超时

  - HttpServerSession 则会被 SessionPool 接管
  - SessionPool 会设置一个新的超时时间

HttpServerSession 最终将关闭

  - 达到 SessionPool 设置的超时时间
  - OServer 主动关闭了TCP连接

## 定义

```
class HttpServerSession : public VConnection
{
public:
  // 构造函数
  HttpServerSession()
    : VConnection(NULL), hostname_hash(), con_id(0), transact_count(0), state(HSS_INIT), to_parent_proxy(false),
      server_trans_stat(0), private_session(false), sharing_match(TS_SERVER_SESSION_SHARING_MATCH_BOTH),
      sharing_pool(TS_SERVER_SESSION_SHARING_POOL_GLOBAL), enable_origin_connection_limiting(false), connection_count(NULL),
      read_buffer(NULL), server_vc(NULL), magic(HTTP_SS_MAGIC_DEAD), buf_reader(NULL)
  {
    ink_zero(server_ip);
  }

  // 销毁自身
  void destroy();
  // 接受新的 NetVConnection
  void new_connection(NetVConnection *new_vc);

  // 重置 MIOBuffer
  void
  reset_read_buffer(void)
  {
    ink_assert(read_buffer->_writer);
    ink_assert(buf_reader != NULL);
    // 释放所有的 IOBufferReader
    // 除了当前正在写入的 IOBufferBlock，之前未读取的 IOBufferBlock 都会被自动释放掉
    read_buffer->dealloc_all_readers();
    // 把当前正在写入的 IOBufferBlock 也释放掉
    read_buffer->_writer = NULL;
    // 重新分配一个 IOBufferReader
    buf_reader = read_buffer->alloc_reader();
  }

  // 获得 IOBufferReader
  IOBufferReader *
  get_reader()
  {
    return buf_reader;
  };

  // 透传给 netvc 的 do_io_read
  virtual VIO *do_io_read(Continuation *c, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0);
  // 透传给 netvc 的 do_io_write
  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false);
  // 除了调用 netvc 的 do_io_close，最后还销毁了 HttpServerSession 自身
  virtual void do_io_close(int lerrno = -1);
  // 透传给 netvc 的 do_io_shutdown
  virtual void do_io_shutdown(ShutdownHowTo_t howto);
  // 透传给 netvc 的 reenable
  virtual void reenable(VIO *vio);

  // 脱离与 HttpSM 或 HttpClientSession 的关联，把自身放入到 SessionPool 中
  // 如果是私有会话，或者关闭了 ServerSession 的共享功能，则直接调用 do_io_close
  void release();
  // 计算给定 hostname 的MD5值，并保存到成员 hostname_hash 中
  void attach_hostname(const char *hostname);
  // 获取成员 server_vc
  NetVConnection *
  get_netvc()
  {
    return server_vc;
  };

  // Keys for matching hostnames
  // 用来在 SessionPool 中查找的凭据
  // 可以根据 OServer的IP，请求的 Hostname 来进行查找
  IpEndpoint server_ip;
  INK_MD5 hostname_hash;

  // HttpServerSession 的唯一 ID
  int64_t con_id;
  // 该 HttpServerSession 上处理过的事务／请求总数
  int transact_count;
  // HttpServerSession 的状态
  HSS_State state;

  // Used to determine whether the session is for parent proxy
  // it is session to orgin server
  // We need to determine whether a closed connection was to
  // close parent proxy to update the
  // proxy.process.http.current_parent_proxy_connections
  // 判断这是否是一个连接到 Parent Proxy 的 HttpServerSession
  // 用于辅助实现对当前ATS与Parent Proxy连接数量的统计
  bool to_parent_proxy;

  // Used to verify we are recording the server
  //   transaction stat properly
  // 该值为 0 表示 HttpServerSession 未被 HttpSM 管理
  // 该值为 1 表示 HttpServerSession 正被 HttpSM 接管
  // 其它值均为异常情况
  int server_trans_stat;

  // Sessions become if authentication headers
  //  are sent over them
  // 用来标记私有会话，通常指该会话有HTTP认证头信息
  bool private_session;

  // Copy of the owning SM's server session sharing settings
  // 在 HttpSM::state_http_server_open 中，复制当前 HttpSM 中的会话共享设置
  TSServerSessionSharingMatchType sharing_match;
  TSServerSessionSharingPoolType sharing_pool;
  //  int share_session;

  LINK(HttpServerSession, ip_hash_link);
  LINK(HttpServerSession, host_hash_link);

  // Keep track of connection limiting and a pointer to the
  // singleton that keeps track of the connection counts.
  // 对连接到 OServer 的 TCP 连接进行控制
  bool enable_origin_connection_limiting;
  ConnectionCount *connection_count;

  // The ServerSession owns the following buffer which use
  //   for parsing the headers.  The server session needs to
  //   own the buffer so we can go from a keep-alive state
  //   to being acquired and parsing the header without
  //   changing the buffer we are doing I/O on.  We can
  //   not change the buffer for I/O without issuing a
  //   an asyncronous cancel on NT
  // 用来接收 OServer 回应的 HTTP Response Header
  // 用来在“间隙”中接收可能的无效数据
  MIOBuffer *read_buffer;

private:
  HttpServerSession(HttpServerSession &);

  NetVConnection *server_vc;
  int magic;

  // 保持 read_buffer 中的数据不会被随意释放
  // 只有执行 reset_read_buffer 才会释放 read_buffer 中的数据
  IOBufferReader *buf_reader;
};
```

