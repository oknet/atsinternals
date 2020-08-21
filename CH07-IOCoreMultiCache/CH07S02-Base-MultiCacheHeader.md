# MultiCache 简介

MultiCache 最大支持 3 层的设计，分别定义为 第 0 层、第 1 层、第 2 层：

```
// 定义了 MultiCache 最大支持的层级
// 注意，该值不可以随意调整，目前的设计没有考虑四层以上的设计，因此修改该值可能造成未知的风险。
#define MULTI_CACHE_MAX_LEVELS 3
```

- 可以将 MultiCache 看成是一个蛋糕，顶层（第 0 层）的容积比较小，容积逐层变大，底层容积最大。
- 按照切蛋糕的方式，从中心点将蛋糕切成将蛋糕切成 n 份（Bucket），然后将连续的几份划分成一个区域，总共划分 64 个区域（Partition）
- 由于切开的份数 n 不是 64 的整数倍，因此每个区域包含的份数可能会相差 1 份（Bucket）。
- 每一份（Bucket）又会被平均切成 m 个小份，每一个小份是一个存储单元叫做 MultiCacheBlock 或 Entry。
- 每一个区域（Partition）使用一个互斥锁进行保护，因此总共有 64 个互斥锁，如果并行线程数较多，可以考虑增加该值（宏定义）。
- 最底层的容量是 MultiCache 的设计容量，上层空间做为底层空间的高速缓存使用，不经常访问的内容会被逐渐推入底层
- 通过哈希方法选择一个区域（Partition）和其中的某一份（Bucket），在这一份（Bucket）内则通过遍历进行查找。
- 上层的容积小，所对应的 Bucket 容量就比较小，容量越小遍历查找的次数就越小，耗时就越短；
- 下层的容积大，所对应的 Bucket 容量就比较大，容量越大遍历查找的次数就越多，耗时就越久；
- 因此为每一层设置合理的容量对性能的影响是非常重要的。

可以看到上面所说的存储单元 MultiCacheBlock 或 Entry 是一种固定容量的存储对象，当需要存储不定长数据时，会遇到麻烦，为此 MultiCache 增设了 Heap 区域，用于存储不定长数据，下面的图以 MultiCache<HostDBInfo> hostdb 的定义为例，大致描述了 MultiCache 对象的格式：

- HostDB 是一个两层 MultiCache 对象：
   - Layer 0 (顶层) 的 Bucket 内有 4 个 Entry
   - Layer 1 (底层) 的 Bucket 内有 32 个 Entry
- Entry 区域后面紧跟着 Heap 区
- 最后是 MultiCacheHeader

内存对象可以直接映射到磁盘上的文件，以实现持久化存储，为了指明用于持久化存储的数据文件的位置，MultiCache 将数据文件的路径以及部分其它信息保存到一个配置文件里，配合数据文件共同完成数据存储的持久化。


