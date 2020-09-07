# 基础组件：MultiCacheBlock

在 MultiCache 中，每一个数据元素叫做 Entry，为了能够对其进行管理，MultiCache 要求数据元素的定义 `class C` 参照 MultiCacheBlock 进行声明。

然后通过 `MultiCache<C>` 的继承类实现不同用途的 MultiCache 数据库。

## 定义

```
struct MultiCacheBlock {
  // 用户获得该 Entry 的唯一识别信息
  uint64_t tag();
  // 返回该 Entry 是否已经被删除
  bool is_deleted();
  // 将 Entry 设置为已删除状态（目前代码中没有看到有任何组件或模块会调用该方法）
  void set_deleted();
  // 判断该 Entry 是否为空
  bool is_empty();
  // 将 Entry 设置为空白状态
  void set_empty();
  // 重置 Entry 的数据
  void reset();
  // 将数据填充到 Entry
  void set_full(uint64_t folded_md5, int buckets);
  // 返回 Entry 在 HEAP 区占用的空间大小
  int
  heap_size()
  {
    return 0;
  }
  // Entry 内有一个 int 类型的成员，用于保存数据在 HEAP 区的位置，
  // 该方法返回指向 int 类型成员的指针。
  int *
  heap_offset_ptr()
  {
    return NULL;
  }
  // 表示该 Entry 被命中的次数
  int hits;
  // 表示该 Entry 已经备份到下一级，例如从 Level 0 备份到 Level 1，从 Level 1 备份到 Level 2
  bool backed;
};
```

对于 Entry 或 Block 的删除状态的用途，目前不明确，只有一处 HostDB.cc 的代码调用了 `is_deleted()` 方法，但是未找到调用 `set_deleted()` 方法的代码。


## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)


# 基础组件：MultiCacheBase

MultiCacheBase 以 MultiCacheHeader 为基类定义了实现 MultiCache 所需要的基本数据接口：

- 所有不需要写入磁盘的内存变量都声明为 MultiCacheBase 的成员
- 所有需要持久化写入磁盘的信息，都在 MultiCacheHeader 中定义
- 定义了与 MultiCache 底层系统相关的内部支持功能及相关的方法

MultiCache 支持将数据同步到磁盘以实现数据的持久化存储，但是也可以关闭该功能，仅在内存中存储数据。如果开启了数据持久化功能，将使用 mmap 系统调用建立内存与文件系统的映射关系，否则会直接使用 malloc 或 memalign 系统调用分配连续内存空间替代 mmap 系统调用。


```
                                      Layout of mmap-ed DB file

           MultiCacheBase::data
MultiCacheBase::level_offset[0] ----> +------+---------------+ <---- database file offset[0]
                                      |      |               |
                                      |      |    Layer 0    |
                                      | D    |               |               +-= member =--------+
MultiCacheBase::level_offset[1] ----> +      +---------------+               |          PtrMutex |
                                      |  A   |               |               |   locks[MC_PARTS] |
                                      |      |    Layer 1    |               +-------------------+
                                      |   T  |               |
MultiCacheBase::level_offset[2] ----> +      +---------------+
                                      |    A |               |               +-= member =--------+
                                      |      |    Layer 2    |               | UnsunkPtrRegistry |
                                      |      |               |               |  unsunk[MC_PARTS] |
           MultiCacheBase::heap ----> +------+---------------+               +-------------------+
                                      |      |  Heap half 0  |
                                      | HEAP +---------------+
                                      |      |  Heap half 1  |               +-= member =--------+
  MultiCacheBase::mapped_header ----> +------+---------------+               |  MultiCacheHeader |
                                      |                      | <=== COPY === |       header_snap |
                                      |   MultiCacheHeader   |               +-------------------+
                                      |                      |
                                      +----------------------+ <---- database file offset[EOF]       

 ** MC_PARTS = MULTI_CACHE_PARTITIONS

```


## 定义

