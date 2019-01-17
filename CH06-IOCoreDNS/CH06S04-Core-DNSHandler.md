# 核心组件：DNSHandler

## 定义

```
/**
  One DNSHandler is allocated to handle all DNS traffic by polling a
  UDP port.

*/
struct DNSHandler : public Continuation {
  /// This is used as the target if round robin isn't set.
  // 保存目标 Name Server 的 IP 地址
  IpEndpoint ip;
  // 保存本地 IP 地址，用于向 Name Server 发起请求的源 IP
  IpEndpoint local_ipv6; ///< Local V6 address if set.
  IpEndpoint local_ipv4; ///< Local V4 address if set.
  // 为每一个 Name Server 建立一个 socket fd
  int ifd[MAX_NAMED];
  // 与多少个 Name Server 建立了连接
  int n_con;
  // 保存 DNSConnection 结构，用于与 Name Server 通信
  DNSConnection con[MAX_NAMED];
  // 保存 DNSEntry 对象的表格
  //   - 横向和纵向都可以无限扩展
  //   - DNSEntry 将被放入到每一个单元里
  //   - 不同的域名解析请求放在不同的行里
  //      - DNSHandler::entries 队列长度表示当前有多少个不同的域名解析请求
  //   - 相同的域名解析请求追加到这一行的末尾
  //      - 每行第一个 DNSEntry 对象的 DNSEntry::dups 队列的长度则表示这一个域名有多少个状态机同时请求进行解析
  Queue<DNSEntry> entries;
  // 收到来自 Name Server 响应的 DNSConnection 队列
  Queue<DNSConnection> triggered;
  // 当前有多少在途请求
  int in_flight;
  // 当前正在使用的 Name Server 的编号
  int name_server;
  // 为了防止 write_dns() 重入，而设置的标志
  int in_write_dns;
  // 为了优化 HostEnt 对象的内存分配，使用此成员指针临时保存创建出来的 HostEnt 对象
  HostEnt *hostent_cache;

  // 用于标记当前无法提供服务的 Name Server
  int ns_down[MAX_NAMED];
  // 用于统计每一个 Name Server 在上一次响应之后又发生的在途请求数
  int failover_number[MAX_NAMED];
  // ????
  int failover_soon_number[MAX_NAMED];
  // 记录自上一次收到 Name Server 响应之后，新发生的在途请求数量超过 dns_failover_number 限定的时间戳
  ink_hrtime crossed_failover_number[MAX_NAMED];
  // 间隔 DNS_PRIMARY_RETRY_PERIOD 重新向第一个 Name Server（Primary）发送请求
  // 用于记录上一次重新发送请求的时间
  ink_hrtime last_primary_retry;
  // 间隔 DNS_PRIMARY_REOPEN_PERIOD 重新打开第一个 Name Server（Primary）的连接
  // 用于记录上一次重新打开连接的时间
  ink_hrtime last_primary_reopen;

  // ink_res_state 在 lib/ts/ink_resolver.h 中定义
  // 用于保存当前 DNS 相关的各种配置，例如：
  //   - 当前有多少个 Name Server，以及他们的域名
  //   - DNS 搜索域的列表
  //   - 默认 DNS 域名
  //   - 等...
  ink_res_state m_res;
  // DNS 查询任务的超时时间，单位：毫秒
  int txn_lookup_timeout;

  // 随机数生成器
  InkRand generator;
  // 通过 bitmap 确保生成的 DNS Query ID 是唯一的
  // bitmap of query ids in use
  uint64_t qid_in_flight[(USHRT_MAX + 1) / 64];

  // 在收到来自 Name Server 的响应后，重置在途请求的统计值
  void
  received_one(int i)
  {
    failover_number[i] = failover_soon_number[i] = crossed_failover_number[i] = 0;
  }

  // 向 Name Server 发送请求后，
  //   - 统计在途请求数量
  //   - 如果在途请求达到 dns_failover_number，还要记录超过限定的时间
  //
  // 最终用于评估 Name Server 的服务状况
  void
  sent_one()
  {
    ++failover_number[name_server];
    Debug("dns", "sent_one: failover_number for resolver %d is %d", name_server, failover_number[name_server]);
    if (failover_number[name_server] >= dns_failover_number && !crossed_failover_number[name_server])
      crossed_failover_number[name_server] = Thread::get_hrtime();
  }

  // 判断指定的 Name Server 是否达到不可用的状态
  //   - 连续发送 dns_failover_number（默认值：6） 次 DNS 解析请求，仍然未收到任何响应
  //   - 上述条件达成后，又等待 dns_failover_period（默认值：60） 秒的时间，仍然未收到任何响应
  bool
  failover_now(int i)
  {
    if (is_debug_tag_set("dns")) {
      Debug("dns", "failover_now: Considering immediate failover, target time is %" PRId64 "",
            (ink_hrtime)HRTIME_SECONDS(dns_failover_period));
      Debug("dns", "\tdelta time is %" PRId64 "", (Thread::get_hrtime() - crossed_failover_number[i]));
    }
    return (crossed_failover_number[i] &&
            ((Thread::get_hrtime() - crossed_failover_number[i]) > HRTIME_SECONDS(dns_failover_period)));
  }

  // 当不满足 failover_now() 状态时，判断是否临近不可用状态
  //   - 连续发送 dns_failover_number（默认值：6） 次 DNS 解析请求，仍然未收到任何响应
  //   - 上述条件达成后，又等待至少 dns_failover_try_period（默认值：31）秒的时间，仍然未收到任何响应
  //   - 此时，达到最基本的 failover_soon 状态判定
  // 
  // 每一次被判定为 failover_soon 状态后，都会重新发送一个 DNS 请求，以测试是否由于丢包等问题导致未收到响应，
  // 每发送一次重试 DNS 请求，下一次对 failover_soon 的判断，就要多等待 FAILOVER_SOON_RETRY（默认值：5）秒的时间，
  // 也就是说，对于重试 DNS 请求的超时时间是 FAILOVER_SOON_RETRY（默认值：5）秒
  bool
  failover_soon(int i)
  {
    return (crossed_failover_number[i] &&
            ((Thread::get_hrtime() - crossed_failover_number[i]) >
             (HRTIME_SECONDS(dns_failover_try_period + failover_soon_number[i] * FAILOVER_SOON_RETRY))));
  }

  // 用于遍历 DNSHandler::triggered 队列，
  //   - 处理从 Name Server 收到的 DNS 响应
  //   - 通过 dns_process() 回调相关的 DNSEntry
  void recv_dns(int event, Event *e);
  // 用于启动全局 DNSHandler 处理器
  int startEvent(int event, Event *e);
  // 用于启动 SplitDNS 系统的 DNSHandler 处理器
  int startEvent_sdns(int event, Event *e);
  // 主要的事件处理函数，通过 Negative Event 周期性回调
  int mainEvent(int event, Event *e);

  // 与指定的 Name Server 建立 UDP socket 连接，并与 DNSConnection 关联
  void open_con(sockaddr const *addr, bool failed = false, int icon = 0);
  // 切换到下一个可用的 Name Server
  void failover();
  // 将指定的 Name Server 标记为服务不可用，不再参与轮询
  void rr_failure(int ndx);
  // 当 Name Server 轮询未开启时，而是采用主、备工作模式时，用于恢复使用主要 Name Server 进行解析的设置
  // 例如：由于某种原因，系统认为 主要 Name Server 无法提供服务，从而将域名解析请求全部发往 备用 Name Server，
  // 系统在后台仍然定期对 主要 Name Server 发送重试请求，如果突然收到了来自 主要 Name Server 的响应，
  // 那么就会调用 recover() 方法将当前使用的 Name Server 切换到 主要 Name Server 上。
  void recover();
  // 
  void retry_named(int ndx, ink_hrtime t, bool reopen = true);
  void try_primary_named(bool reopen = true);
  void switch_named(int ndx);
  uint16_t get_query_id();

  void
  release_query_id(uint16_t qid)
  {
    qid_in_flight[qid >> 6] &= (uint64_t) ~(0x1ULL << (qid & 0x3F));
  };

  void
  set_query_id_in_use(uint16_t qid)
  {
    qid_in_flight[qid >> 6] |= (uint64_t)(0x1ULL << (qid & 0x3F));
  };

  bool
  query_id_in_use(uint16_t qid)
  {
    return (qid_in_flight[(uint16_t)(qid) >> 6] & (uint64_t)(0x1ULL << ((uint16_t)(qid)&0x3F))) != 0;
  };

  DNSHandler();

private:
  // Check the IP address and switch to default if needed.
  void validate_ip();
};
```

