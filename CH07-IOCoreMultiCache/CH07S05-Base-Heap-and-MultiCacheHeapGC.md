# 基础组件：Heap

MultiCache 中存储的数据元素，可能需要保存不定长的数据，对于 HostDB 来说，可能有：主机名字符串，或一个域名存在多个解析记录时。此时无法预估每个数据元素（Entry）的大小，也不能按照该类型数据的最大可能长度来设定数据元素（Entry）的大小，因为这将造成巨大的内存空间的浪费。为了解决该问题，在 MultiCache 中创建一个公共的存储区来存储这些不定长的数据，在 Entry 中记录不定长数据的存储位置即可。

### HEAP 空间的创建：

不同的 MultiCache 存储的数据元素也是多种多样的，对于不同类型的数据元素，其使用到的 HEAP 空间大小也是不一样的，开发人员可以通过对数据元素内各个成员占用的空间进行评估，给出每个 Entry 平均会在 HEAP 空间存储的数据大小，而该值乘以 Entry 的总数，就是 HEAP 空间的大小了。

开发人员需要定义 `estimated_heap_bytes_per_entry()` 方法，该方法返回平均每一个数据元素需要的不定长数据的存储空间。但是 HEAP 空间如果不足，则会出现灾难性的后果，因此要合理预估 HEAP 空间的大小，避免出现 HEAP 空间不足的情况。

```
// 计算 HEAP 区的大小，以字节为单位
MultiCacheBase::initialize(Store *astore, char *afilename, int aelements, int abuckets, unsigned int alevels,
                           int level0_elements_per_bucket, int level1_elements_per_bucket, int level2_elements_per_bucket)
{
  int64_t size = 0;
...
  heap_size = int((float)totalelements * estimated_heap_bytes_per_entry()); 
...
}

// 为 HEAP 区分配内存，
int
MultiCacheBase::mmap_data(bool private_flag, bool zero_fill)
{
...
    if (heap_size) {
      heap = cur;
      cur = mmap_region(bytes_to_blocks(heap_size), fds, cur, total_mapped, private_flag, fd);
      if (!cur) {
        store = saved;
        goto Labort;
      }
    }
...
}
```

### HEAP 空间的使用：

如果一个数据元素（Entry）需要保存一段不定长的数据，在它拿到数据后，就可以调用 `MultiCacheBase::alloc` 方法向 HEAP 区申请一段内存空间，然后将这段不定长的数据复制到这段内存空间。在调用 `alloc` 方法时，需要传入两个参数：

- 第一个是 `int *poffset`：用来记录这段空间的位置，以后想要访问存储在 HEAP 区的数据时，需要通过 `ptr` 方法获得指向这段空间的指针。
- 第二个是 `int asize`：用来声明需要申请的空间大小，以字节为单位，但是会按照 8 字节（MULTI_CACHE_HEAP_ALIGMENT）进行对齐。

返回值：

- 成功返回指向 HEAP 空间的指针
- 失败返回 NULL

可以看出每个数据元素（Entry）需要使用一个 `int` 类型的成员来保存第一个参数的值，为了得到该成员的地址，要求 MultiCache 数据库的设计者，必须参照 `MultiCacheBlock` 的定义，实现 `heap_offset_ptr` 方法，该方法返回 Entry 内 `int` 类型成员的地址，这样就可以通过以下方式在 HEAP 区分配内存，然后再复制数据到 HEAP 区存储：

```
EntryData *data = (EntryData *)demodb.alloc(entry->heap_offset_ptr(), size);
memcpy(data, src, size);
```

需要注意 `alloc` 方法只能为已经存在于 MultiCache 数据区的 Entry 分配 HEAP 区的空间，如果一个数据元素（Entry）还未插入到 MultiCache 数据区，那么是不可以调用 `alloc` 方法的。


