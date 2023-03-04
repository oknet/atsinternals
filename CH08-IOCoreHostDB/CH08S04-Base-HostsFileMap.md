# 基础组件：HostsFileMap

用于保存 `/etc/hosts` 文件的解析结果，并按照主机名进行索引。由 commit a2affb99 实现第一个版本，后期历经多次改动。

```
commit a2affb9916edc53aeb8b7a5d6d223146be9efdc4
Author: Alan M. Carroll <amc@apache.org>
Date:   Sun Sep 21 17:18:19 2014 -0500

    TS-3088: Use /etc/hosts for HostDB.
```

HostDB 的数据最初只有 DNS 解析结果这一个来源，后来由增加了读取 `/etc/hosts` 文件作为其第二数据来源，而且 `/etc/hosts` 的解析结果优先于 DNS 解析结果。

## 定义

```
Source: P_HostDBProcessor.h
// 用于 std::map 的自定义比较方法
// 其中 ptr_len_casecmp() 在 lib/ts/ink_string.h 中定义，是 strcasecmp 的改版，
// 其区别在于输入的字符串可以不是 \0 结尾，而是根据给定的字符串长度判断结尾。
struct CmpConstBuffferCaseInsensitive {
  bool operator()(ts::ConstBuffer a, ts::ConstBuffer b) const { return ptr_len_casecmp(a._ptr, a._size, b._ptr, b._size) < 0; }
};

// Our own typedef for the host file mapping
// 定义 HostsFileMap：
//   Key 为主机名及其字节长度，Value 指向 HostDBInfo 的对象
//   通过自定义方法对 map 成员进行比较
typedef std::map<ts::ConstBuffer, HostDBInfo, CmpConstBuffferCaseInsensitive> HostsFileMap;
// A to hold a ref-counted map
// 通过带有引用计数的结构管理 HostsFileMap，后面可以方便使用 Ptr 自动指针进行管理
struct RefCountedHostsFileMap : public RefCountObj {
  // 成员指向 HostsFileMap 类型，存储 /etc/hosts 文件解析后的内容
  HostsFileMap hosts_file_map;
  // 成员指向一个字符串区域，存储原始的 /etc/hosts 文件内容
  ats_scoped_str HostFileText;
};
```

## 方法

```
// Host file processing globals.

// We can't allow more than one update to be
// proceeding at a time in any case so we might as well make these
// globals.
// 避免多个加载 /etc/hosts 文件的实例同时运行
int HostDBFileUpdateActive = 0;

// Actual ordering doesn't matter as long as it's consistent.
// 未使用
bool
CmpMD5(INK_MD5 const &lhs, INK_MD5 const &rhs)
{
  return lhs[0] < rhs[0] || (lhs[0] == rhs[0] && lhs[1] < rhs[1]);
}
```

### 解析 Hosts 文件内的一行

Hosts 文件的每一行是由空格分隔的多个片段，每一行最少有两个片段，其中第一个片段一定是 IP 地址，其后的每一个片段都是域名或主机名。因此当一行内存在 N 个片段时，将会产生 N-1 个域名解析记录，也就对应了 N-1 个 HostDBInfo 的对象。

```
// 解析 /etc/hosts 文件内的一行内容，由 ParseHostFile 调用
void
ParseHostLine(RefCountedHostsFileMap *map, char *l)
{
  // 允许空格和制表符作为 IP 地址和主机名的分隔符
  Tokenizer elts(" \t");
  // 解析输入的行，返回分段数量
  int n_elts = elts.Initialize(l, SHARE_TOKS);
  // Elements should be the address then a list of host names.
  // Don't use RecHttpLoadIp because the address *must* be literal.
  IpAddr ip;
  // elts[0] 总是为 IP 地址，elts[1..n] 总是为主机名，
  // 至少要有一个 IP 地址和一个主机名才是合法的记录
  if (n_elts > 1 && 0 == ip.load(elts[0])) {
    // 当 IP 地址有效时，遍历所有主机名
    for (int i = 1; i < n_elts; ++i) {
      ts::ConstBuffer name(elts[i], strlen(elts[i]));
      // If we don't have an entry already (host files only support single IPs for a given name)
      // 注意：在 /etc/hosts 文件里，一个主机名只能有一个 IP 地址，不可以对应多个 IP
      // 因此首先查找该主机名是否已经存在，只有不存在时才插入一条记录到 hosts_file_map 容器
      if (map->hosts_file_map.find(name) == map->hosts_file_map.end()) {
        // 由于 [name] 不存在，将会通过 new 方法创建一个 HostDBInfo 对象
        HostsFileMap::mapped_type &item = map->hosts_file_map[name];
        // 设置相关成员的状态将该 HostDBInfo 对象设定为一个正向 DNS 解析记录
        item.round_robin = false;
        item.round_robin_elt = false;
        item.reverse_dns = false;
        item.is_srv = false;
        // 将 IP 地址保存进去，但是这里没有设置 HostDBInfo 的 MD5 信息，
        // 因为 MD5 信息仅用于在 HostDB 中进行检索时使用。
        ats_ip_set(item.ip(), ip);
      }
    }
  }
}
```

