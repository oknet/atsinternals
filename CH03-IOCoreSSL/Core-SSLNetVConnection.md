# 核心组件：SSLNetVConnection

SSLNetVConnection 继承自 UnixNetVConnection，在其基础上构建了对SSL会话的支持。

与介绍UnixNetVConnection时一样，也来一个IOCoreSSL子系统的对比，与EventSystem是一样的，也有Thread，Processor和Event，只是名字不一样了：

|  EventSystem   |         IOCoreNet         |         IOCoreSSL         |
|:--------------:|:-------------------------:|:-------------------------:|
|      Event     |     UnixNetVConnection    |     SSLNetVConnection     |
|     EThread    | NetHandler，InactivityCop | NetHandler, InactivityCop |
| EventProcessor |        NetProcessor       |      sslNetProcessor      |

- 像 Event和UnixNetVConnection 一样，SSLNetVConnection 也提供了面向上层状态机的方法
  - do_io_* 系列
  - (set|cancel)_*_timeout 系列
  - (add|remove)_*_queue 系列
  - 还有一部分面向上层状态机的方法在sslNetProcessor中定义
- SSLNetVConnection 也提供了面向底层状态机的方法
  - 通常由NetHandler来调用
  - 可以把这些方法视作NetHandler状态机的专用回调函数
  - 我个人觉得，应该把所有跟socket打交道的函数都放在NetHandler里面
- SSLNetVConnection 也是状态机
  - 因此它也有自己的handler（回调函数）
    - SSLNetAccept调用acceptEvent
    - InactivityCop调用mainEvent
    - 构造函数会初始化为startEvent，用于调用connectUp()，这是面向sslNetProcessor的
  - 大致有以下三条调用路径：
    - EThread  －－－  SSLNetAccept  －－－ SSLNetVConnection
    - EThread  －－－  NetHandler  －－－  SSLNetVConnection
    - EThread  －－－  InactivityCop  －－－  SSLNetVConnection

由于它既是Event，又是SM，还比UnixNetVConnection增加了SSL的处理，所以从形态上来看，SSLNetVConnection 要比 Event和UnixNetVConnection 复杂的多。

SSLNetVConnection 对 UnixNetVConnection 的部分方法进行了重载：

  - net_read_io
  - load_buffer_and_write
  - SSLNetVConnection
  - do_io_close
  - free
  - reenable
  - getSSLHandShakeComplete
  - setSSLHandShakeComplete
  - getSSLClientConnection
  - setSSLClientConnection

同时增加了一些方法：

  - enableRead
  - getSSLSessionCacheHit
  - setSSLSessionCacheHit
  - read_raw_data
  - initialize_handshake_buffers
  - free_handshake_buffers
  - sslStartHandShake
  - sslServerHandShakeEvent
  - sslClientHandShakeEvent
  - registerNextProtocolSet
  - endpoint
  - advertise_next_protocol
  - select_next_protocol
  - sslContextSet
  - callHooks
  - calledHooks
  - get_ssl_iobuf
  - set_ssl_iobuf
  - get_ssl_reader
  - isEosRcvd


## 定义

在下面的成员方法里，凡是相对于UnixNetVConnection新增的、重载的，都会标记说明。