## 方法

### DNSHandler::startEvent

DNSHandler 由 `DNSProcessor::open()` 创建，其 mutex 与 EThread `ET_DNS[0]` 共享，由于 EThread::mutex 在启动后就一直保持上锁状态，因此 DNSHandler 内的成员只能被运行在 `ET_DNS[0]` 的状态机访问。通过设置，可以让 ET_DNS[0] 共享 `ET_NET[0]`，那么在这种情况下，运行在 `ET_NET[0]` 的状态机可以访问 DNSHandler 内的成员。

在 `DNSProcessor::open()` 完成了对 DNSHandler 以下成员的初始化：

- m_res 指向 dnsProcessor.l_res
- mutex 指向 dnsProcessor.thread->mutex（也就是 `ET_DNS[0]->mutex`）
- ip 被初始化为传入的 target，或采用缺省地址（127.0.0.1）

```
/**
  Initial state of the DNSHandler. Can reinitialize the running DNS
  handler to a new nameserver.

*/
int
DNSHandler::startEvent(int /* event ATS_UNUSED */, Event *e)
{
  //
  // If this is for the default server, get it
  Debug("dns", "DNSHandler::startEvent: on thread %d\n", e->ethread->id);

  // 如果成员 ip 内未包含合法的 Name Server 地址，则：
  //   - 使用第一个 Name Server 的地址填充（m_res->nsaddr_list[0]）
  //   - 如果不存在 Name Server，则使用 127.0.0.1
  this->validate_ip();

  // 避免对 DNSHandler 重复初始化的标志
  if (!dns_handler_initialized) {
    //
    // If we are THE handler, open connection and configure for
    // periodic execution.
    //
    dns_handler_initialized = 1;
    SET_HANDLER(&DNSHandler::mainEvent);
    // 如果 Name Server 轮询开启
    if (dns_ns_rr) {
      // 确定最大可以使用的 Name Server 数量（MAX_NAMED = 32）
      int max_nscount = m_res->nscount;
      if (max_nscount > MAX_NAMED)
        max_nscount = MAX_NAMED;
      // 从 m_res->nsaddr_list[] 获得 Name Server 的 IP 地址，
      // 与每一个 Name Server 建立 UDP 连接
      n_con = 0;
      for (int i = 0; i < max_nscount; i++) {
        ip_port_text_buffer buff;
        sockaddr *sa = &m_res->nsaddr_list[i].sa;
        if (ats_is_ip(sa)) {
          open_con(sa, false, n_con);
          ++n_con;
          Debug("dns_pas", "opened connection to %s, n_con = %d", ats_ip_nptop(sa, buff, sizeof(buff)), n_con);
        }
      }
      // Typo? 这里应该是 dns_ns_rr_init_done ?
      // 表示 Name Server 轮询初始化完成
      dns_ns_rr_init_down = 0;
    } else {
      // Name Server 轮询未开启，只建立到第一个 Name Server（Primary）的 UDP 连接
      open_con(0); // use current target address.
      // 递增 n_con，表示已经打开了一个连接
      n_con = 1;
    }
    // 使用 Negative Event 周期性回调 DNSHandler::mainEvent，这里 DNS_PERIOD == HRTIME_MSECONDS(-100)
    e->ethread->schedule_every(this, DNS_PERIOD);

    return EVENT_CONT;
  } else {
    // 重复初始化 DNSHandler，例如运行了多个全局 DNSHandler，是不被允许的
    // 注：ET_DNS 的数量最大只能为 1，如果设置为 0 则表示与 ET_NET[0] 共享同一个 EThread
    ink_assert(false); // I.e. this should never really happen
    return EVENT_DONE;
  }
}
```

### DNSHandler::mainEvent

`DNSHandler::mainEvent` 用于接收来自 Name Server 的响应，或者向 Name Server 发送请求。

- 首先调用 `DNSHandler::recv_dns` 接收来自 Name Server 的响应
- 然后再遍历 DNSHandler::entries 表格向 Name Server 发送请求

