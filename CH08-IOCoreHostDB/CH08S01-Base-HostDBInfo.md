# 基础组件：HostDBInfo

HostDBInfo 是完全参照 MultiCacheBlock 进行定义的数据结构，用于表示 HostDB 的数据元素。理论上可以让 HostDBInfo 继承 MultiCacheBlock，但是在代码上没有看到这两者存在任何关联，只是在定义上是包含关系，HostDBInfo 完全定义了与 MultiCacheBlock 相同的方法和成员，同时还定义了 HostDB 需要的方法和成员。

根据 HostDBInfo 的定义，其在内存中的格式如下：

```
     \  bits
Bytes \  0        8        16       24     31
        +--------+--------+--------+--------+
   0)   |       HostDBApplicationInfo       |    (3
        +--------+--------+--------+--------+
   4)   |       HostDBApplicationInfo       |    (7
        +--------+--------+--------+--------+
   8)   |         HostDBInfo::data          |    (11
        +--------+--------+--------+--------+
  12)   |         HostDBInfo::data          |    (15
        +--------+--------+--------+--------+
  16)   |         HostDBInfo::data          |    (19
        +--------+--------+--------+--------+
  20)   |         HostDBInfo::data          |    (23
        +--------+--------+--------+--------+
  24)   |         HostDBInfo::data          |    (27
        +--------+--------+--------+--------+
  28)   |         HostDBInfo::data          |    (31
        +--------+--------+--------+--------+
  32)   |         HostDBInfo::data          |    (35
        +--------+--------+--------+--------+
  36)   |           ip_timestamp            |    (39
        +--------+--------+--------+--------+
  40)   |        ip_timeout_interval        |    (43
        +--------+--------+--------+--------+
  44)   |RSHHHDBF|       MD5 Low Low        |    (47    F = full, B = backed, D = deleted, 
        +--------+--------+--------+--------+           H = hits, S = is_srv, R = reverse_dns        
  48)   |              MD5 Low              |    (51
        +--------+--------+--------+--------+
  52)   |xxxxxxEN|      (unused bits)       |    (55    N = round_robin, E = round_robin_elt
        +--------+--------+--------+--------+           x = (unused bits)
  56)   |              MD5 High             |    (59
        +--------+--------+--------+--------+
  60)   |              MD5 High             |    (63
        +--------+--------+--------+--------+


   8 = sizeof(HostDBApplicationInfo)
  28 = sizeof(IpEndpoint)
  12 = sizeof(SRVInfo)
  64 = sizeof(HostDBInfo)

```

由于 commit 0e703e1e3b 的提交，在 HostDBInfo 中增加了 hostname_offset 成员，由此导致自 6.0.0 版本开始出现随机崩溃的问题，因此在上述信息中删掉了 hostname_offset 成员。该 Bug 是因为 hostname_offset 成员为 HostDBInfo 引入了第二个 HEAP 区的存储空间，但是根据 MultiCache 章节的分析可知：一个数据元素只允许在 HEAP 区申请一段连续的存储空间。