```
// 继承自UnixNetVConnection
class SSLNetVConnection : public UnixNetVConnection
{
  // 实现 super ，这个很有趣，也很巧妙
  typedef UnixNetVConnection super; ///< Parent type.
public:
  // 这是一个ssl握手的入口，根据握手是Client与ATS之间，还是ATS与OS之间，再调用后面的两个方法来完成。
  virtual int sslStartHandShake(int event, int &err);
  virtual void free(EThread *t);
  
  // 新增方法
  // 同时激活了read VIO和write VIO
  virtual void
  enableRead()
  {
    read.enabled = 1;
    write.enabled = 1;
  };
  
  // 重载方法
  // 返回成员sslHandShakeComplete，表示是否完成了握手过程
  virtual bool
  getSSLHandShakeComplete()
  {
    return sslHandShakeComplete;
  };
  // 重载方法
  // 设置成员sslHandShakeComplete
  void
  setSSLHandShakeComplete(bool state)
  {
    sslHandShakeComplete = state;
  };
  // 重载方法
  // 返回成员sslClientConnection，表示这是否是来自Client的SSL连接。
  virtual bool
  getSSLClientConnection()
  {
    return sslClientConnection;
  };
  // 重载方法
  // 设置成员sslClientConnection
  virtual void
  setSSLClientConnection(bool state)
  {
    sslClientConnection = state;
  };
  // 新增方法
  // 设置成员sslSessionCacheHit
  virtual void
  setSSLSessionCacheHit(bool state)
  {
    sslSessionCacheHit = state;
  };
  // 新增方法
  // 返回成员sslSessionCacheHit，用来表示这是否是一个Session Reuse的SSL连接
  virtual bool
  getSSLSessionCacheHit()
  {
    return sslSessionCacheHit;
  };
  
  // 新增方法，两个方法都是被sslStartHandShake调用
  // 用于实现Client与ATS之间的握手
  int sslServerHandShakeEvent(int &err);
  // 用于实现ATS与OS之间的握手
  int sslClientHandShakeEvent(int &err);

  // 重载方法
  // 与NetHandler结合，实现对SSL数据的解密（net_read_io）和加密（load_buffer_and_write）
  virtual void net_read_io(NetHandler *nh, EThread *lthread);
  virtual int64_t load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                        int &needs);
  // 实现 NPN 方式的状态机协议流转
  // 注册一个 NPN 状态机的入口
  void registerNextProtocolSet(const SSLNextProtocolSet *);
  // 重载方法
  virtual void do_io_close(int lerrno = -1);

  ////////////////////////////////////////////////////////////
  // Instances of NetVConnection should be allocated        //
  // only from the free list using NetVConnection::alloc(). //
  // The constructor is public just to avoid compile errors.//
  ////////////////////////////////////////////////////////////
  SSLNetVConnection();
  virtual ~SSLNetVConnection() {}

  // 指向用来保存 SSL 会话的对象
  SSL *ssl;
  // 记录ssl会话开始的时间
  ink_hrtime sslHandshakeBeginTime;
  // 记录ssl最后发送数据的时间，由load_buffer_and_write在数据发送之后更新
  ink_hrtime sslLastWriteTime;
  // 记录ssl总共发送的字节数，由load_buffer_and_write在数据发送之后更新
  int64_t sslTotalBytesSent;

  // 实现 NPN/ALPN 方式的状态机协议流转
  // 使用一个 NPN 状态机的入口来测试当前接收的数据，是否能够被该状态机处理
  //     通过 SSL_CTX_set_next_protos_advertised_cb() 设置
  static int advertise_next_protocol(SSL *ssl, const unsigned char **out, unsigned *outlen, void *);
  // 选择下一个 ALPN 状态机
  //     通过 SSL_CTX_set_alpn_select_cb() 设置
  static int select_next_protocol(SSL *ssl, const unsigned char **out, unsigned char *outlen, const unsigned char *in,
                                  unsigned inlen, void *);

  // 新增方法
  // 获取npn状态机，当SSL握手完成后，需要设置接收到的明文数据由哪个状态机来处理
  // 例如https，在完成SSL握手之后，就交给HttpSessionAccept来接管
  Continuation *
  endpoint() const
  {
    return npnEndpoint;
  }
  // 新增方法
  // 这个方法没有被调用，在net_read_io中直接判断了成员的值。
  bool
  getSSLClientRenegotiationAbort() const
  {
    return sslClientRenegotiationAbort;
  };
  // 新增方法
  // 在握手完成后，如果收到了Client发起的Renegotiation（SSL会话重新协商请求）时，
  // 判断 ssl_allow_client_renegotiation 的值来决定是否调用此方法来设置一个Abort状态，以关闭此SSL连接。
  void
  setSSLClientRenegotiationAbort(bool state)
  {
    sslClientRenegotiationAbort = state;
  };
  // 新增方法
  // 返回成员transparentPassThrough
  bool
  getTransparentPassThrough() const
  {
    return transparentPassThrough;
  };
  // 新增方法
  // 当ATS的端口被设置为 tr-pass 类型时，transparentPassThrough为true
  void
  setTransparentPassThrough(bool val)
  {
    transparentPassThrough = val;
  };
  // 新增方法
  // 设置一个session_accept的指针，但是这个指针没有在SSL中使用
  void
  set_session_accept_pointer(SessionAccept *acceptPtr)
  {
    sessionAcceptPtr = acceptPtr;
  };
  // 新增方法
  // 返回成员sessionAcceptPtr
  SessionAccept *
  get_session_accept_pointer(void) const
  {
    return sessionAcceptPtr;
  };

  // Copy up here so we overload but don't override
  // 复用UnixNetVConnection::reenable(VIO *vio)
  using super::reenable;

  // Reenable the VC after a pre-accept or SNI hook is called.
  // 注意这里的reenable与上面的不同，因此SSLNetVConnection::reenable()是多态的
  virtual void reenable(NetHandler *nh);
  
  // Set the SSL context.
  // @note This must be called after the SSL endpoint has been created.
  // 设置SSL的上下文
  virtual bool sslContextSet(void *ctx);

  /// Set by asynchronous hooks to request a specific operation.
  TSSslVConnOp hookOpRequested;

  // 直接从socket fd读取未解密的原始数据，放入handShakeBuffer这个MIOBuffer
  int64_t read_raw_data();
  
  // 初始化handShakeBuffer这个MIOBuffer，以及对应的Reader: handShakeReader 和 handShakeHolder
  //     handShakeBioStored 用来表示当前handShakeBuffer内可消费的数据长度
  void
  initialize_handshake_buffers()
  {
    this->handShakeBuffer = new_MIOBuffer();
    this->handShakeReader = this->handShakeBuffer->alloc_reader();
    this->handShakeHolder = this->handShakeReader->clone();
    this->handShakeBioStored = 0;
  }
  // 释放handShakeBuffer这个MIOBuffer，还有对应的Reader
  void
  free_handshake_buffers()
  {
    if (this->handShakeReader) {
      this->handShakeReader->dealloc();
    }
    if (this->handShakeHolder) {
      this->handShakeHolder->dealloc();
    }
    if (this->handShakeBuffer) {
      free_MIOBuffer(this->handShakeBuffer);
    }
    this->handShakeReader = NULL;
    this->handShakeHolder = NULL;
    this->handShakeBuffer = NULL;
    this->handShakeBioStored = 0;
  }
  
  // Returns true if all the hooks reenabled
  // 回调SSL的Hook，返回true表示所有hooks都执行完了，允许进入下一个流程
  bool callHooks(TSHttpHookID eventId);

  // Returns true if we have already called at
  // least some of the hooks
  // 是否已经开始了对Hook的回调过程
  bool calledHooks(TSHttpHookID /* eventId */) { return (this->sslHandshakeHookState != HANDSHAKE_HOOKS_PRE); }

  // 获取 iobuf
  // iobuf 用来保存解密后的数据，对应的时VIO里的MIOBuffer
  MIOBuffer *
  get_ssl_iobuf()
  {
    return iobuf;
  }

  // 设置 iobuf
  void
  set_ssl_iobuf(MIOBuffer *buf)
  {
    iobuf = buf;
  }
  // 这个reader是对应iobuf的MIOBufferReader
  IOBufferReader *
  get_ssl_reader()
  {
    return reader;
  }

  // 如果遇到了SSL关闭的情况，设置eosRcvd为true
  // 此方法返回这个状态
  bool
  isEosRcvd()
  {
    return eosRcvd;
  }

  // 获取sslTrace状态
  // 这个是一个debug调试SSL的开关
  bool
  getSSLTrace() const
  {
    return sslTrace || super::origin_trace;
  };
  // 设置sslTrace状态
  void
  setSSLTrace(bool state)
  {
    sslTrace = state;
  };

  // 根据特定条件激活sslTrace
  // 例如需要对特定ip，特定域名进行SSL调试的时候
  bool computeSSLTrace();

  // 获得SSL会话的协议版本
  const char *
  getSSLProtocol(void) const
  {
    if (ssl == NULL)
      return NULL;
    return SSL_get_version(ssl);
  };

  // 获得SSL会话的密钥算法套件
  const char *
  getSSLCipherSuite(void) const
  {
    if (ssl == NULL)
      return NULL;
    return SSL_get_cipher_name(ssl);
  }

  /**
   * Populate the current object based on the socket information in in the
   * con parameter and the ssl object in the arg parameter
   * This is logic is invoked when the NetVC object is created in a new thread context
   */
  // 跟UnixNetVConnection里的一样，这个也是2015年10月新增的方法
  virtual int populate(Connection &con, Continuation *c, void *arg);

private:
  SSLNetVConnection(const SSLNetVConnection &);
  SSLNetVConnection &operator=(const SSLNetVConnection &);

  // true = SSL握手完成设置
  bool sslHandShakeComplete;
  // true = 这是一个Client与ATS之间的SSL连接
  bool sslClientConnection;
  // true = 重新协商时被设置禁止，SSL会话讲终止
  bool sslClientRenegotiationAbort;
  // true = 本次TCP连接的SSL会话，是复用了之前的SSL会话
  bool sslSessionCacheHit;
  // 用于SSL握手过程中，存放临时数据的MIOBuffer
  MIOBuffer *handShakeBuffer;
  // handShakeBuffer的reader
  IOBufferReader *handShakeHolder;
  IOBufferReader *handShakeReader;
  // handShakeBuffer内数据的长度
  int handShakeBioStored;

  // true = tr-pass
  bool transparentPassThrough;

  /// The current hook.
  /// @note For @C SSL_HOOKS_INVOKE, this is the hook to invoke.
  // 当前hook点
  class APIHook *curHook;

  // 此处用于标记PreAccept Hook的执行状态
  enum {
    SSL_HOOKS_INIT,     ///< Initial state, no hooks called yet.
                        ///< 初始状态，没有任何hook被调用过
    SSL_HOOKS_INVOKE,   ///< Waiting to invoke hook.
                        ///< 等待对hook的回调，通常在当前hook点没有返回reenable的时候，就停在这个状态
    SSL_HOOKS_ACTIVE,   ///< Hook invoked, waiting for it to complete.
                        ///< 这个没有实现
    SSL_HOOKS_CONTINUE, ///< All hooks have been called and completed
                        ///< 这个也没有实现
    SSL_HOOKS_DONE      ///< All hooks have been called and completed
                        ///< 表示PreAccept Hook执行完了
  } sslPreAcceptHookState;

  // 此处用于标记SNI/CERT Hook的执行状态
  enum SSLHandshakeHookState {
    HANDSHAKE_HOOKS_PRE,     ///< 初始状态，没有任何
    HANDSHAKE_HOOKS_CERT,    ///< 中间状态，只在openssl回调cert callback func的时候，存在
    HANDSHAKE_HOOKS_POST,    ///< 未实现
    HANDSHAKE_HOOKS_INVOKE,  ///< 等待对hook的回调，通常在当前hook点没有返回reenable的时候，就停在这个状态
    HANDSHAKE_HOOKS_DONE     ///< 表示SNI/CERT Hook执行完了
  } sslHandshakeHookState;

  // npnSet 保存了所有被支持协议的 Name String 和 SessionAccept 入口
  const SSLNextProtocolSet *npnSet;
  // 在 SSLaccept() 握手成功后，使用下面的方法进行 NPN/ALPN 协商判断：
  //     this->npnEndpoint = this->npnSet->findEndpoint(proto, len);
  // npnEndpoint 会被设置为协商成功的那个协议的 SessionAccept 入口
  //     如果协商失败则被设置为 NULL
  Continuation *npnEndpoint;
  // 这个SessionAccept好像没用到？
  SessionAccept *sessionAcceptPtr;
  // iobuf 和对应的 reader，实际上是蹦床中 iobuf 的重复，
  // 由于抽象的不够好，导致需要在 SSLNetVConnection 中定义这个成员
  // 而且在上层状态机，如：HttpSM，SpdySM，H2SM中都需要对NetVC进行判断，
  //     如果是sslvc类型则需要从sslvc中获取iobuf
  //     如果是unixnetvc类型则使用从蹦床传递进来的iobuf
  MIOBuffer *iobuf;
  IOBufferReader *reader;
  // 是否接收到了eos
  bool eosRcvd;
  // 是否开启了sslTrace
  bool sslTrace;
};

typedef int (SSLNetVConnection::*SSLNetVConnHandler)(int, void *);

extern ClassAllocator<SSLNetVConnection> sslNetVCAllocator;
```