```
/** Main event for the DNSHandler. Attempt to read from and write to named. */
int
DNSHandler::mainEvent(int event, Event *e)
{
  // 调用 `DNSHandler::recv_dns` 接收来自 Name Server 的响应
  recv_dns(event, e);
  if (dns_ns_rr) {
    // 如果开启 Name Server 轮询
    // 每隔 DNS_PRIMARY_RETRY_PERIOD（默认：5）秒对所有参与轮询的 Name Server 的状态进行检测
    ink_hrtime t = Thread::get_hrtime();
    if (t - last_primary_retry > DNS_PRIMARY_RETRY_PERIOD) {
      for (int i = 0; i < n_con; i++) {
        if (ns_down[i]) {
          // 如果发现无法提供服务的 Name Server，则调用 retry_named() 进行探测，看看是否恢复提供服务的能力
          Debug("dns", "mainEvent: nameserver = %d is down", i);
          retry_named(i, t, true);
        }
      }
      last_primary_retry = t;
    }
    // 遍历所有参与轮询的 Name Server 的状态
    for (int i = 0; i < n_con; i++) {
      // 对于仍然能够提供服务，但是被判定为即将无法提供服务的 Name Server 进行预处理
      if (!ns_down[i] && failover_soon(i)) {
        Debug("dns", "mainEvent: nameserver = %d failover soon", name_server);
        // 进一步判断
        // 如已经处于危急状态，则直接调用 rr_failure() 设置为无法提供服务的状态
        if (failover_now(i))
          rr_failure(i);
        else {
          // 否则，向该 Name Server 发送一个测试请求，看看其是否能够响应该请求
          Debug("dns", "mainEvent: nameserver = %d no failover now - retrying", i);
          retry_named(i, t, false);
          ++failover_soon_number[i];
        }
      }
    }
  } else {
    // 未开启 Name Server 轮询，使用主备模式
    // 判断当前使用的 Name Server 是否处于即将无法提供服务的状态
    if (failover_soon(name_server)) {
      Debug("dns", "mainEvent: will failover soon");
      // 进一步判断
      // 如已经处于危急状态，则直接调用 failover() 切换到备用 Name Server
      if (failover_now(name_server)) {
        Debug("dns", "mainEvent: failing over now to another nameserver");
        failover();
      } else {
        // 否则，向主要 Name Server 发送一个测试请求，看看其是否能够响应该请求
        // 测试主要 Name Server 是否恢复提供服务的能力，
        //   - 如果在当前 Name Server 转为危急状态之前，主要 Name Server 恢复，将会切换回 主要 Name Server 
        //   - 否则，将会使用比当前 Name Server 优先级更低的 备用 Name Server
        try_primary_named(false);
        ++failover_soon_number[name_server];
      }
    } else if (name_server) // not on the primary named
      // 如果当前的 name_server 服务状态良好，而且不是 主要 Name Server
      // 那么就向主要 Name Server 发送一个测试请求，看看其是否恢复提供服务的能力
      try_primary_named(true);
  }

  // 如果 entries 表格不为空，就调用 write_dns() 对其进行遍历，发送 DNS 解析请求到 Name Server
  if (entries.head)
    write_dns(this);

  return EVENT_CONT;
}
```

在轮询模式下，

- 只需要确认哪些 Name Server 是无法提供服务的，这样剩余的 Name Server 参与到轮询中

对于哪些处于即将无法提供服务状态的 Name Server，

- 需要向其发送一个测试请求，判断其是否可以恢复正常，或成为无法提供服务的 Name Server

对于已经处于无法提供服务的 Name Server，

- 需要定期（每隔 5 秒）向其发送一个测试请求，判断其是否可以恢复正常


在主备模式下，

- 第一个 Name Server 的优先级最高（主要），第二个次之（备用），之后优先级依次降低
- 只要高优先级的 Name Server 是可以提供服务的，则永远只向高优先级的 Name Server 发送请求

如果当前 Name Server 处于即将无法提供服务的状态，但是未被判定为无法提供服务之前，

- 如果 主要 Name Server 可以提供服务，可以立即切换到主要 Name Server

如果直到当前 Name Server 被判定为无法提供服务之前，

- 主要 Name Server 都无法提供服务，那么可以切换到优先级更低的备用 Name Server


### write_dns

全局变量 `dns_max_dns_in_flight` 控制了在途 DNS 解析请求的最大数量，默认值为 2048 个，可通过 `proxy.config.dns.max_dns_in_flight` 进行设置。

`write_dns()` 遍历 `DNSHandler::entries` 表格每一行第一个 DNSEntry 对象，如果该 DNSEntry 还未发送 DNS 请求，则构建一个 DNS 请求，并发送出去。当该在途 DNS 解析请求达到最大值 `dns_max_dns_in_flight` 时，则停止对该表格的遍历。`write_dns()` 除了被 DNSEntry 同步调用，更多的是被 DNSHandler 周期性的调用。

```
/** Write up to dns_max_dns_in_flight entries. */
static void
write_dns(DNSHandler *h)
{
  // mutex 用于 DNS_INCREMENT_DYN_STAT
  ProxyMutex *mutex = h->mutex;
  DNS_INCREMENT_DYN_STAT(dns_total_lookups_stat);
  // 计算最大 Name Server，用于在多个 Name Server 之间轮询
  // MAX_NAMED 为 32（在 P_DNSProcessor.h 中定义)
  int max_nscount = h->m_res->nscount;
  if (max_nscount > MAX_NAMED)
    max_nscount = MAX_NAMED;

  // 开始遍历 DNSHandler::entries 表格，将 in_write_dns 设置为 true，
  // 这里主要为了防止重入，因为在 write_dns 可能会递归调用再次进入 write_dns
  if (h->in_write_dns)
    return;
  h->in_write_dns = true;
  // Debug("dns", "in_flight: %d, dns_max_dns_in_flight: %d", h->in_flight, dns_max_dns_in_flight);
  // 当前在途 DNS 请求，小于最大允许的在途请求数，则遍历 DNSHandler::entries 表格，
  // 并尝试发送 DNS 请求，达到表格末尾或最大在途请求后停止遍历
  if (h->in_flight < dns_max_dns_in_flight) {
    DNSEntry *e = h->entries.head;
    while (e) {
      DNSEntry *n = (DNSEntry *)e->link.next;
      // 如果该 DNSEntry 还未发送 DNS 解析请求
      if (!e->written_flag) {
        // 全局变量 dns_ns_rr 控制了是否以轮询方式向多个 Name Server 发送 DNS 解析请求，
        // 可通过 `proxy.config.dns.round_robin_nameservers` 进行设置
        if (dns_ns_rr) {
          // 保存上一次使用过的 Name Server
          int ns_start = h->name_server;
          // 通过轮询算法选择下一个 Name Server
          // 如果选中的 Name Server 无法提供服务，并且不是上一次使用过的 Name Server，则重新轮询
          do {
            h->name_server = (h->name_server + 1) % max_nscount;
          } while (h->ns_down[h->name_server] && h->name_server != ns_start);
          // 如果轮询了一圈，还是无法找到能够提供服务的 Name Server，则继续使用上一次使用的 Name Server
        }
        // 调用 write_dns_event() 构建 DNS 请求的报文，然后发送给选中的 Name Server，
        // 如果发送 DNS 解析请求时遇到 socket 错误，则停止对表格的遍历
        if (!write_dns_event(h, e))
          break;
      }
      // 检查在途请求数量，达到最大值停止对表格的遍历
      if (h->in_flight >= dns_max_dns_in_flight)
        break;
      // 继续表格中的下一行
      e = n;
    }
  }
  // 遍历完成，不需要防止重入了，将 in_write_dns 设置为 false
  h->in_write_dns = false;
}
```

