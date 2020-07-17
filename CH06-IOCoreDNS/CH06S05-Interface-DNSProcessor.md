# 接口：DNSProcessor

dnsProcessor 是全局单例，提供了四个接口，用于实现三种不同类型的 DNS 解析接口：

- 解析 SRV 记录
  - `getSRVbyname(Continuation *cont, const char *name, Options const &opt)`
- 解析域名的 IP 地址
  - `gethostbyname(Continuation *cont, const char *name, Options const &opt)`
  - `gethostbyname(Continuation *cont, const char *name, int len, Options const &opt)`
- 反向地址解析
  - `gethostbyaddr(Continuation *cont, IpAddr const *addr, Options const &opt)`

这四个接口均向调用者返回 Action 对象，以便于调用者取消未完成的 DNS 解析任务。

当解析成功或失败（超时等原因），通过回调 ```DNS_EVENT_LOOKUP``` 事件和 HostEnt 类型向 cont 传递结果：

- 如果 HostEnt 指向 NULL，则表示解析失败，否则指向一个有效的 HostEnt 实例，可从中获得解析结果
- 在回调返回后，HostEnt 对象将会被释放，如在状态机中需要保留 HostEnt 成员的值，必须使用内存复制的方式

## 定义

```
struct DNSProcessor : public Processor {
  // Public Interface
  //

  // 定义 Options 用于在发起 DNS 解析请求时，传递参数
  /// Options for host name resolution.
  struct Options {
    typedef Options self; ///< Self reference type.

    /// Query handler to use.
    /// Default: single threaded handler.
    // 这里的 DNSHandler 可能是全局默认的 DNSHandler，也可能是 SplitDNS 功能的 DNSHandler
    DNSHandler *handler;
    /// Query timeout value.
    /// Default: @c DEFAULT_DNS_TIMEOUT (or as set in records.config)
    // 设置超时时间，单位：秒
    int timeout; ///< Timeout value for request.
    /// Host resolution style.
    /// Default: IPv4, IPv6 ( @c HOST_RES_IPV4 )
    // 设置解析的地址类型，如 IPv4 或 IPv6
    HostResStyle host_res_style;

    /// Default constructor.
    // 构造函数，初始化上述成员
    Options();

    // 用于设置上述三个成员的方法
    /// Set @a handler option.
    /// @return This object.
    self &setHandler(DNSHandler *handler);

    /// Set @a timeout option.
    /// @return This object.
    self &setTimeout(int timeout);

    /// Set host query @a style option.
    /// @return This object.
    self &setHostResStyle(HostResStyle style);

    /// Reset to default constructed values.
    /// @return This object.
    // 重置 Options 对象
    self &reset();
  };

  // DNS lookup
  //   calls: cont->handleEvent( DNS_EVENT_LOOKUP, HostEnt *ent) on success
  //          cont->handleEvent( DNS_EVENT_LOOKUP, NULL) on failure
  // NOTE: the HostEnt *block is freed when the function returns
  //
  // 发起 DNS 解析操作的接口
  // 通过 *cont 指向的状态机接收回调事件（DNS_EVENT_LOOKUP）和解析结果，当结果为：
  //   - NULL 时，表示解析失败
  //   - 非 NULL 时，表示解析成功
  // 回调时传入的 HostEnt 对象，将会在状态机返回后销毁，因此状态机在收到 HostEnt 对象后，
  // 如需要保留某些信息，应该通过内存复制的方式保留一份。
  Action *gethostbyname(Continuation *cont, const char *name, Options const &opt);
  Action *getSRVbyname(Continuation *cont, const char *name, Options const &opt);
  Action *gethostbyname(Continuation *cont, const char *name, int len, Options const &opt);
  Action *gethostbyaddr(Continuation *cont, IpAddr const *ip, Options const &opt);


  // Processor API
  //
  /* currently dns system uses event threads
   * dont pass any value to the call */
  // DNS 子系统的初始化，包含以下几个部分：
  //   - 加载 records.config 内的各种参数
  //   - 启动 ET_DNS 线程
  //   - 如果开了 SplitDNS 功能，则初始化 SplitDNS 的配置
  //   - 最后调用 dns_init() 和 open()
  // 目前 ET_DNS 线程仅支持单线程，可以独立运行，或与 ET_NET[0] 共享
  int start(int no_of_extra_dns_threads = 0, size_t stacksize = DEFAULT_STACKSIZE);

  // Open/close a link to a 'named' (done in start())
  //
  // 创建 DNSHandler，并使其运行在 ET_DNS 线程
  void open(sockaddr const *ns = 0);

  // 构造函数
  // 初始化下面的五个成员
  DNSProcessor();

  // private:
  //
  // 指向 ET_DNS[0] 线程
  EThread *thread;
  // 指向 全局 DNSHandler
  DNSHandler *handler;
  // 由 ink_res_init 初始化，ts_imp_res_state 结构在 ink_resolver.h 中定义
  ts_imp_res_state l_res;
  // 当发起 DNS 解析请求时，使用的本地 IPv6 或 IPv4 地址
  IpEndpoint local_ipv6;
  IpEndpoint local_ipv4;

  /** Internal implementation for all getXbyY methods.
      For host resolution queries pass @c T_A for @a type. It will be adjusted
      as needed based on @a opt.host_res_style.

      For address resolution ( @a type is @c T_PTR ), @a x should be a
      @c sockaddr cast to  @c char @c const* .
   */
  // 前面的四个 DNS 解析操作的接口实际上是这个函数的一个外壳，最终都是向 getby() 传入不同的值来完成 DNS 解析操作
  Action *getby(const char *x, int len, int type, Continuation *cont, Options const &opt);

  // 初始化 Name Server 的配置，包括：
  // 从以下配置项或文件处获得 Name Server 的 IP
  //   - proxy.config.dns.nameservers
  //   - /etc/resolv.conf (路径由 proxy.config.dns.resolv_conf 指定)
  // 是否开启 search default domain 功能：
  //   - proxy.config.dns.search_default_domains
  // 设置本地 IPv4 及 IPv6 地址，用于发起 DNS 解析请求
  void dns_init();
};

//
// Global data
//
// 全局单例
extern DNSProcessor dnsProcessor;
```