```
struct MultiCacheBase : public MultiCacheHeader {
  // 指向 Store 结构，用于指定使用的存储设备或文件
  Store *store;
  // 数据文件的路径及文件名
  char filename[PATH_NAME_MAX];
  // 指向由 mmap() 映射的内存区域，该区域与磁盘文件（例如：host.db）的尾部是映射关系，
  // 对此内存区域的任何修改都会被自动同步到磁盘文件，但是仍然需要定期刷写到磁盘，以免系统崩溃导致文件数据不完整。
  // 在 MultiCacheBase::open() 方法中，通过以下操作将磁盘数据区的 MultiCacheHeader 内容复制到当前 MultiCache 对象
  //   *(MultiCacheHeader *)this = *mapped_header;
  MultiCacheHeader *mapped_header;

  // 在将内存中的 MultiCache 刷新到磁盘时，首先会对当前的 MultiCacheHeader 做一个快照
  // 在 MultiCacheSync::heapEvent() 方法中，通过以下操作完成：
  //   mc->header_snap = *(MultiCacheHeader *)mc;
  // 当所有分区的数据都刷新到磁盘之后，再将 header_snap 的内容复制到 mapped_header 指向的内存区域，
  // 然后调用 msync() 将 MultiCacheHeader 刷写到磁盘。
  MultiCacheHeader header_snap;

  // mmap-ed region
  // 指向被 mmap 映射的内存区域
  // 包含了 Level[0], Level[1], Level[2], Heap, mapped_header 共 5 个区域的数据
  char *data;
  // 用于描述每个 bucket 的数据在 Level 0 ~ 2 的存储情况
  char *lowest_level_data;

  // equal to data + level_offset[3] + bucketsize[3] * buckets;
  // 指向 HEAP 存储区，等价于 data + level_offset[2] + bucketsize[2] * buckets; （原始的注释应该是错误的）
  char *heap;

  // interface functions
  //
  // Heap 空间一半的容量
  int
  halfspace_size()
  {
    return heap_size / 2;
  }

  // Stats support
  //
  // 代码里未使用到该成员，猜测是用来统计查找时每一层的命中的次数，以及查询整个 MultiCache 未命中的次数
  int hit_stat[MULTI_CACHE_MAX_LEVELS];
  int miss_stat;

  /* lowest_level_data 用来描述数据在 bucket 里的分布情况
   * 一个 bucket 跨越了 MultiCache 所有的层级，当最顶层填满后，就会将不经常访问的 Entry 推入同一个 bucket 内更低的层级，
   * 由于 MultiCache 最大支持 3 个层级，因此只需要使用 2 个 bits 就可以为每个 bucket 记录其数据存储的最低层级。
   * 因此使用 buckets 的数量除以 4 就可以计算出存储所有数据需要的字节数，如果余数不足 4 个 bucket 的时候是要额外占一个字节的。
   * 由于默认的整形计算总是向下取整，因此采用以下公式实现向上取整来解决余数不足 4 个时需要多占一个字节的情况：
   *   y = (x + max-1) / max
   */
  // 返回 lowest_level_data 需要的内存字节数
  unsigned int
  lowest_level_data_size()
  {
    return (buckets + 3) / 4;
  }
  // 返回指定 bucket 的 lowest_level
  unsigned int
  lowest_level(unsigned int bucket)
  {
    unsigned int i = (unsigned char)lowest_level_data[bucket / 4];
    // 这里存在 bug，应该是：
    //   return 3 & (i >> ((bucket %4) << 1));
    return 3 & (i >> (buckets % 4));
  }
  // 为指定的 bucket 设置 lowest_level
  void
  set_lowest_level(unsigned int bucket, unsigned int lowest)
  {
    unsigned char p = (unsigned char)lowest_level_data[bucket / 4];
    // 下面两行存在 bug，应该是：
    //   p &= ~(3 << ((bucket % 4) << 1));
    //   p |= (lowest & 3) << ((bucket % 4) << 1);
    p &= ~(3 << (buckets % 4));
    p |= (lowest & 3) << (buckets % 4);
    lowest_level_data[bucket / 4] = (char)p;
  }

  /* 使用 buckets_per_partitionF8 实现快速的向上取整计算，以纠正在将小数转换为整形数时总是向下取整的损失。
   * 分区的数量固定为 64 个，需要将所有 Bucket 平均分配到每一个分区。
   *
   *   buckets_per_partitionF8 = (number_of_buckets << 8) / 64
   *
   * 这里先对 number_of_buckets 乘以 256 再除以 64，相当于保留了最低精度 1/256。
   * 这里需要注意：
   *   - number_of_buckets 是永远大于等于 64 的
   *   - 每一层都有相同数量的 bucket，其编号都是从 0 开始
   *   - 需要同时使用 level 和 bucket 编号，才能唯一确定一个 bucket 
   */
  // Fixed point, 8 bits shifted left
  int buckets_per_partitionF8;

  /* 计算一个 Bucket 所在的分区（需要注意，bucket 的编号是从 0 开始的）
   * 该算法是使用整形计算代替浮点计算的优化算法，并且保留了 1/256 的精度
   * 
   * 当将所有的 bucket 映射到 0 ~ 63 分区时，通常使用以下公式计算分区编号：
   *   partition_of_bucket = index_of_bucket / number_of_buckets * 64
   *
   * 但是该公式需要使用到浮点计算，例如（注意以下步骤都是浮点计算）：
   *     9.0 / 100.0 * 64.0 = 0.09 * 64.0 = 5.76
   *     浮点数直接转换为整形就是向下取整，结果为 5。
   *
   * 使用浮点优化算法，将原浮点计算方式转为整形计算方式后：
   *   partition_of_bucket = (index_of_bucket * 256 + 255) / ((number_of_buckets * 256) / 64)
   *
   * 使用上述例子重新计算，如下（注意以下步骤都是整形计算）：
   *     (9 * 256 + 255) / (100 * 256 / 64) = 2559 / 400 = INT(6.3975) = 6
   *     由于提前在分子上增加了一个 255，因此整个算式的结果 6 是原式结果向上取整的值。
   *     而这里分母可以提前计算并保存在 buckets_per_partitionF8，因此在实际的计算过程里又节省了一些步骤，效率大幅提升。
   *
   * 注：向上取整是为了避免出现余数的问题，但是这里其实没有必要，因为这里不存在需要多占用一个字节的问题。
   */
  int
  partition_of_bucket(int b)
  {
    // 因为 Partition 的值为 0 ~ 63，因此这里的 + 0xFF 可以去掉，
    // 但是去掉 + 0xFF 之后会导致 bucket 与 partition 的对应关系发生变化，
    // 因此，需要同时修改 MultiCache 的数据库版本，重新构造整个 MultiCache 数据库。
    return ((b << 8) + 0xFF) / buckets_per_partitionF8;
  }
  // 计算指定分区的第一个 bucket 的编号
  int
  first_bucket_of_partition(int p)
  {
    return ((buckets_per_partitionF8 * p) >> 8);
  }
  // 计算指定分区的最后一个 bucket 的编号
  int
  last_bucket_of_partition(int p)
  {
    return first_bucket_of_partition(p + 1) - 1;
  }
  // 计算指定分区有多少个 bucket
  int
  buckets_of_partition(int p)
  {
    return last_bucket_of_partition(p) - first_bucket_of_partition(p) + 1;
  }

  // 打开 MultiCache 数据库，
  // 需要同时传入 描述文件 和 数据文件 的文件名，如果 数据文件 的文件名为 NULL，则不支持持久化存储
  int open(Store *store, const char *config_filename, char *db_filename = NULL, int db_size = -1, bool reconfigure = false,
           bool fix = false, bool silent = false);

  // 用于读写 MultiCache 数据库描述文件的前三行：
  //   第 01 行：可容纳的元素数量（参考 MultiCacheHeader::nominal_elements）
  //   第 02 行：MultiCache 的 bucket 数量（参考 MultiCacheHeader::bucket）
  //   第 03 行：MultiCache Heap 空间的大小，以字节为单位（参考 MultiCacheHeader::heap_size）
  // MultiCache 数据库的描述文件以换行符 \n 分隔，每行一个信息，详细说明可以查看 Store 和 Span 章节。
  // 1 for success, 0 for no config file, -1 for failure
  int read_config(const char *config_filename, Store &store, char *fn = NULL, int *pi = NULL, int *pbuckets = NULL);
  int write_config(const char *config_filename, int nominal_size, int buckets);
  
  // 初始化
  int initialize(Store *store, char *filename, int elements, int buckets = 0, unsigned int levels = 2,
                 int level0_elements_per_bucket = 4, int level1_elements_per_bucket = 32, int level2_elements_per_bucket = 1);
  // 通过 mmap 系统调用映射数据区到磁盘
  // 备注：存在遗留的 Unmaper 结构，猜测是用于 Windows 平台支持类似 mmap 系统调用的功能。
  int mmap_data(bool private_flag = false, bool zero_fill = false);
  char *mmap_region(int blocks, int *fds, char *cur, size_t &total_length, bool private_flag, int zero_fill = 0);
  // 计算指定的 Level 有多少个 BLOCK
  int blocks_in_level(unsigned int level);

  // 验证当前 MultiCache 对象的 MultiCacheHeader 的信息与磁盘上数据文件里的 MultiCacheHeader 保存的信息是否是一致的
  // 返回 true 表示两者一致
  // 返回 false 表示验证失败，两者不一致
  bool verify_header();

  // 取消 mmap_data 方法映射的内存对象
  // 成功返回 0，失败返回 -1
  int unmap_data();
  // 重置 MultiCache，先后执行：
  //  - 释放 store
  //  - 释放 lowest_level_data
  //  - 调用 umap_data 方法释放 data
  void reset();
  // 清空当前 MultiCache 存储的内容，包括
  //  - 将 data 指向的内存区域用 0 填充，这包括 MultiCache 数据区的各个层级、Heap区域和最后的 MultiCacheHeader
  //  - 重置 heap_used[] 和 heap_halfspace
  //  - 通过 maped_header 同步 MultiCacheHeader 到磁盘
  void clear(); // this zeros the data
  // 仅清空当前 MultiCache 数据区的各个层级
  void clear_but_heap();

  // 虚函数，用于返回一个 MultiCache<C> 或其继承类的对象实例，可以参考：
  //  - P_MultiCache.h 中 MultiCache<C>::dup() 的定义
  //  - P_HostDBProcessor.h 中 HostDBCache::dup() 的定义
  virtual MultiCacheBase *
  dup()
  {
    ink_assert(0);
    return NULL;
  }

  // 用于返回每个 Entry 平均需要在 Heap 区域占用的空间大小
  // 该值通常为预估值，对于不同类型和用途的 MultiCache 数据库，
  //   - 需要预估 Entry 占用 Heap 空间的比率，因为可能有 Entry 不会使用到 Heap 空间
  //   - 需要预估一个 Entry 可能会占用 Heap 空间的最大值
  // 上述两个值按照一定运算规则（如使用乘法）得出一个固定的值，然后再乘以 MultiCache 设定的 Entry 数量，即可得到 Heap 区占用的空间，
  // 因此每一个 MultiCache 数据库都必须提供该方法，这样才能正确计算 Heap 区域的大小。
  virtual size_t
  estimated_heap_bytes_per_entry() const
  {
    return 0;
  }

  // 由 MultiCacheBase::rebuild 方法调用，打印 totalelements 和 totalsize 到 diags.log 里
  void print_info(FILE *fp);

//
// Rebuild the database, also perform checks, and fixups
// ** cannot be called on a running system **
// "this" must be initialized.
//
#define MC_REBUILD 0
#define MC_REBUILD_CHECK 1
#define MC_REBUILD_FIX 2
  // 重新构建数据库，执行检查和修复工作
  // 返回 0: 成功
  // 返回 -1: 失败
  int rebuild(MultiCacheBase &old, int kind = MC_REBUILD); // 0 on success

  // 仅在 rebuild 方法中调用，定义在 MultiCache<C>::rebuild_element
  virtual void
  rebuild_element(int buck, char *elem, RebuildMC &r)
  {
    (void)buck;
    (void)elem;
    (void)r;
    ink_assert(0);
  }

  //
  // Check the database
  // ** cannot be called on a running system **
  //  assumes that the configuration is correct
  //
  // 检查数据库
  // 是对 rebuild 方法的封装，在其基础上增加了对配置文件的验证功能
  int check(const char *config_filename, bool fix = false);

  // 返回指定的 bucket 所在的 Partition 的互斥锁
  ProxyMutex *
  lock_for_bucket(int bucket)
  {
    return locks[partition_of_bucket(bucket)];
  }
  
  // 计算 Tag 值（Tag 值用于比对两个 Entry 是否一致）
  // 传入的 folded_md5 值是将 MD5 值从 128bit 降低成 64bit 的 hash 值（参考：INK_MD5.h 中 union CryptoHash 的定义）
  // 使用 MultiCache 的 bucket 数量，简单的对 64bit 的 hash 值进行除法运算得到 Tag 值，
  // 通常还会进行取模运算，从而得到 BUCKET 的编号。
  uint64_t
  make_tag(uint64_t folded_md5)
  {
    uint64_t ttag = folded_md5 / (uint64_t)buckets;
    if (!ttag)
      return 1LL;
    // beeping gcc 2.7.2 is broken
    if (tag_bits > 32) {
      uint64_t mask = 0x100000000LL << (tag_bits - 32);
      mask = mask - 1;
      return ttag & mask;
    } else {
      uint64_t mask = 1LL;
      mask <<= tag_bits;
      mask = mask - 1;
      return ttag & mask;
    }
  }

  // 通过系统调用 msync 将所有 mmap 的内存区域的数据刷新到磁盘
  int sync_all();
  // 将与指定 Partition 关联的 HEAP 区域刷新到磁盘，被 sync_all 方法调用（刚方法的实现存在问题）
  int sync_heap(int part); // part varies between 0 and MULTI_CACHE_PARTITIONS
  // 将 MultiCacheHeader 刷新到磁盘，被 sync_all 方法调用
  int sync_header();
  // 将指定的 Partition 内的 BUCKET 刷新到磁盘，被 sync_all 方法调用
  int sync_partition(int partition);
  // 创建 MultiCacheHeapGC 或 MultiCacheSync 来完成数据库的周期性同步到磁盘的功能，以实现数据的持久化存储
  // 无法避免数据丢失，仍然有丢失少量数据的可能性，但是已经将数据丢失的可能性降到最低。
  void sync_partitions(Continuation *cont);

  // 构造函数，主要用于完成 unsunk 部分的初始化
  MultiCacheBase();
  // 析构函数，调用 reset 方法重置 MultiCache
  virtual ~MultiCacheBase() { reset(); }

  // 获得 Entry 或 MultiCacheBlock 对象的大小（字节）
  // 必须在 MultiCache<C> 的继承类中定义
  virtual int
  get_elementsize()
  {
    ink_assert(0);
    return 0;
  }

  // Heap support
  // HEAP 区的数据定时通过 msync 系统调用同步刷新到磁盘，因此部分数据被刷新到磁盘，部分数据未被刷新到磁盘。
  // 为了对上述两种情况进行区分，未被刷新到磁盘的数据指针由 UnsunkPtr 管理，已经刷新到磁盘的数据则由 Entry 直接管理。
  // UnsunkPtrRegistry[x] 是用来管理多个 UnsunkPtr 对象的数组。
  UnsunkPtrRegistry unsunk[MULTI_CACHE_PARTITIONS];

  // -1 on error
  // 对于给定的指针，返回其指向的对象所在的 Partition
  // 返回 -1：该指针指向的地址不在数据区
  // 返回 0 ~ 63：表示所在的 Partition
  int ptr_to_partition(char *);
  // the user must pass in the offset field within the
  // MultiCacheBlock object.  The offset will be inserted
  // into the object on success and a pointer to the data
  // returned.  On failure, NULL is returned;
  // 从 HEAP 上分配指定 size 的一段空间
  //  - 其中 poffset 指向数据区内某个 Entry 或 MultiCacheBlock 内某个 int 类型成员
  //  - 该 int 类型成员用于建立与 UnsunkPtr 的关联（可参考 UnsunkPtr 的成员 int *poffset 的说明）
  void *alloc(int *poffset, int size);
  // 将 SrcEntry 的内容复制到 DstEntry 时，由于在 UnsunkPtr 中的 poffset 成员指向 SrcEntry 内的某个 int 类型的成员，
  // 因此需要为 DstEntry 创建一个新的 UnsunkPtr 使其 poffset 成员指向 DstEntry 内的某个 int 类型的成员，
  // 最后将 UnsunkPtr 中表示数据存储在 HEAP 区偏移量的成员 offset 的值复制到新的 UnsunkPtr。
  // 注意：
  //  - poffset 和 old_poffset 指针指向的位置应位于同一个 Partition，可以是不同的层级，并且两者不能指向同一个地址
  //  - *old_poffset 的值应该小于 0，表示 UnsunkPtr 位于 UnsunkPtrRegister[x] 的位置
  //  - 目前 update 方法存在 bug，没有处理 *old_poffset 大于 0 的情况
  void update(int *poffset, int *old_poffset);
  // 根据指定的 poffset 和 partition，返回在 HEAP 上存储的不定长数据的指针
  void *ptr(int *poffset, int partition);
  // 验证给出的 offset 是否表示合法的 HEAP 位置
  // 注意：offset 应该大于 0，但是这里未做判定
  int
  valid_offset(int offset)
  {
    int max;
    if (offset < halfspace_size())
      max = heap_used[0];
    else
      max = halfspace_size() + heap_used[1];
    return offset < max;
  }
  // 验证给出的指针，是否指向合法的 HEAP 位置
  // 注意：p 应该大于 heap，但是这里未做判定
  int
  valid_heap_pointer(char *p)
  {
    if (p < heap + halfspace_size())
      return p < heap + heap_used[0];
    else
      return p < heap + halfspace_size() + heap_used[1];
  }
  // 将 src 地址开始的 s 个字节复制到当前的 HEAP 半区，由 MultiCache<C>::copy_heap 方法调用
  // 同时通过 pi 和 partition 更新 UnsunkPtr，使其指向新的 HEAP 半区；或将相关信息保存到 HeapGC，稍后统一更新到 Entry
  void copy_heap_data(char *src, int s, int *pi, int partition, MultiCacheHeapGC *gc);
  // 判断给定的 offset 位于 HEAP 的哪一个半区
  // 返回 0：表示位于第一个半区
  // 返回 1：表示位于第二个半区
  int
  halfspace_of(int o)
  {
    return o < halfspace_size() ? 0 : 1;
  }
  // 遍历指定 partition 的 UnsunkPtrRegistry[x]，如其指向 HEAP 区的数据位置在 before_used 之前，
  // 则将 UnsunkPtr 的成员 *poffset 值由负数改为正数，同时回收 UnsunkPtr 对象。
  // 解释：
  //   - UnsunkPtr 字面理解为未沉没的指针，意指漂浮在内存中的指针，属于易失性数据。
  //   - 该方法使 UnsunkPtr “沉没”，落入 Entry，而 Entry 内存区域是通过 mmap 建立了与磁盘文件的映射关系，可以使数据不易丢失。
  // 注意：该方法的实现存在 bug，详见后续分析及修正。
  UnsunkPtrRegistry *fixup_heap_offsets(int partition, int before_used, UnsunkPtrRegistry *r = NULL, int base = 0);

  // 对指定 partiton 进行逐层进行遍历，查找在 HEAP 内存储数据的 Entry，
  // 然后调用 copy_heap_data 方法，将 HEAP 的数据从一个半区迁移到另外一个半区，并更新 Entry 或 UnsunkPtr 的指向。
  virtual void
  copy_heap(int partition, MultiCacheHeapGC *gc)
  {
    (void)partition;
    (void)gc;
  }

  //
  // Private
  //
  // 为每个 Partition 创建一个互斥锁
  void
  alloc_mutexes()
  {
    for (int i = 0; i < MULTI_CACHE_PARTITIONS; i++)
      locks[i] = new_ProxyMutex();
  }
  PtrMutex locks[MULTI_CACHE_PARTITIONS]; // 1 lock per (buckets/partitions)
};
```