## SSL/TLS 实现简介

理论上来说，SSL/TLS 是构建在 TCP 协议之上，因此也可以使用类似 HttpSM 的设计方式，实现 SSL/TLS 的握手和数据加解密的过程。

但是这样的实现，就导致了从 IOCoreNet 到 HttpSM 是一层关系，而从 IOCoreNet 到 SSL/TLS 再到 HttpSM，从而实现 https 的时候，是两层关系。

这样在实现 HttpSM 的时候，就要考虑前面还有一个同层次的上层状态机，而不能直接操作 VIO。

事实上在 IOCoreNet 到 SSL/TLS 再到 HttpSM 的实现方式中，SSL/TLS 更像是一个 TransformVC，在后面我们介绍 TransformVC 时，就会了解到，让 HttpSM 同时兼容 NetVC 和 TransformVC 是很困难的。

因此，在 ATS 对 SSL/TLS 的实现里，选择了把 SSL/TLS 作为 IOCore 的一部分，而不是把 SSL/TLS 放到上层状态机里。

这样的好处，可以让 HttpSM 在处理 HTTPS 协议时使用 SSLNetVConnection，与处理 HTTP 协议时使用 UnixNetVConnection 是完全一样的。

于是 SSLNetVConnection 就诞生了，但是 SSL/TLS 的处理实际上是一个子状态机，它有两个状态：握手状态和数据通信状态。在 UnixNetVConnection 上重载、扩展了一堆的方法来实现这个子状态机。

由于是子状态机，就要有一个切入点，这个切入点，就在：

  - vc->net_read_io
  - write_to_net(vc), write_to_net_io(vc)
    - vc->load_buffer_and_write

还记得 NetHandler::mainNetEvent 部分是如何调用上面的两个方法的吗？

```
  // UnixNetVConnection *
  while ((vc = read_ready_list.dequeue())) {
    if (vc->closed)
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->read.enabled && vc->read.triggered)
      vc->net_read_io(this, trigger_event->ethread);
    else if (!vc->read.enabled) {
      read_ready_list.remove(vc);
    }
  }
  while ((vc = write_ready_list.dequeue())) {
    if (vc->closed)
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->write.enabled && vc->write.triggered)
      write_to_net(this, vc, trigger_event->ethread);
    else if (!vc->write.enabled) {
      write_ready_list.remove(vc);
    }
  }
```

以下是 UnixNetVConnection的读写过程：

  - 在读取 socket fd 到 MIOBuffer 的时候，调用的：
    - vc->net_read_io(this, trigger_event->ethread);
    - 这是一个 netvc 的成员方法
    - 事实上是直接调用了 read_from_net(nh, this, lthread);
    - 这不是一个 netvc 的成员方法
    - 在 read_from_net 中调用了 read/readv 方法实现了读取操作
  - 但是在将 MIOBuffer 写入 socket fd 的时候，调用的：
    - write_to_net(this, vc, trigger_event->ethread);
    - 这不是一个 netvc 的成员方法
    - 事实上是直接调用了 write_to_net_io(nh, vc, thread);
    - 在 write_to_net_io 中调用了vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);
    - 这是一个 netvc 的成员方法
    - 在 vc->load_buffer_and_write 中调用了 write/writev 方法实现了读取操作

那么 SSLNetVConnection 的读写过程呢：

  - 在读取 socket fd 到 MIOBuffer 的时候，调用的：
    - vc->net_read_io(this, trigger_event->ethread);
      - 这是一个 netvc 的成员方法，但是在sslvc中被重载了
    - 在 sslvc->net_read_io 中调用了 ssl_read_from_net(this, lthread, r);
      - 这不是一个 netvc 的成员方法
    - 在 ssl_read_from_net 中调用了 SSLReadBuffer 方法
    - 在 SSLReadBuffer 中调用了 OpenSSL API 的 SSL_read(ssl, buf, (int)nbytes); 实现了读取并解密的操作
  - 在将 MIOBuffer 写入 socket fd 的时候，调用的：
    - write_to_net(this, vc, trigger_event->ethread);
      - 这不是一个 netvc 的成员方法
      - 事实上是直接调用了 write_to_net_io(nh, vc, thread);
    - 在 write_to_net_io 中调用了vc->load_buffer_and_write(towrite, wattempted, total_written, buf, needs);
      - 这是一个 netvc 的成员方法，但是在sslvc中被重载了
    - 在 sslvc->load_buffer_and_write 中调用了 SSLWriteBuffer 方法
    - 在 SSLWriteBuffer 中调用了 OpenSSL API 的 SSL_write(ssl, buf, (int)nbytes); 实现了发送并加密的操作

我们看到：

  - 数据读取过程，基本都被 SSLNetVConnection 重载过了
  - 而数据发送过程则只有后半部分被重载

而 SSL/TLS 的握手过程则在 net_read_io 和 write_to_net_io 中进行了处理：

  - write_to_net_io 不是 netvc 的成员函数，无法被 sslvc 重载，但是在编写时已经对 SSLVC 做了一个判断
  - 当 OpenSSL 遇到 WANT_WRITE 等错误状态时，就需要等待 socket fd 可写

由于 SSL/TLS 的握手过程和数据传输过程是完全不同的，所以：

  - 握手过程实际上需要一个状态机来处理
  - 数据传输过程/加解密过程，则是通过类似 TransformVC 的方式来完成
  - 为了集成上述两种实现，SSL/TLS 设计的真的非常复杂？巧妙？混乱？
  - 反正就是各种难懂、别扭就对了，看不懂就多看两遍，仔细推敲吧

## 方法

### net_read_io(NetHandler *nh, EThread *lthread)

net_read_io() 是 SSLNetVConnection 的成员方法，包含了 SSL握手 和 SSL传输 两个功能的代码。

在 UnixNetVConnection 中，net_read_io 则直接调用了独立函数 read_from_net(nh, this, lthread) 。

