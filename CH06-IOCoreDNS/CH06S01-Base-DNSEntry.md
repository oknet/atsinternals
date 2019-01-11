# 基础组件：DNSEntry

DNSEntry 的意义，相当于 Event System 里的 Event。

DNS 子系统根据 DNSEntry 内填写的解析请求，向互联网发起 DNS 查询，在得到结果之后，回调发起查询请求的状态机，如果状态机提前结束，可以取消 DNSEntry。

## 定义

每一个 DNS 解析请求对应了一个 DNSEntry 对象，其内置了对 DNS 查询操作的超时处理，同时也用于保存和向状态机传递查询结果。

```
/**
  One DNSEntry is allocated per outstanding request. This continuation
  handles TIMEOUT events for the request as well as storing all
  information about the request and its status.

*/
struct DNSEntry : public Continuation {
  // 根据 DNS 协议的要求，为每一次 DNS 解析请求，生成的查询 ID
  // MAX_DNS_RETRIES 默认定义为 9。（P_DNSProcessor.h）
  int id[MAX_DNS_RETRIES];
  // DNS 查询请求的类型：
  //   - T_SRV DNS SRV记录解析
  //   - T_A DNS 正向IPv4地址解析
  //   - T_AAAA DNS 正向IPv6地址解析
  //   - T_PTR DNS 反向地址解析
  // 详细列表可以参考：arpa/nameser.h 和 arpa/nameser_compat.h
  int qtype;                   ///< Type of query to send.
  // 枚举类型：构建 DNS 请求时，首选的 IP 地址类型（在 ink_resolver.h 中定义）
  //   - HOST_RES_NONE = 0  默认值，用于表示未初始化
  //   - HOST_RES_IPV4      希望解析结果首选 IPV4 的地址
  //   - HOST_RES_IPV4_ONLY 希望解析结果仅包含 IPV4 的地址
  //   - HOST_RES_IPV6      希望解析结果首选 IPV6 的地址
  //   - HOST_RES_IPV6_ONLY 希望解析结果仅包含 IPV6 的地址
  HostResStyle host_res_style; ///< Preferred IP address family.
  // 在解析失败时，重新尝试的次数
  int retries;
  // 使用哪一个 Name Server 进行解析
  // 多个 Name Server 及其对应的统计数据被放入数组中，以 which_ns 作为数组的下标来访问这些数组
  int which_ns;
  // DNSEntry 诞生的时间戳
  // 在收到 DNS 查询响应时，可以根据响应结果是成功还是失败，分别统计成功用时或失败用时
  ink_hrtime submit_time;
  // DNS 查询请求发送出去的时间戳
  // 在解析 DNS 查询响应时，可以用当前时间与 send_time 的差，得到 DNS 查询的响应时间
  ink_hrtime send_time;
  // 待解析的域名，如果是反向地址解析则是 [ipv4addr].in-addr.arpa 或 [ipv6addr].ip6.arpa
  // MAXDNAME 为 1025（在 arpa/nameser.h 和 arpa/nameser_compat.h 中定义）
  char qname[MAXDNAME];
  // qname 的长度
  int qname_len;
  // DNS 提供了一个 domain expansion 的功能，会自动把指定的域名添加到主机名后面，然后进行解析
  // 例如：当我们在浏览器地址栏输入 www 时，这是一个主机名，需要在其后追加域名后才可以进行解析，
  // 此时可以执行自动添加的域名后缀，例如：google.com, google.com.hk 等
  // 此时 DNS 子系统会先尝试对 www.google.com 进行解析，如果失败，再尝试对 www.google.com.hk 进行解析。
  // 这里记录了未扩展之前的域名长度（即上述例子中 www 的长度 3），这样方便在多个扩展域名间轮换
  int orig_qname_len;
  // 上述 domain expansion 功能的附带参数。
  // 列出了会进行尝试的域名后缀，是一个字符串数组，可能的字符串如："com", "edu", "gov"
  // 每一次轮换扩展域名之后会重置 retries 成员
  char **domains;
  // 记录 DNS 查询请求是从哪一个线程发起的
  EThread *submit_thread;
  // Action 对象，用于将查询结果回调给状态机，或者取消该次查询的回调
  Action action;
  // 使用定时事件控制 DNS 查询任务的超时
  // 同时也用来保存延迟重试的 Event
  Event *timeout;
  // 保存解析结果
  // 这里采用了智能指针 Ptr<> 来定义，用于在多个 DNSEntry 共享同一个解析结果时，对解析结果自动进行释放
  Ptr<HostEnt> result_ent;
  // 由哪个 DNSHandler 负责此次 DNS 查询的任务
  // 用于在全局 DNSHandler 与 SplitDNS 创建的 DNSHandler 之间进行选择
  // 这些 DNSHandler 都运行在 ET_DNS 线程组里
  DNSHandler *dnsH;
  // 当该 DNS 查询请求发出之后，设置此标志为 true，表示正在等待 DNS 的查询结果
  // 在收到查询结果后，重置为 false，如果遇到查询失败也会重置为 false，下一次重试请求发出后，再设置此标志为 true
  bool written_flag;
  // 只要发出过一次 DNS 查询请求，此标志就设置为 true，直到 DNSEntry 被销毁
  bool once_written_flag;
  // 用于表示这是最后一次尝试发起 DNS 解析请求
  bool last;
  // 由于 DNS 解析请求可能会出现同时请求相同域名的情况，因此 DNS 子系统将对相同域名的解析请求使用 dups 队列保存起来，
  // 这样当第一个解析请求得到结果后，就可以将结果同时传递给所有对同一域名进行 DNS 解析的状态机，避免多次发送 DNS 解析请求
  LINK(DNSEntry, dup_link);
  Que(DNSEntry, dup_link) dups;

  // 主事件处理函数
  int mainEvent(int event, Event *e);
  // 延迟处理事件函数
  // 在 DNSHandler 未准备好时，延迟等待
  int delayEvent(int event, Event *e);
  // 尝试以同步回调的方式将解析结果传递给状态机，最终会调用 postEvent 方法
  int post(DNSHandler *h, HostEnt *ent);
  // 以异步回调的方式将解析结果传递给状态机，回调完成后，负责释放 DNSEntry 的资源
  int postEvent(int event, Event *e);
  // 初始化函数，对上面的成员进行初始化
  // DNSEntry 共享 DNSHandler 的 mutex
  void init(const char *x, int len, int qtype_arg, Continuation *acont, DNSProcessor::Options const &opt);

  // 构造函数，初始化成员
  // DNSEntry 是通过 dnsEntryAllocator 进行对象资源分配的，因此其构造函数并不会真正的执行。
  DNSEntry()
    : Continuation(NULL), qtype(0), host_res_style(HOST_RES_NONE), retries(DEFAULT_DNS_RETRIES), which_ns(NO_NAMESERVER_SELECTED),
      submit_time(0), send_time(0), qname_len(0), orig_qname_len(0), domains(0), timeout(0), result_ent(0), dnsH(0),
      written_flag(false), once_written_flag(false), last(false)
  {
    for (int i = 0; i < MAX_DNS_RETRIES; i++)
      id[i] = -1;
    memset(qname, 0, MAXDNAME);
  }
};

```

