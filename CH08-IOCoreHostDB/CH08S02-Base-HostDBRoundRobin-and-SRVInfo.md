# 基础组件：HostDBRoundRobin

域名解析结果可能返回多个 IP，而且不同的域名返回的 IP 地址有多有少，因此需要将其按照不定长数据存储到 MultiCache 数据库的 HEAP 区。同时支持应用层在使用多个 IP 时，可采用轮询的方式，以平衡负载、规避出现问题的 IP。

为了便于该数据的存储和访问，ATS 定义了 HostDBRoundRobin 数据结构，其内可包含多个 (inner) HostDBInfo 结构，同时也定义了必要的信息表示其在 HEAP 区占用的空间大小、包含了多少个 (inner) HostDBInfo 等。为了描述方便，我们这里把存储在 MultiCacheBlock 中的 HostDBInfo 对象称为 (root) HostDBInfo，存储在 HostDBRoundRobin 中的 HostDBInfo 对象称为 (inner) HostDBInfo。其数据结构如下：

```
     \  bits
Bytes \  0        8        16       24     31
        +--------+--------+--------+--------+
   0)   |     rrcount     |      good       |    (3
        +--------+--------+--------+--------+
   4)   |     current     |     length      |    (7
        +--------+--------+--------+--------+
   8)   |          timed_rr_ctime           |    (11
        +--------+--------+--------+--------+
  12)   |          timed_rr_ctime           |    (15
        +--------+--------+--------+--------+
  16)   |    (inner) HostDBInfo info[0]     |
        |                ...                |
        |                                   |    (80
        +--------+--------+--------+--------+
  81)   |    (inner) HostDBInfo info[1]     |
        |                ...                |
        |                                   |    (143
        +--------+--------+--------+--------+
 144)   |    (inner) HostDBInfo info[...]   |
        |                ...                |
        |                                   |    (m = size(count, 0)
        +--------+--------+--------+--------+
 m+1)   |      SRV hostname one by one      |
        |                ...                |
        |             (optional)            |    (n = size(count, srv_len)
        +--------+--------+--------+--------+

  16 = sizeof(HostDBRoundRobin)
  64 = sizeof(HostDBInfo)

```

可以看到，HostDBRoundRobin 实际上分为头部（16 字节）与 (inner) HostDBInfo 数组区和 SRV 主机名区域三个部分，由于 HEAP 的设计要求：

- MultiCacheBlock 内只能有一个 int 类型成员来表示数据在 HEAP 区的位置，
   - 对于 HostDBRoundRobin 是 (root) HostDBInfo::app.rr.offset。
- HEAP 区的固有数据长度可以保存在 MultiCacheBlock 内，因为 HEAP 区的数据长度一经申请就不会发生改变了，
   - 对于 HostDBRoundRobin 是 rrcount 和 length，在 HEAP 区创建之后，该数据就不再更改了。
   - 但是目前的代码选择把这两个值放在 HostDBRoundRobin 结构里，主要是为了代码编写的方便。
- HEAP 区的动态数据长度不能保存在 MultiCacheBlock 内，因为无法确保 HEAP 区与数据区的磁盘同步是完全对应的，
   - 对于 HostDBRoundRobin 是 good，该值应该与 (inner) HostDBInfo 数组的变动一起保存在 HEAP 区。

因此，个人认为 HostDBRoundRobin 的头部除了 good 成员之外，其它的 4 个成员应该可以定义为 RRInfo 的成员放入 (root) `HostDBInfo::data` 内。但是 64 位程序存在 8 字节对齐的问题，因此 good 如果留在 HostDBRoundRobin 内，仍然会占用 8 个字节，所以只能把  timed_rr_ctime 成员提取出来作为 RRInfo 的成员，另外可以考虑把 HostDBInfo::app.rr.offset 作为 RRInfo 的成员，最终可以为每个 `HostDBRoundRobin` 节省 8 字节的空间，减少 HEAP 区的空间使用。

## 定义