```
// changed by YTS Team, yamsat
void
SSLNetVConnection::net_read_io(NetHandler *nh, EThread *lthread)
{
  int ret;
  int64_t r = 0;
  int64_t bytes = 0;
  NetState *s = &this->read;

  // 如果是 blind tunnel，那么就使用 UnixNetVConnection::net_read_io 替代
  // 相当于保持一致，不进行 SSL 处理，只做TCP Proxy转发
  if (HttpProxyPort::TRANSPORT_BLIND_TUNNEL == this->attributes) {
    this->super::net_read_io(nh, lthread);
    return;
  }

  // 尝试获取对此VC的VIO的Mutex锁
  MUTEX_TRY_LOCK_FOR(lock, s->vio.mutex, lthread, s->vio._cont);
  if (!lock.is_locked()) {
    // 如果没有拿到锁，就重新调度，等下一次再读取
    // read_reschedule 会把此vc放回到read_ready_link的队列尾部
    // 由于NetHandler的处理方式，必然会在本次EventSystem回调NetHandler中完成所有的读操作
    //     不会延迟到下一次EventSystem对NetHandler的回调
    readReschedule(nh);
    return;
  }
  // Got closed by the HttpSessionManager thread during a migration
  // The closed flag should be stable once we get the s->vio.mutex in that case
  // (the global session pool mutex).
  // 拿到锁之后，首先判断该vc是否被异步关闭了，因为 HttpSessionManager 会管理一个全局的session池。
  if (this->closed) {
    // 这里为何不直接调用 close_UnixNetVConnection() ?
    // 通过调用 UnixNetVConnection::net_read_io 来间接调用 close_UnixNetVConnection() 关闭 sslvc
    this->super::net_read_io(nh, lthread);
    return;
  }
  // If the key renegotiation failed it's over, just signal the error and finish.
  // 出现这个状态是在配置里，禁止了 SSL Renegotiation，但是客户端又发起了 SSL Renegotiation 请求
  if (sslClientRenegotiationAbort == true) {
    // 此时直接报错，然后关闭 sslvc
    this->read.triggered = 0;
    readSignalError(nh, (int)r);
    Debug("ssl", "[SSLNetVConnection::net_read_io] client renegotiation setting read signal error");
    return;
  }

  // If it is not enabled, lower its priority.  This allows
  // a fast connection to speed match a slower connection by
  // shifting down in priority even if it could read.
  // 再判断，该vc的读操作是否被异步禁止了
  if (!s->enabled || s->vio.op != VIO::READ) {
    read_disable(nh, this);
    return;
  }
  
  // 下面才开始进入读操作前的准备工作
  // 获取缓冲区，准备写入数据
  MIOBufferAccessor &buf = s->vio.buffer;
  // 在VIO中定义了总共需要读取的数据的总长度，还有已经完成的数据读取长度
  // ntodo 是剩余的，还需要读取的数据长度
  int64_t ntodo = s->vio.ntodo();
  ink_assert(buf.writer());

  // Continue on if we are still in the handshake
  // 前面讲过，SSL/TLS 握手过程是一个子状态机，就嵌入到了 net_read_io 和 write_to_net_io 里面
  // getSSLHandShakeComplete() 返回的是成员 sslHandShakeComplete，表示是否完成了SSL握手过程
  if (!getSSLHandShakeComplete()) {
    // 如果没有完成SSL握手过程，那么就进入SSL握手处理过程
    int err;

    // getSSLClientConnection() 返回的是成员 sslClientConnection，表示这是否是一个由ATS发起的SSL连接
    // 如果由ATS发起，那么ATS就是SSL Client，否则ATS就是SSL Server
    // 调用 sslStartHandShake 来进行握手，传递握手方式，指明这是ATS作为 Client 还是 Server 的握手过程
    if (getSSLClientConnection()) {
      // ATS 作为 SSL Client
      ret = sslStartHandShake(SSL_EVENT_CLIENT, err);
    } else {
      // ATS 作为 SSL Server
      // 会设置 this->attributes 的属性，例如 Blind Tunnel
      ret = sslStartHandShake(SSL_EVENT_SERVER, err);
    }
    // If we have flipped to blind tunnel, don't read ahead
    // 下面的部分主要是判断对Blind Tunnel的处理
    if (this->handShakeReader && this->attributes != HttpProxyPort::TRANSPORT_BLIND_TUNNEL) {
      // 如果不是Blind Tunnel，而且 MIOBuffer *handShakeBuffer 关联的 IOBufferReader *handShakeReader 不为空
      //     就需要对 handShakeBuffer 中的数据进行处理
      // Check and consume data that has been read
      if (BIO_eof(SSL_get_rbio(this->ssl))) {
        this->handShakeReader->consume(this->handShakeBioStored);
        this->handShakeBioStored = 0;
      }
    } else if (this->attributes == HttpProxyPort::TRANSPORT_BLIND_TUNNEL) {
      // 如果是Blind Tunnel，那就需要透传数据，不按照SSL连接来处理
      //     dest_ip=1.2.3.4 action=tunnel ssl_cert_name=servercert.pem ssl_key_name=privkey.pem
      // Now in blind tunnel. Set things up to read what is in the buffer
      // Must send the READ_COMPLETE here before considering
      // forwarding on the handshake buffer, so the
      // SSLNextProtocolTrampoline has a chance to do its
      // thing before forwarding the buffers.
      // 此时SSLVC关联的状态机还是 SSLNextProtocolTrampoline，
      //     因此下面这个READ_COMPLETE是传递给 SSLNextProtocolTrampoline::ioCompletionEvent
      //     通过蹦床重新为 SSLVC 设置状态机，然后 SSLNextProtocolTrampoline 被 delete
      //     对于Blind Tunnel，状态机是HttpSM
      this->readSignalDone(VC_EVENT_READ_COMPLETE, nh);

      // If the handshake isn't set yet, this means the tunnel
      // decision was make in the SNI callback.  We must move
      // the client hello message back into the standard read.vio
      // so it will get forwarded onto the origin server
      if (!this->getSSLHandShakeComplete()) {
        // 如果没有完成SSL握手，就强制设置为已经完成
        this->sslHandShakeComplete = 1;

        // Copy over all data already read in during the SSL_accept
        // (the client hello message)
        // 然后把接收到客户端发送的原始的SSL数据（Raw Data）复制到 SSLVC 的 Read VIO 中
        NetState *s = &this->read;
        MIOBufferAccessor &buf = s->vio.buffer;
        int64_t r = buf.writer()->write(this->handShakeHolder);
        s->vio.nbytes += r;
        s->vio.ndone += r;

        // Clean up the handshake buffers
        // 然后释放掉 SSL 的 MIOBuffer *handShakeBuffer 以及关联的 handShakeReader 和 handShakeHolder
        this->free_handshake_buffers();

        // 如果 handShakeBuffer 中是有数据的，那么上面就执行了数据的复制，Read VIO 中就有了新数据
        //     此时就要向上层状态机发送一个 READ_COMPLETE 的事件。
        if (r > 0) {
          // Kick things again, so the data that was copied into the
          // vio.read buffer gets processed
          this->readSignalDone(VC_EVENT_READ_COMPLETE, nh);
        }
      }
      // 由于是 Blind Tunnel，在设置SSL握手状态为完成后，直接就返回
      //     后面就转为Tunnel的处理，纯TCP的转发
      return;
    }
    
    // 根据 sslStartHandShake() 的返回值进行相应的处理
    // 需要考虑 ATS 作为 Server 或者 Client 的两种情况
    if (ret == EVENT_ERROR) {
      // 错误处理，直接关闭VC，回调蹦床 ioCompletionEvent 传递 VC_EVENT_ERROR 事件类型，lerror = err;
      // err 的值在 sslStartHandShake 中设置，通常为 syscall 或者 openssl 库的错误代码。
      this->read.triggered = 0;
      readSignalError(nh, err);
    } else if (ret == SSL_HANDSHAKE_WANT_READ || ret == SSL_HANDSHAKE_WANT_ACCEPT) {
      // 为了完成握手过程，需要读取更多数据
      if (SSLConfigParams::ssl_handshake_timeout_in > 0) {
        // 如果设置了 SSL 握手超时时间
        double handshake_time = ((double)(Thread::get_hrtime() - sslHandshakeBeginTime) / 1000000000);
        Debug("ssl", "ssl handshake for vc %p, took %.3f seconds, configured handshake_timer: %d", this, handshake_time,
              SSLConfigParams::ssl_handshake_timeout_in);
        if (handshake_time > SSLConfigParams::ssl_handshake_timeout_in) {
          // 发现超时，同样进行错误处理，然后关闭连接，但是固定传递 EOS 的状态
          Debug("ssl", "ssl handshake for vc %p, expired, release the connection", this);
          read.triggered = 0;
          nh->read_ready_list.remove(this);
          readSignalError(nh, VC_EVENT_EOS);
          return;
        }
      }
      // 从 NetHandler 的队列里清除 SSLVC，等下一次有数据时再继续处理
      // 注意跟 disable 的区别，disable 会同时从 epoll 里删除 vc 对应的 socket fd
      // 而这里只是在本次 NetHandler 的处理循环中跳过此 SSLVC，等下一次 epoll_wait 再次触发
      read.triggered = 0;
      nh->read_ready_list.remove(this);
      readReschedule(nh);
    } else if (ret == SSL_HANDSHAKE_WANT_CONNECT || ret == SSL_HANDSHAKE_WANT_WRITE) {
      // 为了完成握手过程，需要发送更多数据
      // 通常是当前缓冲区满了，因此等待下一次缓冲区可写时再继续发送
      // 同样要注意跟 disable 的区别
      write.triggered = 0;
      nh->write_ready_list.remove(this);
      writeReschedule(nh);
    } else if (ret == EVENT_DONE) { // 出现此情况的时候，要进行深入的判断
      // If this was driven by a zero length read, signal complete when
      // the handshake is complete. Otherwise set up for continuing read
      // operations.
      // 原文注释翻译：如果这是由 0 长度读操作驱动的一次调用，
      //     EVENT_DONE 表示完成了握手过程，需要向蹦床 ioCompletionEvent 传递 READ_COMPLETE 事件。
      //     否则，需要继续读取数据，那么 EVENT_DONE 则只是表示没有错误。
      
      // ntodo 表示剩余需要读取的字节数，这里我觉得应该用 nbytes 来判断是否是 0 长度的读操作 ？？？ TS-4216
      if (ntodo <= 0) {
        // 进入 0 长度读操作驱动的调用处理过程
        if (!getSSLClientConnection()) {
          // 当此 SSLVC 对于 ATS 是 SSL Server 端时
          // 由于在 accept 上设置了 TCP_DEFER_ACCEPT 属性，因此对于一个新 Accept 的连接，需要判断是否有新数据。
          // we will not see another ET epoll event if the first byte is already
          // in the ssl buffers, so, SSL_read if there's anything already..
          // 因此要直接做一次读取操作，来试试看是否有新数据
          Debug("ssl", "ssl handshake completed on vc %p, check to see if first byte, is already in the ssl buffers", this);
          // 创建 iobuf 用于接收数据
          this->iobuf = new_MIOBuffer(BUFFER_SIZE_INDEX_4K);
          if (this->iobuf) {
            // 创建 iobuf 成功，分配reader
            this->reader = this->iobuf->alloc_reader();
            // 设置 Read VIO 内的 buffer 为 iobuf
            // 使用writer_for的方法设置之后，再向vio.buffer写入数据时，实际是写入了 this->iobuf 内。
            //     同时，vio.buffer 是不可读的，如果想要读取 this->iobuf 内的数据，需要通过 this->reader 完成。
            // 这里的iobuf在之前指向：SSLNextProtocolAccept::buffer（为了实现设置 0 长度读取操作）
            //     SSLNextProtocolAccept::buffer 是不可以写数据的，因此这里要切换成新分配的 MIOBuffer
            s->vio.buffer.writer_for(this->iobuf);
            // 进行数据读取操作（因为有 TCP_DEFER_ACCEPT）：
            //     从 SSLVC 的 socket 上读取数据，然后通过 vio.buffer 将数据写入 this->iobuf
            ret = ssl_read_from_net(this, lthread, r);
            // 读操作可能会遇到 SSL EOS 的情况，需要做一个判断，例如握手的加密算法不支撑，协商失败等。
            if (ret == SSL_READ_EOS) {
              this->eosRcvd = true;
            }
#if DEBUG
            int pending = SSL_pending(this->ssl);
            if (r > 0 || pending > 0) {
              Debug("ssl", "ssl read right after handshake, read %" PRId64 ", pending %d bytes, for vc %p", r, pending, this);
            }
#endif
          } else {
            Error("failed to allocate MIOBuffer after handshake, vc %p", this);
          }
          // 这里虽然做了 disable，但是下面紧跟着回调蹦床的时候，会再次重新设置 VIO
          read.triggered = 0;
          read_disable(nh, this);
        }
        // 回调蹦床 ioCompletionEvent 传递 READ_COMPLETE 事件，表示完成了握手过程
        // 此时会由蹦床回调上层状态机，上层状态机会重新设置 VIO，激活此 SSLVC
        readSignalDone(VC_EVENT_READ_COMPLETE, nh);
      } else {
        // 这不是一次 0 长度读操作驱动的调用，需要继续读取数据，在 NetHandler 中激活该 NetVC
        read.triggered = 1;
        if (read.enabled)
          nh->read_ready_list.in_or_enqueue(this);
      }
    } else if (ret == SSL_WAIT_FOR_HOOK) {
      // avoid readReschedule - done when the plugin calls us back to reenable
      // 当 plugin 回调 reenable 的时候，避免调用readReschedule
      // 在后面讲解 SSL Hook 的时候，再详细分析这块。
    } else {
      // 其它情况，调用readReschedule
      // 底层调用 read_reschedule(nh, vc); 实现
      //     当 read.triggered == 1 && read.enabled == 1 的时候
      //     调用 nh->read_ready_list.in_or_enqueue(vc);
      readReschedule(nh);
    }
    // 在SSL握手处理过程中，直接返回 NetHandler
    return;
  }
  // 如果SSL握手已经完成
  // 下面的部分就与 UnixNetVConnection::net_read_io 差不多了
  //     只不过这个是调用 ssl_read_from_net 代替 read

  // If there is nothing to do or no space available, disable connection
  // 如果VIO被关闭或者Read VIO的MIOBuffer被写满了，就禁止读
  if (ntodo <= 0 || !buf.writer()->write_avail()) {
    read_disable(nh, this);
    return;
  }

  // At this point we are at the post-handshake SSL processing
  // If the read BIO is not already a socket, consider changing it
  // 握手完成后：
  //     需要把Read BIO从内存块类型切换为Socket类型
  if (this->handShakeReader) {
    // Check out if there is anything left in the current bio
    // 查看内存块类型的Read BIO内是否还有数据
    if (!BIO_eof(SSL_get_rbio(this->ssl))) {
      // BIO_eof() 在遇到EOF时返回 1
      // Still data remaining in the current BIO block
      // 仍然有数据，那么 handshake buffer 应该已经跟当前 SSL会话的 Read BIO 绑定了
      // 等后面调用 ssl_read_from_net 时，就会通过 Read BIO 来消费 handshake buffer 内的数据
    } else {
      // Read BIO内已经被读空了，
      // 此处把handShakeBioStored记录的上一次填充到Read BIO的字节从handShakeReader中消费掉
      // Consume what SSL has read so far.
      this->handShakeReader->consume(this->handShakeBioStored);

      // If we are empty now, switch over
      // 消费完成之后，再看一下handShakeReader中是否还有数据
      if (this->handShakeReader->read_avail() <= 0) {
        // handShakeReader中也没有数据了，那么就可以直接切换Read BIO为Socket类型
        // Switch the read bio over to a socket bio
        SSL_set_rfd(this->ssl, this->get_socket());
        // 然后释放handShake buffers，会将 handShakeReader 设置为 NULL
        this->free_handshake_buffers();
      } else {
        // 如果 handShakeReader 中还有可读取的数据，
        // 那么就要利用剩余的数据，再创建一个内存块类型的 Read BIO 并且绑定到SSL会话上
        // Setup the next iobuffer block to drain
        char *start = this->handShakeReader->start();
        char *end = this->handShakeReader->end();
        this->handShakeBioStored = end - start;

        // Sets up the buffer as a read only bio target
        // Must be reset on each read
        BIO *rbio = BIO_new_mem_buf(start, this->handShakeBioStored);
        // 下面这个 set_mem_eof_return 的详细解读可以参考：
        //     https://www.openssl.org/docs/faq.html#PROG15
        //     https://www.openssl.org/docs/manmaster/crypto/BIO_s_mem.html
        // 其实我没弄明白......
        BIO_set_mem_eof_return(rbio, -1);
        SSL_set_rbio(this->ssl, rbio);
      }
    }
  }
  // Otherwise, we already replaced the buffer bio with a socket bio

  // not sure if this do-while loop is really needed here, please replace
  // this comment if you know
  do {
    // 调用 ssl_read_from_net 读取并解密数据，
    // 这里读取的数据可能来自内存块BIO也可以来自Socket BIO
    ret = ssl_read_from_net(this, lthread, r);
    if (ret == SSL_READ_READY || ret == SSL_READ_ERROR_NONE) {
      bytes += r;
    }
    ink_assert(bytes >= 0);
  } while ((ret == SSL_READ_READY && bytes == 0) || ret == SSL_READ_ERROR_NONE);

  // 如果读取到了数据，则回调 VC_EVENT_READ_READY 到上层状态机
  if (bytes > 0) {
    if (ret == SSL_READ_WOULD_BLOCK || ret == SSL_READ_READY) {
      if (readSignalAndUpdate(VC_EVENT_READ_READY) != EVENT_CONT) {
        Debug("ssl", "ssl_read_from_net, readSignal != EVENT_CONT");
        return;
      }
    }
  }

  // 对最后一次调用 ssl_read_from_net 的返回值进行判断
  switch (ret) {
  case SSL_READ_ERROR_NONE:
  case SSL_READ_READY:
    readReschedule(nh);
    return;
    break;
  case SSL_WRITE_WOULD_BLOCK:
  case SSL_READ_WOULD_BLOCK:
    if (lock.get_mutex() != s->vio.mutex.m_ptr) {
      Debug("ssl", "ssl_read_from_net, mutex switched");
      if (ret == SSL_READ_WOULD_BLOCK)
        readReschedule(nh);
      else
        writeReschedule(nh);
      return;
    }
    // reset the trigger and remove from the ready queue
    // we will need to be retriggered to read from this socket again
    read.triggered = 0;
    nh->read_ready_list.remove(this);
    Debug("ssl", "read_from_net, read finished - would block");
#ifdef TS_USE_PORT
    if (ret == SSL_READ_WOULD_BLOCK)
      readReschedule(nh);
    else
      writeReschedule(nh);
#endif
    break;

  case SSL_READ_EOS:
    // close the connection if we have SSL_READ_EOS, this is the return value from ssl_read_from_net() if we get an
    // SSL_ERROR_ZERO_RETURN from SSL_get_error()
    // SSL_ERROR_ZERO_RETURN means that the origin server closed the SSL connection
    eosRcvd = true;
    read.triggered = 0;
    readSignalDone(VC_EVENT_EOS, nh);

    if (bytes > 0) {
      Debug("ssl", "read_from_net, read finished - EOS");
    } else {
      Debug("ssl", "read_from_net, read finished - 0 useful bytes read, bytes used by SSL layer");
    }
    break;
  case SSL_READ_COMPLETE:
    readSignalDone(VC_EVENT_READ_COMPLETE, nh);
    Debug("ssl", "read_from_net, read finished - signal done");
    break;
  case SSL_READ_ERROR:
    this->read.triggered = 0;
    readSignalError(nh, (int)r);
    Debug("ssl", "read_from_net, read finished - read error");
    break;
  }
}
```

