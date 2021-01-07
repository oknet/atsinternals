# 核心组件：HostDBCache

HostDBCache 继承自 `MultiCache<HostDBInfo>`，是用于存放 HostDBInfo 对象的内存数据库，其具有以下功能：

- 周期性同步内存数据库到 host.db 文件
- 预分配内存空间，总是限定在该空间内保存 HostDBInfo 对象，不会超量分配内存空间

## 定义

```
//
// HostDBCache (Private)
//
// 定义以 HostDBInfo 为存储单元的 MultiCache 内存数据库
// 必须完成三个虚函数的定义：
//  - dup
//  - estimated_heap_bytes_per_entry
// 可选的完成以下虚函数的定义：
//  - rebuild_callout
//  - rebuild_insert_callout
// 初始化以下成员：
//  - tag_bits
//  - max_hits
//  - version
struct HostDBCache : public MultiCache<HostDBInfo> {
  // 定义虚函数
  // 在 MultiCache 执行 rebuild 库操作的时候，通过调用该方法验证数据的完整性和有效性
  // 返回 -1 表示数据损坏
  // 返回 0 表示该数据元素虽然没有损坏，但是已经没有用处了（例如过期的陈旧数据）
  // 返回 1 表示该数据元素是完整有效的数据
  int rebuild_callout(HostDBInfo *e, RebuildMC &r);
  // 启动 HostDB 实例
  int start(int flags = 0);
  // 定义虚函数
  // 返回当前 MultiCache 数据库类型的一个新实例
  MultiCacheBase *
  dup()
  {
    return new HostDBCache;
  }

  // This accounts for an average of 2 HostDBInfo per DNS cache (for round-robin etc.)
  // In addition, we can do a padding for additional SRV records storage.
  // 定义虚函数
  // 返回预估的，平均每个 Entry 占用的 HEAP 空间大小
  // 这里假设每个 Entry 平均包含 2 个 (inner) HostDBInfo 对象，如果 SRV 功能开启，则为 SRV 预留 512 字节的存储空间
  virtual size_t
  estimated_heap_bytes_per_entry() const
  {
    return sizeof(HostDBInfo) * 2 + 512 * hostdb_srv_enabled;
  }

  // 使用两个带引用计数的指针保存 /etc/hosts 文件的解析内容
  // 如果 /etc/hosts 文件发生变动，则将老的内容转移到 prev_hosts_file_ptr 保存，新的内容加载到 hosts_file_ptr，
  // 实现新旧内容的循环覆盖，同时在新旧内容切换的间隙使得部分引用旧内容的操作仍然可以合法的访问旧内容，
  // 待下一次循环覆盖后，保存在 prev_hosts_file_ptr 的第一个副本的引用计数“应该”会降为 0，并自动释放内存空间。
  // 注意：如果以极快的速度进行 /etc/hosts 文件的改变和加载，可能导致出现内存泄露的风险。
  // Map to contain all of the host file overrides, initialize it to empty
  Ptr<RefCountedHostsFileMap> hosts_file_ptr;
  // Double buffer the hosts file becase it's small and it solves dangling reference problems.
  Ptr<RefCountedHostsFileMap> prev_hosts_file_ptr;

  // 用于存储 HostDBContinuation 对象的 MULTI_CACHE_PARTITIONS = 64 个队列
  // 根据主机查询的信息，产生 Hash 值，按照 Hash 值将请求分别放入 64 个队列，
  // 如果该 Hash 值是该队列里是唯一的，那么同时向 IOCoreDNS 子系统发起请求，
  // 当从 IOCoreDNS 子系统接收到结果时，遍历队列里与该 Hash 值相同的请求，共享该结果。
  Queue<HostDBContinuation, Continuation::Link_link> pending_dns[MULTI_CACHE_PARTITIONS];
  // 对于给定的 Hash 值，返回上述 64 个队列之一
  Queue<HostDBContinuation, Continuation::Link_link> &pending_dns_for_hash(INK_MD5 &md5);
  // 构造函数，用于初始化 HostDB 的版本信息等
  HostDBCache();
};

// 全局实例
HostDBCache hostDB;
```