```
                                         [heap_halfspace == 0]
                                                            |
    +-- MultiCacheBase::heap                                |
    |                                                       V
    |                         +-- MultiCacheBase::heap_used[0]  +-- MultiCacheBase::heap_used[1]
    |                         |                                 |
    V                         V                                 V
    +----------+-------------------------------------+----------+-------------------------------------+
    |   Skip   |   Half 0  #  |        Half 0        |   Skip   |               Half 1                |
    | (8bytes) |    Used   #  |         Rest         | (8bytes) |                Rest                 |
    +----------+-----------^-------------------------+----------+-------------------------------------+
                           |
                           | (void *)(heap + offset)
                           |                    +--------------------------------+
                      +----|---+----------+     |             Bucket X           |
                      | offset | *poffset |     +--------------------------------+
                      +--------+------|---+     | +------------+     +---------+ |
                      |     UnsunkPtr |   |     | | Entry M    | ... | Entry N | |
                      +---------------|---+     | +------------+     +---------+ |
                                      |         | | ...        |     |         | |
                                      +-----------> int offset |     | offset <--------+
                                                | | ...        |     |         | |     |
                                                | +------------+     +---------+ |     |
                                                +--------------------------------+     |
                                                                                       |
                                                                       +---------------|---+
                                                                       |     UnsunkPtr |   |
                                                                       +--------+------|---+
                                                                       | offset | *poffset |
                                                                       +----|---+----------+
                                                                            |
                                 (void *)(heap + halfspace_size() + offset) |
                                                                            |
    +----------+-------------------------------------+----------+-----------V-------------------------+
    |   Skip   |               Half 0                |   Skip   |   Half 1  #  |       Half 1         |
    | (8bytes) |                Rest                 | (8bytes) |    Used   #  |        Rest          |
    +----------+-------------------------------------+----------+-------------------------------------+
    ^          ^                                                            ^
    |          |                                                            |
    |          +-- MultiCacheBase::heap_used[0]                             +-- MultiCacheBase::heap_used[1]
    |                                                                                                     ^
    +-- MultiCacheBase::heap                                                                              |
                                                                                                          |
                                                                                       [heap_halfspace == 1]
```

HEAP 空间被平均分成两个半区（参考 MultiCacheHeapGC），每个区域的大小刚好是整个 HEAP 空间的一半，每次仅使用其中的一个半区。通过 `heap_halfspace` 来指明当前使用的是哪个半区（0 表示第一个半区，1 表示第二个半区），通过 `heap_used[0]` 和 `heap_used[1]` 分别保存第一、二两个半区已经使用的字节数，`heap_used[0]` 和 `heap_used[1]` 的默认值都是 8。每次在 HEAP 空间上进行分配时，使用原子操作递增 `heap_used[0]` 或 `heap_used[1]` 的值。每当半区切换之前，修改 `heap_halfspace` 的值，切换完成后，将上一个半区的 `heap_used` 值重置为 8，因此每个 HEAP 半区的前 8 个字节为保留空间，并未存储数据。

```
void *
MultiCacheBase::alloc(int *poffset, int asize)
{
  // 记录当前 HEAP 的工作半区
  int h = heap_halfspace;
  // 将 asize 按照 8 字节进行对齐，向上取整保存到 size
  int size = (asize + MULTI_CACHE_HEAP_ALIGNMENT - 1) & ~(MULTI_CACHE_HEAP_ALIGNMENT - 1);
  // 通过原子操作移动 heap_used[h] 计数器，返回值为移动之前的值，将作为所分配空间的起始偏移量
  // BUG：在执行原子操作之前，需要首先检测 poffset 的合法性，
  // 如果在空间分配完成之后才发现 poffset 不正确时，将会浪费这一段内存区域
  int o = ink_atomic_increment((int *)&heap_used[h], size);

  // 计算所分配的空间是否越过当前工作半区的边界
  if (o + size > halfspace_size()) {
    // 如果越过边界，则通过原子操作回退 heap_used[h]
    ink_atomic_increment((int *)&heap_used[h], -size);
    // 同时设置 int offset 为无效值 0，返回 NULL 表示分配失败
    ink_assert(!"out of space");
    if (poffset)
      *poffset = 0;
    return NULL;
  }
  // 分配成功，根据当前工作半区的情况，计算相对 MultiCacheBase::heap 基址的偏移量保存到 offset
  int offset = (h ? halfspace_size() : 0) + o;
  // 得到指向该空间的数据指针
  char *p = heap + offset;
  // 如果传入 poffset（只有内部调用会传入 NULL，通常都需要填入有效地址）
  if (poffset) {
    // poffset 指向数据区某个数据元素内的 int 类型成员，
    // 通过 ptr_to_partition 可以知道该数据元素位于哪个分区
    int part = ptr_to_partition((char *)poffset);
    if (part < 0)
      return NULL;
    // 从该分区的 UnsunkPtrRegistry 上分配一个 UnsunkPtr 对象，用于临时保存指向该 HEAP 区空间的记录
    UnsunkPtr *up = unsunk[part].alloc(poffset);
    // up->offset 的值是相对 MultiCacheBase::heap 基址的偏移量，已经考虑工作半区的影响
    up->offset = offset;
    // up->poffset 同样指向 Entry 内 int 类型成员的地址，但是分配 UnsunkPtr 成功后 *poffset 的值被设置为负数（参考：UnsunkPtr 章节）
    up->poffset = poffset;
    Debug("multicache", "alloc unsunk %d at %" PRId64 " part %d offset %d", *poffset, (int64_t)((char *)poffset - data), part,
          offset);
  }
  // 返回指向该 HEAP 区空间的指针
  return (void *)p;
}
```

