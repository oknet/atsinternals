# 核心组件：ProtocolProbeSessionAccept

之前介绍 NetAccept 时，讲到创建 UnixNetVConnection 后会在 acceptEvent 回调函数中，回调上层状态机，传递 NET_EVENT_ACCEPT 给上层状态机。

在实现SPDY协议的支持以前，在80端口上只有HTTP协议，在出现了SPDY协议和最近才被支持的HTTP/2协议之后，就需要在80端口上识别客户端采用的是哪种协议，然后再创建对应协议的上层状态机。

ProtocolProbeSessionAccept 就是用来实现这个识别过程的，从字面意思就很容易理解：“协议探测”SessionAccept。

通过 register 方法为每一种协议注册一个上层状态机，然后使用“蹦床”来读取第一部分的数据，对这些数据分析，判定Client发上来的协议是SPDY还是HTTP2，如果都不是，就按照HTTP来处理。

如果也不是HTTP协议，就由HttpSM实现对非HTTP协议的报错处理，或者自动bypass实现blind tunnel等。

为了能够实现首先识别协议类型，再创建处理对应协议的状态机，ATS设计了SessionTrampoline与SessionAccept配合，从而让 UnixNetVC 像是做了一个忍者跳跃一样，从 SessionAccept 这个高台上跳到 SessionTrampoline 上，借助势能转换，再跳到最终的目的地：

```
                                                                              +     +     +
                                                ?                       +   +   +             +
                                          ?                        +  +            +             +
                                    *                          +         + Yeah!     + Yeah!      + Yeah!
                               *                           +             ========    ==========    ========
                           *                            +                = SPDY =    = HTTP/2 =    = HTTP =
                        *                             +                  ========    ==========    ========
                     *                              +                       ==           ==           ==
                  *   +   +                       +                         ==           ==           ==
                * +           +     Ninja       +                           ==           ==           ==
       UnixNetVC                 +      Jump   /                            ==           ==           ==
 =================                 +          /                             ==           ==           ==
 = SessionAccept =                   +       /                              ==           ==           ==
 =================                    +     /                               ==           ==           ==
   ==         ==                       +   /                                ==           ==           ==
   ==         ==                   =====+ /========                         ==           ==           ==
   ==         ==                   =     +        =                         ==           ==           ==
   ==         ==                   =   Session    =                         ==           ==           ==
   ==         ==                   =  Trampoline  =                         ==           ==           ==
   ==         ==                   ================                         ==           ==           ==
   ==         ==                   =              =                         ==           ==           ==
   ==         ==                   =              =                         ==           ==           ==
 =================              ======================                  ====================================

```

## 定义

```
// 枚举类型的结构，用于定义支持的协议种类，将来如果需要扩展支持的协议，可以在这里扩展
struct ProtocolProbeSessionAcceptEnums {
  /// Enumeration for related groups of protocols.
  /// There is a child acceptor for each group which
  /// handles finer grained stuff.
  enum ProtoGroupKey {
    PROTO_HTTP,    ///< HTTP group (0.9-1.1)
    PROTO_HTTP2,   ///< HTTP 2 group
    PROTO_SPDY,    ///< All SPDY versions
    N_PROTO_GROUPS ///< Size value.
  };
};

// SessionAccept 用于实现协议探测的过渡阶段
class ProtocolProbeSessionAccept : public SessionAccept, public ProtocolProbeSessionAcceptEnums
{
public:
  // 构造函数，设置mutex＝NULL，表示没有mutex保护
  // 回调函数，只有 mainEvent 一个
  ProtocolProbeSessionAccept() : SessionAccept(NULL)
  {
    memset(endpoint, 0, sizeof(endpoint));
    SET_HANDLER(&ProtocolProbeSessionAccept::mainEvent);
  }
  ~ProtocolProbeSessionAccept() {}

  // 注册一个与指定协议对应的SessionAccept状态机
  void registerEndpoint(ProtoGroupKey key, SessionAccept *ap);

  // 没有具体实现，只是对基类方法的重载
  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);

private:
  // 主回调函数，用来接受 NET_EVENT_ACCEPT
  int mainEvent(int event, void *netvc);
  ProtocolProbeSessionAccept(const ProtocolProbeSessionAccept &);            // disabled
  ProtocolProbeSessionAccept &operator=(const ProtocolProbeSessionAccept &); // disabled

  /** Child acceptors, index by @c ProtoGroupKey
      We pass on the actual accept to one of these after doing protocol sniffing.
      We make it one larger and leave the last entry NULL so we don't have to
      do range checks on the enum value.
   */
  // 与协议对应的状态机的数组
  // 每种协议只能对应一个状态机
  // 这里的数组元素的数量比协议的数量多一个，最后一个指向NULL，这样在遍历的时候，遇到NULL就知道到达了数组尾部
  SessionAccept *endpoint[N_PROTO_GROUPS + 1];

  friend struct ProtocolProbeTrampoline;
};
```