### 按行解析 Hosts 文件

Hosts 文件是一个文本文件，除去行首的空格后，第一个字符如果是 `#`，则表示该行为注释行，无任何实际用途。因此在解析 Hosts 文件时，要将这类注释信息剔除。

为了避免存在多个同时加载、解析 Hosts 文件的操作，使用 HostDBFileUpdateActive 标明当前是否处于加载 Hosts 文件的进程中。

```
// 解析 /etc/hosts 文件
void
ParseHostFile(char const *path)
{
  Ptr<RefCountedHostsFileMap> parsed_hosts_file_ptr;

  // Test and set for update in progress.
  // 原子操作检测是否有其它实例同时加载 /etc/hosts，避免出现并行操作
  if (0 != ink_atomic_swap(&HostDBFileUpdateActive, 1)) {
    Debug("hostdb", "Skipped load of host file because update already in progress");
    return;
  }
  Debug("hostdb", "Loading host file '%s'", path);

  if (*path) {
    // 只读方式打开 /etc/hosts 文件
    ats_scoped_fd fd(open(path, O_RDONLY));
    if (fd >= 0) {
      // 文件打开成功
      struct stat info;
      if (0 == fstat(fd, &info)) {
        // 读取文件信息成功
        // +1 in case no terminating newline
        // 如果文件最后没有使用回车换行等方式结尾，这里预留一个字节用来添加一个 \0 以结束字符串
        int64_t size = info.st_size + 1;

        // 创建 RefCountedHostsFileMap 对象
        parsed_hosts_file_ptr = new RefCountedHostsFileMap;
        // 为保存 /etc/hosts 的内容，申请内存
        parsed_hosts_file_ptr->HostFileText = static_cast<char *>(ats_malloc(size));
        if (parsed_hosts_file_ptr->HostFileText) {
          // 内存申请成功
          char *base = parsed_hosts_file_ptr->HostFileText;
          char *limit;

          // 将全部文件内容读入内存
          size = read(fd, parsed_hosts_file_ptr->HostFileText, info.st_size);
          // 标记内容尾部
          limit = parsed_hosts_file_ptr->HostFileText + size;
          // 这里将最后一个字节设为 \0，注意上面设置 size 比文件长度多了 1 个字节
          *limit = 0;

          // We need to get a list of all name/addr pairs so that we can
          // group names for round robin records. Also note that the
          // pairs have pointer back in to the text storage for the file
          // so we need to keep that until we're done with @a pairs.
          // 遍历文件内容，查找 \n 以完成分行操作
          while (base < limit) {
            char *spot = strchr(base, '\n');

            // terminate the line.
            // 如果没有找到 \n 那么就把 spot 改为文件尾部，
            // 否则就把找到的 \n 改为 \0
            if (0 == spot)
              spot = limit; // no trailing EOL, grab remaining
            else
              *spot = 0;

            // 跳过行前的空白
            while (base < spot && isspace(*base))
              ++base;                        // skip leading ws
            // 跳过注释掉的行（以 # 开头），
            // 对于有效的行，调用 ParseHostLine 进行解析
            if (*base != '#' && base < spot) // non-empty non-comment line
              ParseHostLine(parsed_hosts_file_ptr, base);
            // 处理下一行
            base = spot + 1;
          }

          // 记录这次加载 /etc/hosts 文件的时间戳
          // BUG: 由于当前 (HOST_DB_TIMEOUT_INTERVAL / HRTIME_SECOND) 的值为 1，可以暂时忽略。
          // 在 backgroundEvent 中，hostdb_hostfile_update_timestamp 被强制转换为 time_t 类型，并与 info.st_mtime 进行比较，
          // 但是这里却是将时间分片的编号保存到了 hostdb_hostfile_update_timestamp。
          // 修复方法 1：hostdb_hostfile_update_timestamp = info.st_mtime;
          // 修复方法 2：请查看 backgroundEvent 里的分析
          hostdb_hostfile_update_timestamp = hostdb_current_interval;
        }
      }
    }
  }

  // Rotate the host file maps down. We depend on two assumptions here -
  // 1) Host files are not updated frequently. Even if updated via traffic_ctl that will be at least 5-10 seconds between reloads.
  // 2) The HostDB clients (essentially the HttpSM) copies the HostDB record or data over to local storage during event processing,
  //    which means the data only has to be valid until the event chaining rolls back up to the event processor. This will certainly
  //    be less than 1s
  // The combination of these means keeping one file back in the rotation is sufficient to keep any outstanding references valid
  // until dropped.
  // 使用不上锁的方式切换 HostsFileMap 对象，将会同时保存新旧两个副本在内存里。
  // 如果在重新加载 /etc/hosts 文件时，有并行的操作也在 HostsFileMap 对象中进行检索操作，那么就不能释放旧
  // 的 HostsFileMap 对象实例；因此通过将旧实例的入口保存起来，把入口指向新实例，那么在指针切换完成之后，新
  // 发起的检索操作会在新实例中进行，而之前发起的检索操作仍然在旧实例中进行；等到再次加载 /etc/hosts 文件时，
  // 才把最老的实例释放掉，那么此时要确保最老的实例不再被访问，那么就要满足以下的假设：
  //  1) /etc/hosts 文件的更新不是很频繁。因为通过 traffic_ctl 触发更新，也要 5-10 秒才能完成。
  //  2) HostDB 的调用者（通常是 HttpSM）在收到回调事件后，需要立即复制 HostDB 的结果到本地变量，这意味着一旦返回，
  //     回调传入的 HostDB 的结果就失效了。上述过程即是 HostDB 的结果的生存周期，该周期的存活时间不会超过 1 秒。
  // 结合上述的假设，只要始终保持一个旧实例，就足以避免出现将正在访问的对象释放的风险。
  hostDB.prev_hosts_file_ptr = hostDB.hosts_file_ptr;
  hostDB.hosts_file_ptr = parsed_hosts_file_ptr;
  // Mark this one as completed, so we can allow another update to happen
  // 修改标记，允许下一个加载 /etc/hosts 文件操作的实例启动
  HostDBFileUpdateActive = 0;
}
```