上述 BUG 的修复补丁：

```
diff --git a/iocore/hostdb/MultiCache.cc b/iocore/hostdb/MultiCache.cc
index fcb03fe7d80..bf531052e93 100644
--- a/iocore/hostdb/MultiCache.cc
+++ b/iocore/hostdb/MultiCache.cc
@@ -1261,8 +1261,14 @@ MultiCacheBase::alloc(int *poffset, int asize)
 {
   int h    = heap_halfspace;
   int size = (asize + MULTI_CACHE_HEAP_ALIGNMENT - 1) & ~(MULTI_CACHE_HEAP_ALIGNMENT - 1);
-  int o    = ink_atomic_increment((int *)&heap_used[h], size);
 
+  int part = ptr_to_partition((char *)poffset);
+  if (poffset && part < 0) {
+    Debug("multicache", "poffset (%p) is invalid, valid range from %p to %p", poffset, data, heap - 1);
+    return NULL;
+  }
+
+  int o = ink_atomic_increment((int *)&heap_used[h], size);
   if (o + size > halfspace_size()) {
     ink_atomic_increment((int *)&heap_used[h], -size);
     ink_assert(!"out of space");
@@ -1270,12 +1276,10 @@ MultiCacheBase::alloc(int *poffset, int asize)
       *poffset = 0;
     return NULL;
   }
+
   int offset = (h ? halfspace_size() : 0) + o;
   char *p    = heap + offset;
   if (poffset) {
-    int part = ptr_to_partition((char *)poffset);
-    if (part < 0)
-      return NULL;
     UnsunkPtr *up = unsunk[part].alloc(poffset);
     up->offset    = offset;
     up->poffset   = poffset;
```

当需要获得存储在 HEAP 区的数据时，可以通过 `MultiCacheBase::ptr` 方法获得指向 HEAP 区的数据指针，需要注意，您不可以直接保存该指针，因为 HEAP 区的数据会在两个工作半区之间发生迁移，因此每次您需要访问某个 Entry 存储在 HEAP 区的数据时，都必须通过 `ptr` 方法获取数据指针后，立即访问该数据。访问该数据时，您同时持有该 Entry 所在的 Partition 的互斥锁，这保证了在访问该数据期间，该数据不受 HeapGC 的影响。在调用 `alloc` 方法时，需要传入两个参数：

- 第一个是 `int *poffset`：用来记录这段空间的位置，该指针指向数据区某个 Entry 内的 `int` 类型的成员
- 第二个是 `int partition`：指定 Entry 所在的分区（实际上可以通过 `ptr_to_partition` 方法获得，但是通常在调用该方法之前，已经取得了分区编号）

返回值：

- 成功返回指向 HEAP 空间的指针
- 失败返回 NULL