## 方法

### HostDBCache::HostDBCache()


设置 `tag_bits` 用于保存 MD5 值，设置 max_hits 用于保存每个 HostDBInfo 对象在查找过程中命中的次数，以在 MultiCache 的多层索引结构中进行数据单元的调整。对于 `HOST_DB_TAG_BITS` 与 `HOST_DB_HITS_BITS` 的取值要参照 HostDBInfo 中留给 MD5 值和 Hits 值的存储空间来确定。

由于 HostDBInfo 的定义在不断的演进，一旦定义发生变化，可通过读取 version 对象的值获得 HostDB 持久化数据的信息是否与当前代码兼容。这需要在 HostDB 创建时就明确，因此在构造函数中进行设置。

最后初始化了用于保存 `/etc/hosts` 记录的智能指针及 HostsFileMap 对象。

```
HostDBCache::HostDBCache()
{
  tag_bits = HOST_DB_TAG_BITS;
  max_hits = (1 << HOST_DB_HITS_BITS) - 1;
  version.ink_major = HOST_DB_CACHE_MAJOR_VERSION;
  version.ink_minor = HOST_DB_CACHE_MINOR_VERSION;
  hosts_file_ptr = new RefCountedHostsFileMap();
}
```

### int HostDBCache::start(int flags = 0)

在调用 `HostDBCache::start()` 方法之前需要先调用 `MultiCacheBase::alloc_mutexes()` 为每一个分区创建 mutex 对象。感觉应该是在 `HostDBCache::start()` 方法里初始化 MultiCache 各个分区的 mutex 对象，但是不知道是基于何种考虑，该操作被放在 `HostDBProcessor::start()` 方法里执行：

```
// source: iocore/hostdb/HostDB.cc
int
HostDBProcessor::start(int, size_t)
{
  hostDB.alloc_mutexes();

  if (hostDB.start(0) < 0)
    return -1;

```

在 `CH07S04-Base-Block-and-MultiCacheBase` 已经对 `HostDBCache::start()` 进行了初步分析，接下来再完整的分析一下。

HostDB 启动时会尝试创建和加载以 MultiCache 作为底层的 HostDB 数据库。参数：

- int flags：按位来控制 HostDBCache 在启动时的处理过程
   - bit 0：PROCESSOR_RECONFIGURE（0x01）
   - bit 1：PROCESSOR_CHECK（0x02）
   - bit 2：PROCESSOR_FIX（0x04）
   - bit 3：PROCESSOR_IGNORE_ERRORS（0x08）

返回值：

- 返回 0 表示成功
- 返回 -1 表示失败