## 方法

DNSEntry 对象在 `DNSProcessor::getby()` 方法内创建，该方法使用全局变量 `dns_retries` 对 DNSEntry::retries 进行赋值后，就调用了 `DNSEntry::init()` 方法，使用传入的参数完成对 DNSEntry 的初始化。

### DNSEntry::init

```
DNSEntry::init(const char *x, int len, int qtype_arg, Continuation *acont, DNSProcessor::Options const &opt)
{
  // 根据传入的参数对 DNSEntry 进行初始化设置
  qtype = qtype_arg;
  host_res_style = opt.host_res_style;
  if (is_addr_query(qtype)) {
    // adjust things based on family preference.
    if (HOST_RES_IPV4 == host_res_style || HOST_RES_IPV4_ONLY == host_res_style) {
      qtype = T_A;
    } else if (HOST_RES_IPV6 == host_res_style || HOST_RES_IPV6_ONLY == host_res_style) {
      qtype = T_AAAA;
    }
  }
  submit_time = Thread::get_hrtime();
  action = acont;
  submit_thread = acont->mutex->thread_holding;

  // 选择 DNSHandler
#ifdef SPLIT_DNS
  if (SplitDNSConfig::gsplit_dns_enabled) {
    dnsH = opt.handler ? opt.handler : dnsProcessor.handler;
  } else {
    dnsH = dnsProcessor.handler;
  }
#else
  dnsH = dnsProcessor.handler;
#endif // SPLIT_DNS

  // BUG? 直接修改 DNSHandler 的成员？
  dnsH->txn_lookup_timeout = opt.timeout;

  // 共享 DNSHandler 的 Mutex
  mutex = dnsH->mutex;

  // 如果解析请求是希望获得对应域名的 A，AAAA 或 SRV 记录
  if (is_addr_query(qtype) || qtype == T_SRV) {
    if (len) {
      len = len > (MAXDNAME - 1) ? (MAXDNAME - 1) : len;
      memcpy(qname, x, len);
      qname[len] = 0;
      orig_qname_len = qname_len = len;
    } else {
      qname_len = ink_strlcpy(qname, x, MAXDNAME);
      orig_qname_len = qname_len;
    }
  // 如果如果解析请求是希望获得对应 IP 地址的 PTR 记录
  // 通过 make_ipv4/ipv6_ptr 转换为 arpa 格式字符串
  } else { // T_PTR
    IpAddr const *ip = reinterpret_cast<IpAddr const *>(x);
    if (ip->isIp6())
      make_ipv6_ptr(&ip->_addr._ip6, qname);
    else if (ip->isIp4())
      make_ipv4_ptr(ip->_addr._ip4, qname);
    else
      ink_assert(!"T_PTR query to DNS must be IP address.");
  }

  // 设置事件处理函数
  SET_HANDLER((DNSEntryHandler)&DNSEntry::mainEvent);
}
```