```
void *
MultiCacheBase::ptr(int *poffset, int partition)
{
  // 取出 Entry 内 int 类型成员的值
  int o = *poffset;
  Debug("multicache", "ptr %" PRId64 " part %d %d", (int64_t)((char *)poffset - data), partition, o);
  // 该值大于 0 表示相对 heap 基址的偏移量 + 1
  if (o > 0) {
    // 通过 valid_offset 验证该偏移值是否正确
    // BUG：严格来说，这里应该使用 o - 1
    // 验证失败则将 int 类型成员设置为 0 表示无效值，同时返回 NULL 表示失败
    if (!valid_offset(o)) {
      ink_assert(!"bad offset");
      *poffset = 0;
      return NULL;
    }
    // 验证通过，则返回指向该地址的指针
    return (void *)(heap + o - 1);
  }
  // 该值为 0 表示无效值，返回 NULL 表示错误
  if (!o)
    return NULL;
  // 该值小于 0 表示需要通过 UnsunkPtr 来查找，通过 -o - 1 转换为 UnsunkPtr 的编号，
  // 然后调用指定分区上的 UnsunkPtrRegistry::ptr 获得 UnsunkPtr 对象
  UnsunkPtr *p = unsunk[partition].ptr(-o - 1);
  // 获取 UnsunkPtr 对象失败，则返回 NULL
  // BUG：是否应该设置 *poffset = 0 ？或者设置一个 ink_assert ？
  if (!p)
    return NULL;
  // 验证 UnsunkPtr 内保存的数据与给定的 poffset 是对应的，否则返回 NULL 表示失败
  // 验证失败可能是在最近一次系统 crash 时，该 Entry 对象的 HEAP 区还未刷新到磁盘，仍然使用 UnsunkPtr 定位 HEAP 区的地址，
  // 在系统 crash 后，内存中的 UnsunkPtr 数据丢失，但是该 Entry 中仍然保留了指向 UnsunkPtr 的记录。
  if (p->poffset != poffset)
    return NULL;
  // 验证通过，使用 UnsunkPtr 内保存的 offset 值生成数据指针，并返回
  // 这里是否应该调用 valid_offset(p->offset) 做一次验证？
  return (void *)(heap + p->offset);
}
```

### HEAP 空间的管理

问题 1：

- 通过原子操作累加计数器实现的 HEAP 内存分配效率很高，但是一旦分配之后很难回收，
- 这些不定长数据可能会被修改，长度发生变化，数据元素也可能被释放，因此 HEAP 空间会慢慢碎片化，

HEAP 的空间回收和半区切换：

- 为解决碎片化和空间回收的问题，HEAP 空间被平分为两部分，每次只使用其中的一个半区，
- 当一个半区满了，就会立即切换到另外一个半区，每一个半区都有一个独立的使用情况计数器。
- 当从第一个半区切换到第二个半区后，HeapGC 会逐个分区遍历所有数据元素，把每一个数据元素使用的 HEAP 空间找出来，
- 如果该空间位于第一个 HEAP 半区里，就在第二个半区里申请一段新的 HEAP 空间，然后把数据复制过去，
- 最后把第一个半区的计数器重置为 8，每个半区头部的 8 个字节保留不使用。
- 通过数据的搬迁，可以把第一个半区里碎片化的数据顺序搬迁到第二个半区里，
- 这样数据就变得连续了，同时也回收了 HEAP 空间，但是缺点也很明显，增加了一倍的内存用量。


问题 2：

- 数据元素被写入到数据区之后，才可以在 HEAP 区分配内存，用来保存不定长数据，因此数据区和 HEAP 区同步到磁盘的顺序就存在先后关系，
- 如果数据区同步到磁盘之后，系统崩溃，HEAP 区还没来得及同步到磁盘，那么在重新启动之后，如何识别哪些数据元素的 HEAP 数据是损坏的呢？

HEAP 数据的重新定位：

- 在数据搬迁的过程里，数据的存储位置变了，那么如何让每个数据节点知道应该去哪里找到它们存储在 HEAP 区的不定长数据呢？
- 数据元素仍然保存在 Bucket 内，如果数据元素需要一个不定长的内存空间，则在其内包含一个 int 类型的成员，
- 通过在数据元素中保存一个 int 类型的成员，HEAP 去的数据可以通过两种方式进行寻址：
   - 当该 int 类型成员的值大于 0 时，该成员的值减去 1 用于表示所存储的数据相对 HEAP 基址的偏移
   - 当该 int 类型成员的值小于 0 时，该成员的值去掉负号后再减去 1 用于表示 UnsunkPtrRegistry 结构中 UnsunkPtr 数组的下标
- 通过下标，可以获得一个 UnsunkPtr 结构，该结构内有两个成员
   - int offset 用于表示所存储的数据相对 HEAP 基址的偏移，由于每个部分的前 8 个字节保留不使用，因此当该值为 0 时表示当前的 UnsunkPtr 是无效的。
   - int *poffset 是一个指向数据元素中 int 类型成员的指针。

HEAP 数据的持久化：