## 方法

```
int
ProtocolProbeSessionAccept::mainEvent(int event, void *data)
{
  // 只处理 NET_EVENT_ACCEPT 事件
  if (event == NET_EVENT_ACCEPT) {
    // data 为 vc，不为NULL
    ink_assert(data);

    VIO *vio;
    // 将data转为netvc
    NetVConnection *netvc = static_cast<NetVConnection *>(data);
    // 尝试转换netvc为sslvc，如果是来自SSL协商后的HTTP/SPDY/HTTP2协议，则其底层是sslvc
    // 否则转换失败，sslvc==NULL
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(netvc);
    // 默认初始化 buf 和 reader 为 NULL
    MIOBuffer *buf = NULL;
    IOBufferReader *reader = NULL;
    if (ssl_vc) {
      // 如果是SSLVC，就从SSLVC获取iobuf和reader
      // 在SSLVC的iobuf里存储了握手完成后读取到的第一部分数据的解密后的内容
      buf = ssl_vc->get_ssl_iobuf();
      reader = ssl_vc->get_ssl_reader();
    }
    // 创建一个协议探测的蹦床，准备将vc向目标状态机跳跃/迁移
    ProtocolProbeTrampoline *probe = new ProtocolProbeTrampoline(this, netvc->mutex, buf, reader);

    // XXX we need to apply accept inactivity timeout here ...

    if (!probe->reader->is_read_avail_more_than(0)) {
      // 这里通常针对 netvc 的处理，sslvc 也有可能会运行到此处
      //     例如 sslvc 的客户端完成握手之后，没有及时发送请求，那么sslvc的iobuf虽然被传入，但是其内是空的。
      // 如果缓冲区内没有可读取的数据，就没法判断
      Debug("http", "probe needs data, read..");
      vio = netvc->do_io_read(probe, BUFFER_SIZE_FOR_INDEX(ProtocolProbeTrampoline::buffer_size_index), probe->iobuf);
      // 因此通过do_io_read操作关联蹦床，由EventSystem回调蹦床来完成判断，reenable后交给EventSystem来处理
      vio->reenable();
    } else {
      // 这里通常是针对 sslvc 的处理，netvc通常不会运行到这里
      //     因为刚刚Accept就有数据可读取，那意味着buf和reader来自sslvc的iobuf
      // 如果缓冲区内已经有可以读取的数据，那就可以根据这些数据来判断是什么协议的类型了
      Debug("http", "probe already has data, call ioComplete directly..");
      // 取消读操作
      vio = netvc->do_io_read(NULL, 0, NULL);
      // 同步回调蹦床来完成判断
      probe->ioCompletionEvent(VC_EVENT_READ_COMPLETE, (void *)vio);
      // 在同步回调之后，蹦床就已经delete了自己，因此之后不可以通过probe再引用蹦床的任何成员
    }
    return EVENT_CONT;
  }

  // 其它事件报错，但是继续运行？
  MachineFatal("Protocol probe received a fatal error: errno = %d", -((int)(intptr_t)data));
  return EVENT_CONT;
}
```

## 参考资料