### Hosts 文件的更新后的动态加载

在 HostDB 子系统启动时，会创建一个 `HostDBContinuation` 实例，并将状态机设置为 `backgroundEvent()`，然后周期性的回调该状态机。

将时间以 `HOST_DB_TIMEOUT_INTERVAL` 为最小单位分片，每个分片里回调一次 `backgroundEvent()` 状态机，其中 `hostdb_current_interval` 记录了该事件分片的唯一编号。

```
#define HOST_DB_TIMEOUT_INTERVAL HRTIME_SECOND

int
HostDBProcessor::start(int, size_t)
{
...
  REC_EstablishStaticConfigInt32U(hostdb_hostfile_check_interval, "proxy.config.hostdb.host_file.interval");

  //
  // Set up hostdb_current_interval
  //
  hostdb_current_interval = (unsigned int)(Thread::get_hrtime() / HOST_DB_TIMEOUT_INTERVAL);

  HostDBContinuation *b = hostDBContAllocator.alloc();
  SET_CONTINUATION_HANDLER(b, (HostDBContHandler)&HostDBContinuation::backgroundEvent);
  b->mutex = new_ProxyMutex();
  // 每秒一次，没有保存返回的 Action，该状态机不可取消
  eventProcessor.schedule_every(b, HOST_DB_TIMEOUT_INTERVAL, ET_DNS);
...
}
```

全局变量 `hostdb_hostfile_check_interval` 用于表示 Hosts 文件的更新周期(以秒为单位)，同时也是解释 Hosts 文件后产生的 `HostDBInfo` 记录的生存周期。