## 方法

四个解析接口都是 getby() 方法的壳函数，通过调用 getby() 方法，传入不同的类型：`T_SRV`，`T_A`，`T_PTR` 完成不同的解析任务。

```
inline Action *
DNSProcessor::getSRVbyname(Continuation *cont, const char *name, Options const &opt)
{
  return getby(name, 0, T_SRV, cont, opt);
}

inline Action *
DNSProcessor::gethostbyname(Continuation *cont, const char *name, Options const &opt)
{
  return getby(name, 0, T_A, cont, opt);
}

inline Action *
DNSProcessor::gethostbyname(Continuation *cont, const char *name, int len, Options const &opt)
{
  return getby(name, len, T_A, cont, opt);
}

inline Action *
DNSProcessor::gethostbyaddr(Continuation *cont, IpAddr const *addr, Options const &opt)
{
  return getby(reinterpret_cast<char const *>(addr), 0, T_PTR, cont, opt);
}
```

### DNSProcessor::getby

用于实现 DNS 解析操作的底层，创建 DNSEntry 对象并启动 DNS 解析任务。

```
Action *
DNSProcessor::getby(const char *x, int len, int type, Continuation *cont, Options const &opt)
{
  Debug("dns", "received query %s type = %d, timeout = %d", x, type, opt.timeout);
  if (type == T_SRV) {
    Debug("dns_srv", "DNSProcessor::getby attempting an SRV lookup for %s, timeout = %d", x, opt.timeout);
  }
  // 创建 DNSEntry 对象
  DNSEntry *e = dnsEntryAllocator.alloc();
  // 使用全局变量 `dns_retries` 对 DNSEntry::retries 进行赋值
  e->retries = dns_retries;
  // 调用 DNSEntry::init 完成初始化
  e->init(x, len, type, cont, opt);
  // 对 DNSHandler 上锁，此时 e->mutex 指向 DNSHandler::mutex，而 DNSHandler 与 ET_DNS[0] 共享同一个 mutex，
  // 因此，这里相当于是判断当前线程是否为 ET_DNS[0]。还需要注意 ET_DNS[0] 还可能等于 ET_NET[0] 或 ET_CALL[0]。
  MUTEX_TRY_LOCK(lock, e->mutex, this_ethread());
  // 成员 thread 指向 ET_DNS[0]
  // 在上锁失败后，将 DNSEntry 重新调度到 ET_DNS[0] 这样就一定可以拿到 e->mutex 的锁
  // 如果上锁成功就直接调用 DNSEntry::mainEvent 进行处理
  if (!lock.is_locked())
    thread->schedule_imm(e);
  else
    e->handleEvent(EVENT_IMMEDIATE, 0);
  // BUG：
  //   - 在 DNSEntry::mainEvent 中可能会出现失败导致 DNSEntry 直接被销毁
  //   - 因此下面在返回 &e->action 时，可能会存在内存泄露
  // 参考：
  //   - [Pull Request #3871](https://github.com/apache/trafficserver/pull/3871)
  //   - 该修复只是一个临时解决访问，截止该文档编写时，未被官方接受 / 合并
  return &e->action;
}
```

