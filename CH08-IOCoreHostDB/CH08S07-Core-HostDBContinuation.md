# 核心组件：HostDBContinuation

HostDBContinuation 是用来实现对 HostDB 缓存查询和 DNS 子系统调用的状态机，其主要实现以下功能：

- `probe()` 和 `probeEvent()`
   - 从 Hosts 文件和 HostDB 中查询域名解析结果，
- `do_dns()` 和 `dnsEvent()`
   - 当找不到需要的数据时，调用 DNS 子系统完成 DNS 解析，
- `lookup_done()` 和 `insert()`
   - 当收到 DNS 子系统返回的 DNS 解析结果后，将其缓存到 HostDB 中，
- `set_check_pending_dns()` 和 `dnsPendingEvent()`
   - 合并相同域名和类型的查询请求，提高 HostDB 查询效率，降低 DNS 解析请求数量
- `check_for_retry()`
   - 实现 IPv4 Only 或 IPv6 Only 查询，
   - 或者先查询 IPv4 或 IPv6，找不到时再自动查询 IPv6 或 
- `setbyEvent()`
   - 直接向 HostDB 内插入数据

其它用途：

- backgroundEvent()
   - 用于自动加载 `/etc/hosts` 内的域名解析信息，
   - 同时查看配置项  `proxy.config.hostdb.host_file.path` 是否发生改变，
   - 如果发生改变则读取其指定的路径最为 `/etc/hosts` 的替代
- iterateEvent()
   - 用于配合 `ShowHostDB` 状态机和 `HostDBProcessor::iterate()` 方法，
   - 可以通过 Web 方式对 HostDB 进行查询和遍历
- clusterEvent()
   - 通过 `do_get_response()` 方法向集群内的 ATS 发起 HostDB 数据库的查询操作，
   - 发送的集群指令代码为：`GET_HOSTINFO_CLUSTER_FUNCTION`，
   - 该状态机用于接收返回的查询结果，
   - 集群节点收到该指令后会调用 `get_hostinfo_ClusterFunction()` 方法，通过 `probeEvent()` 状态机完成查询操作。
- clusterResponseEvent()
   - 通过 `do_put_response()` 方法向集群内的 ATS 发起 HostDB 数据库的插入操作，
   - 发送的集群指令代码为：`PUT_HOSTINFO_CLUSTER_FUNCTION`，
   - 该状态机用于接收对方推送过来的 HostDBInfo 记录，并存储到 HostDB 内，
   - 集群节点收到该指令后会调用 `put_hostinfo_ClusterFunction()` 方法，然后通过该状态机将数据记录存储到 HostDB 内，
   - 如果该记录与之前发出的集群请求相匹配，则直接回调 `clusterEvent()` 状态机。


## 定义

```
//
// Handles a HostDB lookup request
//
struct HostDBContinuation;
typedef int (HostDBContinuation::*HostDBContHandler)(int, void *);

struct HostDBContinuation : public Continuation {
  // 嵌入 Action 成员，用于 cancel 该状态机
  Action action;
  // 保存 md5 值，避免重复计算（与 HostDBInfo 内的 md5 值一致）
  HostDBMD5 md5;
  // 由于在 HostDBMD5 内已经存在 IpAddr ip 成员，因此在 043815e7a7 将该成员删除
  //  IpEndpoint ip;
  // 保存 DNS 解析结果的 TTL 值
  unsigned int ttl;
  // 同样是在 HostDBMD5 内已经存在 HostDBMark db_mark 成员，因此在 043815e7a7 将该成员删除
  //  HostDBMark db_mark; ///< Target type.
  /// Original IP address family style. Note this will disagree with
  /// @a md5.db_mark when doing a retry on an alternate family. The retry
  /// logic depends on it to avoid looping.
  // 请求哪种地址类型的 DNS 解析结果：仅 IPv4，仅 IPv6，首选 iPv4，首选 IPv6
  HostResStyle host_res_style; ///< Address family priority.
  // 如无法在 HostDB 查询到需要的记录，则需要进行 DNS 解析，该值用于设置 DNS 解析操作的超时时间
  int dns_lookup_timeout;
  // 被 HostDBMD5 md5 取代
  //  INK_MD5 md5;
  // 用于 HostDBContinuation 状态机的超时控制
  Event *timeout;
  // 用于集群
  ClusterMachine *from;
  Continuation *from_cont;
  // 用于临时保存设置到 HostDBInfo 上的 app 信息，关联 setby 方法和 setbyEvent() 状态机
  HostDBApplicationInfo app;
  // 用于集群查询时，遍历 HostDB 数据库的深度（HostDB 的底层是 MultiCache DB）
  int probe_depth;
  // 用于支持 iteratorEvent
  int current_iterate_pos;
  // 与 probe_depth 配合，支持集群查询
  ClusterMachine *past_probes[CONFIGURATION_HISTORY_PROBE_DEPTH];
  // 使用 HostDBMD5 md5 内的成员 host_name 替代
  //  char name[MAXDNAME];
  // 使用 HostDBMD5 md5 内的成员 host_len 替代
  //  int namelen;
  // 用于保存 hostname，且 HostDBMD5 md5 内的成员 host_name 总是指向改地址
  char md5_host_name_store[MAXDNAME + 1]; // used as backing store for @a md5
  // 用于 setby_srv() 方法，保存 SRV target 的值，在 do_setby() 方法中 SRV 部分的处理存在 bug
  char srv_target_name[MAXDNAME];
  // 在 043815e7a7 的提交中，该成员的功能由 HostDBMD5 内的 DNSServer *dns_server 和 SplitDNS *pSD 成员替代，
  // 因此在 a2affb99 将该成员删除
  //  void *m_pDS;
  // 用于保存调用 DNS 子系统返回的 Action
  Action *pending_action;

  // 表示解析结果是否存在，用于集群间传递 HostDBInfo 数据
  unsigned int missing : 1;
  // 表示是否强制执行 DNS 解析，从而跳过 HostDB 的查询
  unsigned int force_dns : 1;
  // 表示所传递的信息是否为 Round Robin 的元数据（根节点），用于集群间传递 HostDBInfo 数据
  unsigned int round_robin : 1;

  // 异步查询 HostDB
  int probeEvent(int event, Event *e);
  // 检索 HostDB
  int iterateEvent(int event, Event *e);
  // 集群功能
  int clusterEvent(int event, Event *e);
  int clusterResponseEvent(int event, Event *e);
  // 接收 DNS 子系统的回调
  int dnsEvent(int event, HostEnt *e);
  // 等待接收共享结果
  int dnsPendingEvent(int event, Event *e);
  // 后台任务：关注 /etc/hosts 文件变化并加载
  int backgroundEvent(int event, Event *e);
  // 未定义、未使用
  int retryEvent(int event, Event *e);
  // 根据指定的 Hostname 和 IP 地址，删除指定的 HostDBInfo 对象。未使用，但是在注释掉的代码里有调用。
  int removeEvent(int event, Event *e);
  // 异步向 HostDB 中插入 HostDBInfo 对象
  int setbyEvent(int event, Event *e);

  /// Recompute the MD5 and update ancillary values.
  void refresh_MD5();
  // 发起 DNS 解析
  void do_dns();
  // 当前解析任务是正向域名解析时返回 true
  bool
  is_byname()
  {
    return md5.db_mark == HOSTDB_MARK_IPV4 || md5.db_mark == HOSTDB_MARK_IPV6;
  }
  // 当前解析任务是 SRV 解析时返回 true
  bool
  is_srv()
  {
    return md5.db_mark == HOSTDB_MARK_SRV;
  }
  // 当收到来自 DNS 子系统或集群系统的回调，获取到 DNS 解析结果之后，
  // 通过调用 lookup_done() 方法将数据存储到本地的 HostDB 数据库内，
  //  - 如果该 DNS 解析任务来自集群系统，则回调集群系统，将结果传递给发起任务的集群节点，
  //  - 否则回调状态机，传递解析结果。
  // 返回的指针指向包含解析结果的 HostDB 内的 Entry。
  HostDBInfo *lookup_done(IpAddr const &ip, char const *aname, bool round_robin, unsigned int attl, SRVHosts *s = NULL);
  // 向集群系统发起 HostDB 查询请求
  bool do_get_response(Event *e);
  // 向集群系统发送 HostDB 查询结果
  void do_put_response(ClusterMachine *m, HostDBInfo *r, Continuation *cont);
  // 向集群系统查询失败后，重新发起集群查询请求，简化掉了 probeEvent() 内的一些多余操作
  int failed_cluster_request(Event *e);
  // 根据哈希函数计算出当前查询结果存储到哪个 HostDB / MultiCache 的分区
  int key_partition();
  // 遍历 hostDB.pending_dns 队列，将解析结果分享给所有相同的 DNS 解析任务，是合并 HostDB 查询功能的一部分
  void remove_trigger_pending_dns();
  // 将 HostDBContinuation 对象插入到 hostDB.pending_dns 队列，
  //  - 如果已经存在相同的 HostDB 查询任务，则返回 false，表示仅需要等待 remove_trigger_pending_dns() 分享 DNS 解析结果，
  //  - 否则，返回 true，表示须要立即调用 DNS 子系统完成 DNS 解析。
  // 在 do_dns() 方法内，将根据返回值来确定是否调用 DNS 子系统完成 DNS 解析任务。
  int set_check_pending_dns();

  // 根据当前任务的哈希值和当前节点的信息计算该任务的主节点（目前代码中暂未使用）
  // 可以参考 dnsEvent() 中对 cluster_machine_at_depth() 的调用
  ClusterMachine *master_machine(ClusterConfiguration *cc);

  // 根据当前任务的哈希值向 hostDB 申请一个 HostDBInfo 的存储单元
  // 如果具有相同哈希值的单元已经存在，则先删除已存在的数据
  HostDBInfo *insert(unsigned int attl);

  /** Optional values for @c init.
   */
  // 用于简化向 init() 方法传递参数的结构体
  struct Options {
    typedef Options self; ///< Self reference type.

    int timeout;                 ///< Timeout value. Default 0
    HostResStyle host_res_style; ///< IP address family fallback. Default @c HOST_RES_NONE
    bool force_dns;              ///< Force DNS lookup. Default @c false
    Continuation *cont;          ///< Continuation / action. Default @c NULL (none)

    Options() : timeout(0), host_res_style(HOST_RES_NONE), force_dns(false), cont(0) {}
  };
  static const Options DEFAULT_OPTIONS; ///< Default defaults.
  // 初始化 HostDBContinuation
  void init(HostDBMD5 const &md5, Options const &opt = DEFAULT_OPTIONS);
  // 创建查询请求，用于发送到集群节点
  int make_get_message(char *buf, int len);
  // 创建解析结果，用于发送到集群节点
  int make_put_message(HostDBInfo *r, Continuation *c, char *buf, int len);

  HostDBContinuation()
    : Continuation(NULL),
      ttl(0),
      host_res_style(DEFAULT_OPTIONS.host_res_style),
      dns_lookup_timeout(DEFAULT_OPTIONS.timeout),
      timeout(0),
      from(0),
      from_cont(0),
      probe_depth(0),
      current_iterate_pos(0),
      // 缺少 pending_action(NULL),
      missing(false),
      force_dns(DEFAULT_OPTIONS.force_dns),
      round_robin(false)
  {
    // 此处建议增加对 srv_target_name[] 数组的初始化
    ink_zero(md5_host_name_store);
    ink_zero(md5.hash);
    SET_HANDLER((HostDBContHandler)&HostDBContinuation::probeEvent);
  }
};
```


