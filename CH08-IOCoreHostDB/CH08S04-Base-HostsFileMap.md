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
        // 将 IP 地址保存进去，但是这里没有设置 HostDBInfo 的 MD5 信息
        ats_ip_set(item.ip(), ip);
      }
    }
  }
}

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
          // BUG：hostdb_current_interval 在 HostDB 启动时通过当前时间直接计算得来，
          // 然后通过每秒回调状态机 HostDBContinuation::backgroundEvent 实现每秒递增 1 的操作，
          // 但是如果在 EventLoop 中出现延迟和卡顿时，无法达成每秒一次的回调操作，因此该值可能会比真实的时间戳要慢
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

## 关于 /etc/hosts 实现的问题

可以看到 `ParseHostFile()` 方法将为 `/etc/hosts` 内的每一条记录创建（new）一个 HostDBInfo 的对象，当 RefCountedHostsFileMap 对象被自动释放后，也会将其成员 `HostsFileMap hosts_file_map` 析构，从而释放（delete）这些保存了 `/etc/hosts` 记录的 HostDBInfo 的对象。如果有任何的状态机保存了指向这些 HostDBInfo 的对象的指针，那么就会成为野指针。

而 DNS 返回的结果同样会创建 HostDBInfo 的对象，但是这些 HostDBInfo 的对象都是保存在 MultiCache 为底层存储的结构中，当 HostDBInfo 的对象失效后，这个地址空间可能被标记为无效，或者被填充新的内容，但是其所在的存储空间仍然是可以访问的地址空间，因此即使有状态机保存了指向这些 HostDBInfo 的对象的指针，也永远不会成为野指针。

当然，无论在任何情况下，都不应该保存指向 HostDBInfo 对象的指针，而是应该在收到来自 HostDB 回调时，复制 HostDBInfo 对象的结果信息到本地对象中保存，并在将来使用。

为了将 HostDBInfo 的存储结构统一到 MultiCache 中进行存储，官方于 7.0.0 版本引入了以下提交：

```
commit 5f0a649f7af0f6616b81d577a635e6997344b9e1
Author: Thomas Jackson <jacksontj.89@gmail.com>
Date:   Tue May 10 19:59:24 2016 -0700

    TS-4436: Move hosts file implementation to `do_dns`
    
    This moves the hosts file overrides down to the DNS layer instead of the HostDBInfo layer. This means that each port will get its own entry in hostdb, which will exist as its own HostDBInfo just like all the rest of the entries (with a TTL of the time remaining until the next hostdb sync). This also means we only have to do the check in the map on DNS lookup instead of on each probe().
    
    In addition to fixing the down status issues, this also means we no longer need to keep old copies of the strings etc. since they are copied once lookup_done is called.
```


## 参考资料

- [P_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_HostDBProcessor.h)
- [ink_string.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/ink_string.h)
- [TS-4436](https://issues.apache.org/jira/browse/TS-4436)
- [Pull Request #631](https://github.com/apache/trafficserver/pull/631)


