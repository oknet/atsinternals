# 核心组件：DNSConnection

在 IOCore 的设计里，所有与网络通信相关的功能都在 Net 子系统内实现，因此 NetHandler 除了处理 NetVConnection 的读写事件，同时也处理 DNSConnection 上的读事件。

DNS 协议支持 UDP 和 TCP 两种报文承载方式，但是目前仅支持 UDP 协议承载报文。

为了能够在未来实现兼容 TCP 和 UDP 两种协议发送报文，让 UDP 的操作与 TCP 操作更加接近，

- 使用了 `connect(2)` 方法与目标 Name Server 建立了 UDP Socket 连接
- 这样会得到一个 socket fd，并且在 Linux Kernel 里形成与目标 Name Server 的五元组信息

之后对该 udp socket fd 的读写操作几乎与 tcp socket fd 相同，

- 可以使用 `read(2)`，`write(2)` 读写 udp socket fd
- 而且可以把 udp socket fd 与 tcp socket fd 同时放入同一个 `epoll fd` 内，等待网络事件的触发。

另外仍然可以使用 `recvfrom(2)`，`sendto(2)` 读写 udp socket fd。

在 `NetHandler::mainNetEvent` 中专门对 DNSConnection 类型进行了处理，

- 通过把 udp socket fd 与 DNSConnection 关联
- 就可以通过 `epoll_wait` 获得 udp socket fd 上的读事件
- 然后回调 DNSConnection 上的处理函数完成对此事件的处理

可以看到 DNSConnection 没有继承自 Continuation，它的设计是介于 Connection 与 NetVConnection 之间的，或者说是在 Connection 基础上扩展了一些 NetVConnection 才有的功能，从而在 `ET_NET` 线程组实现了 DNS 响应事件的触发，并通知到 DNSHandler 完成响应数据的接收。

## 定义

```
struct DNSConnection {
  // 用于设定 DNSConnection 的参数
  /// Options for connecting.
  struct Options {
    typedef Options self; ///< Self reference type.

    /// Connection is done non-blocking.
    /// Default: @c true.
    // 在建立连接时，是否采用非阻塞模式
    bool _non_blocking_connect;
    /// Set socket to have non-blocking I/O.
    /// Default: @c true.
    // 在读写数据时，是否采用非阻塞模式
    bool _non_blocking_io;
    /// Use TCP if @c true, use UDP if @c false.
    /// Default: @c false.
    // 是否使用 TCP 协议传输报文
    bool _use_tcp;
    /// Bind to a random port.
    /// Default: @c true.
    // 是否使用随机端口发起请求
    bool _bind_random_port;
    /// Bind to this local address when using IPv6.
    /// Default: unset, bind to IN6ADDR_ANY.
    // 是否指定本地 IPv6 地址
    sockaddr const *_local_ipv6;
    /// Bind to this local address when using IPv4.
    /// Default: unset, bind to INADDRY_ANY.
    // 是否指定本地 IPv4 地址
    sockaddr const *_local_ipv4;

    // 初始化上述六个成员，默认值写在上面的注释中
    Options();

    self &setUseTcp(bool p);
    self &setNonBlockingConnect(bool p);
    self &setNonBlockingIo(bool p);
    self &setBindRandomPort(bool p);
    self &setLocalIpv6(sockaddr const *addr);
    self &setLocalIpv4(sockaddr const *addr);
  };

  // 用于保存 udp socket fd
  int fd;
  // 用于保存目标 Name Server 的 IP
  IpEndpoint ip;
  // 该 DNSConnection 对象位于 DNSHandler::con[] 数组的下标
  // 每一个 DNSHandler 最多创建 MAX_NAMED（32）个 DNSConnection 对象
  int num;
  // 链表节点，允许把 DNSConnection 放入到 DNSHandler::triggered 队列
  LINK(DNSConnection, link);
  // EventIO 对象，用来把 fd 加入到 epoll fd 中
  EventIO eio;
  // 随机数生成种子
  InkRand generator;
  // 指向关联的 DNSHandler
  DNSHandler *handler;

  // 与目标 Name Server 建立 UDP 连接，返回值保存在成员 fd 中
  int connect(sockaddr const *addr, Options const &opt = DEFAULT_OPTIONS);
  /*
                bool non_blocking_connect = NON_BLOCKING_CONNECT,
                bool use_tcp = CONNECT_WITH_TCP, bool non_blocking = NON_BLOCKING, bool bind_random_port = BIND_ANY_PORT);
  */
  // 关闭 fd
  int close();
  // 通知 DNSHandler 准备接收 DNS 响应数据
  void trigger();

  // 析构函数，简单调用了 close() 方法
  virtual ~DNSConnection();
  // 构造函数，初始化了 generator 和 ip 等成员
  DNSConnection();

  // 默认 DNSConnection::Options 配置
  static Options const DEFAULT_OPTIONS;
};
```