## 方法

以下是通过 `getby()` 方法查询 HostDB 的大致流程：

```
       HostDBProcessor::getby()
                        |
                        |
                        V
    HostDBContinuation::probeEvent()  <----------- check_for_retry() -----------+
           |            |                                                       |
           |            |                                                       |
       (force dns)    probe()                                                   |
           |            |                                                       |
           |            |                                                       |
           V            V                                                       |
    HostDBContinuation::do_dns()        HostDBContinuation::dnsPendingEvent()   |
                |                          |                ^                   |
                |                          |                |                   |
                |                          V                |                   |
                |              [ EVENT_HOST_DB_LOOKUP ]     |                   |
                |                          ^                |                   |
                |                          |                |                   |
                |                          |                |                   |
                |                       HostDBContinuation::dnsEvent()  --------+
                |                                           ^
                |                                           |
                V                                           |
   dnsProcessor.getXXXXbyZZZZ()  -------------->  DNSEntry::postOneEvent()
``` 

在 `probe()` 方法内包含了通过 Hosts 文件解析域名到 IP 地址的功能，当解析不到时再查询 HostDB 或通过 DNS 子系统完成域名解析，当收到来自 DNS 子系统的解析结果后，需要先从 HostDB 内申请一个 Entry 用于保存该结果，然后再返回指向该 HostDB Entry 的指针作为最终的域名解析结果。

### init

```
void
HostDBContinuation::init(HostDBMD5 const &the_md5, Options const &opt)
{
  md5 = the_md5;
  // 由于之前传入的 hostname 所在的地址空间是调用者申请的，这部分地址空间有可能在 getby 方法返回后就被释放了，
  // 因此在创建 HostDBContinuation 实例后，要在 init 方法中将 hostname 复制一份保存起来，
  // 然后才能通过 probeEvent 等状态机进行异步的处理。
  if (md5.host_name) {
    // copy to backing store.
    if (md5.host_len > static_cast<int>(sizeof(md5_host_name_store) - 1))
      md5.host_len = sizeof(md5_host_name_store) - 1;
    memcpy(md5_host_name_store, md5.host_name, md5.host_len);
  } else {
    md5.host_len = 0;
  }
  md5_host_name_store[md5.host_len] = 0;
  // 将 md5 中的 host_name 指针切换到 HostDBContinuation 的内部缓冲区
  md5.host_name = md5_host_name_store;

  host_res_style = opt.host_res_style;
  dns_lookup_timeout = opt.timeout;
  // 将 HostDBContinuation 状态机与 buckets 的分区锁绑定
  mutex = hostDB.lock_for_bucket((int)(fold_md5(md5.hash) % hostDB.buckets));
  // 有一种 HostDBContinuation 状态机是 HostDB 内部用来自动更新过期的解析记录，
  // 因此在完成 DNS 解析后不需要回调，所以他的 action.continuation 为 NULL，
  // 但是在操作 action 时，需要先对 action.mutex 上锁才可以判断其成员 cancelled 的值，
  // 因此对于这类特殊的情况，将 action.mutex 设置为 HostDBContinuation 的 mutex。
  if (opt.cont) {
    action = opt.cont;
  } else {
    // ink_assert(!"this sucks");
    action.mutex = mutex;
  }
}
```

### reply_to_cont

本方法负责将 HostDB 查询到的结果（HostDBInfo）通过回调传递给调用者。

参数：

- cont：指向调用者，通常为 HttpSM
- r：指向 HostDB 内的一个 Entry
- is_srv：Entry 内保存的 HostDBInfo 是否为 SRV

BUG？：多处代码，在调用 `reply_to_cont` 时，没有传递第三个参数，有可能导致 SRV 类型回调可能存在异常。