```
+-------+-----------------+-----------------+-------------+--...--+-------------+
|   \   |  Partition 0    |  Partition 1    |     P02     |       |     P63     |
+-------+-----------------+-----------------+-------------+--...--+-------------+
|       | +-------------+ | +-------------+ | +---+ +---+ | +---+ | +---+ +---+ |
|   L   | |  Bucket 0   | | |  Bucket 1   | | | B | | B | | | B | | | B | | B | |
|   A   | | +---------+ | | | +---------+ | | | u | | u | | | u | | | u | | u | |
|   Y   | | | Entry 0 | | | | | Entry 4 | | | | c | | c | | | c | | | c | | c | |
|   E   | | +---------+ | | | +---------+ | | | k | | k | | | k | | | k | | k | |
|   R   | | | ...     | | | | | ...     | | | | e | | e | | | e | | | e | | e | |
|       | | +---------+ | | | +---------+ | | | t | | t | | | t | | | t | | t | |
|       | | | Entry 3 | | | | | Entry 7 | | | |   | |   | | |   | | |   | |   | |
|   0   | | +---------+ | | | +---------+ | | | 2 | | 3 | | | . | | | . | | n | |
|       | +-------------+ | +-------------+ | +---+ +---+ | +---+ | +---+ +---+ |
+-------+-----------------+-----------------+-------------+--...--+-------------+
|       | +-------------+ | +-------------+ | +---+ +---+ | +---+ | +---+ +---+ |
|   L   | |  Bucket 0   | | |  Bucket 1   | | | B | | B | | | B | | | B | | B | |
|   A   | | +---------+ | | | +---------+ | | | u | | u | | | u | | | u | | u | |
|   Y   | | | Entry 0 | | | | | Entry32 | | | | c | | c | | | c | | | c | | c | |
|   E   | | +---------+ | | | +---------+ | | | k | | k | | | k | | | k | | k | |
|   R   | | | ...     | | | | | ...     | | | | e | | e | | | e | | | e | | e | |
|       | | +---------+ | | | +---------+ | | | t | | t | | | t | | | t | | t | |
|       | | | Entry31 | | | | | Entry63 | | | |   | |   | | |   | | |   | |   | |
|   1   | | +---------+ | | | +---------+ | | | 2 | | 3 | | | . | | | . | | n | |
|       | +-------------+ | +-------------+ | +---+ +---+ | +---+ | +---+ +---+ |
+-------+-----------------+-----------------+-------------+--...--+-------------+
|       | +-------------+ | +-------------+ | +---+ +---+ | +---+ | +---+ +---+ |
|   L   | |  Bucket 0   | | |  Bucket 1   | | | B | | B | | | B | | | B | | B | |
|   A   | | +---------+ | | | +---------+ | | | u | | u | | | u | | | u | | u | |
|   Y   | | | Entry   | | | | | Entry   | | | | c | | c | | | c | | | c | | c | |
|   E   | | +---------+ | | | +---------+ | | | k | | k | | | k | | | k | | k | |
|   R   | | | ...     | | | | | ...     | | | | e | | e | | | e | | | e | | e | |
|       | | +---------+ | | | +---------+ | | | t | | t | | | t | | | t | | t | |
|       | | | ...     | | | | | ...     | | | |   | |   | | |   | | |   | |   | |
|   2   | | +---------+ | | | +---------+ | | | 2 | | 3 | | | . | | | . | | n | |
|       | +-------------+ | +-------------+ | +---+ +---+ | +---+ | +---+ +---+ |
+-------+-----------------+-----------------+-------------+--...--+-------------+
| Heap  |            Heap half 0            |           heap half 1             |
+-------+-----------------------------------+-----------------------------------+
| MCHdr |                           MultiCacheHeader                            |
+-------+-----------------------------------------------------------------------+
```

# 基础组件：MultiCacheHeader

## MultiCacheHeader

MultiCacheHeader 主要用于保存 MutiCache 数据文件的描述信息（Meta Data），其每一个字节都将写入到磁盘。

### 定义