```
struct HostDBRoundRobin {
  /** Total number (to compute space used). */
  // 有多少个 (inner) HostDBInfo 对象
  short rrcount;

  /** Number which have not failed a connect. */
  // 处于 alive 状态的 IP 有多少个
  short good;

  // 表示当前轮询的次数，通过对 good 取模，来访问第 n 个 (inner) HostDBInfo 对象
  unsigned short current;
  // 表示在 HEAP 区占用的空间大小，以字节为单位
  unsigned short length;
  // 当基于固定的时间间隔轮询 (inner) HostDBInfo 对象时，用于记录上一次发生轮询切换的时间
  ink_time_t timed_rr_ctime;

  // 用于保存多个 (inner) HostDBInfo 的数组
  HostDBInfo info[];

  // Return the allocation size of a HostDBRoundRobin struct suitable for storing
  // "count" HostDBInfo records.
  // 返回需要在 HEAP 区申请的内存空间的大小，8 字节对齐。参数：
  //  - count：表示保存多少个 (inner) HostDBInfo 对象
  //  - srv_len：表示在最后为 SRV 记录的 hostname 预留多少空间
  // 返回需要在 HEAP 申请的字节数。
  static unsigned
  size(unsigned count, unsigned srv_len = 0)
  {
    ink_assert(count > 0);
    return INK_ALIGN((sizeof(HostDBRoundRobin) + (count * sizeof(HostDBInfo)) + srv_len), 8);
  }

  /** Find the index of @a addr in member @a info.
      @return The index if found, -1 if not found.
  */
  // 在 (inner) HostDBInfo 数组中查找指定的 IP 地址，返回其数组下标。未找到返回 -1
  // 被 find_ip 和 select_next 方法调用
  int index_of(sockaddr const *addr);
  // 在 (inner) HostDBInfo 数组中查找指定的 IP 地址，返回 (inner) HostDBInfo 对象，未找到返回 NULL
  HostDBInfo *find_ip(sockaddr const *addr);
  // Find the srv target
  // 在 (inner) HostDBInfo 数组中根据主机名查找指定的 SRV 记录，返回 (inner) HostDBInfo 对象，未找到返回 NULL
  HostDBInfo *find_target(const char *target);
  /** Select the next entry after @a addr.
      @note If @a addr isn't an address in the round robin nothing is updated.
      @return The selected entry or @c NULL if @a addr wasn't present.
   */
  // 在 (inner) HostDBInfo 数组中查找指定 IP 地址的下一个 HostDBInfo 对象，如果没有可用的 HostDBInfo 对象，返回 NULL
  HostDBInfo *select_next(sockaddr const *addr);
  // 通过轮询算法选择一个 (inner) HostDBInfo 对象，用于域名解析
  HostDBInfo *select_best_http(sockaddr const *client_ip, ink_time_t now, int32_t fail_window);
  // 通过轮询算法选择一个 (inner) HostDBInfo 对象，用于 SRV 记录
  HostDBInfo *select_best_srv(char *target, InkRand *rand, ink_time_t now, int32_t fail_window);
  // 构造函数，完成各成员的初始化
  HostDBRoundRobin() : rrcount(0), good(0), current(0), length(0), timed_rr_ctime(0) {}
};
```

## 方法

### HostDBInfo *HostDBRoundRobin::select\_best\_http(client ip, now, fail window)

该方法用于从域名解析类型的 HostDBRoundRobin 对象中，以轮询算法选择一个 (inner) HostDBInfo 对象。参数：

- `sockaddr const *client_ip`：客户端 IP 地址
- `ink_time_t now`：当前时间
- `int32_t fail_window`：目标服务器不可访问后，等待多长时间再次尝试访问该服务器

返回：

- 找到的 (inner) HostDBInfo 对象，
- 返回 NULL 表示 HostDBRoundRobin 对象无效。

