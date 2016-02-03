# 基础组件：SSLNextProtocolAccept

在 IOCoreNET 部分的介绍的 ProtocolProbeSessionAccept 其实是参照了 SSLNextProtocolAccept 的实现。

SSLNextProtocolAccept 是读取 NPN / ALPN 协议中传递的类型，然后通过“蹦床”跳转到对应的状态机，ProtocolProbeSessionAccept 则是读取客户端发送的第一个请求报文来判断报文的内容是什么类型，然后通过“蹦床”跳转到对应的状态机。

通过 register 方法为每一种协议注册一个上层状态机，然后使用“蹦床”来判断 NPN / ALPN 协议中传递的类型，在完成SSL握手后，通过“蹦床”跳转到对应的状态机。

可以看到与 ProtocolProbeSessionAccept 和 ProtocolProbeSessionTrampoline 组合的功能非常的接近。

## 定义

```
class SSLNextProtocolAccept : public SessionAccept
{
public:
  // 构造函数：
  //     初始化mutex为NULL，使用 new_empty_MIOBuffer() 创建 buffer，
  //     用传入的状态机初始化endpoint，用传入的transparent_passthrough初始化成员
  // 设置回调函数为 mainEvent
  SSLNextProtocolAccept(Continuation *, bool);
  // 析构函数：用来释放成员 buffer
  ~SSLNextProtocolAccept();

  // 没有实现，不可以被调用
  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);

  // Register handler as an endpoint for the specified protocol. Neither
  // handler nor protocol are copied, so the caller must guarantee their
  // lifetime is at least as long as that of the acceptor.
  // 注册一个与指定协议对应的SessionAccept状态机
  // 底层是直接调用了 protoset.registerEndpoint(protocol, handler)
  bool registerEndpoint(const char *protocol, Continuation *handler);

  // Unregister the handler. Returns false if this protocol is not registered
  // or if it is not registered for the specified handler.
  // 注销一个与指定协议对应的SessionAccept状态机
  // 这个方法没有在ATS中使用，在TSAPI中TSNetAcceptNamedProtocol在调用register失败后会调用unregister方法，但是仅仅针对SSL的情况
  // 因此在ProtocolProbeSessionAccept中没有unregister方法
  // 底层是直接调用了 protoset.unregisterEndpoint(protocol, handler)
  bool unregisterEndpoint(const char *protocol, Continuation *handler);

  SLINK(SSLNextProtocolAccept, link);

private:
  // 主回调函数
  int mainEvent(int event, void *netvc);
  SSLNextProtocolAccept(const SSLNextProtocolAccept &);            // disabled
  SSLNextProtocolAccept &operator=(const SSLNextProtocolAccept &); // disabled

  MIOBuffer *buffer; // XXX do we really need this?
  // endpoint 指向 NPN / ALPN 无法匹配时，缺省的处理机制。
  // 对于 SSLNextProtocolAccept 来说，在创建时由调用者传入，通过构造函数设置，
  //     在 HttpProxyServerMain.cc 中创建 SSLNextProtocolAccept 时传入的是 ProtocolProbeSessionAccept 对象。
  Continuation *endpoint;
  // 存储了适用于 NPN / ALPN 协议的，当前注册的所有协议和上层状态机的对应关系
  SSLNextProtocolSet protoset;
  // tr-pass 标志
  bool transparent_passthrough;

  friend struct SSLNextProtocolTrampoline;
};
```

## 方法