```
struct MultiCacheHeader {
  // 用于验证 Header 的有效性
  // 初始值为 MULTI_CACHE_MAGIC_NUMBER 0x0BAD2D8
  // 这里应该是笔误，少写了一个 0，0x00BAD2D8
  unsigned int magic;
  // 用于验证 MultiCacheHeader 的定义
  // 如果开发者修改了 MultiCacheHeader 的定义，应该同时修改这个成员的默认值
  // 初始值为：MULTI_CACHE_MAJOR_VERSION 2 和 MULTI_CACHE_MINOR_VERSION 1
  // 即：version.ink_major = 0x0002, version.ink_minor = 0x0001
  VersionNumber version;

  // 用于表述当前 MultiCache 的数据区分为几层，最多可分为（MULTI_CACHE_MAX_LEVELS 3）层
  unsigned int levels;

  // 在对 MultiCache 内的各个数据区进行索引时，会使用 MD5 值作为 Key，
  // 该值指定我们只使用 MD5 值的多少个 bits 作为 Key 进行索引，
  // 在 HostDB 内，这个值为 56，表示使用 MD5 值的低 56 bits 作为 Key 进行索引。
  int tag_bits;
  // 设置最大的查询命中次数，如果当前层级所有 Entry 的命中次数的平均值超过最大命中次数的50%，
  // 则在遍历 Entry 时，降低其命中次数，这是查找算法中的惩罚策略的逻辑，详细分析见后面章节。
  // 在 struct HostDBCache : public MultiCache<HostDBInfo> 的定义中，用于定义使用多少 bits 来保存 HostDBInfo 的命中次数
  // 对于 HostDB 的来说，
  //   在 P_HostDBProcessor.h 中定义了 HOST_DB_HITS_BITS 为 3，
  //   在 I_HostDBProcessor.h 中定义了 HostDBInfo::hits 成员变量为 3 个 bits
  // 上述两个值需要保持一致。
  int max_hits;
  // 存储到 MultiCache 内，单个数据对象的字节大小
  int elementsize;

  // 指定有多少个 bucket 用于保存数据（最小值为 64）
  // MultiCache 的每一层所持有的 bucket 数量是相同的，但是每一层的 bucket 的容量不同，
  //   - 第 0 层默认每个 bucket 可以保存 4 个对象
  //   - 第 1 层默认每个 bucket 可以保存 32 个对象
  //   - 第 2 层默认每个 bucket 可以保存 1 个对象
  // 每一层的 bucket 能够保存对象的数量，是根据总的对象数量计算得出的。
  // 由于目前代码里没有看到使用 2 层以上的 MultiCache，因此超过 2 层的设计，目前未能分析明白。
  // 由于 MultiCache 会被固定划分为 64 个分区，而每个分区最少持有一个 bucket，所以 buckets 的最小值为 64。
  int buckets;
  // 用于保存每一层的偏移量
  int level_offset[MULTI_CACHE_MAX_LEVELS];
  // 用来为每一层记录每个 bucket 能够保存的对象数量
  int elements[MULTI_CACHE_MAX_LEVELS];
  // 用来为每一层记录每个 bucket 的字节容量
  int bucketsize[MULTI_CACHE_MAX_LEVELS];

  // 累加所有层级上的数据单元数量
  int totalelements;
  // 按照（STORE_BLOCK_SIZE 8192）字节对齐后的 MultiCache 的总字节数，包括：
  // - 所有层级的 buckets 占用的字节数
  //   - 第 2 层所有 buckets 占用的字节数
  //   - 第 1 层所有 buckets 占用的字节数
  //   - 第 0 层所有 buckets 占用的字节数
  //   - 累加上述字节数，并按照（STORE_BLOCK_SIZE 8192）字节对齐
  // - 每一个数据单元保留 estimated_heap_bytes_per_entry() 字节的 heap 空间，
  //   - 累加所有单元的保留字节数，并按照（STORE_BLOCK_SIZE 8192）字节对齐
  // - 一个 MultiCacheHeader 所占用的字节空间，并按照（STORE_BLOCK_SIZE 8192）字节对齐
  // MultiCache 会申请该长度的内存，并使用 mmap() 映射到磁盘文件
  unsigned int totalsize;

  // 初始化 MultiCache 时，计划可容纳多少个不同的数据对象（这也是最外层可容纳的元素总量）
  // 创建 MultiCache 时，由于需要满足于分区和 buckets 的划分要求，最外层可容纳的元素总量可能会大于该值
  int nominal_elements;

  // optional heap
  // HEAP 空间的累计字节数
  int heap_size;
  // heap_used 数组的下标，初始值为 0
  volatile int heap_halfspace;
  // HEAP 空间被平分为 [0] 和 [1] 两个部分
  //   - [0] 和 [1] 分别记录了两个部分使用了多少字节，都是从 8 开始，最大值为 HEAP 空间的一半
  //   - 当其中一半的使用量超过 80% 时，就会创建 MultiCacheHeapGC 切换到另外一半
  //   - 然后 MultiCacheHeapGC 对 64 个分区逐个进行遍历，将数据从原来的部分复制到新的部分，并刷新到磁盘文件
  // 初始值：heap_used[0] 和 heap_used[1] 都为 8
  // HEAP 空间的分配是按照（MULTI_CACHE_HEAP_ALIGNMENT 8）字节对齐
  volatile int heap_used[2];

  // 构造函数，用于初始化上述成员
  MultiCacheHeader();
};
```
## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)