- Data 区域和 HEAP 区域的数据通过 mmap 映射到磁盘的文件，为避免在进程 crash 时丢失数据，需要周期性的将数据刷新到磁盘。
- 而 UnsunkPtrRegistry 和 UnsunkPtr 都是内存中的数据结构，在进程 crash 时就会丢失，
- 因此在周期性刷新数据到磁盘之前，需要把 UnsunkPtr 里的 offset 值刷写入数据元素的 int 类型的成员里，
- 此时该 int 类型的成员作为数据元素的一部分，随 Data 区域被周期性刷新到磁盘，在进程重启后，
- 如果该 int 类型成员的值大于 0，减去 1 之后就可以通过 HEAP 基址定位到对应的不定长数据。
- 如果该 int 类型成员的值小于 0，说明该成员在 crash 之前，其 HEAP 区的数据还未刷新到磁盘，因此其 HEAP 区的数据损坏或丢失了。

在 MultiCacheHeapGC 和 MultiCacheSync 中都会触发 HEAP 区的数据向磁盘同步的操作。（详见：MultiCacheHeapGC 和 MultiCacheSync 的分析）

## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)


# 基础组件：MultiCacheHeapGC

`MultiCacheHeapGC` 状态机由 `MultiCacheBase::sync_partitions` 方法创建并激活。

```
void
MultiCacheBase::sync_partitions(Continuation *cont)
{
  if (data && mapped_header) {
    // HEAP 当前工作半区的空间利用率达到 80%（MULTI_CACHE_HEAP_HIGH_WATER 默认为 0.8）以上，触发 HeapGC 操作；
    if (heap_used[heap_halfspace] > halfspace_size() * MULTI_CACHE_HEAP_HIGH_WATER)
      // 此处应该放入 ET_TASK 线程，应该会更合适一些，不会阻塞 ET_CALL / ET_NET 线程组
      eventProcessor.schedule_imm(new MultiCacheHeapGC(cont, this), ET_CALL);
    else
...
}
```

`MultiCacheHeapGC` 状态机负责将 HEAP 数据从一个工作半区切换到另一个工作半区，然后遍历所有 Entry 对象，将位于旧工作半区的 HEAP 数据复制到新工作半区，并修正 Entry 对象内记录 HEAP 存储位置的变量。

## 定义