## 数据库的创建与加载

### int HostDBCache::start(int flags)

HostDB 启动时会尝试创建和加载以 MultiCache 作为底层的 HostDB 数据库。

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
  //
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

### void stealStore(Store &s, int blocks)

该方法是从缓存的存储设备上分配空间。它首先加载 storage.config 里用于缓存的存储设备的空间配置，然后在其上进行多种类型的空间分配，如：

- hostdb.config 的空间分配
- dir.config 的空间分配（未见代码中有该配置文件）
- alt.config 的空间分配（未见代码中有该配置文件）

然后在剩余的空间上分配 blocks 数量的空间，并将结果存入 Store &s，但是所得空间仍然可能小于所申请的数量。

问题：

- 在缓存的存储设备上为 hostdb.config，dir.config，alt.config 分配空间，然后在剩余的空间里再分配一些空间，
- 但是缓存的存储设备只是为了缓存提供服务，并不会为 HostDB 等其它数据库提供存储服务，
- 如果在缓存的存储设备上分配了一部分空间，那么缓存的空间就会出问题了？

```
void
stealStore(Store &s, int blocks)
{
  // 读取并解析 storage.config，失败直接返回
  // 此时 Store &s 内存储的是 cache 磁盘的总空间
  if (s.read_config())
    return;
  Store tStore;
  // 读取并解析 hostdb.config，分配方案存入 tStore 内
  MultiCacheBase dummy;
  if (dummy.read_config("hostdb.config", tStore) > 0) {
    Store dStore;
    // 如读取成功则在 Store &s 上尝试按照 tStore 的要求分配空间，剩余的空间保存在 Store &s
    s.try_realloc(tStore, dStore);
  }
  // 清空 tStore
  tStore.delete_all();
  // 读取并解析 dir.config，分配方案存入 tStore 内
  if (dummy.read_config("dir.config", tStore) > 0) {
    Store dStore;
    // 如读取成功则在 Store &s 上尝试按照 tStore 的要求分配空间，剩余的空间保存在 Store &s
    s.try_realloc(tStore, dStore);
  }
  // 清空 tStore
  tStore.delete_all();
  // 读取并解析 alt.config，分配方案存入 tStore 内
  if (dummy.read_config("alt.config", tStore) > 0) {
    Store dStore;
    // 如读取成功则在 Store &s 上尝试按照 tStore 的要求分配空间，剩余的空间保存在 Store &s
    s.try_realloc(tStore, dStore);
  }
  // tStore 在最后会通过析构函数完成清理，这里就不调用 delete_all 方法了
  // grab some end portion of some block... so as not to damage the
  // pool header
  // 此时在 Store &s 里存储的是在 cache 磁盘的总空间上分配 hostdb.config，dir.config 和 alt.config 之后的剩余空间
  for (unsigned d = 0; d < s.n_disks;) {
    Span *ds = s.disk[d];
    // 遍历每一个存储设备上的 Span
    while (ds) {
      if (!blocks)
        // 如果已经完成空间分配，就将剩余的 Span 空间清 0
        ds->blocks = 0;
      else {
        // 计算在当前 Span 上的待分配空间
        int b = blocks;
        if ((int)ds->blocks < blocks)
          b = ds->blocks;
        // 如果当前 Span 的存储设备是文件，那么将当前 Span 向尾部缩小到 b 个 BLOCK；
        // 否则就向头部缩小到 b 个 BLOCK，
        if (ds->file_pathname)
          ds->offset += (ds->blocks - b);
        ds->blocks = b;
        // 减去已经分配的空间
        blocks -= b;
      }
      // 链表上的下一个 Span
      ds = ds->link.next;
    }
    // 下一个存储设备
    d++;
  }
  // 最后在 Store &s 里就是从各个 Span 缝隙里分配到的空间，但是这些空间可能小于所申请的数量
}
```

### int MultiCacheBase::open(Store *store, config filename, db filename, db size, reconfigure, fix, silent)

打开一个 MultiCache 数据库，参数：

- store：保存了按照指定的参数分配好空间的 Store 结构（这些参数不是来自 config filename 里的记录）
- config filename：数据库的配置文件（MultiCache 数据库的描述文件以换行符 \n 分隔，每行一个信息，详细说明可以查看 Store 和 Span 章节）
- db filename：数据库的数据文件，默认为 NULL，表示使用配置文件里给出的数据文件的名字
- db size：数据库可存储的条目数（Entry 或 MultiCacheBlock 的数量），默认为 -1，表示使用配置文件中的 db size 的值
- reconfigure：忽略配置文件的内容，使用传入的 store，db filename，db size 重新构建配置文件，默认为 false
- fix：尝试修复错误的数据库，主要是遍历所有 Entry，将存在错误数据的 Entry 删除，默认为 false，不进行修复
- silent：遇到错误时，是否输出相关信息，默认为 false，表示输出错误信息

通过 `read_config` 方法读取 config filename 里的信息，获知当前数据库的配置和空间占用情况，然后与传入的 Store 和 db size 进行匹配以发现数据库的配置是否发生变化。如果未发生变化，则直接调用 `initialize` 方法初始化数据库，然后调用 `mmap_data` 加载数据文件，如指定 fix 参数为 true，则在加载后调用 `check` 方法扫描所有的 entry，并剔除掉存在问题的 entry。如果发生变化，则尝试按照 Store 和 db size 的要求调用 `initialize` 方法重新设定数据库，然后调用 `write_config` 保存配置文件，最后调用 `rebuild` 方法加载并转换数据库的数据（此处存在 bug，HEAP 数据可能全部丢失或错乱）。

返回值：

- 返回 0 表示成功
- 返回 -1 表示失败