```
static bool
reply_to_cont(Continuation *cont, HostDBInfo *r, bool is_srv = false)
{
  // 查找 HostDB 失败（r == NULL）、请求的类型与记录不匹配（r->is_srv != is_srv）、DNS 解析失败（r->failed()），
  // 则回调 NULL 给调用者。
  if (r == NULL || r->is_srv != is_srv || r->failed()) {
    cont->handleEvent(is_srv ? EVENT_SRV_LOOKUP : EVENT_HOST_DB_LOOKUP, NULL);
    // HostDB 需要缓存 r->failed() 的记录，以避免发起大量无用的 DNS 解析请求
    return false;
  }

  // 反向地址解析
  if (r->reverse_dns) {
    if (!r->hostname()) {
      // 记录中不包含域名，可能是之前从 Heap 分配空间失败导致
      // 按照解析失败处理，回调 NULL 给调用者。
      ink_assert(!"missing hostname");
      cont->handleEvent(is_srv ? EVENT_SRV_LOOKUP : EVENT_HOST_DB_LOOKUP, NULL);
      Warning("bogus entry deleted from HostDB: missing hostname");
      // 对于不合规的记录，需要从 HostDB 中删除
      hostDB.delete_block(r);
      return false;
    }
    Debug("hostdb", "hostname = %s", r->hostname());
  }

  // 不是 SRV，但是 Round Robin 类型，应该是域名解析后存在多个 IP 地址的记录
  if (!r->is_srv && r->round_robin) {
    if (!r->rr()) {
      // 记录中不包含 Round Robin 数据，可能是之前从 Heap 分配空间失败导致
      // 按照解析失败处理，回调 NULL 给调用者。
      ink_assert(!"missing round-robin");
      cont->handleEvent(is_srv ? EVENT_SRV_LOOKUP : EVENT_HOST_DB_LOOKUP, NULL);
      Warning("bogus entry deleted from HostDB: missing round-robin");
      // 对于不合规的记录，需要从 HostDB 中删除
      hostDB.delete_block(r);
      return false;
    }
    ip_text_buffer ipb;
    Debug("hostdb", "RR of %d with %d good, 1st IP = %s", r->rr()->rrcount, r->rr()->good, ats_ip_ntop(r->ip(), ipb, sizeof ipb));
  }

  // 将各种失败、数据不完整的情况都过滤之后，回调包含有效数据的 HostDBInfo 给调用者。
  cont->handleEvent(is_srv ? EVENT_SRV_LOOKUP : EVENT_HOST_DB_LOOKUP, r);

  // 回调返回后，判断 Entry 是否被删除，如果 Entry 被删除则从 HostDB 中删除 Entry
  if (!r->full) {
    Warning("bogus entry deleted from HostDB: none");
    hostDB.delete_block(r);
    return false;
  }

  // 返回 true 表示传入的 HostDBInfo 是有效的记录，
  // 在目前的代码里只有与集群功能相关的代码判断了该返回值，
  // 仅当返回值为 true 时，才会将 HostDBInfo 发送到集群里
  return true;
}
```

### probeEvent

异步查询 HostDB 的处理函数。

```
//
// Probe state
//
int
HostDBContinuation::probeEvent(int /* event ATS_UNUSED */, Event *e)
{
  ink_assert(!link.prev && !link.next);
  EThread *t = e ? e->ethread : this_ethread();

  // 首先对 Action 进行上锁
  MUTEX_TRY_LOCK_FOR(lock, action.mutex, t, action.continuation);
  if (!lock.is_locked()) {
    mutex->thread_holding->schedule_in(this, HOST_DB_RETRY_PERIOD);
    return EVENT_CONT;
  }

  // 然后判断当前域名解析任务是否被发起者取消？
  if (action.cancelled) {
    hostdb_cont_free(this);
    return EVENT_DONE;
  }

  // 然后判断 hostdb 是否被禁用？解析请求填充的数据是否有效？
  // 如果无效的话，就要通知调用者，或者通知到集群里的成员
  if (!hostdb_enable || (!*md5.host_name && !md5.ip.isValid())) {
    if (action.continuation)
      action.continuation->handleEvent(EVENT_HOST_DB_LOOKUP, NULL);
    if (from)
      do_put_response(from, 0, from_cont);
    hostdb_cont_free(this);
    return EVENT_DONE;
  }

  // 是否强制进行 DNS 解析
  if (!force_dns) {
    // Do the probe
    //
    // 如果没有强制进行 DNS 解析，那么就调用 probe() 查询 Hosts 文件或 HostDB
    HostDBInfo *r = probe(mutex, md5, false);

    // 如果找到了，将 HostDB 的命中数递增，因此可以将 HostDB 理解为 DNS 解析结果的缓存
    if (r)
      HOSTDB_INCREMENT_DYN_STAT(hostdb_total_hits_stat);

    // 回调结果
    // BUG: 这里在调用 reply_to_cont() 时缺少第三个参数，应改使用 "is_srv()" 作为第三个参数。
    // 该问题由 Vijay Mamidi 发现，并通过 commit bf014061 修复。
    if (action.continuation && r)
      reply_to_cont(action.continuation, r);

    // Respond to any remote node
    //
    // 如果是集群成员发起的请求，则回调结果给集群成员。
    // 结果可能为 NULL，表示在当前节点的 HostDB 里未找到
    if (from)
      do_put_response(from, r, from_cont);

    // If it suceeds or it was a remote probe, we are done
    //
    // 任务完成
    if (r || from) {
      hostdb_cont_free(this);
      return EVENT_DONE;
    }
    // If it failed, do a remote probe
    //
    // 如果在当前节点的 HostDB 里没有解析到，则向集群内其他节点的 HostDB 发起查询请求，
    // 然后并立即返回，等待集群节点查询之后再回调 clusterEvent。
    // 可以看出，在 ATS 诞生的年代，DNS 解析的效率是非常低的，因此才会实现 HostDB 的集群功能。
    if (do_get_response(e))
      return EVENT_CONT;
  }
  // If there are no remote nodes to probe, do a DNS lookup
  //
  // 发起 DNS 解析请求，然后等待来自 DNS 子系统的回调。
  do_dns();
  return EVENT_DONE;
}
```

### probe

查询 HostDB 的底层方法。