虽然每个时间分片都会回调 `backgroundEvent()` ，但是只有达到更新周期时，`backgroundEvent()` 才会重新加载并解析 Hosts 文件，因此对 Hosts 文件及相关配置的变更不会立即生效。

在重新加载 Hosts 文件之前，需要判断以下几种情况：

1. 自上次加载后，`proxy.config.hostdb.host_file.path` 没有发生变化
   - 指向有效的 Hosts 文件，并且文件的最后更新日期在上次加载时间之后，则调用 `ParseHostFile()` 完成加载和解析，
   - 该配置项为空内容，则调用 `ParseHostFile()` 产生空的 HostsFileMap 对象，
   - 如 Hosts 文件无法打开，则不做任何操作，不会清空当前的 HostsFileMap 对象。
2. 自上次加载后，`proxy.config.hostdb.host_file.path` 发生了变化，
   - 指向了新的、有效的 Hosts 文件，重置上一次加载 Hosts 文件的时间，然后调用 `ParseHostFile()` 完成加载和解析，
   -  该配置项为空内容，则调用 `ParseHostFile()` 产生空的 HostsFileMap 对象，
   - 如 Hosts 文件无法打开，也产生空的 HostsFileMap 对象，注意与上面的情况进行区分。

对于 Hosts 文件无法打开的两种情况，可以理解为：

- 配置无修改，但是 Hosts 文件异常无法打开，可以认为是外部故障，有可能在一段时间后故障解除，又能打开了，
   - 所以这里把这种情况视为异常，不对 HostsFileMap 对象做任何的变更。
- 配置有修改，但是 Hosts 文件异常无法打开，可以认为是配置错误，
   - 此时清空之前加载的 Hosts 文件内容，可以让错误尽快出现，并提示管理员解决配置错误。


```
//
// Background event
// Just increment the current_interval.  Might do other stuff
// here, like move records to the current position in the cluster.
//
int
HostDBContinuation::backgroundEvent(int /* event ATS_UNUSED */, Event * /* e ATS_UNUSED */)
{
  // 递增时间分片的编号
  ++hostdb_current_interval;

  // hostdb_current_interval is bumped every HOST_DB_TIMEOUT_INTERVAL seconds
  // so we need to scale that so the user config value is in seconds.
  // 由于 hostdb_hostfile_check_interval 对应的用户配置是以秒为单位的数值，
  // 因此，在进行更新周期判断之前，需要先将 hostdb_current_interval 的数值由时间片的编号转换为秒，
  if (hostdb_hostfile_check_interval && // enabled
      (hostdb_current_interval - hostdb_hostfile_check_timestamp) * (HOST_DB_TIMEOUT_INTERVAL / HRTIME_SECOND) >
        hostdb_hostfile_check_interval) {
    bool update_p = false; // do we need to reparse the file and update?
    struct stat info;
    char path[sizeof(hostdb_hostfile_path)];

    REC_ReadConfigString(path, "proxy.config.hostdb.host_file.path", sizeof(path));
    if (0 != strcasecmp(hostdb_hostfile_path, path)) {
      Debug("hostdb", "Update host file '%s' -> '%s'", (*hostdb_hostfile_path ? hostdb_hostfile_path : "*-none-*"),
            (*path ? path : "*-none-*"));
      // path to hostfile changed
      // 如果 Hosts 文件的路径发生变化，将上一次 Hosts 文件的更新时间重置
      hostdb_hostfile_update_timestamp = 0; // never updated from this file
      if ('\0' != *path) {
        memcpy(hostdb_hostfile_path, path, sizeof(hostdb_hostfile_path));
      } else {
        // 如果路径为空，那表示这里没有 Hosts 文件了，也就意味着需要将 HostsFileMap 里的记录清空
        hostdb_hostfile_path[0] = 0; // mark as not there
      }
      // 标记需要调用 ParseHostFile 对新的 Hosts 文件进行解析
      update_p = true;
    } else {
      // 如果 Hosts 文件的路径没有发生变化，那么就要看上一次检查时，路径是否为空？
      // 注意：这里相当于把 `Hosts 文件路径为空` 看做是 `Hosts 文件的内容为空`，
      // 会产生一个空内容的 HostsFileMap 对象，并保持 hostdb_hostfile_check_interval 的时间
      hostdb_hostfile_check_timestamp = hostdb_current_interval;
      if (*hostdb_hostfile_path) {
        if (0 == stat(hostdb_hostfile_path, &info)) {
          // Hosts 文件路径不为空，并且可以获取其文件状态信息。
          // BUG: 由于当前 (HOST_DB_TIMEOUT_INTERVAL / HRTIME_SECOND) 的值为 1，可以暂时忽略。
          // 在 ParseHostFile 中，hostdb_hostfile_update_timestamp 被赋值为时间分片的编号，
          // 因此在这里应该将其还原回以秒为单位的数值，需要乘以 (HOST_DB_TIMEOUT_INTERVAL / HRTIME_SECOND)
          // 修复该 BUG，只需要在 ParseHostFile 或 backgroundEvent 修改一处即可。
          if (info.st_mtime > (time_t)hostdb_hostfile_update_timestamp) {
            // 如果 Hosts 文件的最后修改时间在上一次解析操作之后，
            // 就标记需要调用 ParseHostFile 对 Hosts 文件进行重新解析
            update_p = true; // same file but it's changed.
          }
        } else {
          // 注意：Hosts 文件如果被删除，那么是不会产生空内容的 HostsFileMap 对象。
          Debug("hostdb", "Failed to stat host file '%s'", hostdb_hostfile_path);
        }
      }
    }
    // 根据设定的标记来调用 ParseHostFile 完成 Hosts 文件的解析
    if (update_p) {
      Debug("hostdb", "Updating from host file");
      ParseHostFile(hostdb_hostfile_path);
    }
  }

  return EVENT_CONT;
}
```