```
struct OffsetTable {
  int new_offset;
  int *poffset;
};

struct MultiCacheHeapGC : public Continuation {
  Continuation *cont;
  MultiCacheBase *mc;
  int partition;
  int n_offsets;
  OffsetTable *offset_table;

  // 由 Event System 回调，首次回调事件为 EVENT_IMMEDIATE，后续事件为 EVENT_INTERVAL
  int
  startEvent(int event, Event *e)
  {
    (void)event;
    // 逐个分区遍历，首次进入 partition 为 0
    if (partition < MULTI_CACHE_PARTITIONS) {
      // copy heap data

      // BUG：此处计算 before 和 after 指针位置的时候，没有把工作半区的偏移量考虑进去
      // 如果当前是第二个半区，那么 before 和 after 的指针的地址就计算错了。
      // 记录开始数据复制之前新 HEAP 半区数据的最后写入位置
      char *before = mc->heap + mc->heap_used[mc->heap_halfspace];
      // 逐层遍历该分区所有的数据元素，并将 HEAP 数据从旧半区复制到新半区
      mc->copy_heap(partition, this);
      // 记录完成数据复制之后新 HEAP 半区数据的最后写入位置
      char *after = mc->heap + mc->heap_used[mc->heap_halfspace];

      // sync new heap data and header (used)
      // 两次位置相减的值大于 0，说明有部分数据从旧半区复制到了新半区，
      // 但是也有可能是有其它分区的 Entry 向 HEAP 区写入了数据
      if (after - before > 0) {
        // 如果发生了 HEAP 区的数据写入，则需要将新写入到 HEAP 区的数据立即同步到磁盘（先将数据落地，在修改 Entry 内保存的 offset 值）
        // BUG：这里不应该是 mc->heap + mc->totalsize 应该是 mc->heap_size
        ink_assert(!ats_msync(before, after - before, mc->heap + mc->totalsize, MS_SYNC));
        // 问题：此处并未执行 *mc->maped_header = mc->header_snap 或 *mc->maped_header = *(MultiCacheHeader *) mc 的操作，
        // 因此 mc->mapped_header 的内容并未发生改变，因此同步到磁盘的内容未产生变化。
        // 猜测应该：
        //  - 在 copy_heap 调用之后立即执行 mc->header_snap = *(MultiCacheHeader *)mc;
        //  - 然后在此处执行 *mc->maped_header = mc->header_snap;
        ink_assert(!ats_msync((char *)mc->mapped_header, STORE_BLOCK_SIZE, (char *)mc->mapped_header + STORE_BLOCK_SIZE, MS_SYNC));
      }
      // update table to point to new entries
      // 将 offset_table 里保存的信息更新到 Entry 里，既将 Entry 里指向旧半区的 offset 更新为指向新半区的 offset，
      // 因为现在同一份数据同时在旧半区和新半区都存在，所以这个数据无论是否更新，都不会影响数据完整性，即使 crash 也没关系。
      for (int i = 0; i < n_offsets; i++) {
        int *i1, i2;
        // BAD CODE GENERATION ON THE ALPHA
        //*(offset_table[i].poffset) = offset_table[i].new_offset + 1;
        i1 = offset_table[i].poffset;
        i2 = offset_table[i].new_offset + 1;
        *i1 = i2;
      }
      n_offsets = 0;
      // 将修改之后的 Entry 数据同步到磁盘，此时该分区的所有 HEAP 数据都从旧半区切换到了新半区，并且都将数据刷新到了磁盘。
      mc->sync_partition(partition);
      // 准备进入下一个分区
      partition++;
      // 将 HeapGC 状态机的互斥锁与下一个分区的互斥锁共享，
      // 在返回到事件系统之前，HeapGC 状态机原来的互斥锁一直保持上锁状态
      if (partition < MULTI_CACHE_PARTITIONS)
        mutex = mc->locks[partition];
      else
      // 如果所有的分区都处理完毕，那么就将HeapGC 状态机的互斥锁与回调的状态机的互斥锁共享
        mutex = cont->mutex;
      // 间隔一定的时间处理下一个分区的数据，避免连续工作导致磁盘 I/O 负载过高。
      // BUG：联合 HostDBSyncer 的代码，数据同步间隔时间的计算步骤，应该不符合 HostDBSyner 要求
      e->schedule_in(MAX(MC_SYNC_MIN_PAUSE_TIME, HRTIME_SECONDS(hostdb_sync_frequency - 5) / MULTI_CACHE_PARTITIONS));
      return EVENT_CONT;
    }
    // 如 partition == 64 则表示已经完成所有分区的遍历操作
    // 将旧 HEAP 半区的计数器重置为 8
    mc->heap_used[mc->heap_halfspace ? 0 : 1] = 8; // skip 0
    // 向状态机回调 MULTI_CACHE_EVENT_SYNC 事件
    cont->handleEvent(MULTI_CACHE_EVENT_SYNC, 0);
    Debug("multicache", "MultiCacheHeapGC done");
    // 释放 HeapGC 状态机及其资源
    delete this;
    return EVENT_DONE;
  }

  // 构造函数
  // 当 HeapGC 执行完成后，会向 acont 回调 MULTI_CACHE_EVENT_SYNC 事件，附加 data 为 NULL
  // 初始化 mc，指向需要进行 HeapGC 操作，同时设置 HeapGC 状态机的锁与 Partition 0 的互斥锁共享
  MultiCacheHeapGC(Continuation *acont, MultiCacheBase *amc)
    : Continuation(amc->locks[0]), cont(acont), mc(amc), partition(0), n_offsets(0)
  {
    // 设置回调函数为 startEvent，将会由 EThread 回调
    SET_HANDLER((MCacheHeapGCHandler)&MultiCacheHeapGC::startEvent);
    // 创建 offset_table，n_offsets 表示已经使用的数量（稍后分析）
    // 需要能够容纳：一个分区的数据元素总数 + 最底层一个 Bucket 内数据元素个数的三倍 + 1 个 OffsetTable 项，因为：
    //  - 每一轮需要遍历一个分区所有的数据元素：
    //     - 如果一个分区所有数据元素的 HEAP 数据都不需要借助 UnsunkPtr 进行查找，那么就需要为该分区的每一个数据元素创建一个 OffsetTable 项
    //     - 通过 (mc->totalelements / MULTI_CACHE_PARTITIONS) 可以得到平均每个分区有多少个数据元素
    //     - 除法操作会向下取整，每个分区的数据元素数量是该分区 bucket 数量的整数倍，因此需要修正计算误差，以避免 OffsetTable 项不足
    //  - 误差修正：除法向下取整的小数部分
    //     - 直接通过 +1 修正
    //  - 误差修正：每个分区所包含的 Bucket 数量最多会相差 1
    //     - 可以额外给各数据层补一个 Bucket 的数据元素数量（最底层/最外层的 Bucket 容量最大），最多会有三层
    //     - 通过 mc->elements[mc->levels - 1] * 3 可以得到最底层一个 Bucket 内数据元素个数的三倍
    // 这里实际上使用 ((mc->buckets / MULTI_CACHE_PARTITIONS) + 1) * mc->elements[mc->levels - 1] * mc->levels 即可，因为
    //  - Bucket 是平均分到 64 个分区，每个分区所包含的 Bucket 数量最多会相差 1 个
    //  - 首先计算平均每个分区有多少个 bucket：(mc->buckets / MULTI_CACHE_PARTITIONS)
    //  - 然后补上可能会多出来的一个 bucket：+1
    //  - 然后乘上最底层一个 Bucket 内数据元素个数：mc->elements[mc->levels - 1]
    //  - 最后再乘上总层数：mc->levels
    // 这样就可以得到一个分区最多可能会有多少个数据元素，最差情况下，每一个数据元素均需要一个 OffsetTable 项。
    offset_table = (OffsetTable *)ats_malloc(sizeof(OffsetTable) *
                                             ((mc->totalelements / MULTI_CACHE_PARTITIONS) + mc->elements[mc->levels - 1] * 3 + 1));
    // flip halfspaces
    mutex = mc->locks[partition];
    // 切换 HEAP 工作半区
    mc->heap_halfspace = mc->heap_halfspace ? 0 : 1;
  }
  // 析构函数，释放 offset_table
  ~MultiCacheHeapGC() { ats_free(offset_table); }
};
```

