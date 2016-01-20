# 核心组件 SSLNetVConnection

SSLNetVConnection 继承自 UnixNetVConnection，对部分方法进行了重载：

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
  - getSSLSessionCacheHit
  - setSSLSessionCacheHit

同时增加了一些方法：

  - enableRead
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

在UnixNetVConnection的基础上构建了对SSL会话的支持。

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

  bool
  isEosRcvd()
  {
    return eosRcvd;
  }

  bool
  getSSLTrace() const
  {
    return sslTrace || super::origin_trace;
  };

  void
  setSSLTrace(bool state)
  {
    sslTrace = state;
  };

  bool computeSSLTrace();

  const char *
  getSSLProtocol(void) const
  {
    if (ssl == NULL)
      return NULL;
    return SSL_get_version(ssl);
  };

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
  virtual int populate(Connection &con, Continuation *c, void *arg);

private:
  SSLNetVConnection(const SSLNetVConnection &);
  SSLNetVConnection &operator=(const SSLNetVConnection &);

  bool sslHandShakeComplete;
  bool sslClientConnection;
  bool sslClientRenegotiationAbort;
  bool sslSessionCacheHit;
  MIOBuffer *handShakeBuffer;
  IOBufferReader *handShakeHolder;
  IOBufferReader *handShakeReader;
  int handShakeBioStored;

  bool transparentPassThrough;

  /// The current hook.
  /// @note For @C SSL_HOOKS_INVOKE, this is the hook to invoke.
  class APIHook *curHook;

  enum {
    SSL_HOOKS_INIT,     ///< Initial state, no hooks called yet.
    SSL_HOOKS_INVOKE,   ///< Waiting to invoke hook.
    SSL_HOOKS_ACTIVE,   ///< Hook invoked, waiting for it to complete.
    SSL_HOOKS_CONTINUE, ///< All hooks have been called and completed
    SSL_HOOKS_DONE      ///< All hooks have been called and completed
  } sslPreAcceptHookState;

  enum SSLHandshakeHookState {
    HANDSHAKE_HOOKS_PRE,
    HANDSHAKE_HOOKS_CERT,
    HANDSHAKE_HOOKS_POST,
    HANDSHAKE_HOOKS_INVOKE,
    HANDSHAKE_HOOKS_DONE
  } sslHandshakeHookState;

  const SSLNextProtocolSet *npnSet;
  Continuation *npnEndpoint;
  SessionAccept *sessionAcceptPtr;
  MIOBuffer *iobuf;
  IOBufferReader *reader;
  bool eosRcvd;
  bool sslTrace;
};

typedef int (SSLNetVConnection::*SSLNetVConnHandler)(int, void *);

extern ClassAllocator<SSLNetVConnection> sslNetVCAllocator;
```


## 参考资料

- [P_SSLNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNetVConnection.h)
- [SSLNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNetVConnection.cc)


  