```
int
MultiCacheBase::open(Store *s, const char *config_filename, char *db_filename, int db_size, bool reconfigure, bool fix, bool silent)
{
  int ret = 0;
  const char *err = NULL;
  char *serr = NULL;
  char t_db_filename[PATH_NAME_MAX];
  int t_db_size = 0;
  int t_db_buckets = 0;
  int change = 0;

  t_db_filename[0] = 0;

  // Set up cache
  {
    Store tStore;
    // 通过 read_config 读取并解析数据库的配置文件，按照配置文件创建 Span 结构并添加到指定的 tStore，同时返回：
    //   - t_db_filename：配置文件里保存的数据文件的名字
    //   - t_db_size：数据库的条目数量（参考 MultiCacheHeader::nominal_elements）
    //   - t_db_buckers：Bucket 数量（参考 MultiCacheHeader::bucket）
    // 返回值 res：
    //   - 无法打开配置文件：0
    //   - 解析配置文件错误：-1
    //   - 成功：1
    int res = read_config(config_filename, tStore, t_db_filename, &t_db_size, &t_db_buckets);

    ink_assert(store_verify(&tStore));
    // 配置文件出错
    if (res < 0)
      goto LfailRead;
    // 无法打开配置文件（不存在或权限不正确）
    if (!res) {
      // 如果：不需要重新配置，或未提供数据库文件名，或未提供数据库记录数量
      // 则认为配置出现错误，需要管理员修改配置
      if (!reconfigure || !db_filename || !db_size)
        goto LfailConfig;
      // 配置文件无误，可能是配置发生了变化，使用新配置重新初始化数据库
      if (initialize(s, db_filename, db_size) <= 0)
        goto LfailInit;
      // 初始化成功后，将新配置写入数据库的配置文件
      write_config(config_filename, db_size, buckets);
      // 映射内存与磁盘的关系
      if (mmap_data() < 0)
        goto LfailMap;
      // 清空数据库（数据库配置发生变化后，无法自动导入旧数据，只能重建数据库）
      clear();
    } else {
      // 成功加载配置文件
      // don't know how to rebuild from this problem
      // 如果传入了数据文件的名字，那么应该与配置文件里保存的名字一致
      ink_assert(!db_filename || !strcmp(t_db_filename, db_filename));
      // 如果没有传入数据库的文件名，则使用配置文件里给出的数据库的文件名
      if (!db_filename)
        db_filename = t_db_filename;

      // Has the size changed?
      // 判断传入的数据库条目数与配置文件里记录的值是否发生变化，
      // change 大于 0 表示增加，小于 0 表示减少，等于 0 表示不变
      change = (db_size >= 0) ? (db_size - t_db_size) : 0;
      if (db_size < 0)
        db_size = t_db_size;
      // 如数据库条目数发生变化，但是未将 reconfigure 设置为 true，则认为配置文件错误
      if (change && !reconfigure)
        goto LfailConfig;

      // 将 tStore 复制一份存入 cStore，tStore 内保存的是配置文件里对数据库如何占用存储空间的信息
      Store cStore;
      tStore.dup(cStore);

      // Try to get back our storage
      Store diff;

      // 在传入的 Store *s 上按照配置文件里定义的存储分配方案（cStore）进行空间分配，
      // 如果 cStore 上的某个 Span 不能在 Store *s 里进行分配，则将其保存到 Store diff。
      s->try_realloc(cStore, diff);
      // 如果出现不能完全分配的情况，同时也未将 reconfigure 设置为 true，则认为配置文件错误
      if (diff.n_disks && !reconfigure)
        goto LfailConfig;

      // Do we need to do a reconfigure?
      // 如果出现了不能完全分配的情况，或者数据库条目数发生了变化，这里隐含了 reconfigure == true
      if (diff.n_disks || change) {
        // find a new store to old the amount of space we need
        // 这里存入 delta 的是数据库条目数的变化，可能为负数，零，正数。
        int delta = change;

        // 这里存入 delta 的是当前存储空间比配置文件中减少的 BLOCK（8KB）数量，可能为零，正数。
        if (diff.n_disks)
          delta += diff.total_blocks();
        // BUG：上面对 delta 的赋值，没有考虑到数据库条目数与存储空间同时发生变化的情况。

        if (delta) {
          // delta 大于 0 的情况可能有以下几种：
          //  - 数据条目增加，存储空间也增加（文档里建议在数据条目增加的同时，同步增加存储空间）
          //  - 数据条目不变，存储空间增加（当条目数与存储空间不匹配时）
          //  - 数据条目增加，存储空间不变（当条目数与存储空间不匹配时）
          if (delta > 0) {
            // 尝试从 storage.config 上借用一些空间，借到的空间分配方案保存在 freeStore 里，delta 是需要借用的空间
            // 一般来说，一个数据库条目不会有 8KB 那么大，因此当数据条目增加时，实际上会多分配了一些空间
            // 备注：从 storage.config 上借用空间，提供给以 MultiCache 为底层的数据库，从目前的代码来看有些奇怪，或者说我没有完全读懂这部分代码？
            Store freeStore;
            stealStore(freeStore, delta);
            // 在 freeStore 上分配 delta 个 BLOCK，结果存储在 more 里。
            Store more;
            freeStore.spread_alloc(more, delta);
            // 如果 more 里的 block 数量小于 delta，表示未能完成空间分配，则认为是配置错误
            if (delta > (int)more.total_blocks())
              goto LfailReconfig;
            // 按照 more 的分配方案，在 Store *s 上进行空间分配，不能分配的空间保存到 more_diff 上
            Store more_diff;
            s->try_realloc(more, more_diff);
            // 如果 more_diff 不为空，表示未能完成空间分配，则认为是配置错误
            if (more_diff.n_disks)
              goto LfailReconfig;
            // 将 more 的分配方案追加到 cStore 里（cStore 是配置文件所描述的分配方案）
            cStore.add(more);
            // 由于 more 的空间来自 storage.config，因此 storage.config 的空间分配发生了变化，因此要把数据文件清空，但是不清理指向目录的 Span
            // 如果清理失败，则认为配置错误
            if (more.clear(db_filename, false) < 0)
              goto LfailReconfig;
          }
          // delta 小于 0 的情况可能有以下几种：
          //  - 数据条目减少，存储空间也减少（同步缩小）
          //  - 数据条目减少，存储空间不变
          if (delta < 0) {
            // 从配置文件的空间分配方案里删掉 delta 个 BLOCK
            Store removed;
            cStore.spread_alloc(removed, -delta);
          }
        }
        // 重新整理 cStore，此时 cStore 里包含了原有配置文件的分配方案，同时也可能包含从 storage.config 里额外得到的分配方案
        cStore.sort();
        // 初始化 MultiCache 数据库，t_db_buckets 的值会被直接存入 this->buckets
        // 最新的存储方案包含了原有的数据文件，新增加的数据文件以附加磁盘的方式，位于 Store 内  disk[x] 数组的最后，
        // BUG：当数据条目（db_size）变化，特别是减少时，bucket 数量应该重新计算，但是这里直接传入配置文件中读取的值，可能造成异常。
        // 例如在 initialize 中计算 elements_per_bucket = elements / buckets 时，可能导致 elements_per_bucket 为 0。
        // 但是下面需要调用 rebuild 方法，这里必须确保新旧两个数据库保持相等的 bucket 数量（详见 rebuild 方法的分析）。
        if (initialize(&cStore, db_filename, db_size, t_db_buckets) <= 0)
          goto LfailInit;

        ink_assert(store_verify(store));

        // 更新配置文件，写入新的 db_size 和 buckets 等数据
        if (write_config(config_filename, db_size, buckets) < 0)
          goto LfailWrite;

        ink_assert(store_verify(store));

        //  rebuild
        // 返回一个 MultiCache<C> 的继承类的对象
        MultiCacheBase *old = dup();
        // 使用配置文件中定义的空间分配方式初始化数据库，用于在 rebuild 过程中加载原来的数据文件
        if (old->initialize(&tStore, t_db_filename, t_db_size, t_db_buckets) <= 0) {
          delete old;
          goto LfailInit;
        }

        // 重建（rebuild）操作流程如下：
        //  1. 先将原数据文件加载到匿名 mmap 内存区域，
        //  2. 然后使用最新的存储方案（cStore）和数据条目数（db_size）建立内存到磁盘的 mmap 映射，
        //     - 此时由于 db_size 发生了变化，但是在 initialize 方法里传入了原来的 bucket 数量，因此 bucket 内 entry 的数量一定发生的变化，
        //     - 进而导致 data 区域的总长度发生了变化，同时其在存储设备上占用的空间大小也发生了变化，
        //     - 而 HEAP 区域是紧挨着 data 区域之后的，因此 HEAP 区的位置也发生了变化，HEAP 区的大小是通过 db_size 计算得到的，所以 HEAP 区的大小也发生了变化。
        //  3. 最后遍历匿名内存区域的原数据，将数据逐个重新插入到新建的 mmap 内存区域
        //     - BUG：分析 rebuild 部分的代码，没有看到 HEAP 区的数据复制过程，感觉是这里假设 HEAP 区的映射位置不变，因此不需要复制 HEAP 区的数据。
        //     - 但是结合第 2 步的分析，HEAP 区在磁盘上的位置和大小都发生了变化，因此这里缺少了对 HEAP 区域数据的处理。
        if (rebuild(*old)) {
          delete old;
          goto LfailRebuild;
        }
        ink_assert(store_verify(store));
        delete old;

      } else {
        // 数据条目和空间均保持一致
        // 初始化 MultiCache 数据库
        if (initialize(&tStore, db_filename, db_size, t_db_buckets) <= 0)
          // 此处应该跳转到 LfailInit
          goto LfailFix;
        ink_assert(store_verify(store));
        // 将数据文件通过 mmap 映射到内存
        if (mmap_data() < 0)
          goto LfailMap;
        // 验证数据文件里的 MultiCacheHeader
        if (!verify_header())
          goto LheaderCorrupt;
        // 将数据文件里的 MultiCacheHeader 加载到内存中的 MultiCache 对象里
        *(MultiCacheHeader *)this = *mapped_header;
        ink_assert(store_verify(store));

        // 如设置 fix 为 true，则调用 check 方法进行修复处理
        // check 方法通过调用 rebuild 并传入 MC_REBUILD_FIX 参数实现 MultiCache 数据库的修复
        if (fix)
          if (check(config_filename, true) < 0)
            goto LfailFix;
      }
    }
  }

  if (store)
    ink_assert(store_verify(store));

// 返回结果
Lcontinue:
  return ret;

// 设置错误信息、根据 errno 转换系统调用的错误信息为字符串等
LheaderCorrupt:
  err = "header missing/corrupt";
  goto Lfail;

LfailWrite:
  err = "unable to write";
  serr = strerror(errno);
  goto Lfail;

LfailRead:
  err = "unable to read";
  serr = strerror(errno);
  goto Lfail;

LfailInit:
  err = "unable to initialize database (too little storage)\n";
  goto Lfail;

LfailConfig:
  err = "configuration changed";
  goto Lfail;

LfailReconfig:
  err = "unable to reconfigure";
  goto Lfail;

LfailRebuild:
  err = "unable to rebuild";
  goto Lfail;

LfailFix:
  err = "unable to fix";
  goto Lfail;

LfailMap:
  err = "unable to mmap";
  serr = strerror(errno);
  goto Lfail;

// 输出 debug 信息
Lfail : {
  unmap_data();
  if (!silent) {
    if (reconfigure) {
      RecSignalWarning(REC_SIGNAL_CONFIG_ERROR, "%s: [%s] %s: disabling database\n"
                                                "You may need to 'reconfigure' your cache manually.  Please refer to\n"
                                                "the 'Configuration' chapter in the manual.",
                       err, config_filename, serr ? serr : "");
    } else {
      RecSignalWarning(REC_SIGNAL_CONFIG_ERROR, "%s: [%s] %s: reinitializing database", err, config_filename, serr ? serr : "");
    }
  }
}
  ret = -1;
  goto Lcontinue;
}
```

### int MultiCacheBase::initialize(Store *store, char *filename, int elements, ...);

根据给定的信息初始化一个 MultiCache 数据库，参数：

- store：保存了按照指定的参数分配好空间的 Store 结构（这些参数不是来自 config filename 里的记录）
- filename：数据库的数据文件，默认为 NULL，表示使用配置文件里给出的数据文件的名字
- elements：同 `MultiCacheHeader::nominal_elements`，计划可容纳多少个不同的数据对象（Entry）
- buckets：指定数据库的 bucket 数量，默认值为 0，表示根据 elements 和最底层每个 bucket 的容量自动计算出适合的 bucket 数量
- levels：表示数据区的层数，最少 1 层，最多 3 层，默认值为 2
- level 0 elements per bucket：第 0 层每个 bucket 可容纳的元素（Entry）数量，默认为 4
- level 1 elements per bucket：第 1 层每个 bucket 可容纳的元素（Entry）数量，需要大于第 0 层的值，默认为 32
- level 2 elements per bucket：第 2 层每个 bucket 可容纳的元素（Entry）数量，需要大于第 1 层的值，默认为 1，因此该默认值不合法，如制定 levels=3 时，需要显示指定该值