### write_dns_event

构造 DNS 解析请求的 UDP 报文，然后发送给选中的 Name Server。

这里使用 `send()` 发送 UDP 报文的前提是：

- 通过 `connect()` 方法与选中的 Name Server 建立了连接
- 操作系统内核为此五元组分配了一个 socket fd

因此可以不使用 `sendto()`，而使用更简便的 `send()`。

```
/**
  Construct and Write the request for a single entry (using send(3N)).

  @return true = keep going, false = give up for now.

*/
static bool
write_dns_event(DNSHandler *h, DNSEntry *e)
{
  // mutex 用于 DNS_INCREMENT_DYN_STAT 等统计系统的宏定义
  ProxyMutex *mutex = h->mutex;
  // 定义一个联合，方便数据单元的存取
  // HEADER 结构长度为 12 字节（在 arpa/nameser_compat.h 中定义）
  // MAX_DNS_PACKET_LEN 为 8192 字节（在 I_DNSProcessor.h 中定义）
  union {
    HEADER _h;
    char _b[MAX_DNS_PACKET_LEN];
  } blob;
  int r = 0;

  // 通过 _ink_res_mkquery() 调用 ink_res_mkquery() 构造一个 DNS 解析请求的 UDP 报文
  // 该方法与 man 3 res_mkquery 的功能一致，在 tslib 里重新实现该方法的原因未知
  if ((r = _ink_res_mkquery(h->m_res, e->qname, e->qtype, blob._b)) <= 0) {
    Debug("dns", "cannot build query: %s", e->qname);
    // 如果构造失败，则通过 dns_result() 进行重试
    // 可能会通过 domain expansion 功能更换域名后缀
    dns_result(h, e, NULL, false);
    // 返回 true 表示会继续尝试，还未得到最终结果
    // 在 write_dns() 方法中，会继续遍历 DNSHandler::entries 表格
    return true;
  }

  // 获得一个有效的，当前未使用的 DNS Query ID
  uint16_t i = h->get_query_id();
  // 从主机字节序转换为网络字节序
  blob._h.id = htons(i);
  // DNSEntry::retries 在初始化时，被初始化为 dns_retries 的值，每次重试时递减
  // 因此这里通过 dns_retries - e->retries 计算数组下标
  // 这样做主要是因为 dns_retries 的值是可以动态热加载
  // BUG? 如果这里把 dns_retries 突然大幅度减少，可能会导致数组下标出现负数
  // 清除之前的 DNS Query ID
  if (e->id[dns_retries - e->retries] >= 0) {
    // clear previous id in case named was switched or domain was expanded
    h->release_query_id(e->id[dns_retries - e->retries]);
  }
  // 赋予新的 DNS Query ID
  e->id[dns_retries - e->retries] = i;
  Debug("dns", "send query (qtype=%d) for %s to fd %d", e->qtype, e->qname, h->con[h->name_server].fd);

  // 发送构造好的 UDP 报文
  int s = socketManager.send(h->con[h->name_server].fd, blob._b, r, 0);
  if (s != r) {
    Debug("dns", "send() failed: qname = %s, %d != %d, nameserver= %d", e->qname, s, r, h->name_server);
    // changed if condition from 'r < 0' to 's < 0' - 8/2001 pas
    // s 小于 0 时表示 -errno 的值，可能为 -EAGAIN
    if (s < 0) {
      // 如果开启了在多个 Name Server 间轮询，
      //   - 则将当前的 Name Server 状态标记为服务不可用
      //   - 这样会在剩余的 Name Server 里进行轮询
      //   - 与该 Name Server 相关的在途 DNS 解析请求将会重新进行发送
      if (dns_ns_rr)
        h->rr_failure(h->name_server);
      // 如果没有开启 Name Server 轮询功能，
      //   - 则选择下一个可用的 Name Server
      else
        h->failover();
    }
    // 由于 Name Server 产生了变化，因此等待下一次 write_dns() 被调用时，重新遍历该表格
    // 返回 false，在 write_dns() 方法中，会停止对 DNSHandler::entries 表格的遍历
    return false;
  }

  // 成功发送了 UDP 报文
  // 设置 written_flag 表示该 DNSEntry 的解析请求已经发往 Name Server
  e->written_flag = true;
  // 记录下 Name Server 的编号
  e->which_ns = h->name_server;
  // 标记该 DNSEntry 的解析请求至少向 Name Server 发送过一次
  e->once_written_flag = true;
  // DNSHandler 在途请求计数器递增
  ++h->in_flight;
  // DNS 统计系统在途请求计数器递增
  DNS_INCREMENT_DYN_STAT(dns_in_flight_stat);

  // 记录发送时间戳
  e->send_time = Thread::get_hrtime();

  // 取消之前的超时事件
  if (e->timeout)
    e->timeout->cancel();

  // 重设超时事件
  // BUG? 在每一个 DNSEntry 创建时，txn_lookup_timeout 会被覆盖掉，也就是这个成员的值是不确定的
  // 除非这里对于每一个 DNS 请求都使用相同的超时控制，那么也没必要每次创建 DNSEntry 都更新 txn_lookup_timeout 的值
  if (h->txn_lookup_timeout) {
    e->timeout = h->mutex->thread_holding->schedule_in(e, HRTIME_MSECONDS(h->txn_lookup_timeout)); // this is in msec
  } else {
    e->timeout = h->mutex->thread_holding->schedule_in(e, HRTIME_SECONDS(dns_timeout));
  }

  Debug("dns", "sent qname = %s, id = %u, nameserver = %d", e->qname, e->id[dns_retries - e->retries], h->name_server);
  /* 用于对 Name Server 的响应能力进行评估的设计
   * 如果向一个 Name Server 连续发送了多个请求，但是一直都没有得到任何一个响应，
   * 那么可以认为这个 Name Server 可能存在以下问题：
   *
   *   - 网络延迟比较大
   *   - 网络丢包严重
   *   - Name Server 服务器压力大
   *
   * 此时为了保证 DNS 子系统的处理性能，会主动通过轮询方式或者选择备用 Name Server 处理后续的 DNS 解析请求。
   * 与之相关的方法：
   *
   *   - received_one()
   *   - failover_now()
   *   - failover_soon()
   *
   * 在 DNSHandler::mainEvent() 中 详细介绍。
   */
  h->sent_one();
  return true;
}
```

