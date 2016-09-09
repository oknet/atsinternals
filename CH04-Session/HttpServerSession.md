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

  - HttpServerSession 则会被 ServerSessionPool 接管
  - ServerSessionPool 会设置一个新的超时时间

HttpServerSession 最终将关闭

  - 达到 ServerSessionPool 设置的超时时间
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

  // 脱离与 HttpSM 或 HttpClientSession 的关联，把自身放入到 ServerSessionPool 中
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
  TSServerSessionSharingMatchType sharing_match; // 匹配方式（None，IP，Host，Both）
  TSServerSessionSharingPoolType sharing_pool;   // 采用全局会话池或线程本地会话池
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

## 状态（State）

在 HttpServerSession 的定义中通过一个枚举类型定义了四个状态值，用来指示 HttpServerSession 的状态：

  - HSS_INIT
    - 初始状态，在构造函数中设置为该状态
    - 在 new_connection 中会再次设置为该状态
  - HSS_ACTIVE
    - 在 HttpSM 调用 new_connection 返回之后会立即设置为该状态
    - 在 HttpSM 从 HttpClientSession 获得之前使用的 HttpServerSession 后，会设置为该状态
    - 在 HttpSessionManager 从 ServerSessionPool 中获取 HttpServerSession 后，会设置为该状态
  - HSS_KA_CLIENT_SLAVE
    - 表示 HttpServerSession 正在被 HttpClientSession 接管
    - 当 HttpServerSession 遇到超时后，会被放入 ServerSessionPool
  - HSS_KA_SHARED
    - 表示 HttpServerSession 正在被 ServerSessionPool 接管

## 方法

HttpServerSession 的方法比较少，这与其作为 HttpClientSession 的附属对象有很大的关系。

一个 HttpServerSession 对象在整个生命周期中会被三个状态机管理：

  - HttpClientSession
  - HttpSM
  - ServerSessionPool

所以，其自身并没有任何的事件处理函数，需要提供的 Interface 基本上就是 VConnection 的 do_io 系列，以及下面几个。

在 HttpSM 向 OServer 发起连接之前，会查看 HttpClientSession 上是否存在已经绑定的 HttpServerSession，
还会对比当前请求的目标 OServer 是否与已经绑定的 HttpServerSession 匹配，如果不匹配就会：

  - 调用 HttpServerSession::release 解除 HttpServerSession 与 HttpSM 的关联
  - 然后调用 HttpClientSession::attach_server_session 解除 HttpServerSession 与 HttpClientSession 的绑定关系
  - 然后发起到 OServer 的连接

在连接成功建立之后会回调 NET_EVENT_OPEN 事件到 HttpSM::state_http_server_open

