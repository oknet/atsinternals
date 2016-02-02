# 基础组件：SSLNextProtocolTrampoline

Trampoline 在英语里是“蹦床”的意思，非常形象的勾勒出这个组件的功能。

在前面我们介绍NetAccept时，提到Acceptor和NetVC与SM的关系，需要Acceptor接受新的NetVC，然后创建SM对象，并与NetVC关联。



在SSL的实现里要增加一个步骤，在Acceptor之后有一个“蹦床”，由蹦床根据NPN/ALPN的协商再跳到对应的SessionAcceptor。

而多个不同的 SessionAcceptor 被注册到蹦床里，通过 SSLNextProtocolSet 来管理。

这里的 SessionAcceptor 实际上已经是上层状态机的一部分了，SessionAcceptor会负责创建对应的状态机。

在 ProtocolTrampoline 的设计中，需要考虑对 NPN/ALPN 协议支持、不支持的两种类型的客户端：

  - 对于支持 NPN/ALPN 协议的客户端所建立的 SSLNetVC
    - 通过解析 NPN/ALPN 协议来确认应用层协议的类型，直接跳转到对应的状态机
  - 对于不支持 NPN/ALPN 协议的客户端所建立的 SSLNetVC
    - 直接跳转到 ProbeSessionAccept，
    - 然后通过 ProbeSessionTrampoline 来判定应用层的协议类型，再跳转到对应的状态机

这个设计就像是有不同本领的忍者，当不能直接到达目标时，通过“忍者跳跃”借助辅助工具“蹦床”达到最终的目标：

  - 有的能一次跳跃达到终点
  - 有的需要借助两次跳跃才能达到终点

双蹦床的设计，如下图所示：

