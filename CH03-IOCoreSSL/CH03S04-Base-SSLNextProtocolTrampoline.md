# 基础组件：SSLNextProtocolTrampoline

## 定义

```
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

// SSLNextProtocolTrampoline is the receiver of the I/O event generated when we perform a 0-length read on the new SSL
// connection. The 0-length read forces the SSL handshake, which allows us to bind an endpoint that is selected by the
// NPN extension. The Continuation that receives the read event *must* have a mutex, but we don't want to take a global
// lock across the handshake, so we make a trampoline to bounce the event from the SSL acceptor to the ultimate session
// acceptor.
struct SSLNextProtocolTrampoline : public Continuation {
  explicit SSLNextProtocolTrampoline(const SSLNextProtocolAccept *npn, ProxyMutex *mutex) : Continuation(mutex), npnParent(npn)
  {
    SET_HANDLER(&SSLNextProtocolTrampoline::ioCompletionEvent);
  }

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
```

## 参考资料

- [SSLNextProtocolAccept.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolAccept.cc)
