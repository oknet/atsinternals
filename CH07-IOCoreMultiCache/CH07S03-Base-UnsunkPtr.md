# 基础组件：UnsunkPtr 和 UnsunkPtrRegistry

HEAP 区的数据定时通过 msync 系统调用周期性同步刷新到磁盘，以实现数据的持久化保存。

但是，在两次同步刷新操作之间，会有一些新数据还来不及刷新到磁盘。因此，在系统意外崩溃时，会有一小部分 Entry 对象存储在 HEAP 区的数据将会丢失，那么在系统重启后，进行崩溃恢复时，就要删除这些 Entry，避免加载错误的数据。

通过将数据在 HEAP 区的偏移量保存在不同的位置，来表示每个 Entry 存储在 HEAP 区的数据刷新状态：

- 未被刷新到磁盘的 HEAP 数据，其偏移量保存在 UnsunkPtr，并在 Entry 中保存 UnsunkPtr 的编号（值为负数），
- 已经刷新到磁盘的 HEAP 数据，其偏移量直接保存在 Entry 中（值为正数）。

而每个 Partition 有一个 UnsunkPtrRegistry 对象来管理 UnsunkPtr，通过编号可以从 UnsunkPtrRegistry 取得对应的 UnsunkPtr 对象，从而进一步得到 HEAP 区数据的存储位置。

UnsunkPtr 字面意思可理解为“未沉没的指针”，意指漂浮在内存中的指针，属于易失性数据。而将 HEAP 区的数据刷新到磁盘后，便使 UnsunkPtr “沉没”，其内存储的偏移量信息则保存到 Entry 中，而 Entry 所在的内存区域通过 mmap 建立了与磁盘文件的映射关系，会随刷新操作保存（“下沉”）到磁盘，变成不易丢失的数据。

## 定义

```
struct UnsunkPtr {
  // 指向 HEAP 区域的偏移量
  int offset;
  // 在定义 MultiCache<HostDBInfo> 时，HostDBInfo 内的成员 hostname_offset 的地址会被传入给 poffset
  // 当 *poffset 的值小于 0 时，表示 UnsunkPtrRegistry[x] 中的第几个 UnsunkPtr 对象
  //   - 需要使用公式转换：-(*poffset)-1
  // 当 *poffset 的值大于 0 时，表示其数据存储在 HEAP 区域的偏移量
  // 当 *poffset 的值为 0 时，表示无效
  // 当 UnsunkPtr 未使用时，作为 freelist 指针指向下一个未使用的 UnsunkPtr 对象。
  int *poffset; // doubles as freelist pointer
};

struct UnsunkPtrRegistry {
  // 指向 MultiCacheBase 对象
  // 在 MultiCacheBase 的构造函数中，完成对每个分区的第一个 UnsunkPtrRegistry 对象内 mc 成员的初始化。
  // 另外，在 UnsunkPtrRegistry::alloc() 为同一分区创建第二个或之后的 UnsunkPtrRegistry 对象时，
  // 会使用每个分区第一个 UnsunkPtrRegistry 对象的 mc 为其赋值。
  MultiCacheBase *mc;
  // 用于表示 ptrs 内有几个 UnsunkPtr 对象
  int n;
  // 指向一个连续保存 UnsunkPtr 对象的地址空间
  UnsunkPtr *ptrs;
  // 指向在 ptrs 地址空间内第一个可用的 UnsunkPtr 对象，在 UnsunkPtrRegistry::alloc_data() 方法中完成初始化
  // 把 next_free 看做是一个 freelist 的头指针，指向第一个可用的 UnsunkPtr 对象，
  // 然后其成员 poffset 指向下一个可用的 UnsunkPtr 对象，最后一个 poffset 指向 NULL
  UnsunkPtr *next_free;
  // 当 ptrs 内的 UnsunkPtr 对象用尽之后，通过 new UnsunkPtrRegistry 创建新的 UnsunkPtrRegistry 对象
  // 使用 next 成员指针指向下一个 UnsunkPtrRegistry 对象，可根据需要扩展更多
  UnsunkPtrRegistry *next;

  // 以每个分区内的第一个 UnsunkPtrRegistry 开始，获取其内保存的第 i 个 UnsunkPtr 对象，
  // 如果 next 指向扩展的 UnsunkPtrRegistry 对象，则会逐个遍历 next 链表。
  UnsunkPtr *ptr(int i);
  // 从 UnsunkPtrRegistry 内获得一个可用的 UnsunkPtr 对象
  UnsunkPtr *alloc(int *p, int base = 0);
  // 用于完成 UnsunkPtrRegistry 对象内以下成员的初始化：
  //   - ptrs
  //   - next_free
  //   - n
  void alloc_data();

  // 构造函数
  // 将成员赋值为 0 或指向 NULL
  UnsunkPtrRegistry();
  // 析构函数
  // 释放 ptrs 指向的内存区域
  ~UnsunkPtrRegistry();
};
```

## 方法

```
// size of block of unsunk pointers with respect to the number of
// elements
#define MULTI_CACHE_UNSUNK_PTR_BLOCK_SIZE(_e) ((_e / 8) / MULTI_CACHE_PARTITIONS)
```

### UnsunkPtrRegistry::alloc_data()

初始化 UnsunkPtrRegistry 对象的成员，并为 UnsunkPtr 对象分配内存。主要被 `UnsunkPtrRegistry::alloc` 方法调用。