上面提到的 BUG，可以使用以下补丁修复：

```
diff --git a/iocore/hostdb/MultiCache.cc b/iocore/hostdb/MultiCache.cc
index bf531052e93..8bec81f0a8a 100644
--- a/iocore/hostdb/MultiCache.cc
+++ b/iocore/hostdb/MultiCache.cc
@@ -1121,14 +1121,14 @@ struct MultiCacheHeapGC : public Continuation {
     if (partition < MULTI_CACHE_PARTITIONS) {
       // copy heap data
 
-      char *before = mc->heap + mc->heap_used[mc->heap_halfspace];
+      char *before = mc->heap + mc->heap_halfspace * mc->halfspace_size() + mc->heap_used[mc->heap_halfspace];
       mc->copy_heap(partition, this);
-      char *after = mc->heap + mc->heap_used[mc->heap_halfspace];
+      char *after = mc->heap + mc->heap_halfspace * mc->halfspace_size() + mc->heap_used[mc->heap_halfspace];
 
       // sync new heap data and header (used)
 
       if (after - before > 0) {
-        ink_assert(!ats_msync(before, after - before, mc->heap + mc->totalsize, MS_SYNC));
+        ink_assert(!ats_msync(before, after - before, mc->heap + mc->heap_size, MS_SYNC));
         ink_assert(!ats_msync((char *)mc->mapped_header, STORE_BLOCK_SIZE, (char *)mc->mapped_header + STORE_BLOCK_SIZE, MS_SYNC));
       }
       // update table to point to new entries
```

## 方法

### void MultiCacheBase::copy\_heap\_data(char *src, int s, ...)

`copy_heap_data` 方法用于将指定的数据从 HEAP 的旧半区复制到新半区，由 `copy_heap` 方法调用。由于 HEAP 区内的数据有两种：

- 第一种是最新存入的数据，从未同步到磁盘，Entry 通过 UnsunkPtr 指针定位该数据存储在 HEAP 区的位置
- 第二种是之前存入的数据，已经同步到磁盘，Entry 内的 int 类型的成员直接记录了该数据在 HEAP 区的位置

在 HEAP 半区进行切换时，对于第一种数据只需要立即更改 UnsunkPtr 指针的记录就可以了，因为其数据本身就是易失数据；对于第二种数据需要先将 HEAP 区的数据复制到新的半区，并且同步到磁盘只后，才能实施 Entry 内 int 类型成员的变更，并同步到磁盘，因此对于第二种数据，对 Entry 内 int 类型成员的变更记录需要暂存在 `gc->offset_table[x]` 数组里。

参数：

- char *src：指向原始数据所在的地址
- int s：原始数据的长度，以字节为单位
- int *pi：指向 Entry 对象内 int 类型的成员，通过该成员保存的数据，可以得到 HEAP 区保存数据内容的地址
- int partition：原始数据所在的分区
- MultiCacheHeapGC *gc：指向调用该方法的 MultiCacheHeapGC 状态机，用于访问 offset_table 数组