```
inline HostDBInfo *
HostDBRoundRobin::select_best_http(sockaddr const *client_ip, ink_time_t now, int32_t fail_window)
{
  // 如果 HostDBRoundRobin 里的 HostDBInfo 对象个数小于等于 0
  // 或者 HostDBInfo 对象个数超出了允许的最大值 16（HOST_DB_MAX_ROUND_ROBIN_INFO）
  // 或者可用的 HostDBInfo 对象个数小于等于 0
  // 或者可用的 HostDBInfo 对象个数超出了允许的最大值 16（HOST_DB_MAX_ROUND_ROBIN_INFO）
  bool bad = (rrcount <= 0 || rrcount > HOST_DB_MAX_ROUND_ROBIN_INFO || good <= 0 || good > HOST_DB_MAX_ROUND_ROBIN_INFO);

  // 如果满足上述条件，则说明此 HostDBRoundRobin 出现了问题，返回 NULL
  if (bad) {
    ink_assert(!"bad round robin size");
    return NULL;
  }

  // 当前正在尝试的 HostDBInfo 对象的数组下标
  int best_any = 0;
  // 当前选中的 HostDBInfo 对象的数组下标
  int best_up = -1;

  // ATS 支持三种轮询模式：
  // Basic round robin, increment current and mod with how many we have
  if (HostDBProcessor::hostdb_strict_round_robin) {
    // 第一种：每从 HostDBRoundRobin 里获得 HostDBInfo 对象后就立即触发轮询切换
    Debug("hostdb", "Using strict round robin");
    // Check that the host we selected is alive
    // 遍历所有可用的 HostDBInfo 对象
    for (int i = 0; i < good; i++) {
      // current 记录了轮询切换的次数，其总是执行递增操作，对 good 取模就可以得到 HostDBInfo 对象的数组下标
      best_any = current++ % good;
      // 通过 alive 方法确认选中的 HostDBInfo 对象是否处于活动状态
      if (info[best_any].alive(now, fail_window)) {
        // 如果是，保存该 HostDBInfo 对象的数组下标值，跳出循环，
        // 否则就继续轮询下一个 HostDBInfo 对象。
        best_up = best_any;
        break;
      }
    }
  } else if (HostDBProcessor::hostdb_timed_round_robin > 0) {
    // 第二种：按照一定时间间隔，周期性触发轮询切换
    Debug("hostdb", "Using timed round-robin for HTTP");
    // 跟上一次发生切换操作时间进行比较，判断当前是否满足切换条件
    if ((now - timed_rr_ctime) > HostDBProcessor::hostdb_timed_round_robin) {
      Debug("hostdb", "Timed interval expired.. rotating");
      // 如果满足条件就立即执行轮询切换操作，并更新切换操作发生的时间
      ++current;
      timed_rr_ctime = now;
    }
    // 遍历所有可用的 HostDBInfo 对象
    for (int i = 0; i < good; i++) {
      // BUG：这里对 current 的递增操作导致多切换了一次
      best_any = current++ % good;
      // 通过 alive 方法确认选中的 HostDBInfo 对象是否处于活动状态
      if (info[best_any].alive(now, fail_window)) {
        // 如果是，保存该 HostDBInfo 对象的数组下标值，跳出循环，
        // 否则就继续轮询下一个 HostDBInfo 对象。
        best_up = best_any;
        break;
      }
      // BUG：应该这里增加一个 else 分支，对 current 进行递增操作
    }
    Debug("hostdb", "Using %d for best_up", best_up);
  } else {
    // 第三种：将客户端 IP 地址与 HostDBInfo 对象里保存的 IP 地址逐个联合起来打分，
    // 在所有处于活动状态的 HostDBInfo 对象里取分数最高的那个。
    // 备注：这种方式不是轮询算法，而是一致性哈希算法。
    Debug("hostdb", "Using default round robin");
    // 用于记录所有 HostDBInfo 最高分
    unsigned int best_hash_any = 0;
    // 用来记录处于活动状态的 HostDBInfo 的最高分
    unsigned int best_hash_up = 0;
    sockaddr const *ip;
    // 遍历所有 HostDBInfo 对象
    for (int i = 0; i < good; i++) {
      ip = info[i].ip();
      // 将 client_ip 与 HostDBInfo 对象里保存的 IP 地址放在一起打分
      unsigned int h = HOSTDB_CLIENT_IP_HASH(client_ip, ip);
      // 如果分数比已知的最高分还要高，则记录该最高分及 HostDBInfo 对象的数组下标
      if (best_hash_any <= h) {
        best_any = i;
        best_hash_any = h;
      }
      // 通过 alive 方法确认选中的 HostDBInfo 对象是否处于活动状态
      if (info[i].alive(now, fail_window)) {
        // 如果是，并且分数比已知的活动状态 HostDBInfo 对象的最高分还要高，
        // 则记录该最高分及 HostDBInfo 对象的数组下标
        if (best_hash_up <= h) {
          best_up = i;
          best_hash_up = h;
        }
      }
    }
  }

  if (best_up != -1) {
    // 返回找到的并且处于可用状态的 HostDBInfo 对象
    ink_assert(best_up >= 0 && best_up < good);
    return &info[best_up];
  } else {
    // 如果没有 HostDBInfo 对象处于活动状态，就返回一个找到的 HostDBInfo 对象
    ink_assert(best_any >= 0 && best_any < good);
    return &info[best_any];
  }
}
```

### HostDBInfo *HostDBRoundRobin::select\_best\_srv(target, rand, now, fail window)