## 方法

在 DNSHandler 启动时，在 `DNSHandler::startEvent` 中会打开 DNSConnection：

- 当 DNS 轮询开启时
  - 遍历 `m_res->nsaddr_list[]`，
  - 通过调用 `DNSHandler::open_con` 为每一个 Name Server 建立一个 DNSConnection
- 当 DNS 轮询未开启时，
  - 仅为主要 Name Server 的建立一个 DNSConnection

### DNSHandler::open_con

```
/**
  Open (and close) connections as necessary and also assures that the
  epoll fd struct is properly updated.

  参数：
    - target
      - 为目标的 IP 及端口
      - 如果为 NULL 表示使用当前地址重建连接
    - failed 当连接失败时，是否允许该 Name Server 继续工作（默认：false）
    - icon 表示使用 DNSHandler::con[] 数组里的哪一个 DNSConnection 发起此连接（默认：0）
*/
void
DNSHandler::open_con(sockaddr const *target, bool failed, int icon)
{
  ip_port_text_buffer ip_text;
  // 获得 ET_DNS 线程的 Poll 描述符
  PollDescriptor *pd = get_PollDescriptor(dnsProcessor.thread);

  // 如果 icon 为 0 并且指定了 target，就将 target 保存到 DNSHandler::ip
  if (!icon && target) {
    ats_ip_copy(&ip, target);
  } else if (!target) {
    // 如果 target 为 NULL，那么就使用 DNSHandler::ip 作为目标 Name Server 的 IP
    target = &ip.sa;
  }

  Debug("dns", "open_con: opening connection %s", ats_ip_nptop(target, ip_text, sizeof ip_text));

  // 如果之前 fd 已经打开，则先将 fd 从 Poll 描述符中取消注册，然后关闭 fd
  if (con[icon].fd != NO_FD) { // Remove old FD from epoll fd
    con[icon].eio.stop();
    con[icon].close();
  }

  // 调用 DNSConnection::connect() 打开连接
  if (con[icon].connect(target, DNSConnection::Options()
                                  .setNonBlockingConnect(true)
                                  .setNonBlockingIo(true)
                                  .setUseTcp(false)
                                  .setBindRandomPort(true)
                                  .setLocalIpv6(&local_ipv6.sa)
                                  .setLocalIpv4(&local_ipv4.sa)) < 0) {
    // 如果打开连接失败，同时不允许该 Name Server 继续工作
    Debug("dns", "opening connection %s FAILED for %d", ip_text, icon);
    if (!failed) {
      // 如果 DNS 轮询开启，则标记此 Name Server 服务不可用
      if (dns_ns_rr)
        rr_failure(icon);
      // 如果 DNS 轮询未开启，则切换到备用 Name Server
      else
        failover();
    }
    return;
  } else {
    // 如果连接建立成功
    // 标记该连接的服务处于可用状态
    ns_down[icon] = 0;
    // 将该连接的 fd 注册到 Poll 描述符
    // 注册完成之后，NetHandler 调用 epoll_wait 时就可能会接收到这个 socket fd 上新数据到达的事件
    // 这里调用的是 EventIO::start(EventLoop l, DNSConnection *vc, int events) 方法
    if (con[icon].eio.start(pd, &con[icon], EVENTIO_READ) < 0) {
      Error("[iocore_dns] open_con: Failed to add %d server to epoll list\n", icon);
    } else {
      // 将下标保存到 num，以作为校验
      con[icon].num = icon;
      Debug("dns", "opening connection %s SUCCEEDED for %d", ip_text, icon);
    }
  }
}
```

以下是：EventIO::start(EventLoop l, DNSConnection *vc, int events) 方法的代码：

```
EventIO::start(EventLoop l, DNSConnection *vc, int events)
{
  // 注意这个类型，在 NetHandler::mainNetEvent 中会进行判断
  type = EVENTIO_DNS_CONNECTION;
  return start(l, vc->fd, (Continuation *)vc, events);
}
```

### DNSConnection::connect