### DNSHandler::recv_dns

`DNSHandler::recv_dns` 负责遍历 DNSConnection 队列 `DNSHandler::triggered`，接收来自 Name Server 的响应，并对响应内容进行分析，找到与之对应的 DNSEntry，回调对同一个域名请求 DNS 解析的所有 DNSEntry 的状态机。

```
void
DNSHandler::recv_dns(int /* event ATS_UNUSED */, Event * /* e ATS_UNUSED */)
{
  DNSConnection *dnsc = NULL;
  ip_text_buffer ipbuff1, ipbuff2;

  // 使用 while 循环遍历 triggered 队列
  while ((dnsc = (DNSConnection *)triggered.dequeue())) {
    // 一个连接上可能会有多个响应
    while (1) {
      // 准备相关参数用于 recvfrom()
      // 用于保存该 UDP 报文的来源 IP
      IpEndpoint from_ip;
      socklen_t from_length = sizeof(from_ip);

      // 预分配 HostEnt 对象，用于存储收到的 UDP 报文
      // 但是 recvfrom() 可能会返回 EAGAIN，为了节约内存分配，使用 hostent_cache 暂存
      // 由于 ET_DNS 只有一个，因此没有使用 ProxyAllocator 为 HostEnt 对象分配内存
      if (!hostent_cache)
        hostent_cache = dnsBufAllocator.alloc();
      HostEnt *buf = hostent_cache;

      // 调用 recvfrom() 接收 UDP 报文
      int res = socketManager.recvfrom(dnsc->fd, buf->buf, MAX_DNS_PACKET_LEN, 0, &from_ip.sa, &from_length);

      // 如果遇到 EAGAIN，则表示当前 DNSConnection 已经没有待接收的报文了，
      // 跳出 while(1) 循环处理下一个 DNSConnection
      if (res == -EAGAIN)
        break;
      // 如果遇到了错误
      if (res <= 0) {
        Debug("dns", "named error: %d", res);
        // 开启了轮询模式则直接设置与该 DNSConnection 关联的 Name Server 为服务不可用状态
        if (dns_ns_rr)
          rr_failure(dnsc->num);
        // 如果是准备模式
        // 如果与该 DNSConnection 关联的 Name Server 就是当前的 Name Server，
        // 就切换到备用 Name Server
        else if (dnsc->num == name_server)
          failover();
        // 跳出 while(1) 循环处理下一个 DNSConnection
        break;
      }

      // 验证来源 IP 地址与我们连接的 Name Server IP 是一致的
      // verify that this response came from the correct server
      if (!ats_ip_addr_eq(&dnsc->ip.sa, &from_ip.sa)) {
        Warning("unexpected DNS response from %s (expected %s)", ats_ip_ntop(&from_ip.sa, ipbuff1, sizeof ipbuff1),
                ats_ip_ntop(&dnsc->ip.sa, ipbuff2, sizeof ipbuff2));
        // 如果不一致，就跳过本次接收到的 UDP 报文，继续接收下一个 UDP 报文
        continue;
      }
      // 接收到的报文是有效的
      // 再次之前，如果重新调用 recvfrom() 接收 UDP 报文，都是可以复用之前保存在 hostent_cache 的 HostEnt 对象
      // 清除 hostent_cache，避免被下一次 recvfrom() 调用时复用
      hostent_cache = 0;
      // 记录收到的报文长度
      buf->packet_size = res;
      Debug("dns", "received packet size = %d", res);

      // 判断运行模式，更新 Name Server 的状态及统计数据
      if (dns_ns_rr) {
        // 轮询模式
        Debug("dns", "round-robin: nameserver %d DNS response code = %d", dnsc->num, get_rcode(buf));
        // 如果该响应是一个有效的 DNS 响应
        if (good_rcode(buf->buf)) {
          // dnsc->num 指向与 DNSConnection 所关联的 Name Server
          // 重置该 Name Server 的服务状态检测的各个计数器
          received_one(dnsc->num);
          // 如果该 Name Server 之前被标记为服务不可用，不能参与轮询，那么就重新标记为服务可用状态，让其参与轮询
          if (ns_down[dnsc->num]) {
            Warning("connection to DNS server %s restored",
                    ats_ip_ntop(&m_res->nsaddr_list[dnsc->num].sa, ipbuff1, sizeof ipbuff1));
            ns_down[dnsc->num] = 0;
          }
        }
      } else {
        // 主备模式
        if (!dnsc->num) {
          // 如果与 DNSConnection 关联的是 主要 Name Server
          Debug("dns", "primary DNS response code = %d", get_rcode(buf));
          // 如果该响应是一个有效的 DNS 响应
          if (good_rcode(buf->buf)) {
            // 当前使用备用 Name Server 说明 主要 Name Server 之前被判定为无法提供服务，
            // 而此时收到了来自 主要 Name Server 的响应，说明 主要 Name Server 恢复了服务能力，
            // 那么就调用 recover() 切回到 主要 Name Server
            if (name_server)
              recover();
            else
              // 当前使用的是 主要 Name Server，就重置其服务状态检测的各个计数器
              received_one(name_server);
          }
        }
      }
      // 将收得到的结果，以回调的方式，传递给所有查询该域名的状态机
      // 通过 Ptr 自动指针对 HostEnt *buf 的计数值递增，
      Ptr<HostEnt> protect_hostent = make_ptr(buf);

      // 调用 dns_process() 从 DNSHandler::entries 里找到记录行
      // 然后向该行所有的 DNSEntry 关联的状态机回调解析结果
      if (dns_process(this, buf, res)) {
        // 如果 dns_process 判断此报文有效，并且 DNSConnection 关联的 Name Server 就是当前正在使用的，
        // 那么就重置其服务状态检测的各个计数器
        if (dnsc->num == name_server)
          received_one(name_server);
      }
      // 在运行到该循环末尾时 Ptr 自动析构，完成对 HostEnt *buf 对象的内存释放
    }
  }
}
```

### get_dns

遍历 `DNSHandler::entries` 表格第一列的 DNSEntry 对象，找出曾经发送过 DNS 解析请求的 DNSEntry 对象，然后在 DNS Query ID 数组中查找与给定 ID 一致的 DNSEntry 对象。