该方法用于从 SRV 解析类型的 HostDBRoundRobin 对象中，以轮询算法选择一个 (inner) HostDBInfo 对象。参数：

- `char *target`：用于返回 SRV 主机名
- `InkRand *rand`：随机数
- `ink_time_t now`：当前时间
- `int32_t fail_window`：目标服务器不可访问后，等待多长时间再次尝试访问该服务器

返回：

- 找到的 (inner) HostDBInfo 对象，
- 返回 NULL 表示 HostDBRoundRobin 对象无效。

```
inline HostDBInfo *
HostDBRoundRobin::select_best_srv(char *target, InkRand *rand, ink_time_t now, int32_t fail_window)
{
  // 如果 HostDBRoundRobin 里的 HostDBInfo 对象个数小于等于 0
  // 或者 HostDBInfo 对象个数超出了允许的最大值 16（HOST_DB_MAX_ROUND_ROBIN_INFO）
  // 或者可用的 HostDBInfo 对象个数小于等于 0
  // 或者可用的 HostDBInfo 对象个数超出了允许的最大值 16（HOST_DB_MAX_ROUND_ROBIN_INFO）
  bool bad = (rrcount <= 0 || rrcount > HOST_DB_MAX_ROUND_ROBIN_INFO || good <= 0 || good > HOST_DB_MAX_ROUND_ROBIN_INFO);

  // 如果满足上述条件，则说明此 HostDBRoundRobin 出现了问题，返回 NULL
  if (bad) {
    ink_assert(!"bad round robin size");
    return NULL;
  }

#ifdef DEBUG
  // 确认 SRV 数据按照优先级从高到低排序，SRV 的优先级数字越小，优先级越高。
  // 也就是 srv_priority 值是从小到大排序的。
  for (int i = 1; i < good; ++i) {
    ink_assert(info[i].data.srv.srv_priority >= info[i - 1].data.srv.srv_priority);
  }
#endif

  // infos 数组与 i 和 len 配合将处于活动状态的 SRV 优先级最高的那一组 SRV 数据保存到了 infos 数组
  int i = 0, len = 0;
  // weight 用与计算所有 SRV 记录的权重值之和
  // p 用于记录最高优先级的值，默认设置为最低，逐步向上升高（数值减小）
  uint32_t weight = 0, p = INT32_MAX;
  HostDBInfo *result = NULL;
  HostDBInfo *infos[HOST_DB_MAX_ROUND_ROBIN_INFO];

  // 以下 do-while 循环与 HostDBInfo::alive 方法的逻辑相似
  do {
    // 如果在上一次故障后，还未经过了 fail_window 指定的时间，则此记录仍然处于故障的状态
    if (info[i].app.http_data.last_failure != 0 && (uint32_t)(now - fail_window) < info[i].app.http_data.last_failure) {
      // 调至底部 while 条件判定，继续遍历下一个 HostDBInfo 对象
      continue;
    }

    // 上一次故障后又经过了 fail_window 指定的时间，那么就认为故障已经恢复，将 last_failure 重置为 0
    if (info[i].app.http_data.last_failure)
      info[i].app.http_data.last_failure = 0;

    // 如果当前 HostDBInfo 对象的 SRV 优先级更高
    if (info[i].data.srv.srv_priority <= p) {
      // 更新最高优先级的值
      p = info[i].data.srv.srv_priority;
      // 累加权重
      weight += info[i].data.srv.srv_weight;
      // 将 HostDBInfo 对象放入 infos 数组
      infos[len++] = &info[i];
    } else
    // 如果下一个 SRV 记录的优先级较低，那么就跳出循环
      break;
  } while (++i < good);

  // 如果 HostDBRoundRobin 内的 HostDBInfo 数组是按照 SRV 记录的优先级从高到低排序，
  //  - 那么在 infos 数组内就应该只有处于存活状态的最高优先级那一组的 SRV 记录，
  //  - 并且 weight 的值是累加了这一组 SRV 记录的权重值之和。
  // 否则，infos 数组内会保存多个不同优先级的 SRV 记录

  if (len == 0) { // all failed
    // 没有找到处于活动状态的 SRV 记录，返回当前正在使用的 HostDBInfo 对象相邻的下一个对象
    result = &info[current++ % good];
  } else if (weight == 0) { // srv weight is 0
    // 如果所有 SRV 记录的权重都为 0，既表示没有权重要求，
    // 只有 SRV 优先级控制是有效的，那么就从优先级最高的 SRV 记录里轮询一个
    // BUG：这里应该使用 infos 数组。
    result = &info[current++ % len];
  } else {
    // SRV 记录对权重值有要求
    // 产生一个随机数，确认落入哪个权重位置
    uint32_t xx = rand->random() % weight;
    // 从优先级最高的那一组 SRV 记录里，查找权重值的位置
    // 这里的进入条件应该是 "i < len - 1"， 逻辑上来说，在跳出 for 循环后，i 的最大值可能为 len，
    // 最后有可能会读取 infos[len]，因此在 clang-analyzer 分析时，会提示这里数组越界。
    for (i = 0; i < len && xx >= infos[i]->data.srv.srv_weight; ++i)
      xx -= infos[i]->data.srv.srv_weight;

    result = infos[i];
  }

  // 找到结果，向 target 复制 SRV 记录的主机名，同时返回保存了 SRV 数据的 HostDBInfo 对象。
  if (result) {
    strcpy(target, result->srvname(this));
    return result;
  }
  
  // 应该不会运行到此处
  return NULL;
}
```