```
HostDBInfo *
probe(ProxyMutex *mutex, HostDBMD5 const &md5, bool ignore_timeout)
{
  // 第一部分：在 Hosts 文件内进行检索，如果找到了就直接返回结果
  // If we have an entry in our hosts file, we don't need to bother with DNS
  // Make a local copy/reference so we don't have to worry about updates.
  Ptr<RefCountedHostsFileMap> current_host_file_map = hostDB.hosts_file_ptr;

  ts::ConstBuffer hname(md5.host_name, md5.host_len);
  HostsFileMap::iterator find_result = current_host_file_map->hosts_file_map.find(hname);
  if (find_result != current_host_file_map->hosts_file_map.end()) {
    return &(find_result->second);
  }

  // 第二部分：查找 HostDB
  // 这里传入的第一个参数是 ProxyMutex *mutex，主要是为了 HOSTDB_INCREMENT_DYN_STAT 等宏定义展开时使用
  // 这个 mutex 应该指向 HostDB 中与 md5 对应的分区的 mutex。
  // 也就是说 mutex == hostDB.lock_for_bucket((int)(fold_md5(md5.hash) % hostDB.buckets))
  ink_assert(this_ethread() == hostDB.lock_for_bucket((int)(fold_md5(md5.hash) % hostDB.buckets))->thread_holding);
  // 此处判断 hostdb_enable 应该是多余的，hostdb_enable 的配置是不支持动态更新的，关闭和开启 HostDB 均需要重启 ATS。
  // 在调用 probe() 之前，对 hostdb_enable 的判定都已经完成了，因此这里不需要考虑 hostdb_enable 的动态变化。
  if (hostdb_enable) {
    uint64_t folded_md5 = fold_md5(md5.hash);
    // 通过 lookup_block() 在 HostDB 内按 MD5 值进行查找，
    // 这里存在一个无影响的小 Bug：hostDB.levels 应改为 "hostDB.levels - 1"，具体可以查看 MultiCache 章节。
    HostDBInfo *r = hostDB.lookup_block(folded_md5, hostDB.levels);
    Debug("hostdb", "probe %.*s %" PRIx64 " %d [ignore_timeout = %d]", md5.host_len, md5.host_name, folded_md5, !!r,
          ignore_timeout);
    // 避免折叠后的 md5 值存在碰撞，再次验证 md5 的高 64 bits
    if (r && md5.hash[1] == r->md5_high) {
      // Check for timeout (fail probe)
      //
      if (r->is_deleted()) {
        // Entry 标记为删除的，直接返回 NULL
        Debug("hostdb", "HostDB entry was set as deleted");
        return NULL;
      } else if (r->failed()) {
        // 由于解析失败的结果也会被缓存到 HostDB 内，因此当 HostDBInfo 标记为失败时，
        // 还需要进一步判断该缓存是否过期。
        Debug("hostdb", "'%.*s' failed", md5.host_len, md5.host_name);
        if (r->is_ip_fail_timeout()) {
          // 如果该缓存过期了，则返回 NULL，否则将缓存的结果返回
          Debug("hostdb", "fail timeout %u", r->ip_interval());
          return NULL;
        }
      } else if (!ignore_timeout && r->is_ip_timeout() && !r->serve_stale_but_revalidate()) {
        // 如果调用 probe() 时，传入的 ignore_timeout == false，那么就表示不能返回过期的 HostDBInfo，
        // 目前只有 dnsEvent 会传入 ignore_timeout == true，因为 dnsEvent 会将最新的解析结果更新到 HostDBInfo 里。
        // 而 serve_stale_but_revalidate() 则是看配置项里是否允许先返回过期的结果，同时在后台发起自动刷新。
        // 因此，这里的 if 判定条件是：
        //   - 当 HostDBInfo 的 IP 记录过期了，
        //   - 同时又不能返回过期的信息，
        //   - 并且没有开启返回过期内容并发起自动刷新的配置时，
        // 直接返回 NULL。
        Debug("hostdb", "timeout %u %u %u", r->ip_interval(), r->ip_timestamp, r->ip_timeout_interval);
        HOSTDB_INCREMENT_DYN_STAT(hostdb_ttl_expires_stat);
        return NULL;
      }
      // error conditions
      // 以下是一些错误情况的判定，通常来说这些 if 条件都不会匹配为 true，
      // 只有出现 bug 时，才有可能出现这些状况，这里其实可以理解为一些 runtime assert
      if (r->reverse_dns && !r->hostname()) {
        // 反向 DNS 解析的记录里没有包含 hostname 的数据
        Debug("hostdb", "missing reverse dns");
        hostDB.delete_block(r);
        return NULL;
      }
      if (r->round_robin && !r->rr()) {
        // 轮询 DNS 解析的记录里没有包含多个轮询记录
        Debug("hostdb", "missing round-robin");
        hostDB.delete_block(r);
        return NULL;
      }
      // Check for stale (revalidate offline if we are the owner)
      // -or-
      // we are beyond our TTL but we choose to serve for another N seconds [hostdb_serve_stale_but_revalidate seconds]
      // 对不新鲜和超时的记录进行刷新，但是有额外的附加条件
      // 对于不新鲜的记录：
      //   - ignore_timeout == true 说明这是来自 dnsEvent 的调用，不需要刷新，
      //   - 对于反向 DNS 解析的记录，不刷新，
      //   - 集群里不属于当前节点管理的部分，不刷新（集群部分的代码暂不做分析），
      //   - 其它情况则直接刷新记录（刷新记录只是将该记录的缓存时间重新计时）。
      // 对于已经过期的记录：
      //   - 需要判断用户配置是否开启了允许返回不新鲜记录的功能
      //   - 只有开启该功能之后，才会刷新记录
      //   - BUG：? 但是这里没有排除反向 DNS 解析
      if ((!ignore_timeout && r->is_ip_stale() && !cluster_machine_at_depth(master_hash(md5.hash)) && !r->reverse_dns) ||
          (r->is_ip_timeout() && r->serve_stale_but_revalidate())) {
        Debug("hostdb", "stale %u %u %u, using it and refreshing it", r->ip_interval(), r->ip_timestamp, r->ip_timeout_interval);
        r->refresh_ip();
        // 这里排除掉 host_name 是 IP 地址的情况，例如 127.0.0.1
        if (!is_dotted_form_hostname(md5.host_name)) {
          // 将不新鲜的或者超时的记录的缓存时间刷新后，立即创建一个 HostDBContinuation 发起 DNS 解析。
          // 这就是由 HostDB 内部发起的 DNS 解析，该解析从 DNS 子系统回调之后，解析结果会更新到 HostDB 内，
          // 但是并不会将解析结果回调到上层状态机，而且可以看到这里也没有设置回调的状态机，
          // 查阅 init 方法的代码，可以看到此时 action.continuation == NULL
          // 这种内部发起的 DNS 解析，仅仅会完成 IPv4 或 IPv6 其中一种类型的地址解析，
          // 不会先解析 IPv4 然后再尝试 IPv6，也不会先解析 IPv6 然后再尝试 IPv4。
          HostDBContinuation *c = hostDBContAllocator.alloc();
          HostDBContinuation::Options copt;
          // 根据过期记录的 IP 地址类型，发起同类型的解析请求（IPv4 或 IPv6）
          copt.host_res_style = host_res_style_for(r->ip());
          c->init(md5, copt);
          // 通过 do_dns() 调用 DNS 子系统完成解析
          c->do_dns();
        }
      }

      // 记录该 HostDB 节点的命中次数，hits 的最大值为 7。
      // hits 主要用来将访问不频繁的 Entry 节点向下一等级移动，空出位置存放新数据。
      // 在 CH07 中的 MultiCache<C>::insert_block 中详细说明了 hits 的用途。
      r->hits++;
      // 由于 hits 的值是 0 ~ 7，很容易溢出，这里为了防止溢出做了一个处理
      if (!r->hits)
        r->hits--;
      return r;
    }
  }
  // 没有开启 HostDB，或者在 HostDB 中没有找到对应的结果，返回 NULL
  return NULL;
}
```

### do\_dns

如果查询 HostDB 失败，那么就要调用 DNS 子系统解析域名，然后再将解析结果存入到 HostDB 内，然后将结果回调到上层状态机。

```
//
// Query the DNS processor
//
void
HostDBContinuation::do_dns()
{
  ink_assert(!action.cancelled);
  // 如果 md5.host_name 里是 IP 地址，则不进行 DNS 解析
  if (is_byname()) {
    Debug("hostdb", "DNS %s", md5.host_name);
    IpAddr tip;
    if (0 == tip.load(md5.host_name)) {
      // check 127.0.0.1 format // What the heck does that mean? - AMC
      if (action.continuation) {
        // 直接插入一条记录到 HostDB：主机名 127.0.0.1 解析成 IP 地址 127.0.0.1
        // 在前面对 HostDB 进行查询时，并不对查询的域名为 IP 地址字符串进行判断，
        // 因此将 127.0.0.1 作为主机名，也可以从 HostDB 内查询到这条插入的记录。
        HostDBInfo *r = lookup_done(tip, md5.host_name, false, HOST_DB_MAX_TTL, NULL);
        reply_to_cont(action.continuation, r);
      }
      hostdb_cont_free(this);
      return;
    }
  }
  // 排除掉 127.0.0.1 的特殊情况后，需要判断用户是否配置了 HostDB 查询的超时设置
  // 并且按照用户的要求设置查询超时时间
  if (hostdb_lookup_timeout) {
    timeout = mutex->thread_holding->schedule_in(this, HRTIME_SECONDS(hostdb_lookup_timeout));
  } else {
    timeout = NULL;
  }
  // 这里的 set_check_pending_dns() 方法用来合并相同域名的 DNS 子系统的调用，
  // 如果在发起调用之前，已经有其他状态机发起了对相同域名的解析请求，那么我们就只需要等待 DNS 子系统的回调，
  // 然后共享回调的解析结果，这样就避免了大量重复的 DNS 解析请求。
  if (set_check_pending_dns()) {
    // 将调用 DNS 子系统的参数设置好
    DNSProcessor::Options opt;
    // DNS 解析的超时时间
    opt.timeout = dns_lookup_timeout;
    // 解析类型：IPv4，IPv6，...
    opt.host_res_style = host_res_style_for(md5.db_mark);
    // dnsEvent 用来接收 DNS 子系统回调的结果，并将 解析结果转存到 HostDBInfo 的数据结构里
    SET_HANDLER((HostDBContHandler)&HostDBContinuation::dnsEvent);
    // 根据解析的类型调用不同的 DNS 子系统的 API
    if (is_byname()) {
      // 正向 DNS 解析
      if (md5.dns_server)
        opt.handler = md5.dns_server->x_dnsH;
      pending_action = dnsProcessor.gethostbyname(this, md5.host_name, opt);
    } else if (is_srv()) {
      // SRV 记录解析
      Debug("dns_srv", "SRV lookup of %s", md5.host_name);
      pending_action = dnsProcessor.getSRVbyname(this, md5.host_name, opt);
    } else {
      // 反向地址解析（这里应该有一个 if 判断 md5.ip 的合法性）
      ip_text_buffer ipb;
      Debug("hostdb", "DNS IP %s", md5.ip.toString(ipb, sizeof ipb));
      pending_action = dnsProcessor.gethostbyaddr(this, &md5.ip, opt);
    }
    // 最后应该有一个 else 兜底其他类型
  } else {
    // dnsPendingEvent 用于接收共享的 DNS 解析结果
    SET_HANDLER((HostDBContHandler)&HostDBContinuation::dnsPendingEvent);
  }
}
```