```
// 由于不同的event传入的可能是netvc，可能是vio
// 这个方法对于不同的event，总是返回sslvc
static SSLNetVConnection *
ssl_netvc_cast(int event, void *edata)
{
  union {
    VIO *vio;
    NetVConnection *vc;
  } ptr;

  switch (event) {
  case NET_EVENT_ACCEPT:
    ptr.vc = static_cast<NetVConnection *>(edata);
    return dynamic_cast<SSLNetVConnection *>(ptr.vc);
  case VC_EVENT_INACTIVITY_TIMEOUT:
  case VC_EVENT_READ_COMPLETE:
  case VC_EVENT_ERROR:
    ptr.vio = static_cast<VIO *>(edata);
    return dynamic_cast<SSLNetVConnection *>(ptr.vio->vc_server);
  default:
    return NULL;
  }
}

int
SSLNextProtocolAccept::mainEvent(int event, void *edata)
{
  // 获取 SSLVC
  SSLNetVConnection *netvc = ssl_netvc_cast(event, edata);
  // 这里应该有一个 assert 判断一下 netvc!=NULL

  Debug("ssl", "[SSLNextProtocolAccept:mainEvent] event %d netvc %p", event, netvc);

  switch (event) {
  case NET_EVENT_ACCEPT:
    // 通常应该只有NET_EVENT_ACCEPT
    ink_release_assert(netvc != NULL);

    // 设置sslvc的tr-pass状态
    netvc->setTransparentPassThrough(transparent_passthrough);

    // Register our protocol set with the VC and kick off a zero-length read to
    // force the SSLNetVConnection to complete the SSL handshake. Don't tell
    // the endpoint that there is an accept to handle until the read completes
    // and we know which protocol was negotiated.
    // 将当前 支持的协议与上层状态机（protoset）注册到SSLVC
    netvc->registerNextProtocolSet(&this->protoset);
    // 为当前的SSLVC设置“蹦床”，准备跳转到匹配的上层状态机
    // 这里设置一个 Read VIO，读取 0字节 长度，this->buffer 是在构造函数中创建的
    // 最后的参数0，对VIO::READ没用。
    netvc->do_io(VIO::READ, new SSLNextProtocolTrampoline(this, netvc->mutex), 0, this->buffer, 0);
    // 设置 sessionAcceptPtr ，但是好像没有用到。
    netvc->set_session_accept_pointer(this);
    return EVENT_CONT;
  default:
    // 如果是其它事件，就直接关闭SSLVC
    // 这里可能会有问题，netvc可能为NULL，如果在ssl_netvc_cast调用的后面有个assert就ok了
    netvc->do_io(VIO::CLOSE);
    return EVENT_DONE;
  }
}
```

## SSL会话创建流程

由于 SSL 位于OSI第六层的表示层，在完成加解密动作后要与OSI第七层的应用层对接，因此 ATS 设计了一个“蹦床”的结构来实现从 SSL 到 HTTP，SPDY，HTTP2的跳转过程，因此ATS的实现方式如下：

  - 首先，创建 ProtocolProbeSessionAccept 对象，用于支持 80 端口 HTTP，SPDY，HTTP2 的“蹦床”结构
    - 但是这个方式是需要读取第一个部分的明文（未加密）数据来判断请求的
    - 有可能判断的不正确
  - 然后，创建 SSLNextProtocolAccept，用于支持 443 端口 SSL 协议加密的 HTTP，SPDY，HTTP2 的“蹦床”结构
    - 使用 NPN / ALPN 的方式注册 http，spdy，http/2 三种协议和对应的状态机
    - 同时把 ProtocolProbeSessionAccept 状态机传入
    - 这样不支持 NPN / ALPN 协议的客户端就可以用原始的方式判断出应用层的协议类型


## 关于 new_empty_MIOBuffer 方法

```
TS_INLINE MIOBuffer *
new_empty_MIOBuffer_internal(
  int64_t size_index)
{
  MIOBuffer *b = THREAD_ALLOC(ioAllocator, this_thread());
  b->size_index = size_index;
  return b;
}
```

对照 new_MIOBuffer 方法

```
TS_INLINE MIOBuffer *
new_MIOBuffer_internal(
  int64_t size_index)
{
  MIOBuffer *b = THREAD_ALLOC(ioAllocator, this_thread());
  b->alloc(size_index);
  return b;
}
```

可以看到差别就是是否执行了 alloc 的内部方法

  - alloc 方法是按照 size_index 的要求为 MIOBuffer 分配内存（IOBufferBlock 和 IOBufferData）
  - 如果不调用 alloc 方法，只是设置了 size_index 的值，则表示在第一次向 MIOBuffer 写入数据的时候，在进行内存的分配。

因此，new_empty_MIOBuffer 就是只创建一个 MIOBuffer 的结构，但是不关联 IOBufferBlock 和 IOBufferData。

## 参考资料

- [P_SSLNextProtocolSet.h](https://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNextProtocolSet.h)
- [P_SSLNextProtocolAccept.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNextProtocolAccept.h)
- [SSLNextProtocolAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolAccept.cc)
- [P_IOBuffer.h](https://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_IOBuffer.h)