提供一个补丁修复随机崩溃的问题，但是由于补丁删除了 hostname_offset 成员，导致主机名无法在 HostDB UI 的结果中显示，如果您需要在结果中总是显示主机名的信息，那么需要您自行改进下面的补丁文件。[下载 diff 文件](https://github.com/oknet/trafficserver/commit/9dd751635590c0ce37df3b150cfd24b34c772019.diff)

```
diff --git a/iocore/hostdb/HostDB.cc b/iocore/hostdb/HostDB.cc
index b228c4441f1..2bece3847fd 100644
--- a/iocore/hostdb/HostDB.cc
+++ b/iocore/hostdb/HostDB.cc
@@ -1342,18 +1342,6 @@ HostDBContinuation::lookup_done(IpAddr const &ip, char const *aname, bool around
     }
   }
 
-  if (aname) {
-    const size_t s_size = strlen(aname) + 1;
-    void *host_dest     = hostDB.alloc(&i->hostname_offset, s_size);
-    if (host_dest) {
-      ink_strlcpy((char *)host_dest, aname, s_size);
-    } else {
-      Warning("Out of room in hostdb for hostname (data area full!)");
-      hostDB.delete_block(i);
-      return NULL;
-    }
-  }
-
   if (from_cont)
     do_put_response(from, i, from_cont);
   ink_assert(!i->round_robin || !i->reverse_dns);
@@ -2305,18 +2293,6 @@ HostDBInfo::hostname()
   return (char *)hostDB.ptr(&data.hostname_offset, hostDB.ptr_to_partition((char *)this));
 }
 
-/*
- * The perm_hostname exists for all records not just reverse dns records.
- */
-char *
-HostDBInfo::perm_hostname()
-{
-  if (hostname_offset == 0)
-    return NULL;
-
-  return (char *)hostDB.ptr(&hostname_offset, hostDB.ptr_to_partition((char *)this));
-}
-
 HostDBRoundRobin *
 HostDBInfo::rr()
 {
@@ -2494,12 +2470,6 @@ struct ShowHostDB : public ShowCont {
       CHECK_SHOW(show("<tr><td>%s</td><td>%s%s %s</td></tr>\n", "Type", r->round_robin ? "Round-Robin" : "",
                       r->reverse_dns ? "Reverse DNS" : "", r->is_srv ? "SRV" : "DNS"));
 
-      if (r->perm_hostname()) {
-        CHECK_SHOW(show("<tr><td>%s</td><td>%s</td></tr>\n", "Hostname", r->perm_hostname()));
-      } else if (rr && r->is_srv && hostdb_rr) {
-        CHECK_SHOW(show("<tr><td>%s</td><td>%s</td></tr>\n", "Hostname", r->srvname(hostdb_rr)));
-      }
-
       // Let's display the MD5.
       CHECK_SHOW(show("<tr><td>%s</td><td>%0.16llx %0.8x %0.8x</td></tr>\n", "MD5 (high, low, low low)", r->md5_high, r->md5_low,
                       r->md5_low_low));
@@ -2517,6 +2487,8 @@ struct ShowHostDB : public ShowCont {
         CHECK_SHOW(show("<tr><td>%s</td><td>%d</td></tr>\n", "Priority", r->data.srv.srv_priority));
         CHECK_SHOW(show("<tr><td>%s</td><td>%d</td></tr>\n", "Port", r->data.srv.srv_port));
         CHECK_SHOW(show("<tr><td>%s</td><td>%x</td></tr>\n", "Key", r->data.srv.key));
+      } else if (r->reverse_dns) {
+        CHECK_SHOW(show("<tr><td>%s</td><td>%s</td></tr>\n", "Hostname", r->hostname() ? r->hostname() : "<none>"));
       } else if (!r->is_srv) {
         CHECK_SHOW(show("<tr><td>%s</td><td>%s</td></tr>\n", "IP", ats_ip_ntop(r->ip(), b, sizeof b)));
       }
@@ -2527,12 +2499,6 @@ struct ShowHostDB : public ShowCont {
       CHECK_SHOW(show("\"%s\":\"%s%s%s\",", "type", (r->round_robin && !r->is_srv) ? "roundrobin" : "",
                       r->reverse_dns ? "reversedns" : "", r->is_srv ? "srv" : "dns"));
 
-      if (r->perm_hostname()) {
-        CHECK_SHOW(show("\"%s\":\"%s\",", "hostname", r->perm_hostname()));
-      } else if (rr && r->is_srv && hostdb_rr) {
-        CHECK_SHOW(show("\"%s\":\"%s\",", "hostname", r->srvname(hostdb_rr)));
-      }
-
       CHECK_SHOW(show("\"%s\":\"%u\",", "app1", r->app.allotment.application1));
       CHECK_SHOW(show("\"%s\":\"%u\",", "app2", r->app.allotment.application2));
       CHECK_SHOW(show("\"%s\":\"%u\",", "lastfailure", r->app.http_data.last_failure));
@@ -2547,6 +2513,8 @@ struct ShowHostDB : public ShowCont {
         CHECK_SHOW(show("\"%s\":\"%d\",", "priority", r->data.srv.srv_priority));
         CHECK_SHOW(show("\"%s\":\"%d\",", "port", r->data.srv.srv_port));
         CHECK_SHOW(show("\"%s\":\"%x\",", "key", r->data.srv.key));
+      } else if (r->reverse_dns) {
+        CHECK_SHOW(show("\"%s\":\"%s\"", "hostname", r->hostname() ? r->hostname() : "<none>"));
       } else if (!r->is_srv) {
         CHECK_SHOW(show("\"%s\":\"%s\"", "ip", ats_ip_ntop(r->ip(), b, sizeof b)));
       }
diff --git a/iocore/hostdb/I_HostDBProcessor.h b/iocore/hostdb/I_HostDBProcessor.h
index bd8bbc24b46..c008a5d120f 100644
--- a/iocore/hostdb/I_HostDBProcessor.h
+++ b/iocore/hostdb/I_HostDBProcessor.h
@@ -155,7 +155,6 @@ struct HostDBInfo {
   }
 
   char *hostname();
-  char *perm_hostname();
   char *srvname(HostDBRoundRobin *rr);
   /// Check if this entry is an element of a round robin entry.
   /// If @c true then this entry is part of and was obtained from a round robin root. This is useful if the
@@ -248,8 +247,6 @@ struct HostDBInfo {
     SRVInfo srv;
   } data;
 
-  int hostname_offset; // always maintain a permanent copy of the hostname for non-rev dns records.
-
   unsigned int ip_timestamp;
   // limited to HOST_DB_MAX_TTL (0x1FFFFF, 24 days)
   // if this is 0 then no timeout.
```

## 定义

与 HostDB 相关的方法与成员集中定义在前部，与 MultiCacheBlock 相同的方法和成员集中定义在后部。

```
struct HostDBInfo {
  /** Internal IP address data.
      This is at least large enough to hold an IPv6 address.
  */
  // 当需要向 HostDBInfo 中写入 IP 地址信息时，可以使用该方法返回指向 data.ip.sa 结构的指针（可写）
  sockaddr *
  ip()
  {
    return &data.ip.sa;
  }
  // 当需要从 HostDBInfo 中读取 IP 地址信息时，可以使用该方法返回指向 data.ip.sa 结构的指针（只读）
  sockaddr const *
  ip() const
  {
    return &data.ip.sa;
  }

  // 当 reverse_dns == true 时，返回反向域名解析得到的主机名，
  // 数据位于 HEAP 区，调用者不可以保留该指针
  char *hostname();
  /* 此处删除了 char *perm_hostname(); */
  // 当 is_srv == true 时，从 RoundRobin 数据中解析出 SRV 记录的指针并返回 ，
  // 数据位于 HEAP 区，调用者不可以保留该指针
  char *srvname(HostDBRoundRobin *rr);
  /// Check if this entry is the root of a round robin entry.
  /// If @c true then this entry needs to be converted to a specific element of the round robin to be used.
  // 返回该 Entry 是否为 Round Robin 数据的根
  // 由于 Round Robin 的记录数量不确定，因此所有的解析记录都以特定格式存储在 HEAP 区，
  // 在 MultiCache 数据库的数据区里存储的只是 Round Robin 数据的元信息。
  bool
  is_rr() const
  {
    return 0 != round_robin;
  }
  /// Check if this entry is an element of a round robin entry.
  /// If @c true then this entry is part of and was obtained from a round robin root. This is useful if the
  /// address doesn't work - a retry can probably get a new address by doing another lookup and resolving to
  /// a different element of the round robin.
  // 返回该 Entry 是否为 Round Robin 数据的的一个数据元素（既一条数据记录）
  bool
  is_rr_elt() const
  {
    return 0 != round_robin_elt;
  }
  // 当 round_robin == true 时，返回指向 Round Robin 记录集合的指针，
  // 数据位于 HEAP 区，调用者不可以保留该指针
  HostDBRoundRobin *rr();

  /** Indicate that the HostDBInfo is BAD and should be deleted. */
  // 将 Entry 设置为空白状态，但是与 set_empty 的区别是，它不会清除 MD5 信息
  // 目前未发现任何代码调用该方法
  void
  bad()
  {
    full = 0;
  }

  /**
    Application specific data. NOTE: We need an integral number of these
    per block. This structure is 32 bytes. (at 200k hosts = 8 Meg). Which
    gives us 7 bytes of application information.

  */
  // 用来存储应用的扩展数据，应用这里指访问 HostDB 数据库的系统
  // 上面的注释说 HostDBInfo 这个结构有 32 字节，但是目前的定义是 64 字节，
  // 然后说 7 个字节用于存储 App Info，但是根据 HostDBApplicationInfo 的定义，应该是 8 个字节。
  HostDBApplicationInfo app;

  // 返回当前 HostDBInfo 对象在 HostDB 中存活的时长，以秒为单位
  unsigned int
  ip_interval()
  {
    return (hostdb_current_interval - ip_timestamp) & 0x7FFFFFFF;
  }

  // 返回当前 HostDBInfo 对象剩余的生存时间，以秒为单位
  int
  ip_time_remaining()
  {
    return static_cast<int>(ip_timeout_interval) - static_cast<int>(this->ip_interval());
  }

  /* 对于 HostDB 里存储的 DNS 解析结果，存在一个保鲜期和保质期的区别：
   *  - 保鲜期之内：表示该记录非常新，不需要重新验证就可以直接使用，
   *  - 保鲜期与保质期之间：表示该记录不重新验证也可直接使用，应尝试进行“刷新/翻新”，
   *  - 保质期之外：表示该记录已经超时，不应该使用该记录，应该重新进行解析。
   */

  // 返回当前 HostDBInfo 对象的内容是否已经陈旧（保鲜期与保质期之间）
  // hostdb_ip_stale_interval 默认为 12 分钟 (720 秒)，表示每隔 12 分钟就要去重新验证 DNS 的解析是否发生变化
  //  - 当允许存活时间 ip_timeout_interval（最大值 HOST_DB_MAX_TTL 为 24 天）大于等于 24 分钟，
  //  - 并且，存活时间达到 12 分钟或以上时，视为陈旧记录，但是陈旧并不意味着过期。
  // ip_timeout_interval 的值受到 proxy.config.hostdb.ttl_mode 配置项的影响，可以是以下几种情况：
  //  - TTL_OBEY：取 DNS 解析结果的 TTL 值
  //  - TTL_IGNORE：取 proxy.config.hostdb.timeout 配置项的值（默认为 24 小时）
  //  - TTL_MIN：在 DNS 解析结果的 TTL 值 和 proxy.config.hostdb.timeout 配置项的值中取较小的值
  //  - TTL_MAX：在 DNS 解析结果的 TTL 值 和 proxy.config.hostdb.timeout 配置项的值中取较大的值
  // 其中 TTL_MIN 表示 HostDBInfo 的最长刷新周期为 proxy.config.hostdb.timeout 配置项的值，
  // 而 TTL_MAX 表示 HostDBInfo 的最短刷新周期为 proxy.config.hostdb.timeout 配置项的值。
  bool
  is_ip_stale()
  {
    return ip_timeout_interval >= 2 * hostdb_ip_stale_interval && ip_interval() >= hostdb_ip_stale_interval;
  }

  // 返回当前 HostDBInfo 对象的内容是否已经超时（保质期之外）
  // 当实际存活时间超过了允许存活时间时表示数据过期，但是允许存活时间设置为 0 表示永不过期
  bool
  is_ip_timeout()
  {
    return ip_timeout_interval && ip_interval() >= ip_timeout_interval;
  }

  // 如果当前 HostDBInfo 保存的是一个失败的解析结果，返回其是否已经超时（保质期之外）
  // HostDB 会缓存解析失败的 DNS 请求，这样就不会大量、重复的解析这些域名，
  // 同时，这些无法解析的记录也会定期重新进行解析。
  // 出现无法解析的情况，可能是 DNS 服务器配置错误，或者 DNS 服务器暂时出现异常导致，也有可能是用户输入了错误的域名。
  bool
  is_ip_fail_timeout()
  {
    return ip_interval() >= hostdb_ip_fail_timeout_interval;
  }

  // “刷新/翻新”当前 HostDBInfo，将导致实际存活时间重新计时
  void
  refresh_ip()
  {
    ip_timestamp = hostdb_current_interval;
  }

  // 是否可以返回过期的数据，同时重新向 DNS 服务器验证
  // 注意这里函数名字虽然体现了“stale”，但是实际上是应用于“expired”状态的数据
  bool
  serve_stale_but_revalidate()
  {
    // the option is disabled
    // 检查该功能的配置项（proxy.config.hostdb.serve_stale_for）是否大于 0
    // 该配置项表示超过有效期多久之后，还可以继续使用过期的数据，也就是将保质期向后延长了一段时间。
    if (hostdb_serve_stale_but_revalidate <= 0)
      return false;

    // ip_timeout_interval == DNS TTL
    // hostdb_serve_stale_but_revalidate == number of seconds
    // ip_interval() is the number of seconds between now() and when the entry was inserted
    // 这里 ip_timeout_interval 是保质期，hostdb_serve_stale_but_revalidate 是保质期之外可以继续使用的时间，
    // 判断当前存活时间超过保质期之后一段时间之内，
    if ((ip_timeout_interval + hostdb_serve_stale_but_revalidate) > ip_interval()) {
      Debug("hostdb", "serving stale entry %d | %d | %d as requested by config", ip_timeout_interval,
            hostdb_serve_stale_but_revalidate, ip_interval());
      // 返回 true，仍然可以继续使用当前 HostDBInfo 的数据
      return true;
    }
    // otherwise, the entry is too old
    // 如果已经超出保质期太多太多了，那么就返回 false，不能继续使用过期太久的数据了
    return false;
  }


  //
  // Private
  //
  // 数据区：用于存储解析结果
  // 如果是将域名解析为 IP，那么 data.ip 内存储了解析的结果，
  // 如果是将 IP 反向解析为域名，那么通过 data.hostname_offset 去 HEAP 区可以获取解析到的域名，
  // 如果是解析 SRV 记录，可以通过 data.srv 可以在 HEAP 区获得 SRV 的结果，
  // 其中 IP 和 SRV 结果可能存在多条解析记录，此时需要联合 RoundRobin 信息在 HEAP 区获取解析结果。
  union {
    IpEndpoint ip;       ///< IP address / port data.
    int hostname_offset; ///< Some hostname thing.
    SRVInfo srv;
  } data;

  /* 此处因造成 BUG 删除了
   * int hostname_offset; // always maintain a permanent copy of the hostname for non-rev dns records.
   */
  // 当前记录的最后更新时间
  unsigned int ip_timestamp;
  // limited to HOST_DB_MAX_TTL (0x1FFFFF, 24 days)
  // if this is 0 then no timeout.
  // 当前数据允许的最大生存时间（保质期）
  unsigned int ip_timeout_interval;

  // Make sure we only have 8 bits of these flags before the @a md5_low_low
  // 表示该 Entry 内已经填充了有效的数据，同时 MD5 值也应该是有效的
  unsigned int full : 1;
  // 表示该 Entry 内的数据已经备份到下一级
  unsigned int backed : 1; // duplicated in lower level
  // 表示该 Entry 内的数据已经被删除
  unsigned int deleted : 1;
  // 表示该 Entry 被查询命中的次数
  unsigned int hits : 3;

  // 表示当前的 HostDBInfo 存储的数据类型是否为 SRV 结果
  unsigned int is_srv : 1;
  // 表示当前的 HostDBInfo 存储的数据类型是否为 PTR 结果（反向地址解析）
  unsigned int reverse_dns : 1;

  // 用于存储折叠后的 MD5 值的低 56 位，高 8 位丢弃（HostDB 的 tag_bits 设置为 56 bits），
  // 这里使用一个 24 bits 的位域和一个 32 bits 的成员组成 56 bits 空间。
  unsigned int md5_low_low : 24;
  unsigned int md5_low;

  // 以下两个成员应该是后期增加（commit 109ba54268），直接导致每个 HostDBInfo 多占用 4 个字节，
  // 上面为了节省 HostDBInfo 占用的字节，直接丢弃 MD5 值的高 8 位，但是这里把上面省出来的空间白白浪费了。

  // 表示当前的 HostDBInfo 存储的数据类型是否为 Round Robin 数据的根节点
  unsigned int round_robin : 1;     // This is the root of a round robin block
  // 表示当前的 HostDBInfo 存储的数据类型是否表示一条 Round Robin 数据记录
  unsigned int round_robin_elt : 1; // This is an address in a round robin block

  // 用于存储 MD5 原始值的高 64 bits
  // 如果可以的话，将 md5_high 的高 2 位丢弃掉，用于存储 Round Robin 相关的 2 个位的数据，可以为 HostDBInfo 节省 4 个字节。
  uint64_t md5_high;

  /*
   * Given the current time `now` and the fail_window, determine if this real is alive
   */
  // 当 ATS 去访问源服务器时，如果遇到故障达到一定次数，则会认为该源服务器不可访问，
  // 同时将发生故障的时间记录在 HostDBInfo 的 app.http_data.last_failure 成员，
  // 并且在一段时间（fail_window）内不再尝试与该源服务器建立连接，
  // 该时间来自配置项 proxy.config.http.down_server.cache_time 。
  // alive 方法通过检测上述值的关系，返回当前 HostDBInfo 对象的源服务器是否处于活动状态（alive）。
  // 注意：该方法仅适用于 Round Robin 的场景，或者强制开启了 Round Robin 功能
  bool
  alive(ink_time_t now, int32_t fail_window)
  {
    unsigned int last_failure = app.http_data.last_failure;

    // 如果没有出现过故障，或在上一次故障后又经过了 fail_window 指定的时间，
    // 则认为该 HostDBInfo 所对应的源服务器是处于活动状态（alive）的。
    // 在 commit be68bd8f47 中，抽取了 HostDBRoundRobin::select_best_srv 方法里的代码形成了 alive 方法，
    // 因此该方法只在 select_best_srv 中调用，但是在 HostDBRoundRobin::select_best_srv 中也有相似的判断。
    if (last_failure == 0 || (unsigned int)(now - fail_window) > last_failure) {
      // 这里并没有重置 last_failure 为 0，因为此时仍然不能断定源服务器已经恢复了，
      // 这里返回 true 之后，上层状态机会尝试向源服务器发起请求
      return true;
    } else {
      // Entry is marked down.  Make sure some nasty clock skew
      //  did not occur.  Use the retry time to set an upper bound
      //  as to how far in the future we should tolerate bogus last
      //  failure times.  This sets the upper bound that we would ever
      //  consider a server down to 2*down_server_timeout
      // 下面的 if 语句用于判定异常情况，在最原始的代码里有一些用于 DEBUG 的辅助代码
      if (now + fail_window < last_failure) {
        // 原始代码里存在 ink_assert(!"extreme clock skew");
        // 因此这里用于重置异常数据
        app.http_data.last_failure = 0;
        return false;
      }
      return false;
    }
  }
  
  // 判断当前 HostDBInfo 的状态是否为不可用状态
  // 如果是 SRV 记录，data.srv.srv_offset 应该大于 0，否则就是无效记录
  // 如果是 PTR 记录，data.hostname_offset 应该不为 0，表示指向 HEAP 区的地址，否则就是无效记录
  // 否则 data.ip 应该是有效的 IP 地址，否则就是无效记录
  bool
  failed()
  {
    return !((is_srv && data.srv.srv_offset) || (reverse_dns && data.hostname_offset) || ats_is_ip(ip()));
  }
  // 将当前 HostDBInfo 设定为不可用状态
  void
  set_failed()
  {
    // 如果是 SRV 记录，就将 data.srv.srv_offset 重置为 0
    if (is_srv)
      data.srv.srv_offset = 0;
    // 如果是 PTR 记录，就将 data.hostname_offset 重置为 0
    else if (reverse_dns)
      data.hostname_offset = 0;
    // 否则就将 data.ip 破坏掉
    else
      ats_ip_invalidate(ip());
  }

  /*************************************************
   * 以下方法和成员与 MultiCacheBlock 中的定义相同
   *************************************************/
  // 将当前 HostDBInfo 设置为已删除状态，该方法没有任何代码调用
  void
  set_deleted()
  {
    deleted = 1;
  }
  // 判断当前 HostDBInfo 是否为已删除状态，虽然有几处调用，但是 set_deleted 方法没有被调用，因此该方法的存在也无意义
  bool
  is_deleted() const
  {
    return deleted;
  }

  // 判断当前 HostDBInfo 是否为空白内容
  bool
  is_empty() const
  {
    return !full;
  }
  // 将当前 HostDBInfo 设置为空白内容
  // 主要重置了 full 成员，并清理了 MD5 值
  void
  set_empty()
  {
    full = 0;
    md5_high = 0;
    md5_low = 0;
    md5_low_low = 0;
  }

  // 向当前 HostDBInfo 填充数据
  // 主要设置 full 成员，并填充折叠后的 MD5 值
  void
  set_full(uint64_t folded_md5, int buckets)
  {
    uint64_t ttag = folded_md5 / buckets;

    if (!ttag)
      ttag = 1;
    md5_low_low = (unsigned int)ttag;
    md5_low = (unsigned int)(ttag >> 24);
    full = 1;
  }

  // 重置当前 HostDBInfo 的数据，通常调用该方法后需要立即调用 set_full 方法
  // 因此该方法没有重置 full 成员和 MD5 值
  // 该方法仅在 MultiCache<C>::insert_block 方法里调用
  void
  reset()
  {
    ats_ip_invalidate(ip());
    app.allotment.application1 = 0;
    app.allotment.application2 = 0;
    backed = 0;
    deleted = 0;
    hits = 0;
    round_robin = 0;
    // BUG：这里可能缺少了对 round_robin_elt 的重置
    reverse_dns = 0;
    is_srv = 0;
  }

  // 该方法由 MultiCache 核心代码调用，以获得表示每个 Entry 唯一性的哈希值
  // 该方法计算过程没有包含 md5_high 成员，因此 md5_high 成员仅在 HostDB 模块内验证。
  uint64_t
  tag()
  {
    uint64_t f = md5_low;
    return (f << 24) + md5_low_low;
  }
  // 该方法没有在 MultiCacheBlock 中定义，也没有在任何地方调用
  bool match(INK_MD5 &, int, int);
  // 根据当前 HostDBInfo 存储数据的类型不同，返回在 HEAP 区实际使用的空间大小
  int heap_size();
  // 根据当前 HostDBInfo 存储数据的类型不同，返回用于保存 HEAP 区地址的 int 类型的成员地址
  int *heap_offset_ptr();
};
```

## HostDBInfo 存储的四种数据类型

HostDBInfo 支持存储四种不同的数据：

第一种：反向地址解析结果（仅出现在 MultiCache 数据区）

如果仅 `reverse_dns == true`，表示当前 HostDBInfo 保存了反向解析的结果，值为主机名且只有一个主机名，不会出现解析为多个主机名的情况。同时主机名一定是存储在 HEAP 区的不定长数据，可以通过 `HostDBInfo::hostname()` 方法获得，在 HEAP 存储的主机名是以 `\0` 结尾的字符串，因此通过 `strlen() + 1` 可以返回其在 HEAP 区占用的空间大小。

第二种：Round Robin 根数据，当域名解析或 SRV 结果为多个记录时（仅出现在 MultiCache 数据区）

此时 `round_robin == true`，如果同时满足：

- `is_srv == true`，表示当前 HostDBInfo 是用于存储 SRV 解析结果的 Round Robin 数据的根记录，
- `is_srv == false`，表示当前 HostDBInfo 是用于存储域名解析结果的 Round Robin 数据的根记录，

通过 `HostDBInfo::rr()` 方法读取该记录中的 `app.rr.offset` 可以获得指向 HEAP 区 HostDBRoundRobin 对象的指针，然后遍历其内包含的多个 HostDBInfo 对象，这些 HostDBInfo 对象的成员 `round_robin_elt` 都设置为 true。对于 HostDBRoundRobin 里存储的 HostDBInfo 对象，仅需要判断 `is_srv` 的值，以区分域名解析结果和 SRV 解析结果，对应到第三种或第四种类型的 HostDBInfo。

第三种：域名解析结果（出现在 MultiCache 数据区或 HEAP 区）

如果满足 `is_srv == false`，`reverse_dns == false`，`round_robin == false`，表示这是一个域名解析记录，有效数据在 data.ip 里，可以通过 `HostDBInfo::ip()` 方法获得。域名解析结果可能出现在 HostDB 的数据区（`round_robin_elt == false`），也可能出现在 HostDBRoundRobin 对象里（`round_robin_elt == true`）。

第四种：SRV 解析结果（仅出现在 HEAP 区）

如果满足 `is_srv == true`，`reverse_dns == false`，`round_robin == false`，`round_robin_elt == true` 表示这是一个 SRV 记录，可以通过 `HostDBInfo::srvname()` 方法获得该 SRV 记录的 target 主机名。SRV 记录总是存储在 HostDBRoundRobin 对象里（`round_robin_elt == true`）。

相关判断的代码如下：

```
if (reverse_dns == true) {
  // reverse dns
} else if (round_robin == true) {
    HostDBRoundRobin *rr = rr();
    for (int i = 0; i< rr->rrcount; i++) {
        ink_assert(rr->info[i].reverse_dns == false);
        ink_assert(rr->info[i].round_robin == false);
        ink_assert(rr->info[i].round_robin_elt == true);
        if (rr->info[i].is_srv == true) {
            // SRV
        } else {
            // IP Address
        }
    }
} else if (is_srv == false) {
    // IP Address
} else {
    ink_assert(!"Bad HostDBInfo");
}
```

## 参考资料

- [I_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/I_HostDBProcessor.h)