### ssl_read_from_net

ssl_read_from_net 并不是 sslvc 的成员方法：

  - 它是对 SSLReadBuffer 的封装，
    - 而 SSLReadBuffer 又是对 OpenSSL API 函数 SSL_read 的封装

```
static int
ssl_read_from_net(SSLNetVConnection *sslvc, EThread *lthread, int64_t &ret)
```

ssl_read_from_net 的返回值是直接对 SSL_read 返回值的映射：

|           SSL_read           |        ssl_read_from_net        |
|:----------------------------:|:-------------------------------:|
|  SSL_ERROR_NONE              |  SSL_READ_ERROR_NONE(0)         |
|  SSL_ERROR_SYSCALL           |  SSL_READ_ERROR(1)              |
|                              |  SSL_READ_READY(2)              |
|                              |  SSL_READ_COMPLETE(3)           |
|  SSL_ERROR_WANT_READ         |  SSL_READ_WOULD_BLOCK(4)        |
|  SSL_ERROR_WANT_X509_LOOKUP  |  SSL_READ_WOULD_BLOCK(4)        |
|  SSL_ERROR_SYSCALL           |  SSL_READ_EOS(5)                |
|  SSL_ERROR_ZERO_RETURN       |  SSL_READ_EOS(5)                |
|  SSL_ERROR_WANT_WRITE        |  SSL_WRITE_WOULD_BLOCK(10)      |