### get_entry

在 `DNSHandler::entries` 表格里，查找与给定域名和请求类型一致的行。

这里是否可以使用 Hash 方式进行优化？

在途 DNS 解析请求的最大数量被控制在 2048 个以内，但是 entries 包含了所有等待发送的 DNS 解析请求，这里如果把 entries 队列分离成两个表，是否可以提高遍历的效率？

```
/** Find a DNSEntry by query name and type. */
inline static DNSEntry *
get_entry(DNSHandler *h, char *qname, int qtype)
{
  // 按行遍历 `DNSHandler::entries` 表格
  for (DNSEntry *e = h->entries.head; e; e = (DNSEntry *)e->link.next) {
    if (e->qtype == qtype) {
      // 如果请求的是 A 或 AAAA 记录类型，那么 qname 里就是域名
      if (is_addr_query(qtype)) {
        // 如果域名相等，则返回该行的第一个 DNSEntry
        if (!strcmp(qname, e->qname))
          return e;
      // 否则就按照二进制方式进行比较，如果相等，则返回该行的第一个 DNSEntry
      } else if (0 == memcmp(qname, e->qname, e->qname_len))
        return e;
    }
  }
  // 找不到匹配的行，则返回 NULL
  return NULL;
}
```

### DNSEntry::mainEvent

在初始化完成后，`DNSProcessor::getby()` 方法尝试回调 DNSEntry 的事件处理函数完成 DNS 查询操作。

如果上锁失败，则重新调度 `DNSEntry` 在 `ET_DNS` 线程组执行回调。

`DNSEntry::mainEvent` 的第一步是通过 `EVENT_IMMEDIATE` 事件启动 DNS 查询请求，在查询请求发出后，则负责处理超时事件。在启动 DNS 查询请求时，要先检查 DNSHandler 是否已经初始化完成，否则应该通过 `DNSEntry::delayEvent` 延迟等待。