```
void
MultiCacheBase::copy_heap_data(char *src, int s, int *pi, int partition, MultiCacheHeapGC *gc)
{
  // 直接在新 HEAP 半区分配指定长度的空间，但是不创建 UnsunkPtr 对象
  char *dest = (char *)alloc(NULL, s);
  Debug("multicache", "copy %p to %p", src, dest);
  if (dest) {
    // 空间分配成功后，就将数据从旧半区复制到新半区
    memcpy(dest, src, s);
    // 判断该 Entry 的数据是哪一种？
    if (*pi < 0) { // already in the unsunk ptr registry, ok to change there
      // 第一种，需要更新 UnsunkPtr 的记录
      UnsunkPtr *ptr = unsunk[partition].ptr(-*pi - 1);
      // 更新时需要验证 UnsunkPtr 的记录与 Entry 的对应关系
      if (ptr->poffset == pi)
        ptr->offset = dest - heap;
      else {
        // 验证失败触发 assert 并将数据清空，但是 HEAP 区的空间不会回退
        ink_assert(0);
        *pi = 0;
      }
    } else {
      // 第二种，需要将更新暂存于 `gc->offset_table[x]` 数组，稍后进行更新
      gc->offset_table[gc->n_offsets].new_offset = dest - heap;
      gc->offset_table[gc->n_offsets].poffset = pi;
      gc->n_offsets++;
    }
  } else {
    // 空间分配失败触发 assert 并将数据清空
    ink_assert(0);
    *pi = 0;
  }
}
```

### void MultiCache<C>::copy_heap(int partition, MultiCacheHeapGC *gc)

`copy_heap` 方法逐层遍历指定分区所有的数据元素，通过调用 `copy_heap_data` 方法将 HEAP 数据从旧半区复制到新半区。

参数：

- int partition：指定分区
- MultiCacheHeapGC *gc：指向调用该方法的 MultiCacheHeapGC 状态机，用于访问 offset_table 数组

```
template <class C>
inline void
MultiCache<C>::copy_heap(int partition, MultiCacheHeapGC *gc)
{
  // 取得指定分区第一个 bucket 的编号
  int b = first_bucket_of_partition(partition);
  // 取得指定分区有多少个 bucket
  int n = buckets_of_partition(partition);
  // 从最底层开始逐层遍历
  for (unsigned int level = 0; level < levels; level++) {
    // 当前分区的 bucket 数量 x 该层每个 bucket 的元素数量 = 当前分区在指定的层有多少个数据元素（Entry）
    int e = n * elements[level];
    // 计算出第一个数据元素的地址
    char *d = data + level_offset[level] + b * bucketsize[level];
    // 转换指针类型，这样可以方便使用 x[i] 的方式遍历数据元素
    C *x = (C *)d;
    // 遍历该分区在当前层的所有数据元素
    for (int i = 0; i < e; i++) {
      // 取得指定元素在 HEAP 空间存储的数据大小，大于 0 表示在 HEAP 空间存储了数据
      int s = x[i].heap_size();
      if (s) {
        // 如果有数据，就先获取 Entry 内的 int 类型成员的地址
        int *pi = x[i].heap_offset_ptr();
        if (pi) {
          // 通过 ptr 方法将 pi 转换为指向 HEAP 数据的指针，src 是指向存储于旧 HEAP 半区的指针
          char *src = (char *)ptr(pi, partition);
          if (src) {
            // 判断当前工作的半区：
            if (heap_halfspace) {
              // 如果当前工作半区是第二半区，则 src 应该指向第一半区（旧半区），
              // 如果 src 指向第二半区就不处理，跳过
              if (src >= heap + halfspace_size())
                continue;
            } else if (src < heap + halfspace_size())
              // 如果当前工作半区是第一半区，则 src 应该指向第二半区（旧半区），
              // 如果 src 指向第一半区就不处理，跳过
              continue;
            // 剩下的就应该是需要进行处理的数据，调用 copy_heap_data 将数据从就半区复制到新半区
            copy_heap_data(src, s, pi, partition, gc);
          }
          // 这里应该有个 else 的判断，触发 assert，指针转换失败应该是数据出错了
        }
        // 这里应该有个 else 的判断，触发 assert，
        // 毕竟数据元素说它在 HEAP 存了数据，但是又不提供 Entry 内 int 类型成员的地址，这绝对是一种异常
      }
    }
  }
}
```

## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)
- [MultiCache.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/MultiCache.cc)