```
                                                    ?       ?                        #     #     #     #
                                            ?                                #      #     #                   #
                                      ?                               #     #      #            #                  #
                                  ?                            #                         #            #               #
                              *                           #     ALPN                          #            #            #
                           *                           #    or                                  #         + #  +    +    #
                        *                           #  NPN                                        #  +   +   #         + #
                      *                          #  h                                             +#         +           +
                    *                          #  t                                             +  + Yeah!   + Yeah!     + Yeah!
                  *                          #  i                                             +    ========  ==========  ========
                * + +      Ninja           #  W                                             +      = SPDY =  = HTTP/2 =  = HTTP =
        SSLNetVC      +        Jump      #                                                +        ========  ==========  ========
 ==================     +               #                                               +             ==         ==         ==
 = ProtocolAccept =       +           #            (Without NPN/ALPN)                  +              ==         ==         ==
 ==================        +         #    #  #  #                         Ninja       +               ==         ==         ==
   ==          ==           +       #  #            # .. .. .. +  +  +        Jump   /                ==         ==         ==
   ==          ==            +     #             =================      +           /                 ==         ==         ==
   ==          ==             +   /              = SessionAccept =        +        /                  ==         ==         ==
   ==          ==          ====+ /=======        =================          +     /                   ==         ==         ==
   ==          ==          =    +       =          ==         ==             +   /                    ==         ==         ==
   ==          ==          =  Protocol  =          ==         ==          ====+ /=======              ==         ==         ==
   ==          ==          = Trampoline =          ==         ==          =    +       =              ==         ==         ==
   ==          ==          ==============          ==         ==          =  Session   =              ==         ==         ==
   ==          ==          =            =          ==         ==          = Trampoline =              ==         ==         ==
   ==          ==          =            =          ==         ==          ==============              ==         ==         ==
   ==          ==          =            =          ==         ==          =            =              ==         ==         ==
   ==          ==          =            =          ==         ==          =            =              ==         ==         ==
 ==================     ====================     =================     ====================     ====================================

```

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

  // 这个状态机是被SSLNextProtocolAccept通过new方法创建的，然后通过“蹦床”方式弹到后面的SessionAccept中，
  // 因此，在弹走SSLNetVC之后要通过delete方法释放对象自身。
  int
  ioCompletionEvent(int event, void *edata)
  {
    VIO *vio;
    Continuation *plugin;
    SSLNetVConnection *netvc;

    // 由于是对 0长度 读操作的I/O事件回调，因此 edata 指向的必然是 SSLNetVC 的read.vio成员，是一个VIO类型
    vio = static_cast<VIO *>(edata);
    // 解析出 VIO 内包含的 SSLNetVC
    netvc = dynamic_cast<SSLNetVConnection *>(vio->vc_server);
    // 如果不是 SSLNetVConnection 类型，那么就 assert 了
    ink_assert(netvc != NULL);

    // 由于是 0长度的读操作，因此只有传入READ_COMPLETE才表示成功，而且不会出现READ_READY，因为READ_COMPLETE优先级较高
    // 其它情况一律是错误的情况
    switch (event) {
    case VC_EVENT_EOS:
    case VC_EVENT_ERROR:
    case VC_EVENT_ACTIVE_TIMEOUT:
    case VC_EVENT_INACTIVITY_TIMEOUT:
      // Cancel the read before we have a chance to delete the continuation
      // 遇到连接超时的时候，注销读操作，关闭SSLNetVC，回收“蹦床”自身
      netvc->do_io_read(NULL, 0, NULL);
      netvc->do_io(VIO::CLOSE);
      delete this;
      return EVENT_ERROR;
    case VC_EVENT_READ_COMPLETE:
      // 遇到我们需要的READ_COMPLETE事件，直接跳出switch
      break;
    default:
      // 感觉不应该有任何运行到这里的可能
      // 是不是要弄个assert在这里？不然要内存泄漏了？
      return EVENT_ERROR;
    }

    // Cancel the read before we have a chance to delete the continuation
    // 注销读操作，因为接下来要回收“蹦床”自身
    netvc->do_io_read(NULL, 0, NULL);
    // 通过SSLNetVConnection的endpoint方法获取 npnEndpoint 成员
    plugin = netvc->endpoint();
    if (plugin) {
      // 优先“弹到” npnEndpoint 状态机
      send_plugin_event(plugin, NET_EVENT_ACCEPT, netvc);
    } else if (npnParent->endpoint) {
      // 如果 npnEndpoint 状态机不存在，就“弹到” npnParent->endpoint 状态机
      // npnParent 是初始化“蹦床”时，传入的 SSLNextProtocolAccept 实例。
      // npnParent->endpoint 对于 SSLNextProtocolAccept 来说，
      //     在 HttpProxyServerMain.cc 中被指向了  ProtocolProbeSessionAccept 对象。
      // Route to the default endpoint
      send_plugin_event(npnParent->endpoint, NET_EVENT_ACCEPT, netvc);
    } else {
      // 如果都不存在，就直接关闭 SSLNetVC
      // No handler, what should we do? Best to just kill the VC while we can.
      netvc->do_io(VIO::CLOSE);
    }

    // 上面从 NET_EVENT_ACCEPT 事件处理函数返回后，SSLNetVC 的 Read VIO 和/或 Write VIO 会被重新设置，并与上层状态机关联。
    // 因此可以安全的删除“蹦床”自身，并返回
    delete this;
    return EVENT_CONT;
  }

  const SSLNextProtocolAccept *npnParent;
};

// 用来回调状态机的方法
// plugin 是即将回调的状态机
// event 是回调的事件
// edata 是NetVC
static void
send_plugin_event(Continuation *plugin, int event, void *edata)
{
  if (plugin->mutex) {
    // 有mutex，则上锁后回调
    EThread *thread(this_ethread());
    MUTEX_TAKE_LOCK(plugin->mutex, thread);
    plugin->handleEvent(event, edata);
    MUTEX_UNTAKE_LOCK(plugin->mutex, thread);
  } else {
    // 没有mutex，直接回调
    plugin->handleEvent(event, edata);
  }
}
```

## 参考资料

- [SSLNextProtocolAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolAccept.cc)
- [HttpProxyServerMain.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyServerMain.cc)
