# Basic component : SessionAccept
`SessionAccept` is a base class that provides an `accept` method and a `mainEvent` call back.

To handle multiple transactions in a TCP connection, ATS abstracts the `SessionAccept` base class.

Transactions here could be:

  - different types, for example, to support SSL,
    - firstly, SSL handshake,
    - secondly, state machines of protocols chosen from NPN/ALPN.
  - the same type, for example, to support a keep-alive HTTP connection,
    - an HTTP transaction,
    - after the first HTTP transaction ends, another HTTP transaction begins.

## Defination

```
class SessionAccept : public Continuation
{
public:
  SessionAccept(ProxyMutex *amutex) : Continuation(amutex) { SET_HANDLER(&SessionAccept::mainEvent); }

  ~SessionAccept() {}

  virtual bool accept(NetVConnection *, MIOBuffer *, IOBufferReader *) = 0;

  static const AclRecord *testIpAllowPolicy(sockaddr const *client_ip);

private:
  virtual int mainEvent(int event, void *netvc) = 0;
};
```

## References

- [I_SessionAccept.h](http://github.com/apache/trafficserver/tree/master/iocore/net/I_SessionAccept.h)