### insert

根据 Hash 值，从 HostDB 获取一个 Entry，并使用当前 HostDBCont 实例的资料填充该 Entry 内的数据。

```
//
// Insert a HostDBInfo into the database
// A null value indicates that the block is empty.
//
HostDBInfo *
HostDBContinuation::insert(unsigned int attl)
{
  // 计算 hash 值
  uint64_t folded_md5 = fold_md5(md5.hash);
  // 计算 bucket 编号
  int bucket = folded_md5 % hostDB.buckets;

  ink_assert(this_ethread() == hostDB.lock_for_bucket(bucket)->thread_holding);
  // 在 3 层 HostDB 数据库内查找相同 hash 值的 Entry
  // remove the old one to prevent buildup
  // 这里存在一个无影响的小 Bug：lookup_block 的参数 level 是从 0 开始的，
  // 因此这里的 "3" 应改为 "hostDB.levels - 1"，具体可以查看 MultiCache 章节。
  // 在 commit b5538298 进行了一次修复，但是没有修改正确。
  HostDBInfo *old_r = hostDB.lookup_block(folded_md5, 3);
  // 如果找到了，就将其清理掉
  if (old_r)
    hostDB.delete_block(old_r);
  // 使用当前 hash 值从 HostDB 内获取一个 Entry，具体过程请查看 MultiCache 章节
  HostDBInfo *r = hostDB.insert_block(folded_md5, NULL, 0);
  // 向 Entry 填充必要的数据
  r->md5_high = md5.hash[1];
  if (attl > HOST_DB_MAX_TTL)
    attl = HOST_DB_MAX_TTL;
  r->ip_timeout_interval = attl;
  r->ip_timestamp = hostdb_current_interval;
  Debug("hostdb", "inserting for: %.*s: (md5: %" PRIx64 ") bucket: %d now: %u timeout: %u ttl: %u", md5.host_len, md5.host_name,
        folded_md5, bucket, r->ip_timestamp, r->ip_timeout_interval, attl);
  // 返回指向 Entry 的指针（该指针的地址范围在 HostDB 空间内）
  return r;
}
```

### lookup\_done

本方法负责将数据存储到本地的 HostDB 数据库内，返回的指针指向包含解析结果的 HostDB 内的 Entry，该 Entry 内的存储格式可参照 HostDBInfo 的定义。

```
// Lookup done, insert into the local table, return data to the
// calling continuation or to the calling cluster node.
//
HostDBInfo *
HostDBContinuation::lookup_done(IpAddr const &ip, char const *aname, bool around_robin, unsigned int ttl_seconds, SRVHosts *srv)
{
  HostDBInfo *i = NULL;

  ink_assert(this_ethread() == hostDB.lock_for_bucket((int)(fold_md5(md5.hash) % hostDB.buckets))->thread_holding);
  // 处理失败的情况：IP 地址无效，没有域名，域名为空字符
  if (!ip.isValid() || !aname || !aname[0]) {
    // 根据记录类型，分别产生日志
    if (is_byname()) {
      Debug("hostdb", "lookup_done() failed for '%.*s'", md5.host_len, md5.host_name);
    } else if (is_srv()) {
      Debug("dns_srv", "SRV failed for '%.*s'", md5.host_len, md5.host_name);
    } else {
      ip_text_buffer b;
      Debug("hostdb", "failed for %s", md5.ip.toString(b, sizeof b));
    }
    // 调用 insert() 插入 HostDBInfo 对象到 HostDB 内
    i = insert(hostdb_ip_fail_timeout_interval); // currently ... 0
    // 设置各种状态
    i->round_robin = false;
    i->round_robin_elt = false;
    i->is_srv = is_srv();
    i->reverse_dns = !is_byname() && !is_srv();

    // 清理并释放旧数据占用的空间
    i->set_failed();
  } else { // 有效数据处理
    // 根据 hostdb_ttl_mode 的设置，确认 HostDBInfo 记录的 TTL 值
    switch (hostdb_ttl_mode) {
    default:
      ink_assert(!"bad TTL mode");
    case TTL_OBEY: // 依传入的 TTL 值
      break;
    case TTL_IGNORE: // 总是设为固定值
      ttl_seconds = hostdb_ip_timeout_interval * 60;
      break;
    case TTL_MIN: // 限制 TTL 的最大值
      if (hostdb_ip_timeout_interval * 60 < ttl_seconds)
        ttl_seconds = hostdb_ip_timeout_interval * 60;
      break;
    case TTL_MAX: // 限制 TTL 的最小值
      if (hostdb_ip_timeout_interval * 60 > ttl_seconds)
        ttl_seconds = hostdb_ip_timeout_interval * 60;
      break;
    }
    HOSTDB_SUM_DYN_STAT(hostdb_ttl_stat, ttl_seconds);

    // Not sure about this - it seems wrong but I can't be sure. If we got a fail
    // in the DNS event, 0 is passed in which we then change to 1 here. Do we need this
    // to be non-zero to avoid an infinite timeout?
    // TTL 值未设置时的默认值（1 秒），避免插入 HostDB 的数据就是过期的数据
    if (0 == ttl_seconds)
      ttl_seconds = 1;

    // 调用 insert() 插入 HostDBInfo 对象到 HostDB 内
    i = insert(ttl_seconds);
    // 主记录该标志设置为 False，只有保存在 Heap 区域的 Round Robin 记录，该标志才会设置为 True
    i->round_robin_elt = false; // only true for elements explicitly added as RR elements.
    // 根据记录类型，填充 HostDBInfo 结构的数据
    if (is_byname()) { // 将域名解析为 IP 地址
      ip_text_buffer b;
      Debug("hostdb", "done %s TTL %d", ip.toString(b, sizeof b), ttl_seconds);
      ats_ip_set(i->ip(), ip);
      i->round_robin = around_robin;
      i->reverse_dns = false;
      // 由于 insert() 和 lookup_done 仅负责填充 A、AAAA 的主记录，也就是 HostDBInfo 部分，
      // 不负责填充 Heap 区域的数据，因此， hostname 的信息被临时保存到 HostDBCont 实例的成员中。
      if (md5.host_name != aname) {
        ink_strlcpy(md5_host_name_store, aname, sizeof(md5_host_name_store));
      }
      i->is_srv = false;
    } else if (is_srv()) { // SRV 解析结果
      ink_assert(srv && srv->srv_host_count > 0 && srv->srv_host_count <= 16 && around_robin);

      i->data.srv.srv_offset = srv->srv_host_count;
      i->reverse_dns = false;
      i->is_srv = true;
      i->round_robin = around_robin;

      // 由于 insert() 和 lookup_done 仅负责填充 SRV 的主记录，也就是 HostDBInfo 部分，
      // 不负责填充 Heap 区域的数据，因此， hostname 的信息被临时保存到 HostDBCont 实例的成员中。
      if (md5.host_name != aname) {
        ink_strlcpy(md5_host_name_store, aname, sizeof(md5_host_name_store));
      }

    } else { // PTR 解析结果
      Debug("hostdb", "done '%s' TTL %d", aname, ttl_seconds);
      const size_t s_size = strlen(aname) + 1;
      void *s = hostDB.alloc(&i->data.hostname_offset, s_size);
      if (s) {
        ink_strlcpy((char *)s, aname, s_size);
        i->round_robin = false;
        i->reverse_dns = true;
        i->is_srv = false;
      } else {
        // 如果从 Heap 区分类空间失败，则
        // 标记该 HostDB Entry 为已删除状态，并清理 hash 数据，返回 NULL
        ink_assert(!"out of room in hostdb data area");
        Warning("out of room in hostdb for reverse DNS data");
        hostDB.delete_block(i);
        return NULL;
      }
    }
  }

  /*
   * 此处删除了 commit 0e703e1e3b 和 commit 0cd1ef3e 提交的代码，
   * 详细分析可以查看 HostDBInfo 章节。
  if (aname) {
    const size_t s_size = strlen(aname) + 1;
    void *host_dest = hostDB.alloc(&i->hostname_offset, s_size);
    if (host_dest) {
      ink_strlcpy((char *)host_dest, aname, s_size);
      *((char *)host_dest + s_size) = '\0';
    } else {
      Warning("Out of room in hostdb for hostname (data area full!)");
      hostDB.delete_block(i);
      return NULL;
    }
  }*/



  // 来自集群的请求
  if (from_cont)
    do_put_response(from, i, from_cont);
  ink_assert(!i->round_robin || !i->reverse_dns);
  // 返回指向 HostDBInfo 的指针（该指针的地址位于 HostDB 范围）
  return i;
}
```