```
void
UnsunkPtrRegistry::alloc_data()
{
  // 每个元素都对应一个 UnsunkPtr 对象，通过一个 UnsunkPtrRegistry 管理多个 UnsunkPtr 对象
  // 元素总数均分到 64 个分区，每个分区内的元组再分成 8 个组，
  // 每个小组的元素对应的 UnsunkPtr 对象，由一个 UnsunkPtrRegistry 对象管理
  int bs = MULTI_CACHE_UNSUNK_PTR_BLOCK_SIZE(mc->totalelements);
  // 为该小组的 UnsunkPtr 对象，创建内存空间
  size_t s = bs * sizeof(UnsunkPtr);
  ptrs = (UnsunkPtr *)ats_malloc(s);
  // 初始化该小组内的 UnsunkPtr 对象，在 UnsunkPtr 对象未被使用之前，使用 poffset 指针指向下一个空闲的 UnsunkPtr 对象
  for (int i = 0; i < bs; i++) {
    ptrs[i].offset = 0;
    ptrs[i].poffset = (int *)&ptrs[i + 1];
  }
  // 最后一个空闲的 UnsunkPtr 对象的 poffset 指向 NULL
  ptrs[bs - 1].poffset = NULL;
  // next_free 指针指向第一个空闲的 UnsunkPtr 对象
  next_free = ptrs;
  // 小组内 UnsunkPtr 对象的数量
  n = bs;
}
```

### UnsunkPtrRegistry::alloc(int *poffset, int base)

从每个分区的 UnsunkPtrRegistry 池里获得一个可用的 UnsunkPtr 对象，如果没有剩余可用的 UnsunkPtr，则调用 `alloc_data` 方法追加一个 UnsunkPtrRegistry 到链表上。

参数 base 的默认值为 0，表示 UnsunkPtr 对象的编号在当前 UnsunkPtrRegistry 池里的起始值。用于支持在多个 UnsunkPtrRegistry 构成的链表上进行分配。

```
UnsunkPtr *
UnsunkPtrRegistry::alloc(int *poffset, int base)
{
  // 如果当前小组有空闲的 UnsunkPtr 对象可用
  if (next_free) {
    // 从 freelist 上获取一个空闲的 UnsunkPtr 对象
    UnsunkPtr *res = next_free;
    // 让 freelist 指向下一个空调的 UnsunkPtr 对象
    next_free = (UnsunkPtr *)next_free->poffset;
    // 将 UnsunkPtr 对象的编号存入 '*poffset'
    // 这里使用小于 0 的数字表示 UnsunkPtr 对象在每一个分区的 UnsunkPtrRegistry 池里的编号
    // 由于 '*poffset' 不能为 0，因此通过以下转换让其总是小于 0
    //   - 第一个 UnsunkPtr 对象，其值为 -1
    //   - 第二个 UnsunkPtr 对象，其值为 -2
    //   - 以此类推
    //   - 可以在 MultiCacheBase::ptr() 方法里看到其还原逻辑
    *poffset = -(base + (res - ptrs)) - 1;
    ink_assert(*poffset);
    // 返回空闲的 UnsunkPtr 对象
    return res;
  } else {
    // 当前小组没有空闲的 UnsunkPtr 对象可用
    // 检查 UnsunkPtr 对象池是否已经初始化
    if (!ptrs) {
      // 初始化当前 UnsunkPtrRegistry 对象
      alloc_data();
      // 然后重新调用 alloc 方法
      return alloc(poffset, base);
    }
    // 当前 UnsunkPtrRegistry 池里的 UnsunkPtr 对象都被分配出去了
    // 是否还有下一个可用的 UnsunkPtrRegistry 池？
    if (!next) {
      // 如果没有就创建一个
      next = new UnsunkPtrRegistry;
      // 传递 mc 到新建的 UnsunkPtrRegistry 池
      next->mc = mc;
    }
    // 尝试在下一个 UnsunkPtrRegistry 池里寻找空闲的 UnsunkPtr 对象
    int s = MULTI_CACHE_UNSUNK_PTR_BLOCK_SIZE(mc->totalelements);
    // 这里使用 base + s 表示下一个 UnsunkPtrRegistry 池里的 UnsunkPtr 对象的编号是从 base + s 开始的
    // 感觉这里使用成员 n 就可以了，没必要使用 mc->totalelements 重新计算 s
    return next->alloc(poffset, base + s);
  }
}
```

### UnsunkPtrRegistry::ptr(int i)

使用 UnsunkPtr 对象的编号获得 UnsunkPtr 对象的指针。

输入的参数 i，其值已经进行了转换，大于等于 0，但是代码中并未做强制性判断。

```
UnsunkPtr *
UnsunkPtrRegistry::ptr(int i)
{
  // 此处应当首先判断 i >= 0
  // 如果编号超过当前 UnsunkPtrRegistry 池里可容纳的最大 UnsunkPtr 对象数量，
  // 则继续查找下一个 UnsunkPtrRegistry 池
  if (i >= n) {
    if (!next)
      return NULL;
    else
      return next->ptr(i - n);
  } else {
    // 否则从当前 UnsunkPtrRegistry 池中直接取出 UnsunkPtr 对象的指针
    if (!ptrs)
      return NULL;
    return &ptrs[i];
  }
}
```

## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)