ssl_read_from_net 的调用者需要对上面的返回值进行处理：

  - SSL_READ_WOULD_BLOCK 和 SSL_WRITE_WOULD_BLOCK
    - 表示需要等待下一次 NetHandler 回调
  - SSL_READ_ERROR_NONE 和 SSL_READ_READY
    - 表示已经成功读取了一些数据，但是Kernel TCP/IP Buffer里还有数据，还可以继续读
  - SSL_READ_COMPLETE
    - 表示 VIO 设定的数据读取长度已经完成
  - SSL_READ_EOS
    - 表示连接中断了
  - SSL_READ_ERROR
    - 表示读取操作遇到了错误

在整个SSL的实现里，还使用了下面这些映射：

|       SSL_accept/connect     |  sslServer/ClientHandShakeEvent |
|:----------------------------:|:-------------------------------:|
|  SSL_ERROR_WANT_READ         |  SSL_HANDSHAKE_WANT_READ(6)     |
|  SSL_ERROR_WANT_WRITE        |  SSL_HANDSHAKE_WANT_WRITE(7)    |
|  SSL_ERROR_WANT_ACCEPT       |  SSL_HANDSHAKE_WANT_ACCEPT(8)   |
|  SSL_ERROR_WANT_CONNECT      |  SSL_HANDSHAKE_WANT_CONNECT(9)  |
|                              |  SSL_WAIT_FOR_HOOK(11)          |

这几个值在SSL握手过程中使用，上面所有这些值都是在 SSLNetVConnection.cc 的头部定义的宏。