### set\_check\_pending\_dns

在 HostDB 内有多个底层的 MultiCache 分区，HostDB 为每个分区创建了一个队列，用于保存操作该分区的 HostDBContinuation 状态机。

在 HostDBCache 中定义了一个队列数组 `pending_dns[MULTI_CACHE_PARTITIONS]`，该数组内包含的队列数量与 HostDB 的分区数量一致。可以通过对 MD5 值取模的方法得到与之对应的队列，为了简化该操作，HostDB 定义了 `pending_dns_for_hash()` 方法。

```
struct HostDBCache : public MultiCache<HostDBInfo> {
...
  Queue<HostDBContinuation, Continuation::Link_link> pending_dns[MULTI_CACHE_PARTITIONS];
  Queue<HostDBContinuation, Continuation::Link_link> &pending_dns_for_hash(INK_MD5 &md5);
...
}
```

`set_check_pending_dns()` 的功能是对某一个 `pending_dns` 队列进行 `set` 和 `check` 操作，具体分析如下：

```
int
HostDBContinuation::set_check_pending_dns()
{
  // 根据 MD5 值获得对应的 pending_dns 队列
  Queue<HostDBContinuation> &q = hostDB.pending_dns_for_hash(md5.hash);
  // 使用一个 for 循环自队列头部开始遍历
  HostDBContinuation *c = q.head;
  for (; c; c = (HostDBContinuation *)c->link.next) {
    // 由于队列内包含的 HostDBContinuation 对象是对 MD5 取模后结果一致的情况，
    // 因此还需要完全比对一次 MD5 的值，才能确定当前将要发起 DNS 解析的域名是否与队列中的一致。
    if (md5.hash == c->md5.hash) {
      // 如果与队列中已经存在的 HostDBContinuation 的域名是一致的，
      // 那么就插入队列后返回 false，表示只需要等待并分享 DNS 解析的结果即可。
      Debug("hostdb", "enqueuing additional request");
      q.enqueue(this);
      return false;
    }
  }
  // 如果未在 pending_dns 队列中找到域名相同的请求，那么就说明当前的 HostDBContinuation 是第一个解析该域名的对象，
  // 那么就要插队队列后返回 true，表示需要调用 DNS 子系统发起 DNS 解析，还需要负责将 DNS 解析的结果写入到 HostDB，最后再分享解析结果。
  q.enqueue(this);
  return true;
}
```

### remove\_trigger\_pending\_dns

用来重新触发 pending_dns 队列里相同查询任务的 HostDBCont 实例。

```
void
HostDBContinuation::remove_trigger_pending_dns()
{
  // 通过 hash 算法找到对应的 HostDB 分区的 pending_dns 队列。
  Queue<HostDBContinuation> &q = hostDB.pending_dns_for_hash(md5.hash);
  // 将当前实例从 pending_dns 队列里删除。
  q.remove(this);
  // 遍历 pending_dns 队列
  HostDBContinuation *c = q.head;
  // 创建本地临时队列
  Queue<HostDBContinuation> qq;
  while (c) {
    HostDBContinuation *n = (HostDBContinuation *)c->link.next;
    // 找到相同的查询任务
    if (md5.hash == c->md5.hash) {
      Debug("hostdb", "dequeuing additional request");
      // 将任务从 pending_dns 队列里删除
      q.remove(c);
      // 然后放入本地临时队列
      qq.enqueue(c);
    }
    c = n;
  }
  // 遍历本地临时队列，逐个回调（重新触发）HostDBCont 实例。
  while ((c = qq.dequeue()))
    c->handleEvent(EVENT_IMMEDIATE, NULL);
}
```

### dnsEvent

通过 `set_check_pending_dns()` 查询到任务队列里不存在相同 DNS 解析任务后，就会将当前状态机的处理函数设置为 `dnsEvent()`，然后调用 DNS 子系统解析域名，通过 `dnsEvent()` 等待并接收来自 DNS 子系统返回的结果。收到来自 DNS 子系统的回调，将结果存入 HostDB，然后再让那些具有相同查询任务的 `HostDBContinuation` 重新去 HostDB 里查询一次。

如果遇到 DNS 解析任务超时等情况，那么 HostDB 里就没有保存相关的信息，具有相同查询任务的 `HostDBContinuation` 重新去 HostDB 里查询后发现无法找到数据时，会再由其中一个实例触发 DNS 解析任务，其它相同查询任务的实例则继续等待。如等待任务超时，则会提前返回调用者 NULL 的结果。

`dnsEvent` 的处理流程，可以分为三个部分：

1. 接收 DNS 解析结果超时，
2. 保存 DNS 结果到 HostDB 内，
3. 回调解析结果及后续处理。

第一部分需要处理：

- 任务超时：回调解析失败给调用者，然后会扩展等待  hostdb_insert_timeout 的时间，以尽可能的收到来自 DNS 的解析结果；
- 扩展超时：销毁当前实例，但是相同查询任务里的一个实例会重新发起 DNS 解析。

第二部分需要处理：

- SRV 类型的记录：总是以 Round Robin 方式保存数据到 Heap 区，
- Round Robin 类型的记录：需要过滤到非法记录，剩余记录保存到 Heap 区，
- A、AAAA 类型的单一记录：直接保存到 HostDB Entry 内，
- PTR 类型的单一记录：hostname 保存到 Heap 区。

第三部分需要处理：

- 集群模式下，多个节点间的任务回调
- 回调状态机：
   - 如果解析失败，调用者接受备选 IP 协议的结果，则尝试发起备选 IP 协议的结果；
   - 上锁失败，通过 probeEvent 重新查询 HostDB，再次回调；同时重新触发所有相同查询任务的 HostDBCont 实例；
   - 上锁成功，直接回调
- 回调成功，或者不需要回调，则重新触发所有相同查询任务的 HostDBCont 实例，然后销毁当前 HostDBCont 实例。

以下是详细的代码注释：