```
/** Handle timeout events. */
int
DNSEntry::mainEvent(int event, Event *e)
{
  switch (event) {
  default:
    ink_assert(!"bad case");
    return EVENT_DONE;
  case EVENT_IMMEDIATE: {
    // 检查 DNSHandler 是否已经准备好
    if (!dnsH)
      dnsH = dnsProcessor.handler;
    // 如果没有准备好，通过 delayEvent 延迟处理
    if (!dnsH) {
      Debug("dns", "handler not found, retrying...");
      SET_HANDLER((DNSEntryHandler)&DNSEntry::delayEvent);
      return handleEvent(event, e);
    }

    // 全局变量 dns_search 表示 proxy.config.dns.search_default_domains 是否开启
    // 如果开启 dns_search 功能，并且请求的域名不是以 “.” 结尾（参考前面以请求 www 为例的讲解）
    // trailing '.' indicates no domain expansion
    if (dns_search && ('.' != qname[orig_qname_len - 1])) {
      // 将成员 domains 指向 搜索域名列表
      domains = dnsH->m_res->dnsrch;
      // start domain expansion straight away
      // if lookup name has no '.'
      // 如果 domains 不为空，而即将解析的域名里面不包含任何的 “.”
      // 那么，直接对该域名进行 domain 扩展，将第一个 domain 追加到该域名的尾部
      if (domains && !strnchr(qname, '.', MAXDNAME)) {
        qname[orig_qname_len] = '.';
        // 这里的 qname + orig_qname_len + 1 是对上面新加入的 “.” 增加计数
        // 这里的 MAXDNAME - (orig_qname_len + 1) 是对末尾的 \0 增加计数
        qname_len = orig_qname_len + 1 + ink_strlcpy(qname + orig_qname_len + 1, *domains, MAXDNAME - (orig_qname_len + 1));
        // 移动到下一个可用的 domain
        ++domains;
      }
    } else {
      // 否则关闭 domain 扩展功能
      domains = NULL;
    }
    // 将 DNSEntry 插入到 DNSHandler 的队列里
    // 这里可以想象 DNSHandler::entries 是一个表格，
    //   - 横向和纵向都可以无限扩展
    //   - DNSEntry 将被放入到每一个单元里
    //   - 不同的域名解析请求放在不同的行里
    //      - DNSHandler::entries 队列长度表示当前有多少个不同的域名解析请求
    //   - 相同的域名解析请求追加到这一行的末尾
    //      - 每行第一个 DNSEntry 对象的 DNSEntry::dups 队列的长度则表示这一个域名有多少个状态机同时请求进行解析
    Debug("dns", "enqueing query %s", qname);
    DNSEntry *dup = get_entry(dnsH, qname, qtype);
    if (dup) {
      Debug("dns", "collapsing NS request");
      dup->dups.enqueue(this);
    } else {
      Debug("dns", "adding first to collapsing queue");
      dnsH->entries.enqueue(this);
      // 放入到 DNSHandler 的队列之后，就调用 write_dns() 方法发送 DNS 解析请求
      // BUG: 在执行 write_dns 之后，DNSEntry 可能会被释放
      //   - 在返回到 DNSProcessor::getby() 方法后，又将 DNSEntry::action 的地址返回给了调用者
      //   - 此时这个 Action 对象的地址可能指向一个已经被释放的 DNSEntry 对象
      // 参考：
      //   - [Pull Request #3871](https://github.com/apache/trafficserver/pull/3871)
      //   - 该修复只是一个临时解决访问，截止该文档编写时，未被官方接受 / 合并
      //   - 在本章节全部完成时，我会考虑进行一个完整的修复
      //
      // write_dns() 遍历 DNSHandler::entries 表格每一行第一个 DNSEntry 对象，
      // 如果该 DNSEntry 还未发送 DNS 请求，则构建一个 DNS 请求，并发送出去。
      // 当该在途 DNS 解析请求达到最大值（dns_max_dns_in_flight）时，则停止对该表格的遍历。
      // write_dns() 除了被 DNSEntry 同步调用，更多的是被 DNSHandler 周期性的调用。
      write_dns(dnsH);
      // 接下来等待 DNSEntry::post() 被调用，并传递解析结果
    }
    return EVENT_DONE;
  }
  case EVENT_INTERVAL:
    // 这里有两个功能，
    //   - 首先是处理 DNS 解析请求超时
    //   - 然后是在回调状态机无法上锁时，重新尝试回调状态机
    Debug("dns", "timeout for query %s", qname);
    // 如果 DNSHandler 有超时设置
    if (dnsH->txn_lookup_timeout) {
      // Event *timeout 是定时事件，一次性执行
      timeout = NULL;
      // 通过 dns_result 发起回调操作，将结果通知所有相关的 DNSEntry 对象及其关联的状态机
      dns_result(dnsH, this, result_ent, false); // do not retry -- we are over TXN timeout on DNS alone!
      return EVENT_DONE;
    }

    // 下面是重新尝试回调状态机的逻辑，
    // 如果没有超时控制，那么回调 EVENT_INTERVAL 事件的就只能是延迟重试

    // 如果已经发出了 DNS 解析请求
    if (written_flag) {
      Debug("dns", "marking %s as not-written", qname);
      // 重置标志
      written_flag = false;
      // 减少 DNSHandler 中在途请求的计数
      --(dnsH->in_flight);
      // 减少 DNS 解析在途请求的计数统计
      DNS_DECREMENT_DYN_STAT(dns_in_flight_stat);
    }
    // Event *timeout 是定时事件，一次性执行，这里应该放在前面一起处理
    timeout = NULL;
    dns_result(dnsH, this, result_ent, true);
    return EVENT_DONE;
  }
}
```