## 关于 /etc/hosts 实现的问题

可以看到 `ParseHostFile()` 方法将为 `/etc/hosts` 内的每一条记录创建（new）一个 HostDBInfo 的对象，当 RefCountedHostsFileMap 对象被自动释放后，也会将其成员 `HostsFileMap hosts_file_map` 析构，从而释放（delete）这些保存了 `/etc/hosts` 记录的 HostDBInfo 的对象。如果有任何的状态机保存了指向这些 HostDBInfo 的对象的指针，那么就会成为野指针。

而 DNS 返回的结果同样会创建 HostDBInfo 的对象，但是这些 HostDBInfo 的对象都是保存在 MultiCache 为底层存储的结构中，当 HostDBInfo 的对象失效后，这个地址空间可能被标记为无效，或者被填充新的内容，但是其所在的存储空间仍然是可以访问的地址空间，因此即使有状态机保存了指向这些 HostDBInfo 的对象的指针，也永远不会成为野指针。

当然，无论在任何情况下，都不应该保存指向 HostDBInfo 对象的指针，而是应该在收到来自 HostDB 回调时，复制 HostDBInfo 对象的结果信息到本地对象中保存，并在将来使用。

由于通过不上锁的方式切换 HostsFileMap 对象存在一定的风险，因此官方对 HostDBInfo 的存储进行了改进。为了将 HostDBInfo 对象统一到 MultiCache 中，官方于 7.0.0 版本引入了以下提交：

```
commit 5f0a649f7af0f6616b81d577a635e6997344b9e1
Author: Thomas Jackson <jacksontj.89@gmail.com>
Date:   Tue May 10 19:59:24 2016 -0700

    TS-4436: Move hosts file implementation to `do_dns`
    
    This moves the hosts file overrides down to the DNS layer instead of the HostDBInfo layer. This means that each port will get its own entry in hostdb, which will exist as its own HostDBInfo just like all the rest of the entries (with a TTL of the time remaining until the next hostdb sync). This also means we only have to do the check in the map on DNS lookup instead of on each probe().
    
    In addition to fixing the down status issues, this also means we no longer need to keep old copies of the strings etc. since they are copied once lookup_done is called.
```

该提交将 Hosts 文件的检索移到了 HostDB 之后，但是在调用 DNS 子系统之前。这个改进将 Hosts 文件内获得的结果看做是一种特殊的 DNS 解析结果，一并存储到 HostDB 中。

## 参考资料

- [P_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_HostDBProcessor.h)
- [ink_string.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/ink_string.h)
- [TS-4436](https://issues.apache.org/jira/browse/TS-4436)
- [Pull Request #631](https://github.com/apache/trafficserver/pull/631)