## SRV 记录权重值的匹配

根据 SRV 的相关资料，SRV 的权重记录仅在同一个优先级里进行分配，每一个优先级可以有不同的权重分配。例如：

```
# _service._proto.name.  TTL   class SRV priority weight port target.
_sip._tcp.example.com.   86400 IN    SRV 1       4     5060 smallbox1.example.com.
_sip._tcp.example.com.   86400 IN    SRV 1       6     5060 bigbox1.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       4     5060 smallbox2.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       3     5060 smallbox3.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       4     5060 smallbox4.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       2     5060 tinybox1.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       6     5060 bigbox2.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       10     5060 hugebox.example.com.
_sip._tcp.example.com.   86400 IN    SRV 3       6     5060 bigbox3.example.com.
_sip._tcp.example.com.   86400 IN    SRV 10      0      5060 backupbox1.example.com.
_sip._tcp.example.com.   86400 IN    SRV 10      0      5060 backupbox2.example.com.
```

优先级为 1 的记录有两条，其权重值分别为 4，6，权重值总和为 10。只要优先级为 1 的两条记录可用，那么就仅在这两条记录之间，按照 4/10，6/10 的权重来分担所有的业务负载。优先级为 3 的记录暂时不参与业务负载的分担，只有当优先级为 1 的记录全部不可用之后，才会启用下一个优先级的记录，而优先级为 10 的记录其权重值为 0，既表示其不支持按权重分配业务负载，此时默认按照轮询方式来分担业务负载。

由于 SRV 记录按照优先级从高到低在 HostDBRoundRobin 内排序，因此 `HostDBRoundRobin::select_best_srv` 按顺序从高优先级的记录开始遍历，找到第一组处于可用状态的 SRV 记录，将其保存在 infos 指针数组里，同时累加了该组 SRV 记录的权重值之和。然后产生一个随机数对权重值之和取模，取得随机位置，然后按照下图定位该随机位置落在哪一个 SRV 记录里。

先按照上面的例子建立 `HostDBRoundRobin::info[]` 数组的内容，假设优先级为 1 的两个 SRV 记录当前处于不可用状态（Alive 为 N）：

```
+---------+--------------+---+---+---+---+---+---+---+----+---+----+----+
|         |  Array Index | 0 | 1 | 2 | 3 | 4 | 5 | 6 |  7 | 8 |  9 | 10 |
|         +--------------+---+---+---+---+---+---+---+----+---+----+----+
|         | SRV Priority | 1 | 1 | 3 | 3 | 3 | 3 | 3 |  3 | 3 | 10 | 10 |
|  info[] +--------------+---+---+---+---+---+---+---+----+---+----+----+
|         |   SRV Weight | 4 | 4 | 4 | 3 | 4 | 2 | 6 | 10 | 6 |  0 |  0 |
|         +--------------+---+---+---+---+---+---+---+----+---+----+----+
|         |        Alive | N | N | Y | Y | Y | Y | Y |  N | N |  Y |  Y |
+---------+--------------+---+---+---+---+---+---+---+----+---+----+----+ 
```

通过对 `HostDBRoundRobin::info[]` 数组的遍历，产生了 `infos[]` 数组的内容，由于优先级为 1 的 SRV 记录不可用，因此可用的 SRV 记录中，最高优先级为 3，而优先级为 3 的 SRV 记录中也有两个记录不可用，因此最终生成的 `infos[]` 数组只有 5 个 SRV 记录。下面分别产生两个随机数，模拟两次调用的结果，说明 SRV 记录的优先级与权重选取过程：