```
int
HostDBCache::start(int flags)
{
  Store *hostDBStore;
  Span *hostDBSpan;
  char storage_path[PATH_NAME_MAX];
  int storage_size = 33554432; // 32MB default

  bool reconfigure = ((flags & PROCESSOR_RECONFIGURE) ? true : false);
  bool fix = ((flags & PROCESSOR_FIX) ? true : false);

  storage_path[0] = '\0';

  // Read configuration
  // Command line overrides manager configuration.
  // 是否开启 hostdb
  REC_ReadConfigInt32(hostdb_enable, "proxy.config.hostdb");
  REC_ReadConfigString(hostdb_filename, "proxy.config.hostdb.filename", sizeof(hostdb_filename));
  REC_ReadConfigInt32(hostdb_size, "proxy.config.hostdb.size");
  REC_ReadConfigInt32(hostdb_srv_enabled, "proxy.config.srv_enabled");
  REC_ReadConfigString(storage_path, "proxy.config.hostdb.storage_path", sizeof(storage_path));
  REC_ReadConfigInt32(storage_size, "proxy.config.hostdb.storage_size");

  // If proxy.config.hostdb.storage_path is not set, use the local state dir. If it is set to
  // a relative path, make it relative to the prefix.
  if (storage_path[0] == '\0') {
    ats_scoped_str rundir(RecConfigReadRuntimeDir());
    ink_strlcpy(storage_path, rundir, sizeof(storage_path));
  } else if (storage_path[0] != '/') {
    Layout::relative_to(storage_path, sizeof(storage_path), Layout::get()->prefix, storage_path);
  }

  Debug("hostdb", "Storage path is %s", storage_path);

  if (access(storage_path, W_OK | R_OK) == -1) {
    Warning("Unable to access() directory '%s': %d, %s", storage_path, errno, strerror(errno));
    Warning("Please set 'proxy.config.hostdb.storage_path' or 'proxy.config.local_state_dir'");
  }

  // 创建一个 Store 和 Span
  hostDBStore = new Store;
  hostDBSpan = new Span;
  // 根据 HostDB 的路径和大小，初始化 Span，然后将 Span 添加到 Store
  // 注意这里传入的 storage_path 是一个路径，不是文件，因此 Span 会按照目录来处理。
  hostDBSpan->init(storage_path, storage_size);
  hostDBStore->add(hostDBSpan);
  // 可以将这个 Store 理解为空白的存储空间，后面会尝试按照 HostDB/MultiCache 的配置文件的设定在这个空白的存储空间上进行空间分配，
  // 如果可以成功完成空间的分配，则说明 HostDB/MultiCache 的配置文件是正确的，因此这个 Store 仅仅使用来做一个配置文件合法性的验证。
  // 对 HostDB 来说，MultiCache 的配置文件就是 hostdb.config 文件，通常位于 /run/trafficserver 或 /var/run/trafficserver 目录下

  // hostdb_filename 默认为 host.db；hostdb_size 是记录的条目数量
  Debug("hostdb", "Opening %s, size=%d", hostdb_filename, hostdb_size);
  // 第一次调用 MultiCache::open，尝试打开 HostDB/MultiCache，这里 reconfigure，fix 和 slient 都为 false
  if (open(hostDBStore, "hostdb.config", hostdb_filename, hostdb_size, reconfigure, fix, false /* slient */) < 0) {
    // 返回值小于 0 表示失败，可能是：
    //  - 首次运行，HostDB 的配置文件和数据文件都未创建
    //  - HostDB 的配置发生的变化，条目数或数据库的大小可能变大或变小
    //  - 数据库的配置文件与传入的 Store 结构不匹配，不能完成空间分配
    ats_scoped_str rundir(RecConfigReadRuntimeDir());
    ats_scoped_str config(Layout::relative_to(rundir, "hostdb.config"));

    Note("reconfiguring host database");

    // 删除 HostDB/MultiCache 的配置文件，将由 open 方法按照传入的参数重建
    if (unlink(config) < 0)
      Debug("hostdb", "unable to unlink %s", (const char *)config);

    // 删除之前创建的 Store，并重新建立 Store 和 Span
    delete hostDBStore;
    hostDBStore = new Store;
    hostDBSpan = new Span;
    hostDBSpan->init(storage_path, storage_size);
    hostDBStore->add(hostDBSpan);
    // 第二次调用 MultiCache::open，尝试打开 HostDB/MultiCache，这里 reconfigure 强制传入 true，fix 和 slient 仍然为 false
    if (open(hostDBStore, "hostdb.config", hostdb_filename, hostdb_size, true, fix) < 0) {
      // 如果仍然失败，则说明无法完成数据库的创建，只能关闭 HostDB 功能。
      Warning("could not initialize host database. Host database will be disabled");
      hostdb_enable = 0;
      delete hostDBStore;
      return -1;
    }
  }
  HOSTDB_SET_DYN_COUNT(hostdb_bytes_stat, totalsize);
  //  XXX I don't see this being reference in the previous function calls, so I am going to delete it -bcall
  delete hostDBStore;
  return 0;
}
```

### int HostDBCache::rebuild_callout(HostDBInfo *e, RebuildMC &r)


由于某些原因，MultiCache 需要重建数据库，此时需要遍历整个数据库里每一个数据元素，保留正确的、有效的数据元素，删除破损的、无效的数据元素。在 `rebuild()` 方法里调用 `rebuild_callout()` 方法来完成对数据元素的验证，通过返回值确认指定数据元素的状态：