- [ProtocolProbeSessionAccept.h](http://github.com/apache/trafficserver/tree/master/proxy/ProtocolProbeSessionAccept.h)
- [ProtocolProbeSessionAccept.cc](http://github.com/apache/trafficserver/tree/master/proxy/ProtocolProbeSessionAccept.cc)

# 核心组件：ProtocolProbeTrampoline

“协议探测 蹦床”

## 定义

```
struct ProtocolProbeTrampoline : public Continuation, public ProtocolProbeSessionAcceptEnums {
  static const size_t minimum_read_size = 1;
  // 在 iocore/net/P_Net.h 中，定义了：
  //     static size_t const CLIENT_CONNECTION_FIRST_READ_BUFFER_SIZE_INDEX = BUFFER_SIZE_INDEX_4K;
  static const unsigned buffer_size_index = CLIENT_CONNECTION_FIRST_READ_BUFFER_SIZE_INDEX;
  IOBufferReader *reader;

  // 构造函数
  // mutex 继承自 netvc 的 mutex
  // probeParent 指向创建此蹦床实例的 ProtocolProbeSessionAccept 对象
  explicit ProtocolProbeTrampoline(const ProtocolProbeSessionAccept *probe, ProxyMutex *mutex, MIOBuffer *buffer,
                                   IOBufferReader *reader)
    : Continuation(mutex), probeParent(probe)
  {
    // 如果SSLVC的客户端不支持NPN/ALPN协议，那么就会通过此处来判断具体的协议类型
    // 对于 sslvc，传递进来的 buffer和reader 都来自 sslvc 的成员，则继承 buffer和reader
    // 对于 netvc，传递进来的 buffer和reader 都是 NULL，则创建独立的 buffer和reader
    // buffer和reader 用于读取第一个来自客户端的请求数据，这样可以判断客户端采用什么样的协议
    this->iobuf = buffer ? buffer : new_MIOBuffer(buffer_size_index);
    this->reader = reader ? reader : iobuf->alloc_reader(); // reader must be allocated only on a new MIOBuffer.
    // 设置回调函数为 ioCompletionEvent
    SET_HANDLER(&ProtocolProbeTrampoline::ioCompletionEvent);
  }

  int
  ioCompletionEvent(int event, void *edata)
  {
    VIO *vio;
    NetVConnection *netvc;
    ProtoGroupKey key = N_PROTO_GROUPS; // use this as an invalid value.

    vio = static_cast<VIO *>(edata);
    netvc = static_cast<NetVConnection *>(vio->vc_server);

    // 只有 VC_EVENT_READ_READY / VC_EVENT_READ_COMPLETE 才是正确的事件
    //     这表示读取到了内容，接下来就可以根据读取到的内容判断出来
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // Error ....
      netvc->do_io_close();
      goto done;
    case VC_EVENT_READ_READY:
    case VC_EVENT_READ_COMPLETE:
      break;
    default:
      return EVENT_ERROR;
    }

    // 必须要有netvc，因为这里是 READ_READY / READ_COMPLETE 事件的处理过程
    ink_assert(netvc != NULL);

    // 如果数据量不足，这里默认为读取至少1个字节
    if (!reader->is_read_avail_more_than(minimum_read_size - 1)) {
      // Not enough data read. Well, that sucks.
      // 不再尝试第二次，直接关闭
      netvc->do_io_close();
      goto done;
    }

    // SPDY clients have to start by sending a control frame (the high bit is set). Let's assume
    // that no other protocol could possibly ever set this bit!
    // 首先判断 spdy
    if (proto_is_spdy(reader)) {
      key = PROTO_SPDY;
    // 然后判断 http2
    } else if (proto_is_http2(reader)) {
      key = PROTO_HTTP2;
    } else {
    // 最后默认 http
      key = PROTO_HTTP;
    }

    // 既然协议已经被probe出来了，那就不需要再读取数据了，关闭Read VIO
    netvc->do_io_read(NULL, 0, NULL); // Disable the read IO that we started.

    // 通过probeParent找到在ProtocolProbeSessionAccept中注册的对应协议的状态机
    // 如果对应的状态机没有设置，则说明不支持这种类型的状态机
    if (probeParent->endpoint[key] == NULL) {
      Warning("Unregistered protocol type %d", key);
      // 打印输出的信息，然后关闭netvc
      netvc->do_io_close();
      goto done;
    }

    // Directly invoke the session acceptor, letting it take ownership of the input buffer.
    // 直接回调选中协议的状态机的accept方法
    //     上层协议的accept方法会设置Read VIO / Write VIO等，来接管后续的IO操作
    //     设置完成后就会迅速返回
    probeParent->endpoint[key]->accept(netvc, this->iobuf, reader);
    // 释放状态机自身
    delete this;
    return EVENT_CONT;

  done:
    // 异常处理（由于SSLVC也会在客户端不支持NPN/ALPN时使用此状态机判定应用层的协议类型，因此要对SSLVC做判断和处理）
    // 如果：
    //     传入的 iobuf 是由 SSLVC 提供的，但不是 SSLVC 内部的 iobuf
    //  或
    //     netvc不是一个SSLVC
    // iobuf 需要释放掉，因为该 iobuf 用于 probe 操作临时存放接收到的第一部分数据
    SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(netvc);
    if (!ssl_vc || (this->iobuf != ssl_vc->get_ssl_iobuf())) {
      free_MIOBuffer(this->iobuf);
    }
    this->iobuf = NULL;
    // 释放状态机自身
    delete this;
    return EVENT_CONT;
  }

  MIOBuffer *iobuf;
  // probeParent 指向创建此“蹦床”实例的 ProtocolProbeSessionAccept 状态机
  const ProtocolProbeSessionAccept *probeParent;
};
```

## 方法

```
static bool
proto_is_spdy(IOBufferReader *reader)
{
  // 判断spdy协议，首字节为 0x80
  // SPDY clients have to start by sending a control frame (the high bit is set). Let's assume
  // that no other protocol could possibly ever set this bit!
  return ((uint8_t)(*reader)[0]) == 0x80u;
}

static bool
proto_is_http2(IOBufferReader *reader)
{
  // 判断http2协议，需要至少4个字节
  char buf[HTTP2_CONNECTION_PREFACE_LEN];
  char *end;
  ptrdiff_t nbytes;

  end = reader->memcpy(buf, sizeof(buf), 0 /* offset */);
  nbytes = end - buf;

  // Client must send at least 4 bytes to get a reasonable match.
  if (nbytes < 4) {
    return false;
  }

  ink_assert(nbytes <= (int64_t)HTTP2_CONNECTION_PREFACE_LEN);
  return memcmp(HTTP2_CONNECTION_PREFACE, buf, nbytes) == 0;
}
```

## 无锁设计 / Mutex Free

ProtocolProbeSessionAccept 状态机的mutex在构造函数中设置为NULL，表示该状态机被回调时不需要加锁，可以被并发调用

  - 从NetAccept回调一个状态机的时候，通过下面的方式回调 ProtocolProbeSessionAccept::mainEvent
    - action_.continuation->handleEvent(NET_EVENT_ACCEPT, vc);
  - 由于 ProtocolProbeSessionAccept 的mutex为NULL，因此可以被并发调用

在 ProtocolProbeSessionAccept::mainEvent 中

  - 为每一个新的 netvc 创建了 ProtocolProbeTrampoline 状态机，共享 netvc 的 mutex
    - ProtocolProbeTrampoline *probe = new ProtocolProbeTrampoline(this, netvc->mutex, buf, reader);
  - 如果 iobuf 内没有数据，则设置 Read VIO，等待对 ProtocolProbeTrampoline 的回调
  - 如果 iobuf 内有数据，则取消 Read VIO，同步回调 ProtocolProbeTrampoline
    - probe->ioCompletionEvent(VC_EVENT_READ_COMPLETE, (void *)vio);

创建 ProtocolProbeTrampoline 状态机后，由构造函数完成初始化：

  - 对于非SSLVC：需要通过 new_MIOBuffer 为 iobuf 分配内存
  - 对于SSLVC：iobuf 则指向 SSLVC 的 iobuf
  - reader 则与 iobuf 对应
  - 设置 probeParent 指向 ProtocolProbeSessionAccept，以便获取 ProtocolProbeSessionAccept::endpoint 来完成跳转
  - 设置回调函数为：ioCompletionEvent

回调函数 ProtocolProbeTrampoline::ioCompletionEvent 接受两种事件类型：

  - VC_EVENT_READ_READY
  - VC_EVENT_READ_COMPLETE

然后根据读取到的信息判断应用层的协议是 SPDY 还是 HTTP2，如果都不是则按照 HTTP 协议来处理。

根据协议的类型，通过下面的方法回调上层状态机的 SessionAccept 的 accept 方法

  - probeParent->endpoint[key]->accept(netvc, this->iobuf, reader);

上层状态机的 accept 方法用来：

  - 接管 netvc，
  - 重新设置 Read/Write VIO，
  - 接管 iobuf 指向的 MIOBuffer，因此在返回后不需要在 ProtocolProbeTrampoline 释放
  - 因此在调用后快速返回。

此时 ProtocolProbeTrampoline 完成了从 NetAccept 到上层状态机的 SessionAccept 的跳转，释放自身：

  - delete this;
  - return EVENT_CONT;

如果 ProtocolProbeTrampoline 没有成功将 netvc 弹到上层协议的 SessionAccept 状态机，就需要考虑是否释放 iobuf

  - 考虑到 iobuf 在 sslvc 时直接指向 sslvc 的 iobuf，因此在释放前要做一个判断

```
  SSLNetVConnection *ssl_vc = dynamic_cast<SSLNetVConnection *>(netvc);
  if (!ssl_vc || (this->iobuf != ssl_vc->get_ssl_iobuf())) {
    free_MIOBuffer(this->iobuf);
  }
```

总结：

  - ProtocolProbeSessionAccept 是：
    - 每个 TCP Port 创建一个
    - 没有 Mutex 保护
  - ProtocolProbeTrampoline 是：
    - 每个 NetVC 会创建一个
    - 生命周期非常的短，在跳转到上层协议的 SessionAccept 状态机之后，就会被回收
    - 共享 NetVC 的 Mutex