```
/** Find a DNSEntry by id. */
inline static DNSEntry *
get_dns(DNSHandler *h, uint16_t id)
{
  // 遍历 `DNSHandler::entries`
  for (DNSEntry *e = h->entries.head; e; e = (DNSEntry *)e->link.next) {
    // 曾经发送过 DNS 解析请求
    if (e->once_written_flag) {
      // 检查在途的 DNS Query ID 是否与给定 ID 一致
      for (int j = 0; j < MAX_DNS_RETRIES; j++) {
        if (e->id[j] == id) {
          // 如果一致则立即返回找到的 DNSEntry 对象，因为 ID 通过 bitmap 确保是唯一的
          return e;
        } else if (e->id[j] < 0) {
          // 如果 ID 值小于 0 说明后面没有更多的 ID 了，继续处理表格下一行的 DNSEntry
          goto Lnext;
        }
      }
    }
  Lnext:
    ;
  }
  // 最终如果没有找到，返回 NULL
  return NULL;
}
```

### dns_process

`dns_process()` 用于解析收到的 UDP 报文，一个 DNS 响应的 UDP 报文可以包含多个不同类型的记录，对于这部分的分析我直接跳过了，有兴趣的可以详细进行分析，其功能大致是循环调用 man 3 dn_expand 完成对 UDP 报文的解析。

```
/** Decode the reply from "named". */
static bool
dns_process(DNSHandler *handler, HostEnt *buf, int len)
{
  // mutex 用于统计系统的宏定义
  ProxyMutex *mutex = handler->mutex;
  HEADER *h = (HEADER *)(buf->buf);
  // 根据 ID 值，在 DNSHandler::entries 表格里查找对应的 DNSEntry 行
  DNSEntry *e = get_dns(handler, (uint16_t)ntohs(h->id));
  bool retry = false;
  bool server_ok = true;
  uint32_t temp_ttl = 0;

  //
  // Do we have an entry for this id?
  //
  // 未找到对一个的 DNSEntry 行，或者该 DNSEntry 没有在途的请求
  // 这说明收到的响应与 DNSEntry 不匹配
  // 返回 false，表示该次响应报文无效
  if (!e || !e->written_flag) {
    Debug("dns", "unknown DNS id = %u", (uint16_t)ntohs(h->id));
    return false; // cannot count this as a success
  }
  //
  // It is no longer in flight
  //
  // 报文有效，重置该 DNSEntry 的在途状态
  e->written_flag = false;
  // 递减 DNSHandler 的在途请求计数器
  --(handler->in_flight);
  // 递减 DNS 统计系统的在途请求计数
  DNS_DECREMENT_DYN_STAT(dns_in_flight_stat);

  // 统计 DNS 平均响应时间（当前时间 - 请求发送时间）
  DNS_SUM_DYN_STAT(dns_response_time_stat, Thread::get_hrtime() - e->send_time);

  // 这里 h 不是 DNSHandler
  // 如果 DNS 响应里的状态码为出错类型，或者 DNS 响应里不包含任何记录
  // 则进入 DNS 响应错误处理
  if (h->rcode != NOERROR || !h->ancount) {
    Debug("dns", "received rcode = %d", h->rcode);
    switch (h->rcode) {
    default:
      Warning("Unknown DNS error %d for [%s]", h->rcode, e->qname);
      retry = true;
      server_ok = false; // could be server problems
      goto Lerror;
    case SERVFAIL: // recoverable error
      retry = true;
    case FORMERR: // unrecoverable errors
    case REFUSED:
    case NOTIMP:
      Debug("dns", "DNS error %d for [%s]", h->rcode, e->qname);
      server_ok = false; // could be server problems
      goto Lerror;
    case NOERROR:
    case NXDOMAIN:
    case 6:  // YXDOMAIN
    case 7:  // YXRRSET
    case 8:  // NOTAUTH
    case 9:  // NOTAUTH
    case 10: // NOTZONE
      Debug("dns", "DNS error %d for [%s]", h->rcode, e->qname);
      goto Lerror;
    }
  } else {
    // 收到了一个正常的 DNS 响应
    //
    // Initialize local data
    //
    //    struct in_addr host_addr;            unused
    u_char tbuf[MAXDNAME + 1];
    buf->ent.h_name = NULL;

    int ancount = ntohs(h->ancount);
    unsigned char *bp = buf->hostbuf;
    int buflen = sizeof(buf->hostbuf);
    u_char *cp = ((u_char *)h) + HFIXEDSZ;
    u_char *eom = (u_char *)h + len;
    int n;
    ink_assert(buf->srv_hosts.srv_host_count == 0 && buf->srv_hosts.srv_hosts_length == 0);
    buf->srv_hosts.srv_host_count = 0;
    buf->srv_hosts.srv_hosts_length = 0;
    unsigned &num_srv = buf->srv_hosts.srv_host_count;
    int rname_len = -1;

    // 解析出原始 DNS 请求的内容
    //
    // Expand name
    //
    if ((n = ink_dn_expand((u_char *)h, eom, cp, bp, buflen)) < 0)
      goto Lerror;

... 省略部分代码 ...

    // 初始化 HostEnt 的数据结构，准备用于保存多个 DNS 解析记录
    // 例如：CNAME 和 多个 A 记录等
    //
    // Configure HostEnt data structure
    //
    u_char **ap = buf->host_aliases;
    buf->ent.h_aliases = (char **)buf->host_aliases;
    u_char **hap = (u_char **)buf->h_addr_ptrs;
    *hap = NULL;
    buf->ent.h_addr_list = (char **)buf->h_addr_ptrs;

    //
    // INKqa10938: For customer (i.e. USPS) with closed environment, need to
    // build up try_server_names[] with names already successfully resolved.
    // try_server_names[] gets filled up with every success dns response.
    // Once it's full, a new entry get inputted into try_server_names round-
    // robin style every 50 success dns response.

    // TODO: Why do we do strlen(e->qname) ? That should be available in
    // e->qname_len, no ?
    if (local_num_entries >= DEFAULT_NUM_TRY_SERVER) {
      if ((attempt_num_entries % 50) == 0) {
        try_servers = (try_servers + 1) % countof(try_server_names);
        ink_strlcpy(try_server_names[try_servers], e->qname, MAXDNAME);
        memset(&try_server_names[try_servers][strlen(e->qname)], 0, 1);
        attempt_num_entries = 0;
      }
      ++attempt_num_entries;
    } else {
      // fill up try_server_names for try_primary_named
      try_servers = local_num_entries++;
      ink_strlcpy(try_server_names[try_servers], e->qname, MAXDNAME);
      memset(&try_server_names[try_servers][strlen(e->qname)], 0, 1);
    }

    /* added for SRV support [ebalsa]
       this skips the query section (qdcount)
     */
    unsigned char *here = (unsigned char *)buf->buf + HFIXEDSZ;
    if (e->qtype == T_SRV) {
      for (int ctr = ntohs(h->qdcount); ctr > 0; ctr--) {
        int strlen = dn_skipname(here, eom);
        here += strlen + QFIXEDSZ;
      }
    }

    // 使用 while 循环，遍历所有的 DNS 解析记录，并放入 HostEnt 结构
    //
    // Decode each answer
    //
    int answer = false, error = false;

    while (ancount-- > 0 && cp < eom && !error) {
      // 调用 ink_dn_expand() 解释数据
      n = ink_dn_expand((u_char *)h, eom, cp, bp, buflen);
      if (n < 0) {
        ++error;
        break;
      }
      cp += n;
      short int type;
      NS_GET16(type, cp);
      cp += NS_INT16SZ;       // NS_GET16(cls, cp);
      NS_GET32(temp_ttl, cp); // NOTE: this is not a "long" but 32-bits (from nameser_compat.h)
      if ((temp_ttl < buf->ttl) || (buf->ttl == 0))
        buf->ttl = temp_ttl;
      NS_GET16(n, cp);

      //
      // Decode cname
      //
      if (is_addr_query(e->qtype) && type == T_CNAME) {
... 省略部分代码 ...
      }
      if (e->qtype != type) {
        ++error;
        break;
      }
      //
      // Decode names
      //
      if (type == T_PTR) {
... 省略部分代码 ...
      } else if (type == T_SRV) {
... 省略部分代码 ...
        ++num_srv;
      } else if (is_addr_query(type)) {
... 省略部分代码 ...
      } else
        goto Lerror;

      // 有效记录计数器
      ++answer;
    }
    // 存在有效记录
    if (answer) {
      *ap = NULL;
      *hap = NULL;
      //
      // If the named didn't send us the name, insert the one
      // the user gave us...
      //
      if (!buf->ent.h_name) {
        Debug("dns", "inserting name = %s", e->qname);
        ink_strlcpy((char *)bp, e->qname, sizeof(buf->hostbuf) - (bp - buf->hostbuf));
        buf->ent.h_name = (char *)bp;
      }
      // 通过 dns_result() 向与该域名关联的所有 DNSEntry 的状态机回调解析结果
      dns_result(handler, e, buf, retry);
      // server_ok 为 true
      // 返回 true，表示此 UDP 报文是正确的
      return server_ok;
    }
  }
Lerror:
  ;
  // 错误处理
  // 统计 DNS 出错次数
  DNS_INCREMENT_DYN_STAT(dns_lookup_fail_stat);
  // 通过 dns_result() 向与该域名关联的所有 DNSEntry 的状态机回调空的解析结果
  // 如果还有重试次数 dns_result() 会自动安排重新尝试 DNS 解析
  dns_result(handler, e, NULL, retry);
  // 根据错误处理的部分，server_ok 可能为 true 或 false
  return server_ok;
}
```

