# 基础组件：SSLNextProtocolTrampoline

Trampoline 在英语里是“蹦床”的意思，非常形象的勾勒出这个组件的功能。

在前面我们介绍NetAccept时，提到Acceptor和NetVC与SM的关系，需要Acceptor接受新的NetVC，然后创建SM对象，并与NetVC关联。

在SSL的实现里要增加一个步骤，在Acceptor之后有一个“蹦床”，由蹦床根据NPN/ALPN的协商再跳到对应的SessionAcceptor。

而多个不同的 SessionAcceptor 被注册到蹦床里，通过 SSLNextProtocolSet 来管理。

这里的 SessionAcceptor 实际上已经是上层状态机的一部分了，SessionAcceptor会负责创建对应的状态机。

## 定义

```
// SSLNextProtocolTrampoline is the receiver of the I/O event generated when we perform a 0-length read on the new SSL
// connection. The 0-length read forces the SSL handshake, which allows us to bind an endpoint that is selected by the
// NPN extension. The Continuation that receives the read event *must* have a mutex, but we don't want to take a global
// lock across the handshake, so we make a trampoline to bounce the event from the SSL acceptor to the ultimate session
// acceptor.
// 对源代码的注释做直接翻译，如下：
// 当在一个SSL连接上，执行长度为0字节的读操作时，EventSystem 将回调 SSLNextProtocolTrampoline 来接受这个I/O事件。
//     注：这里0长度的读取操作里的0长度，指的是解密后的数据长度。例如：对于握手过程，就是0长度。
// 0字节读操作的关注点就是SSL握手过程，实现了绑定一个NPN扩展选中的协议末端。
// 被这个读事件回调的“延续”必须要有一个mutex锁，但是我们又不希望在整个握手过程中一直保持上锁状态，
// 因此设计了这个蹦床，将这个事件从SSL Acceptor弹到最终的Session Acceptor。

struct SSLNextProtocolTrampoline : public Continuation {
  // 构造函数
  // 初始化mutex和npnParent成员，并设置回调函数为ioCompletionEvent
  explicit SSLNextProtocolTrampoline(const SSLNextProtocolAccept *npn, ProxyMutex *mutex) : Continuation(mutex), npnParent(npn)
  {
    SET_HANDLER(&SSLNextProtocolTrampoline::ioCompletionEvent);
  }

  // 
  int
  ioCompletionEvent(int event, void *edata)
  {
    VIO *vio;
    Continuation *plugin;
    SSLNetVConnection *netvc;

    vio = static_cast<VIO *>(edata);
    netvc = dynamic_cast<SSLNetVConnection *>(vio->vc_server);
    ink_assert(netvc != NULL);

    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // Cancel the read before we have a chance to delete the continuation
      netvc->do_io_read(NULL, 0, NULL);
      netvc->do_io(VIO::CLOSE);
      delete this;
      return EVENT_ERROR;
    case VC_EVENT_READ_COMPLETE:
      break;
    default:
      return EVENT_ERROR;
    }

    // Cancel the read before we have a chance to delete the continuation
    netvc->do_io_read(NULL, 0, NULL);
    plugin = netvc->endpoint();
    if (plugin) {
      send_plugin_event(plugin, NET_EVENT_ACCEPT, netvc);
    } else if (npnParent->endpoint) {
      // Route to the default endpoint
      send_plugin_event(npnParent->endpoint, NET_EVENT_ACCEPT, netvc);
    } else {
      // No handler, what should we do? Best to just kill the VC while we can.
      netvc->do_io(VIO::CLOSE);
    }

    delete this;
    return EVENT_CONT;
  }

  const SSLNextProtocolAccept *npnParent;
};

static void
send_plugin_event(Continuation *plugin, int event, void *edata)
{
  if (plugin->mutex) {
    EThread *thread(this_ethread());
    MUTEX_TAKE_LOCK(plugin->mutex, thread);
    plugin->handleEvent(event, edata);
    MUTEX_UNTAKE_LOCK(plugin->mutex, thread);
  } else {
    plugin->handleEvent(event, edata);
  }
}
```

## 参考资料

- [SSLNextProtocolAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolAccept.cc)