### DNSEntry::delayEvent

等待 DNSHandler 初始化完毕，然后重新回到 `DNSEntry::mainEvent` 继续执行 DNS 查询操作。

```
int
DNSEntry::delayEvent(int event, Event *e)
{
  (void)event;
  // 判断 DNSHandler 是否初始化完毕
  if (dnsProcessor.handler) {
    // 初始化完毕则回到 `DNSEntry::mainEvent` 继续执行
    SET_HANDLER((DNSEntryHandler)&DNSEntry::mainEvent);
    return handleEvent(EVENT_IMMEDIATE, e);
  }
  // 否则，延迟重试
  e->schedule_in(DNS_DELAY_PERIOD);
  return EVENT_CONT;
}

```

### DNSEntry::post

当 DNSHandler 收到来自 Name Server 的响应后，就会从 DNSHandler::entries 表格里找到与这个响应匹配的那一行记录，然后逐个调用这一行所有 DNSEntry 对象的 `post()` 方法。

```
int
DNSEntry::post(DNSHandler *h, HostEnt *ent)
{
  // 取消超时事件
  if (timeout) {
    timeout->cancel(this);
    timeout = NULL;
  }
  // 将传入的解析结果保存到 result_ent 成员
  result_ent = ent;
  // 由于 `post()` 方法是从 DNSHandler 发起的调用，因此在 `post()` 方法内不能阻塞 DNSHandler 的运行
  // 如果发起 DNS 解析查询的状态机所在的线程刚好与 DNSHandler 在同一个线程
  // 注：ET_DNS 可以是独立的线程组，也可以与 ET_NET 共享
  // 由于 ET_DNS 通常只有一个线程，当与 ET_NET 共享时，DNSHandler 运行在 ET_NET[0]
  if (h->mutex->thread_holding == submit_thread) {
    // 尝试对状态机上锁
    MUTEX_TRY_LOCK(lock, action.mutex, h->mutex->thread_holding);
    // 上锁失败返回 1，表示这个 DNSEntry 没有回调成功需要重新放回到 DNSHandler::entries 表格里
    // 此时 submit_thread 应该就是 ET_NET[0]，所以没有通过事件系统安排回调，而是放回表格。
    if (!lock.is_locked()) {
      Debug("dns", "failed lock for result %s", qname);
      return 1;
    }
    // 上锁成功，通过 postEvent() 回调状态机
    postEvent(0, 0);
  } else {
    // ?? 是否应该先判断 action.cancelled ??
    // 切换 mutex 为状态机的 mutex，脱离与 DNSHandler 的关联
    mutex = action.mutex;
    // 切换状态处理函数为 postEvent()
    SET_HANDLER(&DNSEntry::postEvent);
    // 将该 DNSEntry 调度到状态机所在的线程，以更快捷的完成对状态机的回调
    submit_thread->schedule_imm_signal(this);
  }
  // 返回 0，表示这个 DNSEntry 回调成功，不需要重新放回到 DNSHandler::entries 表格里
  return 0;
}
```

### DNSEntry::postEvent

```
int
DNSEntry::postEvent(int /* event ATS_UNUSED */, Event * /* e ATS_UNUSED */)
{
  // 如果该 DNSEntry 没有被取消，则回调状态机 DNS_EVENT_LOOKUP 事件和 DNS 解析结果
  if (!action.cancelled) {
    Debug("dns", "called back continuation for %s", qname);
    action.continuation->handleEvent(DNS_EVENT_LOOKUP, result_ent);
  }
  // 然后重置成员，并释放 DNSEntry 占用的内存
  result_ent = NULL;
  action.mutex = NULL;
  mutex = NULL;
  dnsEntryAllocator.free(this);
  return EVENT_DONE;
}
```

# 参考资料

- /usr/include/arpa/nameser.h
- /usr/include/arpa/nameser_compat.h
- [P_DNSProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/P_DNSProcessor.h)
- [DNS.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/DNS.cc)