返回值：

- 返回 < 0 表示出错
- 返回 > 0 表示实际分配的 BLOCK 数量，既 MultiCache 的占用内存、磁盘空间的大小

建议 elements 值为 level X elements per bucket 的整数倍，否则：

- 最底层（最外层）可容纳的数据对象（Entry）的数量会低于 elements 的值
- 各 level X elements per bucket 的值也会重新计算
- 详见下面的分析

```
//
// Initialize MultiCache
// The outermost level of the cache contains ~aelements.
// The higher levels (lower in number) contain fewer.
//
int
MultiCacheBase::initialize(Store *astore, char *afilename, int aelements, int abuckets, unsigned int alevels,
                           int level0_elements_per_bucket, int level1_elements_per_bucket, int level2_elements_per_bucket)
{
  int64_t size = 0;

  Debug("multicache", "initializing %s with %d elements, %d buckets and %d levels", afilename, aelements, abuckets, alevels);
  ink_assert(alevels <= MULTI_CACHE_MAX_LEVELS);
  // MultiCache 数据区最大支持 3 层设计
  if (alevels > MULTI_CACHE_MAX_LEVELS) {
    Warning("Alevels too large %d, cannot initialize MultiCache", MULTI_CACHE_MAX_LEVELS);
    return -1;
  }
  // 设置数据库的层级参数
  levels = alevels;
  // 设置每个元素（Entry）占用的字节数，get_elementsize 方法在 MultiCache<C> 的继承类中定义
  elementsize = get_elementsize();
  // 初始化所有层级可容纳的总元素数量为 0，接下来将逐层计算并累加
  totalelements = 0;
  // 最底层（最外层）可存储的数据元素数量
  nominal_elements = aelements;
  // 设置 bucket 数量
  buckets = abuckets;

  // 复制数据库文件名
  ink_strlcpy(filename, afilename, sizeof(filename));
  //
  //  Allocate level 2 as the outermost
  //
  // 如果层级为 3 层，从最底层（最外层）开始计算
  if (levels > 2) {
    // 如果层级设定为 3 层，并且 buckets 设置为 0，将根据 elements 和最底层每个 bucket 的容量自动计算出适合的 bucket 数量
    if (!buckets) {
      // 如果 aelements 不是 level2_elements_per_bucket 的整数倍，这里会向下取整，那么会导致该数据层可存储的元素数量小于 aelements 指定的值。
      buckets = aelements / level2_elements_per_bucket;
      // 确保每个分区至少有一个 bucket
      // BUG：此处应同步调整 aelements 的值，否则 level2_elements_per_bucket 的值可能为 0
      if (buckets < MULTI_CACHE_PARTITIONS)
        buckets = MULTI_CACHE_PARTITIONS;
    }
    // 如果层级设定为 3 层，重新计算 level2_elements_per_bucket
    if (levels == 3)
      level2_elements_per_bucket = aelements / buckets;
    // elements[x] 数组用于存储每个层级 bucket 可容纳的元素（Entry）数量
    elements[2] = level2_elements_per_bucket;
    // 累加可容纳的总元素数量
    totalelements += buckets * level2_elements_per_bucket;
    // bucketsize[x] 数组用于存储每个层级 bucket 占用的字节数
    bucketsize[2] = elementsize * level2_elements_per_bucket;
    // 累加占用的总内存字节数
    size += (int64_t)bucketsize[2] * (int64_t)buckets;

    // 确认 level2_elements_per_bucket > level1_elements_per_bucket
    if (!(level2_elements_per_bucket / level1_elements_per_bucket)) {
      Warning("Size change too large, unable to reconfigure");
      return -1;
    }

    // 按照每一级 bucket 可容纳的元素数量，计算上一级可容纳的总元素数量
    aelements /= (level2_elements_per_bucket / level1_elements_per_bucket);
  }
  //
  //  Allocate level 1
  //
  // 如果层级为 2 层或 3 层，计算第 1 级的数据
  if (levels > 1) {
    // 如果层级设定为 2 层，并且 buckets 设置为 0，将根据 elements 和最底层每个 bucket 的容量自动计算出适合的 bucket 数量
    if (!buckets) {
      // 如果 aelements 不是 level1_elements_per_bucket 的整数倍，这里会向下取整，那么会导致该数据层可存储的元素数量小于 aelements 指定的值。
      buckets = aelements / level1_elements_per_bucket;
      // 确保每个分区至少有一个 bucket
      // BUG：此处应同步调整 aelements 的值，否则 level1_elements_per_bucket 的值可能为 0
      if (buckets < MULTI_CACHE_PARTITIONS)
        buckets = MULTI_CACHE_PARTITIONS;
    }
    // 如果层级设定为 2 层，重新计算 level1_elements_per_bucket
    if (levels == 2)
      level1_elements_per_bucket = aelements / buckets;
    // elements[x] 数组用于存储每个层级 bucket 可容纳的元素（Entry）数量
    elements[1] = level1_elements_per_bucket;
    // 累加可容纳的总元素数量
    totalelements += buckets * level1_elements_per_bucket;
    // bucketsize[x] 数组用于存储每个层级 bucket 占用的字节数
    bucketsize[1] = elementsize * level1_elements_per_bucket;
    // 累加占用的总内存字节数
    size += (int64_t)bucketsize[1] * (int64_t)buckets;
    // 确认 level1_elements_per_bucket > level0_elements_per_bucket
    if (!(level1_elements_per_bucket / level0_elements_per_bucket)) {
      Warning("Size change too large, unable to reconfigure");
      return -2;
    }
    // 按照每一级 bucket 可容纳的元素数量，计算上一级可容纳的总元素数量
    aelements /= (level1_elements_per_bucket / level0_elements_per_bucket);
  }
  //
  //  Allocate level 0
  //
  // 计算第 0 级的数据
  // 如果层级设定为 1 层，并且 buckets 设置为 0，将根据 elements 和最底层每个 bucket 的容量自动计算出适合的 bucket 数量
  if (!buckets) {
    // 如果 aelements 不是 level0_elements_per_bucket 的整数倍，这里会向下取整，那么会导致该数据层可存储的元素数量小于 aelements 指定的值。
    buckets = aelements / level0_elements_per_bucket;
    // 确保每个分区至少有一个 bucket
    // BUG：此处应同步调整 aelements 的值，否则 level0_elements_per_bucket 的值可能为 0
    if (buckets < MULTI_CACHE_PARTITIONS)
      buckets = MULTI_CACHE_PARTITIONS;
  }
  // 如果层级设定为 1 层，重新计算 level0_elements_per_bucket
  if (levels == 1)
    level0_elements_per_bucket = aelements / buckets;
  // elements[x] 数组用于存储每个层级 bucket 可容纳的元素（Entry）数量
  elements[0] = level0_elements_per_bucket;
  // 累加可容纳的总元素数量
  totalelements += buckets * level0_elements_per_bucket;
  // bucketsize[x] 数组用于存储每个层级 bucket 占用的字节数
  bucketsize[0] = elementsize * level0_elements_per_bucket;
  // 累加占用的总内存字节数
  size += (int64_t)bucketsize[0] * (int64_t)buckets;

  // 计算 buckets_per_partitionF8，参考定义部分的说明
  buckets_per_partitionF8 = (buckets << 8) / MULTI_CACHE_PARTITIONS;
  ink_release_assert(buckets_per_partitionF8);

  // 根据 size 值计算出需要多少个 BLOCK（8KB），向上取整（这里可以调用 bytes_to_blocks 方法）
  unsigned int blocks = (size + (STORE_BLOCK_SIZE - 1)) / STORE_BLOCK_SIZE;

  // 根据总元素（所有层级）数量和每个元素预计占用的 HEAP 空间大小，计算出 HEAP 空间的容量（以字节计）
  heap_size = int((float)totalelements * estimated_heap_bytes_per_entry());
  // 转换 HEAP 空间的大小为 BLOCK 数量，并累加到 blocks
  blocks += bytes_to_blocks(heap_size);

  // MultiCacheHeader 占用一个 BLOCK
  blocks += 1; // header
  // 计算 MultiCache 数据库占用的总内存/磁盘空间
  totalsize = (int64_t)blocks * (int64_t)STORE_BLOCK_SIZE;

  Debug("multicache", "heap_size = %d, totalelements = %d, totalsize = %d", heap_size, totalelements, totalsize);

  //
  //  Spread alloc from the store (using storage that can be mmapped)
  //
  delete store;
  store = new Store;
  // 在传入的 Store 结构上尝试分配 blocks 数量的空间，且要求空间必须是支持 mmap 的
  astore->spread_alloc(*store, blocks, true);
  // 分配结果保存到 MultiCacheBase::store 内，通过 total_blocks 得到实际完成分配的空间
  unsigned int got = store->total_blocks();

  // 判断得到的空间是否小于请求的空间数量
  if (got < blocks) {
    // 空间分配失败，提示用户增加空间
    astore->free(*store);
    delete store;
    store = NULL;
    Warning("Configured store too small, unable to reconfigure");
    return -3;
  }
  // 空间分配成功，该行代码可删除，重复计算 totalsize
  totalsize = (STORE_BLOCK_SIZE)*blocks;

  // 保存每个层级数据相对 data 指针的偏移，方便后续使用
  level_offset[1] = buckets * bucketsize[0];
  level_offset[2] = buckets * bucketsize[1] + level_offset[1];

  // 重设 lowest_level_data
  if (lowest_level_data)
    delete[] lowest_level_data;
  lowest_level_data = new char[lowest_level_data_size()];
  ink_assert(lowest_level_data);
  memset(lowest_level_data, 0xFF, lowest_level_data_size());

  // 返回实际分配的 BLOCK 数量，既 MultiCache 的占用内存、磁盘空间的大小
  return got;
}
```

### int MultiCacheBase::mmap_data(bool private flag = false, bool zero fill = false)

在调用 `initialize` 方法完成数据库初始化之后，就需要调用 `mmap_data` 方法将数据文件加载（映射）到内存中，参数：

- private flag：默认为 false
   - 当设置为 true 时，在调用 mmap 系统调用时会传递 `MAP_PRIVATE` 参数，这表示磁盘文件映射到内存后，在内存中的改变不会同步到磁盘文件。
   - 放设置为 false 时，在调用 mmap 系统调用时会传递 `MAP_SHARED` 参数，这表示磁盘文件映射到内存后，在内存中的改变将会同步到磁盘文件。
- zero fill：默认为 false
   - 当设置为 true 时，如果在将 `MultiCacheBase::store` 映射到内存时遇到文件无法打开时，会使用匿名映射来代替该文件的映射，通常与 private flag 同时设置为 true。
   - 当设置为 false 时，会直接在内存中为 MultiCache 分配所需的空间，因此将不会支持数据持久化存储的功能。