```
// 传入已经完成TCP连接建立的 NetVConnection
void
HttpServerSession::new_connection(NetVConnection *new_vc)
{
  ink_assert(new_vc != NULL);
  // 保存 NetVC 到 server_vc 成员
  server_vc = new_vc;

  // Used to do e.g. mutex = new_vc->thread->mutex; when per-thread pools enabled
  // 使用 NetVC 的 mutex 来设置 HttpServerSession 的 mutex
  mutex = new_vc->mutex;

  // Unique client session identifier.
  // 生成 HttpServerSession 的唯一 ID，用来在 Debug 信息中追踪
  con_id = ink_atomic_increment((int64_t *)(&next_ss_id), 1);

  // 设置 magic 值，用来发现内存分配错误
  magic = HTTP_SS_MAGIC_ALIVE;
  // 更新当前源服务器并发连接数
  HTTP_SUM_GLOBAL_DYN_STAT(http_current_server_connections_stat, 1); // Update the true global stat
  // 更新源服务器连接数总计数
  HTTP_INCREMENT_DYN_STAT(http_total_server_connections_stat);
  // Check to see if we are limiting the number of connections
  // per host
  // 如果开启了对源服务器的连接数限制
  // 在 HttpSM 发起对源服务器的TCP连接之前会首先判断指定 server_ip 上的连接数，
  //     如已经超出最大值 origin_max_connections，则在一段时间之后重新调度 HttpSM
  // 在 ServerSessionPool 收到 HttpServerSession 的超时事件时会判断指定 server_ip 的连接数，
  //     如已经超过 origin_min_keep_alive_connections，则直接关闭 HttpServerSession
  if (enable_origin_connection_limiting == true) {
    if (connection_count == NULL)
      connection_count = ConnectionCount::getInstance();
    // 对该 server_ip 的连接数增量
    connection_count->incrementCount(server_ip);
    char addrbuf[INET6_ADDRSTRLEN];
    Debug("http_ss", "[%" PRId64 "] new connection, ip: %s, count: %u", con_id,
          ats_ip_ntop(&server_ip.sa, addrbuf, sizeof(addrbuf)), connection_count->getCount(server_ip));
  }
  // 创建 MIOBuffer
#ifdef LAZY_BUF_ALLOC
  read_buffer = new_empty_MIOBuffer(HTTP_SERVER_RESP_HDR_BUFFER_INDEX);
#else
  read_buffer = new_MIOBuffer(HTTP_SERVER_RESP_HDR_BUFFER_INDEX);
#endif
  // 为 MIOBuffer 分配 IOBufferReader
  buf_reader = read_buffer->alloc_reader();
  Debug("http_ss", "[%" PRId64 "] session born, netvc %p", con_id, new_vc);
  // 设置状态
  state = HSS_INIT;
}
```

在 HttpSM::state_http_server_open 中调用 new_connection 之后，

  - 继续调用 HttpSM::attach_server_session
    - 设置 HttpServerSession 的回调函数为 HttpSM::state_send_server_request_header
    - 通过 do_io_read 接收数据
    - 通过 do_io_write 关闭数据发送
  - 然后调用 HttpSM::handle_http_server_open 完成发送请求给 OServer 的操作
    - 如果客户端是 HTTP POST 请求，可能流程会有变化

HttpSM 会先处理 OServer 的回应头，然后会建立 HttpTunnel 传输返回的内容，此时 HttpServerSession 的以下事件：

  - VC_EVENT_EOS
  - VC_EVENT_ERROR
  - VC_EVENT_READ_COMPLETE
  - VC_EVENT_ACTIVE_TIMEOUT
  - VC_EVENT_INACTIVITY_TIMEOUT

由 HttpSM::tunnel_handler_server 接管。

在 HttpSM::tunnel_handler_server 中会进行如下的判断：

  - 如果已经收到了 EOS 事件，
    - 那么就调用 HttpServerSession::do_io_close 关闭。
  - 如果不允许 OServer 的连接复用，
    - 那么就调用 HttpClientSession::attach_server_session 把 HttpServerSession 附到 HttpClientSession 上。
  - 如果允许连接复用，
    - 设置 keep alive 超时时间，然后调用 HttpServerSession::release 与 HttpSM 脱离