```
    weight = 4 + 3 + 4 + 2 + 6 = 19
        x1 = RAND() % weight = 7354728 % 19 = 18
        x2 = RAND() % weight =  912357 % 19 = 15
        
                                             x2 x1
                                             |  |
                                             V  V
+---------+--------------+----+---+----+--+------+
|         |  Array Index |  0 | 1 |  2 | 3|   4  |
|         +--------------+----+---+----+--+------+
| infos[] | SRV Priority |  3 | 3 |  3 | 3|   3  |
|         +--------------+----+---+----+--+------+
|         |   SRV Weight |  4 | 3 |  4 | 2|   6  |
+---------+--------------+----+---+----+--+------+
|       Weight indicator |1234|123|1234|12|123456|
+------------------------+----+---+----+--+------+
```

## 参考资料

- [I_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/I_HostDBProcessor.h)
- [P_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_HostDBProcessor.h)
- [Wikipedia::SRV_record](https://en.wikipedia.org/wiki/SRV_record)

# 基础组件：SRVInfo

SRVInfo 用于存储 DNS SRV 记录，其格式如下：

```
_service._proto.name.  TTL   class SRV priority weight port target.
```

DNS 客户端尝试解析域名 `_service._proto.name.` 的 SRV 记录，DNS 服务器返回结果将包含多条 SRV 记录，每一条 SRV 记录都包含：

- Service name：DNS 客户端请求的服务名称，将协议与服务名称组合为域名的格式：`_service._proto.name.`。通过哈希函数转换为数值后，保存在 `HostDBInfo::data.srv.key`，
- TTL：指示该记录的过期时间，将保存在 `HostDBInfo::ip_timeout_interval`，
- priority：指示该 SRV 记录的优先级，值越小，优先级越高，保存在 `HostDBInfo::data.srv.srv_priority`
- weight：指示该 SRV 记录的权重，值越大，被选中的机会越多，保存在 `HostDBInfo::data.srv.srv_weight`
- port：指示该 SRV 记录提供的服务端口，转换为整型数后保存在 `HostDBInfo::data.srv.srv_port`
- target：指示该 SRV 记录提供的服务域名，保存在 HEAP 区 HostDBRoundRobin 对象的后部，其地址可通过 `HostDBInfo::data.srv.srv_offset` 获得

除了 target 信息，其它都是保存在 `HostDBRoundRobin::info[]` 数组中一个 HostDBInfo 对象里，而 target 信息的内容则保存在 `HostDBRoundRobin::info[]` 数组的后部，这些信息一次性写入不会修改。

注意：`srv_offset` 在不同场景表示的含义不同：

- 在独立 `HostDBInfo` 对象中，
   - `data.srv.srv_offset` 表示 SRV 记录的数量，最大为 16，
- 在 `HostDBRoundRobin::info[]` 数组中的 `HostDBInfo` 对象中，
   - `data.srv.srv_offset` 表示 SRV target 相对 HostDBRoundRobin 对象基址的偏移地址，
   - 只有该数组中的对象 `HostDBInfo` 才可以调用 `rr->info[x].srvname(rr)` 方法。

## 定义

```
struct SRVInfo {
  // 前四个成员都是 2 bytes
  unsigned int srv_offset : 16;
  unsigned int srv_weight : 16;
  unsigned int srv_priority : 16;
  unsigned int srv_port : 16;
  // 这个成员是 4 bytes
  unsigned int key;
};
```

需要注意与 `CH06S02-Base-HostEnt` 章节的 SRV 结构区分，DNS 解析的结果首先被存入 SRV 结构，然后再存入 SRVInfo 结构，但是在 SRVInfo 结构里不会存储服务名称，这主要是为了节省内存考虑。

```
struct SRV {
  unsigned int weight;
  unsigned int port;
  unsigned int priority;
  unsigned int ttl;
  unsigned int host_len;
  unsigned int key;
  char host[MAXDNAME];

  SRV() : weight(0), port(0), priority(0), ttl(0), host_len(0), key(0) { host[0] = '\0'; }
};
```

## 参考资料

- [I_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/I_HostDBProcessor.h)
- [Wikipedia::SRV_record](https://en.wikipedia.org/wiki/SRV_record)