仅在 `rebuld` 方法中调用 `mmap_data(true, true)` 用于读取原始数据文件的内容，以避免改变原始内容。

最终，`mmap_data` 调用 `mmap_region` 方法完成映射关系的建立，返回值：

- 返回 0 表示成功
- 返回 -1 表示失败


```
int
MultiCacheBase::mmap_data(bool private_flag, bool zero_fill)
{
  // 用于匿名 mmap 映射
  ats_scoped_fd fd;
  // 用于保存所有 MultiCacheBase::store 内各个 Span 的文件的描述符，最大支持 256 个文件。
  int fds[MULTI_CACHE_MAX_FILES] = {0};
  // 记录 fds 数组实际打开的文件数
  int n_fds = 0;
  // 实际完成映射的字节数
  size_t total_mapped = 0; // total mapped memory from storage.

  // open files
  //
  // 如果 MultiCacheBase::store 没有指向有效的存储，则直接分配一段连续的内存，此时将无法实现数据的持久化存储
  if (!store || !store->n_disks)
    goto Lalloc;
  // 按顺序遍历 MultiCacheBase::store，逐个打开 Span 对应的文件，将描述符保存到 fds 数组，并使用 n_fds 记录打开的文件数量
  for (unsigned i = 0; i < store->n_disks; i++) {
    Span *ds = store->disk[i];
    for (unsigned j = 0; j < store->disk[i]->paths(); j++) {
      char path[PATH_NAME_MAX];
      Span *d = ds->nth(j);
      // 获取文件名的绝对路径
      int r = d->path(filename, NULL, path, PATH_NAME_MAX);
      if (r < 0) {
        Warning("filename too large '%s'", filename);
        goto Labort;
      }
      // 以读写方式打开文件，如果不存在就创建该文件（适用于首次配置）
      fds[n_fds] = socketManager.open(path, O_RDWR | O_CREAT, 0644);
      if (fds[n_fds] < 0) {
        // 如果打开或创建文件失败，但是 zero_fill 为 false 则直接分配一段连续的内存，此时将无法实现数据的持久化存储
        if (!zero_fill) {
          Warning("unable to open file '%s': %d, %s", path, errno, strerror(errno));
          goto Lalloc;
        }
        //
        fds[n_fds] = 0;
      }
      // 当 Span 的底层存储指向一个目录，不是块设备或字符设备时
      if (!d->file_pathname) {
        struct stat fd_stat;

        // 此时会在 Span 指定的目录下创建 MultiCacheBase::filename 为名称的文件
        // 如果文件在打开之前已经存在，那么就要获得其文件长度
        if (fstat(fds[n_fds], &fd_stat) < 0) {
          Warning("unable to stat file '%s'", path);
          goto Lalloc;
        } else {
          // 计算当前 MultiCache 需要的文件长度
          int64_t size = (off_t)(d->blocks * STORE_BLOCK_SIZE);
          // 比较实际文件长度是否与所需的文件长度一致，如果不一致就重设文件长度，并将数据文件用 0x00 填充
          if (fd_stat.st_size != size) {
            int err = ink_file_fd_zerofill(fds[n_fds], size);

            // 如果调整文件长度失败，则直接分配一段连续的内存，此时将无法实现数据的持久化存储
            if (err != 0) {
              Warning("unable to set file '%s' size to %" PRId64 ": %d, %s", path, size, err, strerror(err));
              goto Lalloc;
            }
          }
        }
      }
      n_fds++;
    }
  }

  data = 0;


  // mmap levels
  //
  {
    // make a copy of the store
    // 将 MultiCacheBase::store 的内容复制到 tStore
    Store tStore;
    store->dup(tStore);
    Store *saved = store;
    // 将 tStore 作为临时操作空间，遇到任何错误时，使用 saved 恢复原来的 store 内容
    store = &tStore;

    char *cur = 0;

// find a good address to start
// 打开 /dev/zero 用于接下来创建匿名 mmap 映射
#if !defined(darwin)
    fd = socketManager.open("/dev/zero", O_RDONLY, 0645);
    if (fd < 0) {
      store = saved;
      Warning("unable to open /dev/zero: %d, %s", errno, strerror(errno));
      goto Labort;
    }
#endif

// lots of useless stuff
// 创建匿名 mmap 映射 totalsize 的空间，确认是否有足够的连续的内存空间
#if defined(darwin)
    cur = (char *)mmap(0, totalsize, PROT_READ, MAP_SHARED_MAP_NORESERVE | MAP_ANON, -1, 0);
#else
    cur = (char *)mmap(0, totalsize, PROT_READ, MAP_SHARED_MAP_NORESERVE, fd, 0);
#endif
    // 如果映射失败，则还原 store，并返回错误信息
    if (cur == NULL || cur == (caddr_t)MAP_FAILED) {
      store = saved;
#if defined(darwin)
      Warning("unable to mmap anonymous region for %u bytes: %d, %s", totalsize, errno, strerror(errno));
#else
      Warning("unable to mmap /dev/zero for %u bytes: %d, %s", totalsize, errno, strerror(errno));
#endif
      goto Labort;
    }
    // 映射成功后，则立即取消映射
    if (munmap(cur, totalsize)) {
      store = saved;
#if defined(darwin)
      Warning("unable to munmap anonymous region for %u bytes: %d, %s", totalsize, errno, strerror(errno));
#else
      Warning("unable to munmap /dev/zero for %u bytes: %d, %s", totalsize, errno, strerror(errno));
#endif
      goto Labort;
    }

    /* We've done a mmap on a target region of the maximize size we need. Now we drop that mapping
       and do the real one, keeping at the same address space (stored in @a data) which should work because
       we just tested it.
    */
    // coverity[use_after_free]
    // 此时 cur 地址指向的是一段连续的内存空间，并且可以满足 totalsize 的空间要求，
    // 后面会以此地址为基准，连续执行 mmap 操作，将多个文件映射到以 cur 为基址的连续内存空间。
    data = cur;

    // 映射第 0 层，失败则还原 store
    cur = mmap_region(blocks_in_level(0), fds, cur, total_mapped, private_flag, fd);
    if (!cur) {
      store = saved;
      goto Labort;
    }
    // 映射第 1 层，失败则还原 store
    if (levels > 1)
      cur = mmap_region(blocks_in_level(1), fds, cur, total_mapped, private_flag, fd);
    if (!cur) {
      store = saved;
      goto Labort;
    }
    // 映射第 2 层，失败则还原 store
    if (levels > 2)
      cur = mmap_region(blocks_in_level(2), fds, cur, total_mapped, private_flag, fd);
    if (!cur) {
      store = saved;
      goto Labort;
    }

    // 映射 HEAP 区，失败则还原 store
    if (heap_size) {
      heap = cur;
      cur = mmap_region(bytes_to_blocks(heap_size), fds, cur, total_mapped, private_flag, fd);
      if (!cur) {
        store = saved;
        goto Labort;
      }
    }
    // 映射 MultiCacheHeader，失败则还原 store
    // BUG：此处在 aarch 等架构可能存在问题，因为 aarch 架构的 page size 可能为 64KB，而代码里目前定义一个 BLOCK 为 8KB
    mapped_header = (MultiCacheHeader *)cur;
    if (!mmap_region(1, fds, cur, total_mapped, private_flag, fd)) {
      store = saved;
      goto Labort;
    }
#if !defined(darwin)
    // 关闭用于匿名映射的文件描述符
    ink_assert(!socketManager.close(fd));
#endif
    store = saved;
  }

  // 关闭已经打开的 Span 文件
  for (int i = 0; i < n_fds; i++) {
    if (fds[i] >= 0)
      ink_assert(!socketManager.close(fds[i]));
  }

  return 0;
Lalloc : { // 直接分配一段连续内存替代 mmap 映射
  free(data);
  char *cur = 0;

  data = (char *)ats_memalign(ats_pagesize(), totalsize);
  cur = data + STORE_BLOCK_SIZE * blocks_in_level(0);
  if (levels > 1)
    cur = data + STORE_BLOCK_SIZE * blocks_in_level(1);
  if (levels > 2)
    cur = data + STORE_BLOCK_SIZE * blocks_in_level(2);
  if (heap_size) {
    heap = cur;
    cur += bytes_to_blocks(heap_size) * STORE_BLOCK_SIZE;
  }
  mapped_header = (MultiCacheHeader *)cur;
  for (int i = 0; i < n_fds; i++) {
    if (fds[i] >= 0)
      socketManager.close(fds[i]);
  }

  return 0;
}

Labort: // 遇到不可恢复错误，关闭所有文件描述符，并取消已经创建的映射，返回 -1
  for (int i = 0; i < n_fds; i++) {
    if (fds[i] >= 0)
      socketManager.close(fds[i]);
  }
  if (total_mapped > 0)
    munmap(data, total_mapped);

  return -1;
}
```

### char * MultiCacheBase::mmap_region(int blocks, int fds[], char *cur, size\_t &total length, bool private flag, int zero fill = 0)

`mmap_region` 方法用于在指定的内存地址 `cur` 上映射 blocks 数量的 BLOCK。用于映射的磁盘文件数据将会从 `MultiCacheBase::disk[x]` 数组中按顺序取得。调用之后 `total_length` 为已经完成 mmap 映射的字节数，同时返回一个指针地址，指向已经完成映射区域的尾部的下一字节，如果需要连续进行 mmap 映射，可以将该返回值继续传递给下一次 `mmap_region` 的调用。如果未能完成映射，则返回 NULL。参数：

- blocks：要映射的 BLOCK 数量
- fds：已经打开的 Span 的文件描述符
- cur：指定进行映射的内存地址，如果为 NULL，则在任意地址完成映射
- total length：累加已经完成 mmap 映射的字节数
- private flag：同 `mmap_data`，由 `mmap_data` 传入
   - 当设置为 true 时，在调用 mmap 系统调用时会传递 `MAP_PRIVATE` 参数，这表示磁盘文件映射到内存后，在内存中的改变不会同步到磁盘文件。
   - 放设置为 false 时，在调用 mmap 系统调用时会传递 `MAP_SHARED` 参数，这表示磁盘文件映射到内存后，在内存中的改变将会同步到磁盘文件。
- zero fill：为匿名映射时使用的文件描述符，由 `mmap_data` 传入

注意：`mmap_region` 映射的内存空间是地址连续的内存空间，但是可能会映射到多个位于不同的存储设备上的文件。