```
// DNS lookup result state
//
int
HostDBContinuation::dnsEvent(int event, HostEnt *e)
{
  ink_assert(this_ethread() == hostDB.lock_for_bucket(fold_md5(md5.hash) % hostDB.buckets)->thread_holding);
  // 关闭 timeout 事件
  if (timeout) {
    timeout->cancel(this);
    timeout = NULL;
  }
  EThread *thread = mutex->thread_holding;
  if (event == EVENT_INTERVAL) {
    // 处理超时事件
    if (!action.continuation) {
      // HostDB 内部发起的 DNS 解析请求，如果遇到超时直接就结束了。
      // 调用 remove_trigger_pending_dns 释放等待队列中的任务，重新触发 HostDB 查询。
      // give up on insert, it has been too long
      remove_trigger_pending_dns();
      hostdb_cont_free(this);
      return EVENT_DONE;
    }
    // 锁定 action.mutex，尝试回调上层状态机
    MUTEX_TRY_LOCK_FOR(lock, action.mutex, thread, action.continuation);
    if (!lock.is_locked()) {
      // 锁定失败则延迟 HOST_DB_RETRY_PERIOD 重试
      timeout = thread->schedule_in(this, HOST_DB_RETRY_PERIOD);
      return EVENT_CONT;
    }
    // [amc] Callback to client to indicate a failure due to timeout.
    // We don't try a different family here because a timeout indicates
    // a server issue that won't be fixed by asking for a different
    // address family.
    // 锁定成功，将 NULL 作为解析结果回调上层状态机。
    if (!action.cancelled && action.continuation)
      action.continuation->handleEvent(EVENT_HOST_DB_LOOKUP, NULL);
    // 这里没有调用 remove_trigger_pending_dns，因此仍然有很多 `HostDBContinuation` 在等待队列中。
    // 清理 action，使其转换为 HostDB 内部请求，继续等待 DNS 子系统的返回。
    action = NULL;
    // do not exit yet, wait to see if we can insert into DB
    timeout = thread->schedule_in(this, HRTIME_SECONDS(hostdb_insert_timeout));
    return EVENT_DONE;
  } else {
    // 处理 DNS 子系统的回调
    // DNS 子系统回调时传入的是 DNS_EVENT_LOOKUP 事件，数据对象为 HostEnt 类型，如果指向 NULL，则说明 DNS 解析超时或失败。
    // 首先是一部分基础变量的初始化
    bool failed = !e;

    bool rr = false;
    pending_action = NULL;

    // 判断 SRV 和 Round Robin 类型的对象
    // 感觉这个逻辑写的很奇怪，如使用 failed 作为主要判定条件，内部再区分 SRV 类型和非 SRV 类型可读性是否会更好一些？
    // 如 DNS 子系统返回了解析记录，
    //  - 类型为 SRV 并且记录数大于 0 则设置 rr 为 true，
    //  - 类型为非 SRV，并且记录数大于 1 则设置 rr 为 true。
    if (is_srv()) {
      rr = !failed && (e->srv_hosts.srv_host_count > 0);
    } else if (!failed) {
      rr = 0 != e->ent.h_addr_list[1];
    } else {
    }

    // TTL 以分钟为单位和以秒为单位
    ttl = failed ? 0 : e->ttl / 60;
    int ttl_seconds = failed ? 0 : e->ttl; // ebalsa: moving to second accuracy

    // 查询一下是否有老的 HostDBInfo 记录，如果存在说明我们这一次是更新过期的数据
    // 同时要考虑到 RoundRobin 类型
    HostDBInfo *old_r = probe(mutex, md5, true);
    HostDBInfo old_info;
    if (old_r)
      old_info = *old_r;
    HostDBRoundRobin *old_rr_data = old_r ? old_r->rr() : NULL;
#ifdef DEBUG
    // 这里是对保存在 heap 区内的 RoundRobin 数据做一次校验，
    // 应该是之前存在 bug，导致 HEAP 区经常被写乱掉，才添加了这部分代码及时发现问题。
    if (old_rr_data) {
      for (int i = 0; i < old_rr_data->rrcount; ++i) {
        if (old_r->md5_high != old_rr_data->info[i].md5_high || old_r->md5_low != old_rr_data->info[i].md5_low ||
            old_r->md5_low_low != old_rr_data->info[i].md5_low_low)
          ink_assert(0);
      }
    }
#endif
    // 这部分是对 DNS 返回的数据进行解析，
    // 确认地址类型 af，确认记录数量 n，大于 1 就是 RoundRobin
    int n = 0, nn = 0;
    void *first = 0;
    uint8_t af = e ? e->ent.h_addrtype : AF_UNSPEC; // address family
    if (rr) {
      // 根据对前述代码的分析，当 rr 为 true 时，failed 一定为 false
      if (is_srv() && !failed) {
        // 当结果为 SRV 记录时，保存 SRV 记录数量到变量 n
        n = e->srv_hosts.srv_host_count;
      } else {
        // 结果为非 SRV 记录时，在 ent.h_addr_list[] 内保存了至少 2 条记录，
        // 使用 for 循环遍历所有记录，统计有效的地址数量，递增变量 n
        void *ptr; // tmp for current entry.
        for (; nn < HOST_DB_MAX_ROUND_ROBIN_INFO && 0 != (ptr = e->ent.h_addr_list[nn]); ++nn) {
          if (is_addr_valid(af, ptr)) {
            if (!first)
              first = ptr;
            ++n;
          } else {
            Warning("Zero address removed from round-robin list for '%s'", md5.host_name);
          }
          // what's the point of @a n? Should there be something like
          // if (n != nn) e->ent.h_addr_list[n] = e->ent->h_addr_list[nn];
          // with a final copy of the terminating null? - AMC
        }
        // 如果所有地址都无效，则将 failed 和 rr 设置为无效
        if (!first) {
          failed = true;
          rr = false;
        }
      }
    } else if (!failed) {
      // 不是 SRV 记录，不是 RoundRobin 记录，可能是单一 A、AAAA 或 PTR 类型，直接保存地址到 first 指针。
      // 注意：这里的 first 可能仍然指向 NULL，这表示返回的记录类型可能是反向地址解析
      first = e->ent.h_addr_list[0];
    } // else first is 0.

    HostDBInfo *r = NULL;
    IpAddr tip; // temp storage if needed.

    if (is_byname()) {
      // 如果请求的是将域名解析为 IP 地址（A、AAAA）
      // 将 first 指向的 IP 地址格式化到 IpAddr tip 内，
      // 注意：这里 first 如果为 NULL 的话，此时仍然会保存到 HostDB，以下判定条件也应该成立
      //  - failed == true
      //  - ttl_seconds == 0
      //  - tip.isValid() == false
      if (first)
        ip_addr_set(tip, af, first);
      // 调用 lookup_done() 向 HostDB 内插入 DNS 解析的结果
      r = lookup_done(tip, md5.host_name, rr, ttl_seconds, failed ? 0 : &e->srv_hosts);
    } else if (is_srv()) {
      // 如果请求的是 SRV 记录，将 IP 地址类型设置为 AF_INET 避免后续判定时出错
      // 在 lookup_done 中会调用 IpAddr::isValid() 方法判断 IP 的有效性
      if (!failed)
        tip._family = AF_INET;       // force the tip valid, or else the srv will fail
      // 同样这里也会缓存 failed == true 的记录到 HostDB
      r = lookup_done(tip,           /* junk: FIXME: is the code in lookup_done() wrong to NEED this? */
                      md5.host_name, /* hostname */
                      rr,            /* is round robin, doesnt matter for SRV since we recheck getCount() inside lookup_done() */
                      ttl_seconds,   /* ttl in seconds */
                      failed ? 0 : &e->srv_hosts);
    } else if (failed) {
      // 保存解析失败的结果
      // 如果把这个 if 判定放在这第三步，是因为 HostDB 需要将“解析错误的结果”也缓存起来，避免频繁发起 DNS 解析。
      r = lookup_done(tip, md5.host_name, false, ttl_seconds, 0);
    } else {
      // 保存 PTR 反向地址解析的结果
      r = lookup_done(md5.ip, e->ent.h_name, false, ttl_seconds, &e->srv_hosts);
    }

    // BUG: 没有处理 r == NULL 的情况，也就是 HostDB 的 Heap 区域满了的情况，
    // 只是在 Debug 开启时，通过 assert 来检查该情况，但是生产环境通常不会使用 Debug 版本。
    // @c lookup_done should always return a valid value so @a r should be null @c NULL.
    ink_assert(r && r->app.allotment.application1 == 0 && r->app.allotment.application2 == 0);

    // 如果是 Round Robin 或者 SRV 类型
    if (rr) {
      // 从 HostDB Heap 区分配空间，用来存储 Round Robin 或 SRV 数据。
      const int rrsize = HostDBRoundRobin::size(n, e->srv_hosts.srv_hosts_length);
      HostDBRoundRobin *rr_data = (HostDBRoundRobin *)hostDB.alloc(&r->app.rr.offset, rrsize);

      Debug("hostdb", "allocating %d bytes for %d RR at %p %d", rrsize, n, rr_data, r->app.rr.offset);

      // 成功分配 Heap 区空间
      if (rr_data) {
        rr_data->length = rrsize;
        int i = 0, ii = 0;
        // 如果是 SRV 类型
        if (is_srv()) {
          int skip = 0;
          char *pos = (char *)rr_data + sizeof(HostDBRoundRobin) + n * sizeof(HostDBInfo);
          SRV *q[HOST_DB_MAX_ROUND_ROBIN_INFO];
          ink_assert(n <= HOST_DB_MAX_ROUND_ROBIN_INFO);
          // 对 SRV 记录进行排序：优先级越小越靠前，优先级相同，key 越小越靠前
          // sort
          for (i = 0; i < n; ++i) {
            q[i] = &e->srv_hosts.hosts[i];
          }
          for (i = 0; i < n; ++i) {
            for (ii = i + 1; ii < n; ++ii) {
              if (*q[ii] < *q[i]) {
                SRV *tmp = q[i];
                q[i] = q[ii];
                q[ii] = tmp;
              }
            }
          }

          // 保存 SRV 记录到 HostDB
          for (i = 0; i < n; ++i) {
            SRV *t = q[i];
            HostDBInfo &item = rr_data->info[i];
.
.
.
            Debug("dns_srv", "inserted SRV RR record [%s] into HostDB with TTL: %d seconds", t->host, ttl_seconds);
          }
          rr_data->good = rr_data->rrcount = n;
          rr_data->current = 0;

          // 将旧数据里与新数据相同的记录的状态信息复制回来
          // restore
          if (old_rr_data) {
.
.
.
          }
        } else { // 不是 SRV 就是 Round Robin 类型了
          // 保存 Round Robin 记录到 HostDB，
          // 同时调用 restore_info() 将旧数据里与新数据相同的记录的状态信息复制回来
          for (ii = 0; ii < nn; ++ii) {
            if (is_addr_valid(af, e->ent.h_addr_list[ii])) {
.
.
.
            }
          }
          rr_data->good = rr_data->rrcount = n;
          rr_data->current = 0;
        }
      } else {
        // 从 HostDB Heap 区分配空间失败，丢弃 Round Robin 数据。
        ink_assert(!"out of room in hostdb data area");
        Warning("out of room in hostdb for round-robin DNS data");
        r->round_robin = 0;
        r->round_robin_elt = 0;
      }
    }
    // 没有失败、不是 Round Robin 类型、不是 SRV 类型，
    // 这里应该是 A、AAAA 或 PTR 类型的数据，
    // 仅需要调用 restore_info() 将旧数据记录的状态信息复制回来
    if (!failed && !rr && !is_srv())
      restore_info(r, old_r, old_info, old_rr_data);
    ink_assert(!r || !r->round_robin || !r->reverse_dns);
    ink_assert(failed || !r->round_robin || r->app.rr.offset);

    // 支持 HostDB 的 Cluster 模式
    // if we are not the owner, put on the owner
    //
    ClusterMachine *m = cluster_machine_at_depth(master_hash(md5.hash));
    if (m)
      do_put_response(m, r, NULL);

    // action.continuation 不为 NULL，则需要回调结果
    // try to callback the user
    //
    if (action.continuation) {
      // 如果解析失败，但是允许尝试第二种 IP 协议
      // Check for IP family failover
      if (failed && check_for_retry(md5.db_mark, host_res_style)) {
        // 调用 refresh_MD5() 切换到第二种 IP 协议
        this->refresh_MD5(); // family changed if we're doing a retry.
        // 由于调用 refresh_MD5() 之后，mutex 可能会发生改变，因此不能直接调用 probeEvent()，
        // 需要切换状态机到 probeEvent，由事件系统重新回调。
        SET_CONTINUATION_HANDLER(this, (HostDBContHandler)&HostDBContinuation::probeEvent);
        thread->schedule_in(this, MUTEX_RETRY_DELAY);
        return EVENT_CONT;
      }

      // 解析成功、解析失败 的记录都已经保存到 HostDB，
      // 解析成功、解析失败都可能执行到这里。
      MUTEX_TRY_LOCK_FOR(lock, action.mutex, thread, action.continuation);
      if (!lock.is_locked()) {
        // 上锁失败后，先重新触发相同查询 HostDB 的任务，这些任务会通过 probeEvent 重新查询 HostDB。
        remove_trigger_pending_dns();
        // 将当前 HostDBCont 的状态机也设置为 probeEvent，重新查询 HostDB 后再回调。
        SET_HANDLER((HostDBContHandler)&HostDBContinuation::probeEvent);
        thread->schedule_in(this, HOST_DB_RETRY_PERIOD);
        return EVENT_CONT;
      }
      // 上锁成功，且 HostDB 查询认为没有被取消，则调用 reply_to_cont 完成回调
      if (!action.cancelled)
        reply_to_cont(action.continuation, r, is_srv());
    }
    // 新触发相同查询 HostDB 的任务，这些任务会通过 probeEvent 重新查询 HostDB。
    // wake up everyone else who is waiting
    remove_trigger_pending_dns();

    // 销毁当前 HostDBCont 实例
    // all done
    //
    hostdb_cont_free(this);
    return EVENT_DONE;
  }
}

```

