# 核心组件：ProtocolProbeSessionAccept

之前介绍NetAccept时，讲到创建UnixNetVConnection后会在

## 定义

```
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

class ProtocolProbeSessionAccept : public SessionAccept, public ProtocolProbeSessionAcceptEnums
{
public:
  ProtocolProbeSessionAccept() : SessionAccept(NULL)
  {
    memset(endpoint, 0, sizeof(endpoint));
    SET_HANDLER(&ProtocolProbeSessionAccept::mainEvent);
  }
  ~ProtocolProbeSessionAccept() {}

  void registerEndpoint(ProtoGroupKey key, SessionAccept *ap);

  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);

private:
  int mainEvent(int event, void *netvc);
  ProtocolProbeSessionAccept(const ProtocolProbeSessionAccept &);            // disabled
  ProtocolProbeSessionAccept &operator=(const ProtocolProbeSessionAccept &); // disabled

  /** Child acceptors, index by @c ProtoGroupKey
      We pass on the actual accept to one of these after doing protocol sniffing.
      We make it one larger and leave the last entry NULL so we don't have to
      do range checks on the enum value.
   */
  SessionAccept *endpoint[N_PROTO_GROUPS + 1];

  friend struct ProtocolProbeTrampoline;
};
```

## 参考资料

- [ProtocolProbeSessionAccept.h](http://github.com/apache/trafficserver/tree/master/proxy/ProtocolProbeSessionAccept.h)
- [ProtocolProbeSessionAccept.cc](http://github.com/apache/trafficserver/tree/master/proxy/ProtocolProbeSessionAccept.cc)
