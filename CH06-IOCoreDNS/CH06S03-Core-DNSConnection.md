# 核心组件：DNSConnection

## 定义

```
struct DNSConnection {
  /// Options for connecting.
  struct Options {
    typedef Options self; ///< Self reference type.

    /// Connection is done non-blocking.
    /// Default: @c true.
    bool _non_blocking_connect;
    /// Set socket to have non-blocking I/O.
    /// Default: @c true.
    bool _non_blocking_io;
    /// Use TCP if @c true, use UDP if @c false.
    /// Default: @c false.
    bool _use_tcp;
    /// Bind to a random port.
    /// Default: @c true.
    bool _bind_random_port;
    /// Bind to this local address when using IPv6.
    /// Default: unset, bind to IN6ADDR_ANY.
    sockaddr const *_local_ipv6;
    /// Bind to this local address when using IPv4.
    /// Default: unset, bind to INADDRY_ANY.
    sockaddr const *_local_ipv4;

    Options();

    self &setUseTcp(bool p);
    self &setNonBlockingConnect(bool p);
    self &setNonBlockingIo(bool p);
    self &setBindRandomPort(bool p);
    self &setLocalIpv6(sockaddr const *addr);
    self &setLocalIpv4(sockaddr const *addr);
  };

  int fd;
  IpEndpoint ip;
  int num;
  LINK(DNSConnection, link);
  EventIO eio;
  InkRand generator;
  DNSHandler *handler;

  int connect(sockaddr const *addr, Options const &opt = DEFAULT_OPTIONS);
  /*
                bool non_blocking_connect = NON_BLOCKING_CONNECT,
                bool use_tcp = CONNECT_WITH_TCP, bool non_blocking = NON_BLOCKING, bool bind_random_port = BIND_ANY_PORT);
  */
  int close();
  void trigger();

  virtual ~DNSConnection();
  DNSConnection();

  static Options const DEFAULT_OPTIONS;
};
```

# 参考资料

- [P_DNSConnection.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/P_DNSConnection.h)
- [DNS.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/DNS.cc)