### dnsPendingEvent

通过 `set_check_pending_dns()` 查询到任务队列里存在相同 DNS 解析任务后，就不会重复调用 DNS 子系统解析域名了，而是将当前状态机设置为 `dnsPendingEvent()`，等待接收来自 DNS 子系统返回的结果。因此对于同一个域名的解析请求，只有一个 `HostDBContinuation` 状态机发起了 DNS 解析请求，待该状态机收到 DNS 解析结果后，会在任务队列中查找相同任务的 `HostDBContinuation`，与其共享 DNS 解析结果。

这些没有发起 DNS 解析请求的 `HostDBContinuation` 状态机的处理函数都被设置为 `dnsPendingEvent()`。

```
int
HostDBContinuation::dnsPendingEvent(int event, Event *e)
{
  // 确保任务队列上锁
  ink_assert(this_ethread() == hostDB.lock_for_bucket(fold_md5(md5.hash) % hostDB.buckets)->thread_holding);
  // 取消超时控制 或 取消重试
  if (timeout) {
    timeout->cancel(this);
    timeout = NULL;
  }
  if (event == EVENT_INTERVAL) {
    // 出现超时，回调 NULL 作为解析结果
    // we timed out, return a failure to the user
    MUTEX_TRY_LOCK_FOR(lock, action.mutex, ((Event *)e)->ethread, action.continuation);
    if (!lock.is_locked()) {
      // 上锁失败后，重试操作的 Event 保存到 timeout，
      // 如果在回调期间又收到了来自 DNS 解析的结果，可以在进入处理函数时就被取消掉。
      timeout = eventProcessor.schedule_in(this, HOST_DB_RETRY_PERIOD);
      return EVENT_CONT;
    }    
    if (!action.cancelled && action.continuation)
      action.continuation->handleEvent(EVENT_HOST_DB_LOOKUP, NULL);
    // 超时回调完成后，需要从任务队列里删除任务，并回收内存
    hostDB.pending_dns_for_hash(md5.hash).remove(this);
    hostdb_cont_free(this);
    return EVENT_DONE;
  } else {
    // 设置处理函数为 probeEvent，重新查询 HostDB
    SET_HANDLER((HostDBContHandler)&HostDBContinuation::probeEvent);
    return probeEvent(EVENT_INTERVAL, NULL);
  }
}
```

## 参考资料



- [HostDB.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/HostDB.cc)
- [P_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_HostDBProcessor.h)
 