```
// 关闭并释放 HttpServerSession
void
HttpServerSession::do_io_close(int alerrno)
{
  // 如果是从 HttpSM 中发起的
  if (state == HSS_ACTIVE) {
    // 更新当前源服务器的并发事务数，该值在 HttpSM::attach_server_session 中做增量
    HTTP_DECREMENT_DYN_STAT(http_current_server_transactions_stat);
    // HttpServerSession 的引用数减少
    this->server_trans_stat--;
  }

  Debug("http_ss", "[%" PRId64 "] session closing, netvc %p", con_id, server_vc);

  // 关闭 NetVConnection
  server_vc->do_io_close(alerrno);
  // 解除与 NetVConnection 的关联
  server_vc = NULL;

  // 更新当前源服务器的并发连接数
  HTTP_SUM_GLOBAL_DYN_STAT(http_current_server_connections_stat, -1); // Make sure to work on the global stat
  // 统计平均每一个源服务器上完成的事务数量
  HTTP_SUM_DYN_STAT(http_transactions_per_server_con, transact_count);

  // Check to see if we are limiting the number of connections
  // per host
  // 如果开启了对源服务器的连接数限制
  if (enable_origin_connection_limiting == true) {
    if (connection_count->getCount(server_ip) > 0) {
      // 对该 server_ip 的连接数减量
      connection_count->incrementCount(server_ip, -1);
      char addrbuf[INET6_ADDRSTRLEN];
      Debug("http_ss", "[%" PRId64 "] connection closed, ip: %s, count: %u", con_id,
            ats_ip_ntop(&server_ip.sa, addrbuf, sizeof(addrbuf)), connection_count->getCount(server_ip));
    } else {
      Error("[%" PRId64 "] number of connections should be greater than zero: %u", con_id, connection_count->getCount(server_ip));
    }
  }

  // 如果是连接到 Parent Proxy 的
  if (to_parent_proxy) {
    // 更新当前到 Parent Proxy 的并发连接数，该值在 HttpSM::state_http_server_open 中做增量
    HTTP_DECREMENT_DYN_STAT(http_current_parent_proxy_connections_stat);
  }
  // 释放资源，回收对象内存
  destroy();
}
```

可以看到，只要调用了 HttpServerSession::do_io_close 则一定会关闭 NetVConnection 同时释放该对象，如果需要把 HttpServerSession 留下来复用，就需要通过：

  - HttpServerSession::release
    - 放入 ServerSessionPool
  - HttpClientSession::attach_server_session
    - 关联到 HttpClientSession

```
void
HttpServerSession::release()
{
  Debug("http_ss", "Releasing session, private_session=%d, sharing_match=%d", private_session, sharing_match);
  // Set our state to KA for stat issues
  state = HSS_KA_SHARED;

  // Private sessions are never released back to the shared pool
  // 如果是 私有会话，或者配置文件中关闭了 ServerSessionPool 功能
  //     则直接调用 do_io_close 关闭
  if (private_session || TS_SERVER_SESSION_SHARING_MATCH_NONE == sharing_match) {
    this->do_io_close();
    return;
  }

  // 通过 HttpSessionManager 把 HttpServerSession 放入 ServerSessionPool
  HSMresult_t r = httpSessionManager.release_session(this);

  // 根据返回值判断是否成功放入
  if (r == HSM_RETRY) {
    // Session could not be put in the session manager
    //  due to lock contention
    // 由于没有拿到 ServerSessionPool 的锁，导致放入失败，需要再次重试
    // FIX:  should retry instead of closing
    // 但是目前没有实现重试的机制，所以直接调用 do_io_close 关闭，
    //     重试的逻辑需要将来实现了。
    this->do_io_close();
  } else {
    // The session was successfully put into the session
    //    manager and it will manage it
    // 成功放入到 ServerSessionPool 内
    // (Note: should never get HSM_NOT_FOUND here)
    // 此时 r 应该等于HSM_DONE
    ink_assert(r == HSM_DONE);
  }
}
```

释放和回收资源的 destroy 方法

```
void
HttpServerSession::destroy()
{
  ink_release_assert(server_vc == NULL);
  ink_assert(read_buffer);
  ink_assert(server_trans_stat == 0);
  // 设置 magic
  magic = HTTP_SS_MAGIC_DEAD;
  // 释放 MIOBuffer
  if (read_buffer) {
    free_MIOBuffer(read_buffer);
    read_buffer = NULL;
  }

  // 清理 mutex
  mutex.clear();
  // 回收对象占用的内存
  if (TS_SERVER_SESSION_SHARING_POOL_THREAD == sharing_pool)
    THREAD_FREE(this, httpServerSessionAllocator, this_thread());
  else
    httpServerSessionAllocator.free(this);
}
```

## 参考资料

- [HttpServerSession.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpServerSession.h)
- [HttpServerSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpServerSession.cc)