- 返回 -1 表示数据损坏
- 返回 0 表示该数据元素虽然没有损坏，但是已经没有用处了（例如过期的陈旧数据）
- 返回 1 表示该数据元素是完整有效的数据

```
int
HostDBCache::rebuild_callout(HostDBInfo *e, RebuildMC &r)
{
  // 非法记录：反向 DNS 解析不存在 Round Robin 类型
  if (e->round_robin && e->reverse_dns)
    return corrupt_debugging_callout(e, r);
  // 反向 DNS 计息类型合法性判定
  if (e->reverse_dns) {
    // 陈旧记录：如果 data.hostname_offset 的值小于 0 表示 hostname 保存在 UnsunkPtr 里，
    // 那么这些数据是无法重建的，因为重建操作是扫描磁盘上持久化数据来完成重建的。
    if (e->data.hostname_offset < 0)
      return 0;
    // 如果 data.hostname_offset 的值大于 0 表示 hostname 保存在 HEAP 区
    if (e->data.hostname_offset > 0) {
      // 判断 data.hostname_offset 的值是否指向合法的 HEAP 区的地址
      if (!valid_offset(e->data.hostname_offset - 1))
        return corrupt_debugging_callout(e, r);
      char *p = (char *)ptr(&e->data.hostname_offset, r.partition);
      if (!p)
        return corrupt_debugging_callout(e, r);
      // 判断 hostname 的长度是否超过最大长度
      char *s = p;
      while (*p && p - s < MAXDNAME) {
        if (!valid_heap_pointer(p))
          return corrupt_debugging_callout(e, r);
        p++;
      }
      if (p - s >= MAXDNAME)
        return corrupt_debugging_callout(e, r);
    }
  }
  // Round Robin 类型合法性判定
  if (e->round_robin) {
    // 陈旧记录：如果 app.rr.offset 的值小于 0 表示 Round Robin 记录保存在 UnsunkPtr 里，
    // 那么这些数据是无法重建的，因为重建操作是扫描磁盘上持久化数据来完成重建的。
    if (e->app.rr.offset < 0)
      return 0;
    // 判断 app.rr.offset 的值是否指向合法的 HEAP 区的地址
    if (!valid_offset(e->app.rr.offset - 1))
      return corrupt_debugging_callout(e, r);
    HostDBRoundRobin *rr = (HostDBRoundRobin *)ptr(&e->app.rr.offset, r.partition);
    if (!rr)
      return corrupt_debugging_callout(e, r);
    // 验证 Round Robin 的数据成员，以下情况为无效数据：
    //  - rrcount 小于等于 0 或者大于最大值，
    //  - good 小于等于 0 或者大于最大值，
    //  - good 大于 rrcount （有效主机大于总主机数）
    if (rr->rrcount > HOST_DB_MAX_ROUND_ROBIN_INFO || rr->rrcount <= 0 || rr->good > HOST_DB_MAX_ROUND_ROBIN_INFO ||
        rr->good <= 0 || rr->good > rr->rrcount)
      return corrupt_debugging_callout(e, r);
    // 遍历所有有效的主机记录
    for (int i = 0; i < rr->good; i++) {
      // 确认主机记录指向合法的 HEAP 区的地址
      if (!valid_heap_pointer(((char *)&rr->info[i + 1]) - 1))
        return -1;
      // 确认该记录是合法的 IP 地址
      if (!ats_is_ip(rr->info[i].ip()))
        return corrupt_debugging_callout(e, r);
      // 对比数据单元里存储的 Hash 值与 Round Robin 主机记录的 Hash 值，如果不一致则为非法值
      if (rr->info[i].md5_high != e->md5_high || rr->info[i].md5_low != e->md5_low || rr->info[i].md5_low_low != e->md5_low_low)
        return corrupt_debugging_callout(e, r);
    }
  }
  // 陈旧记录：判断记录是否超时
  if (e->is_ip_timeout())
    return 0;
  // 有效记录
  return 1;
}
```

## 参考资料

- [HostDB.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/HostDB.cc)