### dns_result

通过 `dns_result()` 向 DNSEntry 的状态机回调解析结果：

- 可能是有效的解析结果
- 可能是无结果，例如
  - 超时
  - 得到的响应有问题

对于无结果的情况，需要根据错误的类型，选择是否可以重新尝试解析，部分错误是不可恢复的，重新尝试也不会得到正确结果，有些错误则可以通过重新尝试得到正确的解析结果。

`dns_result()` 需要区分上述情况，

`dns_result()` 的调用路径有以下几种：

传入的 HostEnt 指向 NULL，Retry 为 false：

- `write_dns()` --> `write_dns_event()` --> `dns_result(DNSHandler, DNSEntry, NULL, false)`
  - 构建 DNS 解析请求失败时
- `DNSEntry::mainEvent(EVENT_INTERNAL, )` --> `dns_result(DNSHandler, DNSEntry, NULL, false)`
  - 出现超时，并且 DNSHandler 有超时设置

传入的 HostEnt 指向 NULL，Retry 为 true：

- `DNSEntry::mainEvent(EVENT_INTERNAL, )` --> `dns_result(DNSHandler, DNSEntry, NULL, true)`
  - 出现超时，但是 DNSHandler 没有超时设置

传入的 HostEnt 指向 NULL，Retry 不确定：

- `DNSHandler::mainEvent()` --> `recv_dns()` --> `dns_process()` --> `dns_result(DNSHandler, DNSEntry, NULL, retry)`
  - 得到了来自 Name Server 的响应，但是响应不包含有效的内容
  - 需要根据响应代码判断是否要重试，所以 retry 不确定

传入的 HostEnt 指向 有效内容，Retry 为 false：

- `DNSHandler::mainEvent()` --> `recv_dns()` --> `dns_process()` --> `dns_result(DNSHandler, DNSEntry, HostEnt, false)`
  - 得到了来自 Name Server 的响应，并且解析正确，得到了有效的 DNS 解析记录
  - 此时 retry 为 false