```
int
DNSConnection::connect(sockaddr const *addr, Options const &opt)
//                       bool non_blocking_connect, bool use_tcp, bool non_blocking, bool bind_random_port)
{
  ink_assert(fd == NO_FD);
  ink_assert(ats_is_ip(addr));

  int res = 0;
  short Proto;
  uint8_t af = addr->sa_family;
  IpEndpoint bind_addr;
  size_t bind_size = 0;

  // connect 方法允许与 Name Server 建立 TCP 连接或 UDP 连接
  // 但是目前没有看到设置 use_tcp 为 true 的代码
  if (opt._use_tcp) {
    Proto = IPPROTO_TCP;
    if ((res = socketManager.socket(af, SOCK_STREAM, 0)) < 0)
      goto Lerror;
  } else {
    Proto = IPPROTO_UDP;
    if ((res = socketManager.socket(af, SOCK_DGRAM, 0)) < 0)
      goto Lerror;
  }

  // 将 socket fd 保存到成员 fd
  fd = res;

  memset(&bind_addr, 0, sizeof bind_addr);
  bind_addr.sa.sa_family = af;

  // 根据目标 IP 的地址类型选择本地 IP 的地址
  if (AF_INET6 == af) {
    if (ats_is_ip6(opt._local_ipv6)) {
      ats_ip_copy(&bind_addr.sa, opt._local_ipv6);
    } else {
      bind_addr.sin6.sin6_addr = in6addr_any;
    }
    bind_size = sizeof(sockaddr_in6);
  } else if (AF_INET == af) {
    if (ats_is_ip4(opt._local_ipv4))
      ats_ip_copy(&bind_addr.sa, opt._local_ipv4);
    else
      bind_addr.sin.sin_addr.s_addr = INADDR_ANY;
    bind_size = sizeof(sockaddr_in);
  } else {
    ink_assert(!"Target DNS address must be IP.");
  }

  // 选择随机的本地端口
  if (opt._bind_random_port) {
    int retries = 0;
    // BUG? 重复定义 bind_addr 和 bind_size
    // 上面通过 opt 传入的 local_ipv4 和 local_ipv6 地址将失效
    IpEndpoint bind_addr;
    size_t bind_size = 0;
    memset(&bind_addr, 0, sizeof bind_addr);
    bind_addr.sa.sa_family = af;
    if (AF_INET6 == af) {
      bind_addr.sin6.sin6_addr = in6addr_any;
      bind_size = sizeof bind_addr.sin6;
    } else {
      bind_addr.sin.sin_addr.s_addr = INADDR_ANY;
      bind_size = sizeof bind_addr.sin;
    }
    // 随机尝试选择 10000 个本地端口，当 bind 成功后，停止尝试
    while (retries++ < 10000) {
      ip_port_text_buffer b;
      // 生成随机数
      uint32_t p = generator.random();
      // 在 16000 ~ 60000 之间随机选择一个端口
      p = static_cast<uint16_t>((p % (LAST_RANDOM_PORT - FIRST_RANDOM_PORT)) + FIRST_RANDOM_PORT);
      ats_ip_port_cast(&bind_addr.sa) = htons(p); // stuff port in sockaddr.
      Debug("dns", "random port = %s\n", ats_ip_nptop(&bind_addr.sa, b, sizeof b));
      // 尝试执行 bind 操作，失败继续下一次尝试，成功则跳出
      if ((res = socketManager.ink_bind(fd, &bind_addr.sa, bind_size, Proto)) < 0) {
        continue;
      }
      goto Lok;
    }
    // 如果尝试 10000 次都无法成功输出警告信息
    Warning("unable to bind random DNS port");
    // BUG？ 这里 bind 失败的时候，是否应该跳转到失败处理？
  Lok:
    ;
  } else if (ats_is_ip(&bind_addr.sa)) {
    // 由系统自动分配一个本地端口
    ip_text_buffer b;
    res = socketManager.ink_bind(fd, &bind_addr.sa, bind_size, Proto);
    if (res < 0)
      Warning("Unable to bind local address to %s.", ats_ip_ntop(&bind_addr.sa, b, sizeof b));
    // BUG？ 这里 bind 失败的时候，是否应该跳转到失败处理？
  }

  // 是否采用非阻塞 connect
  if (opt._non_blocking_connect)
    if ((res = safe_nonblocking(fd)) < 0)
      goto Lerror;

  // 设置 TCP 选项，这里应该判断一下 opt._use_tcp 的值
// cannot do this after connection on non-blocking connect
#ifdef SET_TCP_NO_DELAY
  if (opt._use_tcp)
    if ((res = safe_setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, SOCKOPT_ON, sizeof(int))) < 0)
      goto Lerror;
#endif
#ifdef RECV_BUF_SIZE
  socketManager.set_rcvbuf_size(fd, RECV_BUF_SIZE);
#endif
#ifdef SET_SO_KEEPALIVE
  // enables 2 hour inactivity probes, also may fix IRIX FIN_WAIT_2 leak
  if ((res = safe_setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, SOCKOPT_ON, sizeof(int))) < 0)
    goto Lerror;
#endif

  // 将目标 addr 保存到成员 ip 内
  ats_ip_copy(&ip.sa, addr);
  // 发起连接，如果是 TCP 模式会触发三次握手，如果是 UDP 模式仅仅在 Kernel 中建立映射关系
  res = ::connect(fd, addr, ats_ip_size(addr));

  // 以下情况，我们认为 connect(2) 没有遇到错误
  //   - 返回值为 0
  //     - 表示阻塞模式下完成了 TCP 三次握手 或 在 Kernel 内建立了 UDP 报文的地址映射关系
  //   - 返回值为 -1 但是 errno 为 EINPROGRESS
  //     - 表示非阻塞模式下发起了 TCP 三次握手，但是是否完成连接建立仍然要判断 socket fd 可写
  // BUG? 以下情况，通常表示遇到了错误，但是可以通过重试解决
  //   - 返回值为 -1 但是 errno 为 EWOULDBLOCK / EAGAIN
  //     - 表示非阻塞模式下连接未能建立，需要再次调用 connect 方法重建连接
  //   - 返回值为 -1 但是 errno 为 EINTR
  //     - 表示阻塞模式下连接未能建立，需要再次调用 connect 方法重建连接
  if (!res || ((res < 0) && (errno == EINPROGRESS || errno == EWOULDBLOCK))) {
    // 连接建立成功
    // 如果使用阻塞方式进行 connect 但是设置了非阻塞通信时，将 socket fd 切换到非阻塞模式
    if (!opt._non_blocking_connect && opt._non_blocking_io)
      if ((res = safe_nonblocking(fd)) < 0)
        goto Lerror;
    // Shouldn't we turn off non-blocking when it's a non-blocking connect
    // and blocking IO?
    // 上面的注释提到：
    // 是否要在选择了采用非阻塞方式进行 connect 但是设置了阻塞通信时，将 socket fd 切换到阻塞模式？
    // 这部分的代码没有实现，到底应该不应该实现？
  } else
    goto Lerror;

  return 0;

// 任何时候遇到错误只需要简单的调用 close() 关闭 socket fd
Lerror:
  if (fd != NO_FD)
    close();
  return res;
}
```