```
char *
MultiCacheBase::mmap_region(int blocks, int *fds, char *cur, size_t &total_length, bool private_flag, int zero_fill)
{
  // 验证 blocks 数大于 0
  if (!blocks)
    return cur;
  // 用于记录 fds 数组使用的情况
  int p = 0;
  // 保存 mmap 的返回值，指向映射成功的内存地址，失败时为 NULL
  char *res = 0;
  // 按照与 mmap_data 中相同的顺序遍历 MultiCacheBase::store，按顺序消费每个 Span 的 BLOCK，
  // 该操作确保 Span 文件与 fds 数组完全对应。
  for (unsigned i = 0; i < store->n_disks; i++) {
    // 首先尝试将映射请求平均分配到多个存储设备，以平衡磁盘 I/O 的负载
    unsigned int target = blocks / (store->n_disks - i);
    unsigned int following = store->total_blocks(i + 1);
    // 如果空间不足以平均分配，则增加当前设备的分配容量
    if (blocks - target > following)
      target = blocks - following;
    Span *ds = store->disk[i];
    for (unsigned j = 0; j < store->disk[i]->paths(); j++) {
      Span *d = ds->nth(j);

      ink_assert(d->is_mmapable());

      if (target && d->blocks) {
        // 计算当前 Span 可以进行映射的 BLOCK 数量
        int b = d->blocks;
        if (d->blocks > target)
          b = target;
        // 已经完成映射的空间，从 Span 中删除
        d->blocks -= b;
        // 计算当前 Span 上可以映射的字节数
        unsigned int nbytes = b * STORE_BLOCK_SIZE;
        // 如果 fds[p] 为 0 表示对应的文件打开失败，此时使用传入的 zero_fill 文件描述符完成匿名 mmap 映射（此处表示调用 mmap_data 时 zero_fill 为 true）
        int fd = fds[p] ? fds[p] : zero_fill;
        ink_assert(-1 != fd);
        // 如设置 private_flag 为 true，则为 mmap 调用设置 MAP_PRIVATE 标志，否则设置 MAP_SHARED 标志
        int flags = private_flag ? MAP_PRIVATE : MAP_SHARED_MAP_NORESERVE;

        // 如果传入的 cur 不为 NULL，则通过 MAP_FIXED 标志要求 mmap 在指定的内存地址完成映射
        if (cur)
          res = (char *)mmap(cur, nbytes, PROT_READ | PROT_WRITE, MAP_FIXED | flags, fd, d->offset * STORE_BLOCK_SIZE);
        else
          res = (char *)mmap(cur, nbytes, PROT_READ | PROT_WRITE, flags, fd, d->offset * STORE_BLOCK_SIZE);

        // 已经完成映射的空间，从 Span 中删除
        d->offset += b;

        // 映射失败，直接返回 NULL，但是未回退 Span 空间
        if (res == NULL || res == (caddr_t)MAP_FAILED)
          return NULL;
        ink_assert(!cur || res == cur);
        // cur 指向已经完成映射区域的尾部的下一字节
        cur = res + nbytes;
        // 从 blocks 中减去已经完成映射的 BLOCK 数量
        blocks -= b;
        // 向 total_length 累加已经完成映射的字节数
        total_length += nbytes; // total amount mapped.
      }
      // 继续处理下一个 Span 文件描述符
      p++;
    }
  }
  // 如还有剩余 blocks 未完成映射，则返回 NULL，表示失败
  // 否则返回 cur，指向已经完成映射区域的尾部的下一字节
  return blocks ? 0 : cur;
}
```


### int MultiCacheBase::rebuild(MultiCacheBase &old, int kind = MC_REBUILD)

`rebuild` 方法从传入的旧 MultiCache 数据库的实例读取数据，并在当前 MultiCache 实例重建数据库。`rebuild` 方法要求当前 MultiCache 实例已经完成初始化操作。参数：

- old：旧 MultiCache 数据库的实例
- kind：操作类型，可以是 MC_REBUILD（重建），MC_REBUILD_CHECK（检查），MC_REBUILD_FIX（修复）

调用 rebuild 方法后，不论返回值如何，当前实例不可以继续使用，返回值：

- 返回 0 表示成功
- 返回 < 0 表示失败

```
// file: iocore/hostdb/P_MultiCache.h
struct RebuildMC {
  bool rebuild;
  bool check;
  bool fix;
  char *data;
  int partition;

  int deleted;
  int backed;
  int duplicates;
  int corrupt;
  int stale;
  int good;
  int total;
};

// file: iocore/hostdb/MultiCache.cc
//
// We need to preserve the buckets
// while moving the existing data into the new locations.
//
// if data == NULL we are rebuilding (as opposed to check or fix)
//
int
MultiCacheBase::rebuild(MultiCacheBase &old, int kind)
{
  char *new_data = 0;

  ink_assert(store_verify(store));
  ink_assert(store_verify(old.store));

  // map in a chunk of space to use as scratch (check)
  // or to copy the database to.
  // 创建私有匿名内存映射，用于临时存储 MultiCache 的数据文件内容
  ats_scoped_fd fd(socketManager.open("/dev/zero", O_RDONLY));
  if (fd < 0) {
    Warning("unable to open /dev/zero: %d, %s", errno, strerror(errno));
    return -1;
  }

  new_data = (char *)mmap(0, old.totalsize, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);

  // 创建私有匿名内存映射失败，返回 -1
  ink_assert(data != new_data);
  if (new_data == NULL || new_data == (caddr_t)MAP_FAILED) {
    Warning("unable to mmap /dev/zero for %u bytes: %d, %s", totalsize, errno, strerror(errno));
    return -1;
  }
  // if we are rebuilding get the original data
  // 后面的代码会以遍历 old.data 内的数据为基础，对数据进程检查、修复或重建
  // 当 kind == MC_REBUILD，表示须要重建数据库，此时 data 应该指向 NULL，表示仅完成了初始化操作，还未加载数据文件。
  // 数据库的重建是逐项遍历老的数据，然后将数据逐个插入到当前数据库实例中来。
  if (!data) {
    ink_assert(kind == MC_REBUILD);
    // 使用 old 实例加载原数据文件，并以私有内存映射模式映射文件数据到内存，以避免修改磁盘上的文件内容
    if (old.mmap_data(true, true) < 0)
      return -1;
    // 然后将原数据文件内容保存到刚刚创建的私有匿名内存映射空间
    memcpy(new_data, old.data, old.totalsize);
    // 解除 old 实例的内存映射
    old.unmap_data();
    // now map the new location
    // 然后使用当前实例加载数据文件，加载完成后 totalsize != old.totalsize, heap_size != old.heap_size
    if (mmap_data() < 0)
      return -1;
    // old.data is the copy
    // 将 old 实例的数据切换到私有匿名内存映射，此时 old 实例的数据仅在内存中存在。
    old.data = new_data;
    // 在重建完成后，虽然合法的数据元素（Entry）都被复制回来了，但是由于存储区域可能发生变化，因此 HEAP 区域的数据有可能被全部破坏了（BUG？）
  } else {
    ink_assert(kind == MC_REBUILD_CHECK || kind == MC_REBUILD_FIX);
    // 当 kind != MC_REBUILD，当前实例应该完成了初始化操作和数据文件映射操作，因此 data 应该指向有效的内存映射地址
    // 当 kind == MC_REBUILD_CHECK，表示仅对数据进行检测
    if (kind == MC_REBUILD_CHECK) {
      // old.data is the original, data is the copy
      // old.data 切换到 data，而 data 指向空白的私有匿名内存映射区
      old.data = data;
      data = new_data;
      // 在检查完成之后，也没有将 data 切换回去，因此在检查完数据库之后，当前实例不可用。
    } else {
      // 否则 kind == MC_REBUILD_FIX，此时默认当前实例与 old 实例的配置应该完全一致，因此 totalsize == old.totalsize
      // 将当前实例的数据备份到私有匿名内存映射区，验证正确的数据元素（Entry）会从 old 区复制回来，但是不会复制 HEAP 区的内容
      memcpy(new_data, data, old.totalsize);
      // old.data is the copy, data is the original
      // 将 old.data 切换到私有匿名内存映射区，
      old.data = new_data;
      // 在修复完成之后，只是删除了不合法的数据，因此当前实例可立即使用。
    }
  }

  // 要求 bucket 的数量保持一致
  // 原因可能是为了确保数据库的容量保持一致，但是数据库的记录数发生变化后，会影响到数据区和 HEAP 区的大小
  ink_assert(buckets == old.buckets);

  FILE *diag_output_fp = stderr;

  // RebuildMC 结构用于存在在遍历 old 对象时发现的一些问题的统计
  RebuildMC r;

  r.data = old.data;

  r.rebuild = kind == MC_REBUILD;
  r.check = kind == MC_REBUILD_CHECK;
  r.fix = kind == MC_REBUILD_FIX;

  r.deleted = 0;
  r.backed = 0;
  r.duplicates = 0;
  r.stale = 0;
  r.corrupt = 0;
  r.good = 0;
  r.total = 0;

  // 如果需要进行重建操作，则打印当前数据库实例的信息
  if (r.rebuild)
    fprintf(diag_output_fp, "New:\n");
  // BUG：应该包含到上面的 if 语句里
  print_info(diag_output_fp);
  
  // 如果需要进行重建或修复操作，则打印旧实例的信息，同时清除当前实例 HEAP 区之外的内容
  if (r.rebuild || r.fix) {
    fprintf(diag_output_fp, "Old:\n");
    old.print_info(diag_output_fp);
    clear_but_heap();
  }

  // 开始遍历 old 区的数据元素（Entry）
  fprintf(diag_output_fp, "    [processing element.. ");

  // 记录遍历元素的个数
  int scan = 0;
  // 从最底层/最外层开始遍历
  for (int l = old.levels - 1; l >= 0; l--)
    for (int b = 0; b < old.buckets; b++) {
      // 如果 bucket 的数量保持一致，那么 bucket 所在的 partition 也不会发生变化
      r.partition = partition_of_bucket(b);
      for (int e = 0; e < old.elements[l]; e++) {
        scan++;
        // 每遍历 32767 个数据元素（Entry），就打印一条 debug 信息
        if (!(scan & 0x7FFF))
          fprintf(diag_output_fp, "%d ", scan);
        // x 指向一个数据元素（Entry）
        char *x = old.data + old.level_offset[l] + b * old.bucketsize[l] + e * elementsize;
        // 调用 rebuild_element 方法处理该元素（将在 MultiCache 章节讲解）
        // 该方法检测 x 是否为空，进一步调用 rebuild_callout 方法让业务系统检测该数据元素（Entry）的完整性和合法性，
        // 通过 debug 日志记录有问题的数据元素（Entry），只有完整合法的数据元素（Entry）才会被重新插入到当前数据库的第 0 层。
        rebuild_element(b, x, r);
      }
    }
  // BUG：此处不应该设置 if 判断语句，直接输出 done 信息即可
  if (scan & 0x7FFF)
    printf("done]\n");
  // 如果完成了重建或修复操作，需要同步各个分区的数据到磁盘
  if (r.rebuild || r.fix)
    for (int p = 0; p < MULTI_CACHE_PARTITIONS; p++)
      sync_partition(p);

  // 输出各项统计信息
  fprintf(diag_output_fp, "    Usage Summary\n");
  fprintf(diag_output_fp, "\tTotal:      %-10d\n", r.total);
  if (r.good)
    fprintf(diag_output_fp, "\tGood:       %.2f%% (%d)\n", r.total ? ((r.good * 100.0) / r.total) : 0, r.good);
  if (r.deleted)
    fprintf(diag_output_fp, "\tDeleted:    %5.2f%% (%d)\n", r.deleted ? ((r.deleted * 100.0) / r.total) : 0.0, r.deleted);
  if (r.backed)
    fprintf(diag_output_fp, "\tBacked:     %5.2f%% (%d)\n", r.backed ? ((r.backed * 100.0) / r.total) : 0.0, r.backed);
  if (r.duplicates)
    fprintf(diag_output_fp, "\tDuplicates: %5.2f%% (%d)\n", r.duplicates ? ((r.duplicates * 100.0) / r.total) : 0.0, r.duplicates);
  if (r.stale)
    fprintf(diag_output_fp, "\tStale:      %5.2f%% (%d)\n", r.stale ? ((r.stale * 100.0) / r.total) : 0.0, r.stale);
  if (r.corrupt)
    fprintf(diag_output_fp, "\tCorrupt:    %5.2f%% (%d)\n", r.corrupt ? ((r.corrupt * 100.0) / r.total) : 0.0, r.corrupt);

  // 释放 old 对象占用的资源
  old.reset();

  return 0;
}
```