### DNSProcessor::dns_init

加载 Name Server 的 IP 地址列表，设置 default domain，search list 等基础信息。

```
//
// Initialization
//
void
DNSProcessor::dns_init()
{
  // 获取本机的 hostname
  gethostname(try_server_names[0], 255);
  Debug("dns", "localhost=%s\n", try_server_names[0]);
  Debug("dns", "Round-robin nameservers = %d\n", dns_ns_rr);

  // 遍历 dns_ns_list (proxy.config.dns.nameservers)
  // 结果保存在 nameserver 本地数组，数量保存在 nserv 本地变量
  IpEndpoint nameserver[MAX_NAMED];
  size_t nserv = 0;

  if (dns_ns_list) {
    Debug("dns", "Nameserver list specified \"%s\"\n", dns_ns_list);
    int i;
    char *last;
    char *ns_list = ats_strdup(dns_ns_list);
    char *ns = (char *)strtok_r(ns_list, " ,;\t\r", &last);

    for (i = 0, nserv = 0; (i < MAX_NAMED) && ns; ++i) {
      Debug("dns", "Nameserver list - parsing \"%s\"\n", ns);
      bool err = false;
      int prt = DOMAIN_SERVICE_PORT;
      char *colon = 0; // where the port colon is.
      // Check for IPv6 notation.
      if ('[' == *ns) {
        char *ndx = strchr(ns + 1, ']');
        if (ndx) {
          if (':' == ndx[1])
            colon = ndx + 1;
        } else {
          err = true;
          Warning("Unmatched '[' in address for nameserver '%s', discarding.", ns);
        }
      } else
        colon = strchr(ns, ':');

      if (!err && colon) {
        *colon = '\0';
        // coverity[secure_coding]
        if (sscanf(colon + 1, "%d%*s", &prt) != 1) {
          Debug("dns", "Unable to parse port number '%s' for nameserver '%s', discardin.", colon + 1, ns);
          Warning("Unable to parse port number '%s' for nameserver '%s', discarding.", colon + 1, ns);
          err = true;
        }
      }

      if (!err && 0 != ats_ip_pton(ns, &nameserver[nserv].sa)) {
        Debug("dns", "Invalid IP address given for nameserver '%s', discarding", ns);
        Warning("Invalid IP address given for nameserver '%s', discarding", ns);
        err = true;
      }

      if (!err) {
        ip_port_text_buffer buff;

        ats_ip_port_cast(&nameserver[nserv].sa) = htons(prt);

        Debug("dns", "Adding nameserver %s to nameserver list", ats_ip_nptop(&nameserver[nserv].sa, buff, sizeof(buff)));
        ++nserv;
      }

      ns = (char *)strtok_r(NULL, " ,;\t\r", &last);
    }
    ats_free(ns_list);
  }
  // 调用 ink_res_init 初始化域名解析的底层数据
  // 该方法主要对 l_res 的以下成员完成初始化：
  //   - nscount （Name Server 的数量）
  //   - nsaddr_list[] （保存 Name Server 的 IP 地址）
  //   - dnsrch （domain search list）
  //   - defdname （default domain）
  // The default domain (5th param) and search list (6th param) will
  // come from /etc/resolv.conf.
  // 其中第五个参数（default domain）和第六个参数（search list）会从 /etc/resolv.conf 中读取
  if (ink_res_init(&l_res, nameserver, nserv, dns_search, NULL, NULL, dns_resolv_conf) < 0)
    Warning("Failed to build DNS res records for the servers (%s).  Using resolv.conf.", dns_ns_list);

  // Check for local forced bindings.
  // 如果指定了本地 IPv6 地址，那么将在发起 DNS 解析请求时作为客户端地址
  if (dns_local_ipv6) {
    if (0 != ats_ip_pton(dns_local_ipv6, &local_ipv6)) {
      ats_ip_invalidate(&local_ipv6);
      Warning("Invalid IP address '%s' for dns.local_ipv6 value, discarding.", dns_local_ipv6);
    } else if (!ats_is_ip6(&local_ipv6.sa)) {
      ats_ip_invalidate(&local_ipv6);
      Warning("IP address '%s' for dns.local_ipv6 value was not IPv6, discarding.", dns_local_ipv6);
    }
  }
  // 如果指定了本地 IPv4 地址，那么将在发起 DNS 解析请求时作为客户端地址
  if (dns_local_ipv4) {
    if (0 != ats_ip_pton(dns_local_ipv4, &local_ipv4)) {
      ats_ip_invalidate(&local_ipv4);
      Warning("Invalid IP address '%s' for dns.local_ipv4 value, discarding.", dns_local_ipv4);
    } else if (!ats_is_ip4(&local_ipv4.sa)) {
      ats_ip_invalidate(&local_ipv4);
      Warning("IP address '%s' for dns.local_ipv4 value was not IPv4, discarding.", dns_local_ipv4);
    }
  }
}
```

### DNSProcessor::open

用于创建 DNSHandler 对象，并使其运行在 `ET_DNS[0]` 线程

```
void
DNSProcessor::open(sockaddr const *target)
{
  // 创建 DNSHandler 对象
  DNSHandler *h = new DNSHandler;

  // DNSHandler 与 ET_DNS[0] 共享同一个 Mutex
  // 这意味着只有在 ET_DNS[0] 才可以访问 DNSHandler
  h->mutex = thread->mutex;
  h->m_res = &l_res;
  ats_ip_copy(&h->local_ipv4.sa, &local_ipv4.sa);
  ats_ip_copy(&h->local_ipv6.sa, &local_ipv6.sa);

  if (target)
    ats_ip_copy(&h->ip, target);
  else
    ats_ip_invalidate(&h->ip); // marked to use default.

  // 将 DNSHandler 保存到成员 handler
  if (!dns_handler_initialized)
    handler = h;

  // 在 ET_DNS[0] 上启动 DNSHandler
  SET_CONTINUATION_HANDLER(h, &DNSHandler::startEvent);
  thread->schedule_imm(h);
}
```

# 参考资料

- [I_DNSProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/I_DNSProcessor.h)
- [DNS.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/DNS.cc)