### write_to_net_io (write_to_net)

write_to_net_io 并不是 sslvc 的成员方法，SSLNetVConnection 与 UnixNetVConnection 共享此函数。

在 write_to_net_io 中同时考虑了 NetVC 类型为 SSLNetVConnection 与 UnixNetVConnection 的两种情况。

但是在实际发送数据时则调用了 vc->load_buffer_and_write 来完成，以实现SSL会话上的数据加密传输。

在 CH02-IOCoreNet 的 CH02S09-Core-UnixNetVConnection 中的 [nethandler的延伸：从miobuffer到socket](https://github.com/oknet/atsinternals/blob/master/CH02-IOCoreNET/CH02S09-Core-UnixNetVConnection.md#nethandler的延伸从miobuffer到socket) 已经对 write_to_net_io 进行了分析，但是跳过了 SSL 的部分，下面就要翻回来继续看 SSL 部分的代码。

```
void
write_to_net_io(NetHandler *nh, UnixNetVConnection *vc, EThread *thread)
{
  // ...... 略去部分代码
  
  // This function will always return true unless
  // vc is an SSLNetVConnection.
  // 如果 vc 的类型是 SSLNetVConnection，那么返回 SSL 握手是否完成的状态值，
  // 如果 vc 的类型不是 SSLNetVConnection，那么返回值固定为 true，取反后就是 false 了。
  // 由于我们要分析的是 SSLVC，因此这里假定 vc 的类型是 SSLNetVConnection。
  if (!vc->getSSLHandShakeComplete()) {
    // 没有完成 SSL 握手
    int err, ret;

    // 确认 sslvc 的方向，是作为 SSL Client，还是 SSL Server？
    // 此处与 net_read_io 的部分一样
    if (vc->getSSLClientConnection())
      ret = vc->sslStartHandShake(SSL_EVENT_CLIENT, err);
    else
      ret = vc->sslStartHandShake(SSL_EVENT_SERVER, err);

    // 根据 sslStartHandShake() 的返回值进行相应的处理
    // 需要考虑 ATS 作为 Server 或者 Client 的两种情况
    if (ret == EVENT_ERROR) {
      // 遇到错误，关闭写，向上层状态机回调
      vc->write.triggered = 0;
      write_signal_error(nh, vc, err);
    } else if (ret == SSL_HANDSHAKE_WANT_READ || ret == SSL_HANDSHAKE_WANT_ACCEPT) {
      // 需要读取更多数据，但是缓冲区已经空了
      // 因此重新调度 read 操作，等 NetHandler::mainNetEvent 的下一次回调
      vc->read.triggered = 0;
      nh->read_ready_list.remove(vc);
      read_reschedule(nh, vc);
    } else if (ret == SSL_HANDSHAKE_WANT_CONNECT || ret == SSL_HANDSHAKE_WANT_WRITE) {
      // 需要发送更多数据，但是缓冲区已经满了
      // 因此重新调度 write 操作，等 NetHandler::mainNetEvent 的下一次回调
      vc->write.triggered = 0;
      nh->write_ready_list.remove(vc);
      write_reschedule(nh, vc);
    } else if (ret == EVENT_DONE) {
      // 完成了握手操作
      // 激活 write 操作，让数据传输的部分来判断Write VIO的情况
      // 但是需要注意，只有 triggered 和 enabled 都设置为 1 的时候，NetHandler::mainNetEvent 才会回调
      //       enabled 表示之前曾经调用过 do_io()
      //     triggered 表示 epoll_wait 发现此 vc 的 socket fd 上有数据活动
      vc->write.triggered = 1;
      if (vc->write.enabled)
        nh->write_ready_list.in_or_enqueue(vc);
    } else
      // 其它返回值，例如：SSL_WAIT_FOR_HOOK，SSL_READ_WOULD_BLOCK，SSL_WRITE_WOULD_BLOCK 等
      // 直接重新调度 write 操作，等 NetHandler::mainNetEvent 的下一次回调
      write_reschedule(nh, vc);
    return;
  }
  
  // ...... 略去部分代码
}
```

### load_buffer_and_write

load_buffer_and_write 的功能是将 buf 中的 towrite 字节数据通过 SSL 加密通道发送。

  - total_written 用来表示成功发送的字节数
  - wattempted 用来表示尝试发送的字节数
  - needs 表示在返回调用者后，调用者需要再次激活 read 和/或 write

返回值

  - 如果 towrite 字节全部成功发送，那么返回值等于 total_written
  - 如果 towrite 字节部分成功发送，那么：
    - 返回值大于0，表示 buf 中可发送的数据长度可能小于 towrite 字节
    - 返回值小于0，表示最后一次发送时返回的错误值

```
int64_t
SSLNetVConnection::load_buffer_and_write(int64_t towrite, int64_t &wattempted, int64_t &total_written, MIOBufferAccessor &buf,
                                         int &needs)
{
  ProxyMutex *mutex = this_ethread()->mutex;
  int64_t r = 0;
  int64_t l = 0;
  uint32_t dynamic_tls_record_size = 0;
  ssl_error_t err = SSL_ERROR_NONE;

  // XXX Rather than dealing with the block directly, we should use the IOBufferReader API.
  int64_t offset = buf.reader()->start_offset;
  IOBufferBlock *b = buf.reader()->block;

  // 在SSL发送的时候，是把一个明文数据块加密后发出，加密之后被叫做 TLS record。
  // 接收端必须收到一个完整的 TLS record 才能进行解密，但是受限于 TCP 连接的传输情况，
  //   每次能够发送的 TCP 报文大小是不同的，如果 TLS record 的长度需要跨越多个 TCP 报文，
  //   接收端就必须将多个 TCP 报文合并之后，得到一个完整的 TLS record 才可以进行解密。
  // 在 ATS 中采用了一种动态调整明文数据块大小的方法，让加密后的 TLS record 尽可能在一个 TCP 报文中传递，
  //   这样接收端每收到一个 TCP 报文，就可以得到一个完整的 TLS record，可以立即解密并获得明文内容。
  // Dynamic TLS record sizing
  ink_hrtime now = 0;
  if (SSLConfigParams::ssl_maxrecord == -1) {
    now = Thread::get_hrtime_updated();
    int msec_since_last_write = ink_hrtime_diff_msec(now, sslLastWriteTime);

    if (msec_since_last_write > SSL_DEF_TLS_RECORD_MSEC_THRESHOLD) {
      // reset sslTotalBytesSent upon inactivity for SSL_DEF_TLS_RECORD_MSEC_THRESHOLD
      sslTotalBytesSent = 0;
    }
    Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, now %" PRId64 ",lastwrite %" PRId64 " ,msec_since_last_write %d", now,
          sslLastWriteTime, msec_since_last_write);
  }

  // 判断 Blind Tunnel 的情况，通过 UnixNetVConnection::load_buffer_and_write() 执行
  //   感觉这里应该放在 Dyn TLS record sizing 之前处理会比较好啊。
  if (HttpProxyPort::TRANSPORT_BLIND_TUNNEL == this->attributes) {
    return this->super::load_buffer_and_write(towrite, wattempted, total_written, buf, needs);
  }

  // 用于 SSL 跟踪调试
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  do {
    // 准备用于发送的数据块，计算其长度
    // check if we have done this block
    l = b->read_avail();
    l -= offset;
    if (l <= 0) {
      offset = -l;
      b = b->next;
      continue;
    }
    // check if to amount to write exceeds that in this buffer
    int64_t wavail = towrite - total_written;

    if (l > wavail) {
      l = wavail;
    }

    // 仍然是为了 Dyn TLS record sizing
    // TS-2365: If the SSL max record size is set and we have
    // more data than that, break this into smaller write
    // operations.
    int64_t orig_l = l;
    if (SSLConfigParams::ssl_maxrecord > 0 && l > SSLConfigParams::ssl_maxrecord) {
      l = SSLConfigParams::ssl_maxrecord;
    } else if (SSLConfigParams::ssl_maxrecord == -1) {
      if (sslTotalBytesSent < SSL_DEF_TLS_RECORD_BYTE_THRESHOLD) {
        dynamic_tls_record_size = SSL_DEF_TLS_RECORD_SIZE;
        SSL_INCREMENT_DYN_STAT(ssl_total_dyn_def_tls_record_count);
      } else {
        dynamic_tls_record_size = SSL_MAX_TLS_RECORD_SIZE;
        SSL_INCREMENT_DYN_STAT(ssl_total_dyn_max_tls_record_count);
      }
      if (l > dynamic_tls_record_size) {
        l = dynamic_tls_record_size;
      }
    }

    if (!l) {
      break;
    }

    // 设置 wattempted 为本次期望发送的字节数
    wattempted = l;
    // 设置 total_writen 为本次成功发送后的总发送字节数
    //   这个设置好奇怪？？
    total_written += l;
    Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, before SSLWriteBuffer, l=%" PRId64 ", towrite=%" PRId64 ", b=%p", l,
          towrite, b);
    // 调用 SSLWriteBuffer 进行数据的加密和发送
    //   ssl 为SSL CTX描述符
    //   b->start() + offset 为准备发送的明文数据内容的起始地址
    //   l 为期望发送的字节数
    //   r 为成功发送的字节数
    //   当出现错误时，err为非0值
    err = SSLWriteBuffer(ssl, b->start() + offset, l, r);

    // 根据需要显示 SSL 的调试信息
    if (!origin_trace) {
      TraceOut((0 < r && trace), get_remote_addr(), get_remote_port(), "WIRE TRACE\tbytes=%d\n%.*s", (int)r, (int)r,
               b->start() + offset);
    } else {
      char origin_trace_ip[INET6_ADDRSTRLEN];
      ats_ip_ntop(origin_trace_addr, origin_trace_ip, sizeof(origin_trace_ip));
      TraceOut((0 < r && trace), get_remote_addr(), get_remote_port(), "CLIENT %s:%d\ttbytes=%d\n%.*s", origin_trace_ip,
               origin_trace_port, (int)r, (int)r, b->start() + offset);
    }

    if (r == l) {
      // 本次数据发送成功，设置 wattempted 为总发送字节数
      wattempted = total_written;
    }
    
    // 判断当前 IOBufferBlock 的发送是否受到 Dyn TLS record sizing 的影响而分片发送
    if (l == orig_l) {
      // 没有分片，则继续发送下一个 IOBufferBlock
      // on to the next block
      offset = 0;
      b = b->next;
    } else {
      // 出现分片，则继续发送当前 IOBufferBlock 剩余的部分
      offset += l;
    }

    Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite,Number of bytes written=%" PRId64 " , total=%" PRId64 "", r,
          total_written);
    NET_INCREMENT_DYN_STAT(net_calls_to_write_stat);
    // 如果：
    //   本轮发送成功：r == l
    //   没有完全全部发送任务：total_written < towrite
    //   还有承载数据的 IOBufferBlock：b
    // 那么就继续下一个循环
  } while (r == l && total_written < towrite && b);

  if (r > 0) {
    // 从 while 跳出时，没有遇到错误
    sslLastWriteTime = now;
    sslTotalBytesSent += total_written;
    if (total_written != wattempted) {
      // 没有完成所有的数据发送，此时 b==NULL
      // 需要回调上层状态机 WRITE_READY，重新填充数据，然后重新调度写操作，
      // 等待下一次NetHandler回调，继续完成数据的发送。
      Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, wrote some bytes, but not all requested.");
      // I'm not sure how this could happen. We should have tried and hit an EAGAIN.
      // 告知调用者（write_to_net_io），需要重新调度写操作
      needs |= EVENTIO_WRITE;
      // 返回最后一次成功发送的字节数
      return (r);
    } else {
      // 完成了全部的数据发送，此时 total_written >= towrite
      // !!! 这里需要注意 !!!
      //   如果IOBufferBlock中可用于发送的数据量大于 towrite 值，会出现 total_written > towrite 的情况
      //   这是bug？？
      Debug("ssl", "SSLNetVConnection::loadBufferAndCallWrite, write successful.");
      // 返回实际发送成功的字节数
      return (total_written);
    }
  } else {
    // 从 while 跳出时，遇到了错误
    // 根据最后一次调用 SSLWriteBuffer 的返回值，判断错误类型
    switch (err) {
    case SSL_ERROR_NONE:
      // 没有错误
      Debug("ssl", "SSL_write-SSL_ERROR_NONE");
      break;
    case SSL_ERROR_WANT_READ:
      // 需要读取数据
      needs |= EVENTIO_READ;
      r = -EAGAIN;
      SSL_INCREMENT_DYN_STAT(ssl_error_want_read);
      Debug("ssl.error", "SSL_write-SSL_ERROR_WANT_READ");
      break;
    case SSL_ERROR_WANT_WRITE:
      // 需要发送数据
    case SSL_ERROR_WANT_X509_LOOKUP: {
      // 需要查询X509
      if (SSL_ERROR_WANT_WRITE == err) {
        SSL_INCREMENT_DYN_STAT(ssl_error_want_write);
      } else if (SSL_ERROR_WANT_X509_LOOKUP == err) {
        SSL_INCREMENT_DYN_STAT(ssl_error_want_x509_lookup);
        TraceOut(trace, get_remote_addr(), get_remote_port(), "Want X509 lookup");
      }

      needs |= EVENTIO_WRITE;
      r = -EAGAIN;
      Debug("ssl.error", "SSL_write-SSL_ERROR_WANT_WRITE");
      break;
    }
    case SSL_ERROR_SYSCALL:
      // 在OpenSSL API调用系统API时遇到了错误
      TraceOut(trace, get_remote_addr(), get_remote_port(), "Syscall Error: %s", strerror(errno));
      r = -errno;
      SSL_INCREMENT_DYN_STAT(ssl_error_syscall);
      Debug("ssl.error", "SSL_write-SSL_ERROR_SYSCALL");
      break;
    // end of stream
    case SSL_ERROR_ZERO_RETURN:
      // 通常表示连接中断
      // 区别于 read 操作，不返回EOS，因为不会对Write VIO回调VC_EVENT_EOS
      TraceOut(trace, get_remote_addr(), get_remote_port(), "SSL Error: zero return");
      r = -errno;
      SSL_INCREMENT_DYN_STAT(ssl_error_zero_return);
      Debug("ssl.error", "SSL_write-SSL_ERROR_ZERO_RETURN");
      break;
    case SSL_ERROR_SSL:
      // 表示遇到 SSL 内部错误
    default: {
      char buf[512];
      unsigned long e = ERR_peek_last_error();
      ERR_error_string_n(e, buf, sizeof(buf));
      TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL Error: sslErr=%d, ERR_get_error=%ld (%s) errno=%d", err, e, buf,
              errno);
      r = -errno;
      SSL_CLR_ERR_INCR_DYN_STAT(this, ssl_error_ssl, "SSL_write-SSL_ERROR_SSL errno=%d", errno);
    } break;
    }
    // 返回调用者
    return (r);
  }
}
```


## 参考资料

- [P_SSLNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNetVConnection.h)
- [SSLNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNetVConnection.cc)
- [SSLUtils.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLUtils.cc)

  