### int MultiCacheBase::check(const char *config_filename, bool fix = false)

当前的 MultiCache 对象已经完成初始化和加载的过程，调用 `check` 方法加载同一 MultiCache 的配置文件，可实现对 MultiCache 数据库完成性的检查，或通过修复（fix == true）操作删除不合法的数据元素（Entry）。返回值：

- 返回 0 表示成功
- 返回 < 0 表示失败

```
int
MultiCacheBase::check(const char *config_filename, bool fix)
{
  //  rebuild
  Store tStore;
  char t_db_filename[PATH_NAME_MAX];
  t_db_filename[0] = 0;
  int t_db_size = 0, t_db_buckets = 0;
  // 读取数据库配置信息
  // tStore 存储了空间分配信息，t_db_filename 存储了数据文件的名称，t_db_size 存储了记录数，t_db_buckets 存储了 Bucket 数量
  if (read_config(config_filename, tStore, t_db_filename, &t_db_size, &t_db_buckets) <= 0)
    return -1;

  // 传建一个临时的 MultiCache<C> 的继承类对象实例
  MultiCacheBase *old = dup();

  // 按照配置文件提供的空间分配信息和当前 MultiCache 数据库的信息初始化 old 实例
  if (old->initialize(&tStore, filename, nominal_elements, buckets) <= 0) {
    delete old;
    return -1;
  }

  // 调用 rebuild 方法，以 old 实例为数据源，检查或修复当前 MultiCache 数据库实例
  int res = rebuild(*old, fix ? MC_REBUILD_FIX : MC_REBUILD_CHECK);
  // 释放临时 MultiCache 对象 old
  delete old;
  // 返回 rebuild 的结果
  return res;
}
```

## 数据的持久化


### sync_heap(int part)

将与指定 Partition 关联的 HEAP 区域刷新到磁盘，被 sync_all 方法调用（但是方法的实现存在问题）

```
int
MultiCacheBase::sync_heap(int part)
{
  // 确认存在 HEAP 分区
  if (heap_size) {
    // 强制将 HEAP 区分为 64 段
    int b_per_part = heap_size / MULTI_CACHE_PARTITIONS;
    // 通过 part 计算出各个分段的偏移地址，将指定分段的数据刷写到磁盘，如遇失败返回 -1
    if (ats_msync(data + level_offset[2] + buckets * bucketsize[2] + b_per_part * part, b_per_part, data + totalsize, MS_SYNC) < 0)
      return -1;
  }
  // 成功返回 0
  return 0;
}
```

由于 HEAP 区是全局使用，因此 HEAP 区并不像数据区一样分成 64 个分区，因此强制将 HEAP 区平均分为 64 个分段，分多次刷新到磁盘并无实际意义。

### sync_partition(int partition)

将指定的 Partition 内的 BUCKET 刷新到磁盘，被 sync_all 方法调用。

```
//
// Sync a single partition
//
// Since we delete from the higher levels
// and insert into the lower levels,
// start with the higher levels to reduce the risk of duplicates.
//
// 由于向 MultiCache 中插入数据时，是从最顶层（第 0 层）入手，当顶层的空间用尽后，会将所有数据备份（插入）到下一层（第 1 层）。
// 当插入新数据时，首先在顶层空间查找空位置，如果没有空位置就在已经备份的 Entry 里找到命中率最低的那个覆盖掉，但是该数据已经备份到下一层，并不会消失。
// 在上述过程里，插入操作与备份操作是递归进行的，数据落入最底层（第 2 层）后，同时最底层也放满时，备份操作只是将最底层的数据标记为已备份，但是并不实际执行备份操作。
// 因此最底层的数据会不断的被覆盖掉，也可以将这种发生在最低层的覆盖理解为删除操作，因为一旦覆盖，这个数据就从 MultiCache 中完全的消失了。
// 由于数据是从最顶层插入，并在最底层删除，因此刷新操作要从最底层开始刷新，以减少数据的重复。例如：
//   - 先刷新第 0 层到磁盘，
//   - 然后第 0 层的数据被备份到了第 1 层，
//   - 接下来如果再刷新第 1 层的数据到磁盘，那么其中有很多数据就跟第 1 层是一样的。
// 反之：
//   - 先刷新第 2 层到磁盘，
//   - 然后第 1 层的数据被备份到了第 2 层，
//   - 再刷新第 1 层的数据到磁盘，那么并不会影响到已经刷新完成的第 2 层数据。
int
MultiCacheBase::sync_partition(int partition)
{
  int res = 0;
  int b = first_bucket_of_partition(partition);
  int n = buckets_of_partition(partition);
  // 刷新第 2 层的数据到磁盘（如果有第 2 层）
  // L3
  if (levels > 2) {
    if (ats_msync(data + level_offset[2] + b * bucketsize[2], n * bucketsize[2], data + totalsize, MS_SYNC) < 0)
      res = -1;
  }
  // L2
  // 刷新第 1 层的数据到磁盘（如果有第 2 层）
  if (levels > 1) {
    if (ats_msync(data + level_offset[1] + b * bucketsize[1], n * bucketsize[1], data + totalsize, MS_SYNC) < 0)
      res = -1;
  }
  // L1
  // 刷新第 0 层的数据到磁盘（如果有第 2 层）
  if (ats_msync(data + b * bucketsize[0], n * bucketsize[0], data + totalsize, MS_SYNC) < 0)
    res = -1;
  // 失败返回 -1，并设置 errno；成功返回 0
  return res;
}
```

### sync_header()

将 MultiCacheHeader 刷新到磁盘，被 sync_all 方法调用。

```
int
MultiCacheBase::sync_header()
{
  // 将当前内存对象的 MultiCache 的 Header 复制到 mapped_header 指向的内存地址，该地址通过 mmap 与磁盘文件建立了映射关系
  *mapped_header = *(MultiCacheHeader *)this;
  // 调用 msync 将该区域的数据刷新到磁盘，由于 MultiCacheHeader 的长度肯定小于 STORE_BLOCK_SIZE，因此长度就直接设置为 STORE_BLOCK_SIZE，便于对齐。
  // 如遇到失败 msync 返回 -1，并设置 errno；成功返回 0
  return ats_msync((char *)mapped_header, STORE_BLOCK_SIZE, (char *)mapped_header + STORE_BLOCK_SIZE, MS_SYNC);
}
```


### sync_all()

通过系统调用 msync 将所有 mmap 的内存区域的数据刷新到磁盘。但是该方法没有被任何功能调用，也许是一个未完成的设计。

```
int
MultiCacheBase::sync_all()
{
  int res = 0, i = 0;
  // 遍历 64 个分区，逐个分区调用 sync_heap 方法将 HEAP 区的数据刷新到磁盘，如遇到错误设置 res 为 -1
  for (i = 0; i < MULTI_CACHE_PARTITIONS; i++)
    if (sync_heap(i) < 0)
      res = -1;
  // 遍历 64 个分区，逐个分区调用 sync_partition 方法将数据区的数据刷新到磁盘，如遇到错误设置 res 为 -1
  for (i = 0; i < MULTI_CACHE_PARTITIONS; i++)
    if (sync_partition(i) < 0)
      res = -1;
  // 将 MultiCacheHeader 刷新到磁盘，如遇到错误设置 res 为 -1
  if (sync_header())
    res = -1;
  // 返回 res 值，成功为 0，错误为 -1
  return res;
}
```

### sync_partitions(Continuation *cont)

创建 MultiCacheHeapGC 来完成 HEAP 空间的碎片回收，或创建 MultiCacheSync 来实现将数据库数据同步到磁盘的功能。通过周期性重复调用该方法，即可实现 MultiCache 数据的持久化存储。

由于存在同步间隔，因此无法完全避免数据丢失，仍然会有可能丢失少量数据，降低同步的时间间隔可以减少丢失的数据量。

```
// MultiCacheHeapGC 或 MultiCacheSync 完成任务之后，会向 cont 回调 MULTI_CACHE_EVENT_SYNC 事件，表示操作完成，可以再次调用 sync_partitions 方法
void
MultiCacheBase::sync_partitions(Continuation *cont)
{
  // don't try to sync if we were not correctly initialized
  // 确认 MultiCache 已经初始化完成，并建立了与磁盘文件的内存映射关系
  if (data && mapped_header) {
    // HEAP 当前工作半区的空间利用率达到 80%（MULTI_CACHE_HEAP_HIGH_WATER 默认为 0.8）以上，触发 HeapGC 操作；
    // 否则执行 MultiCache 的整体数据同步操作。
    // MultiCacheHeapGC 和 MultiCacheSync 通过 HostDBSyncer::sync_event 周期性的交替执行。
    if (heap_used[heap_halfspace] > halfspace_size() * MULTI_CACHE_HEAP_HIGH_WATER)
      eventProcessor.schedule_imm(new MultiCacheHeapGC(cont, this), ET_CALL);
    else
      eventProcessor.schedule_imm(new MultiCacheSync(cont, this), ET_CALL);
  }
}
```

## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)
- [HostDB.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/HostDB.cc)