```
/**
  We have a result for an entry, return it to the user or retry if it
  is a retry-able and we have retries left.
*/
static void
dns_result(DNSHandler *h, DNSEntry *e, HostEnt *ent, bool retry)
{
  // mutex 用于统计系统的宏定义
  ProxyMutex *mutex = h->mutex;
  bool cancelled = (e->action.cancelled ? true : false);

  // 如果没有得到正确的 DNS 响应，DNSEntry 也没有被取消
  if (!ent && !cancelled) {
    // 如果这个不正确的 DNS 响应是可以通过重试来解决的
    // 同时，DNSEntry 的重试次数还没有耗尽，则调用 write_dns() 重新发送 DNS 请求
    // try to retry operation
    if (retry && e->retries) {
      Debug("dns", "doing retry for %s", e->qname);

      // 同时重试的次数
      DNS_INCREMENT_DYN_STAT(dns_retries_stat);

      // DNSEntry 的重试次数递减
      --(e->retries);
      // 这里调用 write_dns 不仅仅是重新发送这一个 DNSEntry
      // 如果 write_dns() 重入的话，他是什么都不做，直接返回的，
      // 需要等到下一次 DNSHandler::mainEvent 被回调才会重新发送 DNS 请求
      write_dns(h);
      return;
    } else if (e->domains && *e->domains) {
      // 如果是不可恢复的 DNS 响应错误，但是开启了 domain extension 功能
      // 那么就在当前域名的尾部，拼接上 domain 之后，再调用 write_dns() 重试
      do {
        Debug("dns", "domain extending, last tried '%s', original '%.*s'", e->qname, e->orig_qname_len, e->qname);

        // Make sure the next try fits
        // 如果拼接上 domain 之后超过最大允许的长度则选择下一个 domain
        if (e->orig_qname_len + strlen(*e->domains) + 2 > MAXDNAME) {
          Debug("dns", "domain too large %.*s + %s", e->orig_qname_len, e->qname, *e->domains);
        } else {
          e->qname[e->orig_qname_len] = '.';
          e->qname_len =
            e->orig_qname_len + 1 + ink_strlcpy(e->qname + e->orig_qname_len + 1, *e->domains, MAXDNAME - (e->orig_qname_len + 1));
          ++(e->domains);
          e->retries = dns_retries;
          Debug("dns", "new name = %s retries = %d", e->qname, e->retries);
          // 拼接好 domain 之后调用 write_dns()
          write_dns(h);

          return;
        }

        // Try another one
        ++(e->domains);
      } while (*e->domains);
      // 如果遍历所有的 domain 都无法找到一个有效的 domain，那么就不进行重试，继续向下
    } else {
      // 如果所有 domain 都尝试过，那么就尝试一下直接解析主机名
      // 当开启了 domain extension 功能时，在 DNSEntry::mainEvent 里就直接把第一个 domain 拼接到主机名的尾部，
      // 因此当所有的 domain 都尝试过之后，最后才对主机名进行解析的尝试
      // BUG? 这里是不是应该使用 orig_qname_len ?
      e->qname[e->qname_len] = 0;
      // 在域名里找不到 "."（这就是一个主机名），那么就尝试做最后一次解析
      if (!strchr(e->qname, '.') && !e->last) {
        e->last = true;
        write_dns(h);
        return;
      }
    }
    if (retry) {
      DNS_INCREMENT_DYN_STAT(dns_max_retries_exceeded_stat);
    }
  }
  // 对于 ent 为 NULL，没有正确解析结果的情况，上面对符合重试条件的都安排了重试
  
  // 这个 BAD_DNS_RESULT 应该是没删干净
  if (ent == BAD_DNS_RESULT)
    ent = NULL;

  // 如果没有取消
  if (!cancelled) {
    if (!ent) {
      // ent 为 NULL，统计 DNS 解析失败时的平均响应时间
      DNS_SUM_DYN_STAT(dns_fail_time_stat, Thread::get_hrtime() - e->submit_time);
    } else {
      // ent 不为 NULL，统计 DNS 解析成功时的平均响应时间
      DNS_SUM_DYN_STAT(dns_success_time_stat, Thread::get_hrtime() - e->submit_time);
    }
  }
  // 将 DNSEntry 从 DNSHandler::entries 的表格中移除
  // 这里移除的是一整行
  h->entries.remove(e);

  // debug 调试功能
  if (is_debug_tag_set("dns")) {
    if (is_addr_query(e->qtype)) {
      ip_text_buffer buff;
      char const *ptr = "<none>";
      char const *result = "FAIL";
      if (ent) {
        result = "SUCCESS";
        ptr = inet_ntop(e->qtype == T_AAAA ? AF_INET6 : AF_INET, ent->ent.h_addr_list[0], buff, sizeof(buff));
      }
      Debug("dns", "%s result for %s = %s retry %d", result, e->qname, ptr, retry);
    } else {
      if (ent) {
        Debug("dns", "SUCCESS result for %s = %s af=%d retry %d", e->qname, ent->ent.h_name, ent->ent.h_addrtype, retry);
      } else {
        Debug("dns", "FAIL result for %s = <not found> retry %d", e->qname, retry);
      }
    }
  }

  // 统计 DNS 解析成功次数和失败次数
  if (ent) {
    DNS_INCREMENT_DYN_STAT(dns_lookup_success_stat);
  } else {
    DNS_INCREMENT_DYN_STAT(dns_lookup_fail_stat);
  }

  // 遍历解析同一个域名的 DNSEntry 行
  DNSEntry *dup = NULL;
  while ((dup = e->dups.dequeue())) {
    // 调用 DNSEntry::post() 回调状态机
    // 如果没有成功回调，则把该 DNSEntry 放回到队列尾部，然后进入重试步骤
    if (dup->post(h, ent)) {
      e->dups.enqueue(dup);
      goto Lretry;
    }
  }

  // 以下代码与 DNSEntry::post() 非常相似，但是多了清除 DNS Query ID 的部分。
  // 感觉以下代码的逻辑应该是可以进行优化，或者重用 DNSEntry::post() 方法，有兴趣的可以搞一下。
  // 取消之前的超时设置
  if (e->timeout) {
    e->timeout->cancel(e);
    e->timeout = NULL;
  }
  e->result_ent = ent;

  if (h->mutex->thread_holding == e->submit_thread) {
    // 尝试对状态机上锁
    MUTEX_TRY_LOCK(lock, e->action.mutex, h->mutex->thread_holding);
    // 上锁失败进入重试步骤
    if (!lock.is_locked()) {
      Debug("dns", "failed lock for result %s", e->qname);
      goto Lretry;
    }
    // 清除 DNS Query ID
    for (int i = 0; i < MAX_DNS_RETRIES; i++) {
      if (e->id[i] < 0)
        break;
      h->release_query_id(e->id[i]);
    }
    // 上锁成功，通过 postEvent() 回调状态机
    e->postEvent(0, 0);
  } else {
    // 清除 DNS Query ID
    for (int i = 0; i < MAX_DNS_RETRIES; i++) {
      if (e->id[i] < 0)
        break;
      h->release_query_id(e->id[i]);
    }
    // ?? 是否应该先判断 action.cancelled ??
    // 切换 mutex 为状态机的 mutex，脱离与 DNSHandler 的关联
    e->mutex = e->action.mutex;
    // 切换状态处理函数为 postEvent()
    SET_CONTINUATION_HANDLER(e, &DNSEntry::postEvent);
    // 将该 DNSEntry 调度到状态机所在的线程，以更快捷的完成对状态机的回调
    e->submit_thread->schedule_imm_signal(e);
  }
  return;
Lretry:
  // 重试步骤
  // 将 ent 保存到 Ptr<HostEnt> result_ent，这样在返回之后，就不会被释放了
  e->result_ent = ent;
  // 没有重试的必要了，重置
  e->retries = 0;
  // 通过事件系统安排对 DNSEntry 的重试回调
  if (e->timeout)
    e->timeout->cancel();
  // 延迟回调 DNSEntry
  // BUG：这里 DNS_PERIOD 为负数
  // 参考：
  //   - [Pull Request #2574](https://github.com/apache/trafficserver/pull/2574)
  e->timeout = h->mutex->thread_holding->schedule_in(e, DNS_PERIOD);
}
```

# 参考资料

- /usr/include/arpa/nameser.h
- /usr/include/arpa/nameser_compat.h
- [P_DNSProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/P_DNSProcessor.h)
- [DNS.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/DNS.cc)
- [Pull Request #2574](https://github.com/apache/trafficserver/pull/2574)