### DNSConnection::close

```
int
DNSConnection::close()
{
  // 避免关闭标准输入输出，从而导致无法输入 debug 日志
  // don't close any of the standards
  if (fd >= 2) {
    // 安全的关闭 socket fd
    // 这里先将成员 fd 保存到本地变量，然后将成员 fd 重置为 -1 后，再关闭 fd_save 是为了避免竞争问题
    // 成员变量仅有当前运行的 close() 实例可以访问
    int fd_save = fd;
    // fd 关闭后，将其设置为 NO_FD（-1）
    fd = NO_FD;
    return socketManager.close(fd_save);
  } else {
    // 这里应该算是异常情况，说明该对象的内存可能被破坏了，
    // 任何时候 fd 都不应该小于 2
    fd = NO_FD;
    return -EBADF;
  }
}
```

### DNSConnection::trigger

当 NetHandler 通过 `epoll_wait` 发现 DNSConnection 的 socket fd 上有数据待接收时，将会调用此方法，将 DNSConnection 放入 `DNSHandler::triggered` 的队列。

由于在 `DNSHandler::open_con` 中，将 socket fd 注册到了 DNSHandler 所在线程 ET_DNS[0] 内的 Poll 描述符，所以只有线程 ET_DNS[0] 内运行的 NetHandler 才会收到 DNSConnection 上的数据可读事件。

所以也就只有 ET_DNS[0] 内的 NetHandler 才会调用 `DNSConnection::trigger` 方法，而 DNSHandler 的 mutex 与 线程 ET_DNS[0] 共享，因此只能被线程 ET_DNS[0] 里运行的状态机访问，所以 `DNSHandler::triggered` 队列可以不是原子队列。

```
void
DNSConnection::trigger()
{
  handler->triggered.enqueue(this);
}
```

以下是 NetHandler::mainNetEvent 的相关代码：

```
  vc = NULL;
  for (int x = 0; x < pd->result; x++) {
    epd = (EventIO *)get_ev_data(pd, x);
    if (epd->type == EVENTIO_READWRITE_VC) {

... 省略部分代码 ...

    } else if (epd->type == EVENTIO_DNS_CONNECTION) {
      // 判断为 EVENTIO_DNS_CONNECTION 类型，即可知这是一个 DNSConnection
      if (epd->data.dnscon != NULL) {
        // 调用 trigger 方法将其放入到 DNSHandler::triggered 队列内
        epd->data.dnscon->trigger(); // Make sure the DNSHandler for this con knows we triggered
#if defined(USE_EDGE_TRIGGER)
        epd->refresh(EVENTIO_READ);
#endif
      }
    } else if (epd->type == EVENTIO_ASYNC_SIGNAL) {
      net_signal_hook_callback(trigger_event->ethread);
    }
    ev_next_event(pd, x);
  }
```


# 参考资料

- [P_DNSConnection.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/P_DNSConnection.h)
- [DNS.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/DNS.cc)
- [P_UnixNet.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/net/P_UnixNet.h)
- [UnixNet.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/net/UnixNet.cc)

