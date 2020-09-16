# 核心组件：MultiCache

MultiCache 是一个模板，其底层又定义了 MultiCacheBase 和 MultiCacheHeader 两个基类。

由于需要实现 HostDB 存储的持久化，需要将内存里的内容不间断的定期保存到磁盘，并且在 ATS 进程意外崩溃时，仍然能确保数据记录的完整性，ATS 设计了 MultiCache 作为数据文件的基础结构。

在向 MultiCache 内存入数据时，是从最顶层（第 0 层）开始存入数据，如果存满了，就把数据备份到下一层（第 1 层）同一个 Bucket 里，然后把第 0 层查询命中率较低的空间让出来，用于存储新数据元素。同样的，第 1 层满了也会向第 2 层备份，但是最底层满了的话，则无法备份数据，直接把命中率较低的数据元素丢弃，空间让给新的数据元素。这个备份的过程是逐层递归的将数据元素在同一个 Bucket 内，从最高层推向最底层的，不同 Bucket 的使用情况各不相同。

这个数据插入的过程就像是往一个最高为 3 层的蛋糕上倒蜂蜜：

- 蜂蜜首先流淌到最顶层，然后会慢慢的流淌到下一层，
- 最终流淌到最底层，可能还会继续流淌到桌面上，桌面上的蜂蜜就是溢出而丢弃的数据元素，
- 但是蜂蜜在各个方位上的覆盖情况却也不完全相同，而谁也不能保证蜂蜜能够把蛋糕完全覆盖。

MultiCache 使用 `lowest_level_data` 结构来记录每一个 Bucket 数据备份的深度，以此来了解数据在各个 Bucket 的分布情况。

## 定义

```
template <class C> struct MultiCache : public MultiCacheBase {
  // 返回 MultiCache 内单个元素的大小，以字节为单位
  int
  get_elementsize()
  {
    return sizeof(C);
  }

  // 创建相同类型的 MultiCache<C> 对象
  MultiCacheBase *
  dup()
  {
    return new MultiCache<C>;
  }

  // 由 MultiCacheBase::rebuild 方法调用，在判断 elem 不是空白内容后，将其插入到当前 MultiCache 数据库
  void rebuild_element(int buck, char *elem, RebuildMC &r);
  // -1 is corrupt, 0 == void (do not insert), 1 is OK
  // 用于在业务层判断指定的数据元素 C *c 的完整性和有效性，需要在 MultiCache<C> 的继承类实现该方法，
  // 由 rebuild_element 方法在调用 insert_block 方法插入数据之前调用，用于避免将损坏、无用的数据插入到数据库中
  // 返回 -1 表示数据损坏，
  // 返回 0 表示该数据元素虽然没有损坏，但是已经没有用处了（例如过期的陈旧数据）
  // 返回 1 表示该数据元素是完整有效的数据
  virtual int
  rebuild_callout(C *c, RebuildMC &r)
  {
    (void)c;
    (void)r;
    return 1;
  }

  // 用于在插入数据完成之后进行后续处理工作，例如建立 HEAP 区的数据
  // 由 rebuild_element 方法在调用 insert_block 方法之后调用，没有看到代码中有该方法的定义
  // 猜测是用来复制 HEAP 区数据，因为 insert_block 方法只能复制 Entry 的内容，没法复制 HEAP 区的数据，
  // 另外在 Entry 建立之后，才能申请 HEAP 区的空间，然后复制数据到 HEAP 区。
  virtual void
  rebuild_insert_callout(C *c, RebuildMC &r)
  {
    (void)c;
    (void)r;
  }

  //
  // template operations
  //
  // 返回指定的数据元素（Entry）位于 MultiCache 的层级
  int level_of_block(C *b);
  // 判断指定的数据元素（Entry）是否与 hash 值匹配
  bool match(uint64_t folded_md5, C *block);
  // 根据 hash 值，在指定的 MultiCache 层级找到 bucket，并返回 bucket 中的第一个数据元素（Entry）
  C *cache_bucket(uint64_t folded_md5, unsigned int level);
  // 将数据元素插入到 MultiCache 数据库指定的层级，并返回该数据元素所在的 Entry
  C *insert_block(uint64_t folded_md5, C *new_block, unsigned int level);
  // 将位于 level 层级，指定 bucket 内的数据元素全部备份（重新插入）到下一级
  void flush(C *b, int bucket, unsigned int level);
  // 将指定的 block 所在的 Entry 清空，如果该 block 已经备份到下一级，
  // 则遍历 MultiCache 数据库，将备份有该 block 内容的 Entry 也全部清空
  void delete_block(C *block);
  // 从最顶层开始遍历 MultiCache 数据库，最多遍历 level 层，查找与指定 hash 值匹配的第一个数据元素
  C *lookup_block(uint64_t folded_md5, unsigned int level);
  // 在两个工作半区之间复制 HEAP 区的数据，具体参考 Heap and MultiCacheHeapGC 章节
  void copy_heap(int paritition, MultiCacheHeapGC *);
};
```






   - MultiCache 内的对象使用 Hash 值进行索引和访问，最初版本采用 MD5 算法计算 Hash 值
   - 可以通过 Hash 值直接定位到 Bucket，当 Hash 值出现冲突时，使用链表法解决冲突
   - 因此每个 Bucket 的第一个 Entry 是该链表的表头
   - 在获取空白 Entry 对象时，首先从 Bucket 内部逐个获取
   - 当一个 Bucket 内部的空白 Entry 耗尽时，会从该层级的全局空白 Entry 列表中获取
   - 全局空白 Entry 列表是在 MultiCache 对象创建时一同建立
      - 从第一个 Bucket 开始，逐个获取每个 Bucket 的第二个空白 Entry 压入链表头
      - 然后再从第一个 Bucket 开始，逐个获取每个 Bucket 的第三个空白 Entry 压入链表头
      - 直到把所有 Entry 都压入链表头
      - 由于每个 Bucket 的第一个 Entry 要做为 Hash 冲突链表的表头使用，因此不能放入全局空白 Entry 列表
      - 该列表的存取按照先进后出原则，仅在表头进行操作
   - ？？？当一层所有 Entry 全部用完后，会将不常用的 Entry 推入下一层，因此
      - 上层总是保存了下层 Entry 的热点内容
      - 上层每个 Bucket 可容纳的 Entry 数量总要小于下层
      - 上层冲突链表的最大深度是 (“上层每个 Bucket 可容纳的 Entry 数量” - 1 ) * Bucket 数量 + 1## MultiCacheHeader

以下内容待确认：

- 在进行插入或检索时，总是从第 0 层开始，因此第 0 层是优先级最高的。
- 当第 0 层存满之后，会将命中率较低的元素复制到下一层。
- 因此，最外层存储的是原始数据，内部各层存储的内容是最外层某个元素的拷贝。
- 那么每一个 Bucket 内的数据是否复制到了下一层，以及向下复制了几层？MultiCache 中使用成员 lowest_level_data 来描述（该机制目前不完善，从代码上来看应该是仅用于调试）。

问题：

- 会从外层向内层复制元素吗？
- 最外层存满之后，会怎么处理？



## 方法

### C *MultiCache\<C\>::cache_bucket(folded md5, level)

根据被“折叠的” MD5 值，对当前 MultiCache 数据库内总 Bucket 的数量 `buckets` 取模，得到该 MD5 值所对应的 Bucket 编号。然后在指定的层级找到该 Bucket，并返回 bucket 中的第一个数据元素（Entry）。将数据元素的关键信息传入 openssl 库函数 `MD5_Finial()`，将会得到 16 个 8 字节构成的 MD5 值，将高 64 位和低 64 位进行异或操作，即可得到被折叠（压缩）后的 MD5 值。参数：

- `uint64_t folded_md5`：被“折叠的” MD5 值，方便使用 64 位直接运算，
- `level`：指定层级。

该方法不会失败，总会返回指向选取 Bucket 内的第一个数据元素的指针。

```
template <class C>
inline C *
MultiCache<C>::cache_bucket(uint64_t folded_md5, unsigned int level)
{
  // 注意：这里缺少对 level 的验证
  // 通过简单的哈希方法得到 bucket 的编号
  int bucket = (int)(folded_md5 % buckets);
  // 使用数据区基址 data，加上指定层级的偏移 level_offset[level]，
  // 再加上指定层级每个 Bucket 内存储的数据元素数量 bucketsize[level] 乘以 Bucket 的编号
  char *offset = data + level_offset[level] + bucketsize[level] * bucket;
  // 得到指向该 Bucket 的指针，强制指针类型转换，得到指向该 Bucket 内第一个元素的指针，并返回
  return (C *)offset;
}
```

### C *MultiCache\<C\>::insert_block(folded md5, new block, level)

返回一个位于指定层级的 Entry，该 Entry 将用于存储新数据，如果给出一个新数元素，则在返回之前将新数据元素复制到该 Entry 里。当调用者得到返回值后，可以立即向其内填充数据。参数：

- `uint64_t folded_md5`：被“折叠的” MD5 值，方便使用 64 位直接运算，
- `C *new_block`：新数据元素，如果不为 NULL，则在返回之前将新数据元素复制到该 Entry 里，
- `unsigned int level`：指定插入操作的层级，从 0 开始到 levels - 1，因为 levels 最大为 3，因此该值仅为 0，1，2 中的一个，

向 MultiCache 数据库插入数据的流程如下：

1. 首先要通过给定的 hash 值在第 0 层找到一个 Bucket，
2. 然后在这个 Bucket 内查找空白的 Entry 或者与给定 hash 值一致的 Entry，如果找到了就跳转到第 5 步，
3. 如果没有找到空白的 Entry，表示选中的 Bucket 存满了，就在这个 Bucket 内查找 backed 为 true 的 Entry，
4. 如果这个 Bucket 内没有找到任何一个 backed 为 true 并且 hits 值最低的那个的 Entry，
   - 那么就调用 flush 方法将当前 Bucket 内所有的数据都备份（重新插入）到下一层（第 1 层）的同一个编号的 Bucket 里，
   - 然后重新在这个 Bucket 内查找 backed 为 true 的 Entry，
   - 注意：在重新插入到下一层时，可能会触发递归调用 flush 方法。
5. 此时应该找到了一个 Entry，可能是处于以下状态：
   - 空白：empty == true，
   - 重复：tag == block->tag()，
   - 已备份：backed == true，
   - 不管是哪种情况其内部的数据都将被重置或覆盖。
6. 如果参数 `C *new_block`：
   - 非 NULL，那么就使用其内容直接填充到所找到的 Entry 里，并调用 `update` 方法处理 HEAP 数据，
   - 为 NULL，那么就调用 reset 方法重置 Entry。
7. 最后，将找到的 Entry 返回给调用者。

因此，调用 `insert_block` 方法，传入的 `level` 值一般总是为 0，既表示总是从第 0 层向 MultiCache 数据库中插入数据，只有 `flush` 方法调用 `insert_block` 时会传入大于 0 的 `level` 值。

当调用者获得返回的 Entry 后，

- 可以向 Entry 内填入新的内容或更新原有内容，
- 如果需要存储不定长数据，也可以在此时向 HEAP 区申请空间，然后再使用 `memcpy` 方法填入数据
- 如果需要修改 HEAP 区的内容，也可以在此时进行修改
   - 修改是要注意：所占 HEAP 区空间只能变小不能变大，
   - 如果新数据超过原有 HEAP 空间长度，可重新申请新的 HEAP 区空间，
   - 旧的 HEAP 空间可直接丢弃，在下一次 HeapGC 时会自动回收丢弃的空间。
- 任何一个数据元素只能在 HEAP 区存储一个连续的 HEAP 空间，因为：
   - 通过 `MultiCacheBlock::heap_offset_ptr` 方法只能得到一个 HEAP 空间指针，
   - 通过 `MultiCacheBlock::heap_size` 方法也只能知道一个 HEAP 空间的大小。

```
//
// Insert an entry
//
template <class C>
inline C *
MultiCache<C>::insert_block(uint64_t folded_md5, C *new_block, unsigned int level)
{
  // 根据 MD5 值进行 hash 后得到 Bucket 的编号，然后得到该 Bucket 所在指定层级的第一个数据元素
  C *b = cache_bucket(folded_md5, level);
  C *block = NULL, *empty = NULL;
  // 根据 MD5 值进行 hash 后得到 Bucket 的编号
  int bucket = (int)(folded_md5 % buckets);
  int hits = 0;

  // Find the entry
  //
  // 根据 MD5 值计算 TAG，TAG 受到 MultiCacheHeader::tag_bits 的限制，默认为 56 bits 的长度，
  // 这里 tag = md5 / buckets 之后的低 56 bits
  uint64_t tag = make_tag(folded_md5);
  int n_empty = 0;

  // 在指定层级选中的 Block 内查找是否有空白的 Entry，或者与给定 MD5 值一致的 Entry，
  // 既：要么插入新的元素，要么覆盖具有相同 MD5 值的数据元素
  for (block = b; block < b + elements[level]; block++) {
    // 保存第一个空白 Entry 到 empty 变量
    if (block->is_empty() && !empty) {
      // 这个 n_empty++ 比较奇怪，这里只会执行一次
      n_empty++;
      empty = block;
    }
    // 如果找到具有相同 MD5 值的数据元素，任务就结束了，直接套转到 Lfound
    // 如果是空白 Entry 也不应该比较 tag 值，这里应该加个 else 优化一下
    if (tag == block->tag())
      goto Lfound;
    // 累加已经遍历过的数据元素的 hits 值，
    // 这里应该是对非空的 Entry 才进行累加，空白 Entry 的 hits 值为 0
    hits += block->hits;
  }
  // 这里跳转到 Lfound 为何没有放在上面的 for 循环里？
  // 找到第一个空白的 Entry 就可以跳转到 Lfound 了
  if (empty) {
    block = empty;
    goto Lfound;
  }

  // 此时 hits 内累加了该 Block 内所有数据元素被查询的次数（但是达到最大值 max_hits 后就不再增加了）
  {
    C *best = NULL;
    int again = 1;
    // 由于当前 Bucket 内所有的 Entry 都填满了数据元素，因此接下来要看哪些 Entry 内的数据备份到下一层了，
    // 如过 Entry 里的数据元素已经备份到下一层，那么就可以把查询次数最低的那个 Entry 拿来保存新数据元素。
    do {
      // Find an entry previously backed to a higher level.
      // self scale the hits number within the bucket
      //
      unsigned int dec = 0;
      // 这里 max_hits / 2 表示每个数据元素最大命中数的一半，考虑到整数除法会向下取整，因此再 + 1 得到大于最大命中数一半的最小整数，
      // 再乘以该 Bucket 内数据元素的数量，既表示：该 Bucket 内所有数据元素的查询命中次数均大于 50% 时的总命中次数。
      // 当该 Block 没有空白 Entry 时，如果所有元素的平均查询次数超过最大查询次数的 50%，就设置 dec 为 1。
      if (hits > ((max_hits / 2) + 1) * elements[level])
        dec = 1;
      // 在指定层级选中的 Block 内查找已经备份到下一级的元素，并在这些元素中找到查询次数（hits）最少的那个
      // 该操作必须完整遍历 Block 内所有的数据元素
      for (block = b; block < b + elements[level]; block++) {
        // 将已经备份的数据元素保存到 best 变量，如果新找到的元素的 hits 更小，则将 best 指向新找到的元素
        if (block->backed && (!best || best->hits > block->hits))
          best = block;
        // 如果当前 Block 内数据元素的平均查询次数超过 50%，就将每个数据元素的查询次数减 1
        // 这是一个退火处理，同时降低所有数据元素的查询热度，以避免所有数据的查询次数都达到最大值后，无法找到 hits 比较小的 Entry
        if (block->hits)
          block->hits -= dec;
      }
      // 如果找到已经做过数据备份的 Entry，就跳转到 Lfound
      if (best) {
        block = best;
        goto Lfound;
      }
      // 如果当前 Bucket 内所有的元素都没有做过备份，那么就调用 flush 方法将 Bucket 内所有元素备份到下一层
      flush(b, bucket, level);
      // 然后再从当前 Bucket 内查找做过备份的 Entry
    } while (again--);
    ink_assert(!"cache flush failure");
  }

// 找到了可以用来存储新数据元素的 Entry
Lfound:
  // 如需要复制数据到 Entry 里
  if (new_block) {
    // 使用对象复制的方式复制数据到 Entry 里
    *block = *new_block;
    // 判断是否有 Heap 数据需要复制
    int *hop = new_block->heap_offset_ptr();
    // 如过需要的话，调用 update 方法复制 HEAP 数据
    if (hop)
      update(block->heap_offset_ptr(), hop);
    // 重置新数据元素的备份状态为未备份
    block->backed = 0;
  } else
    // 不需要复制数据，仅将选中的 Entry 重置，避免遗留的旧数据产生影响
    block->reset();
  // 将新数据的 MD5 值设置到 Entry 上，同时设置 Entry 的状态为 full
  block->set_full(folded_md5, buckets);
  ink_assert(block->tag() == tag);
  // 将找到的 Entry 返回给调用者，
  // 调用者可以向 Entry 继续填充数据，或为其申请 HEAP 空间等
  return block;
}
```

### void MultiCache\<C\>::flush(block, bucket, level)

遍历指定层级、指定 Bucket 内的所有数据元素，首先将数据元素插入到下一层，然后将其标记为已备份（backed = true）。参数：

- C *b：指定层级、指定 Bucket 内的第一个数据元素
- int bucket：指定 Bucket 的编号，从 0 开始到 buckets - 1
- unsigned int level：指定层级，从 0 开始到 levels - 1，因为 levels 最大为 3，因此该值仅为 0，1，2 中的一个

如果指定的 level 已经是最底层了，没有下一层，那么就不将数据元素插入到下一层，只将其标记为已备份状态。

```
// 该宏定义根据输入的 TAG 值 `_cl->tag()` 还原出原来的 MD5 值
#define REBUILD_FOLDED_MD5(_cl) ((_cl->tag() * (uint64_t)buckets + (uint64_t)bucket))

//
// This function ejects some number of entries.
//
template <class C>
inline void
MultiCache<C>::flush(C *b, int bucket, unsigned int level)
{
  C *block = NULL;
  // The comparison against the constant is redundant, but it
  // quiets the array_bounds error generated by g++ 4.9.2
  if (level < levels - 1 && level < (MULTI_CACHE_MAX_LEVELS - 1)) {
    // 如果不是最底层
    // 设置该 bucket 的深度
    if (level >= lowest_level(bucket))
      set_lowest_level(bucket, level + 1);
    // 将数据元素一个一个插入到下一层的同一个 Bucket 里，然后设置 backed 为 true
    for (block = b; block < b + elements[level]; block++) {
      ink_assert(!block->is_empty());
      // 通过 TAG 还原出 MD5 值，确保数据元素重新插入（备份）到下一层的同一个 Bucket 编号
      insert_block(REBUILD_FOLDED_MD5(block), block, level + 1);
      // 备份到下一层之后，将已经备份的数据元素设置为已备份状态
      block->backed = true;
    }
  } else {
    // 如果是最底层，不进行重新插入操作，只设置 backed 为 true
    for (block = b; block < b + elements[level]; block++)
      if (!block->is_empty())
        block->backed = true;
  }
}
```

### void MultiCacheBase::update(poffset, old_poffset)

在 `MultiCache<C>::insert_block` 方法中，会将数据元素重新插入（备份）到下一层，会通过内存复制的方式拷贝数据元素的内容，如果该数据元素在 HEAP 区存储了数据，那么其内部会存在一个 int 类型的成员用于保存数据存储在 HEAP 区的位置：

- 该值如果为正数则没有问题，两个数据元素指向同一个 HEAP 区
- 该值如果为负数则存在问题，因为两个数据元素指向同一个 UnsunkPtr 对象，此时在 HeapGC 的遍历操作中会出现问题

因此，需要为新的 Entry 创建一个 UnsunkPtr 对象，新的 UnsunkPtr 对象可以与旧的 UnsunkPtr 对象都指向同一个 HEAP 区上的空间。

参数：

- int *poffset：指向新 Entry 内 int 类型成员的指针
- int *old_poffset：指向旧 Entry 内 int 类型成员的指针
- 要求新 Entry 内的数据完全由旧 Entry 直接通过内存复制而来

```
void
MultiCacheBase::update(int *poffset, int *old_poffset)
{
  // 获取新 Entry 内 int 类型成员的值，该值直接从旧 Entry 拷贝而来
  int o = *poffset;
  Debug("multicache", "updating %" PRId64 " %d", (int64_t)((char *)poffset - data), o);
  // 如果该值大于 0，说明其能够直接表示数据位于 HEAP 区的偏移量，无需做任何操作，直接返回
  if (o > 0) {
    // 验证偏移量是否正确，如果不正确将其重置
    if (!valid_offset(o)) {
      ink_assert(!"bad poffset");
      *poffset = 0;
    }
    return;
  }
  // 该值为 0，是非法值
  if (!o)
    return;

  // 此时，该值小于 0，说明其表示 UnsunkPtr 的编号
  // 根据 poffset 的指针地址计算该 Entry 所在的分区
  int part = ptr_to_partition((char *)poffset);

  // 分区错误，表示指针指向的地址不在 MultiCache 的数据区内
  if (part < 0)
    return;

  // 得到指向 UnsunkPtr 对象的指针
  // 因为新 Entry 的数据是由旧 Entry 直接通过内存复制得来，所以：
  //   - *poffset == *old_poffset
  //   - poffset == old_poffset
  UnsunkPtr *p = unsunk[part].ptr(-*old_poffset - 1);
  // 这个判断应该是之前出现过内存写乱掉的情况
  // 如果 int 类型成员的值不能表示合法有效的 UnsunkPtr 的编号，或者出现其他异常的情况，则重置并返回
  if (!p || p->poffset != old_poffset) {
    *poffset = 0;
    return;
  }
  // 确认 UnsunkPtr 对象没有指向新 Entry
  // 个人认为这里应该判断 UnsunkPtr 对象指向旧 Entry：p->poffset == old_poffset
  ink_assert(p->poffset != poffset);
  // 申请一个新的 UnsunkPtr
  UnsunkPtr *n = unsunk[part].alloc(poffset);
  // 指向新 Entry 内的 int 类型成员
  n->poffset = poffset;
  // 两个 UnsunkPtr 共同指向同一段 HEAP 空间
  n->offset = p->offset;
}
```

### C *MultiCache<C>::lookup_block(folded md5, level)

从第 0 层级开始，逐层查找指定 MD5 值的数据元素，最多向下查找指定的层数，返回找到的数据元素。参数：

- `uint64_t folded_md5`：被“折叠的” MD5 值，方便使用 64 位直接运算，
- `unsigned int level`：指定在第 0 层查找不到时，继续向下查找几层？

如果没有找到指定的数据元素，返回 NULL，否则返回找到的数据元素。

```
//
// Lookup an entry up to some level in the cache
//
template <class C>
inline C *
MultiCache<C>::lookup_block(uint64_t folded_md5, unsigned int level)
{
  // 根据 MD5 值进行 hash 后得到 Bucket 的编号，然后得到该 Bucket 所在指定层级的第一个数据元素
  C *b = cache_bucket(folded_md5, 0);
  // 根据 MD5 值计算 TAG，TAG 受到 MultiCacheHeader::tag_bits 的限制，默认为 56 bits 的长度，
  // 这里 tag = md5 / buckets 之后的低 56 bits
  uint64_t tag = make_tag(folded_md5);
  // 从第 0 层开始
  int i = 0;
  // Level 0
  // 遍历指定 Bucket 内所有的数据元素，找到就立即返回该元素
  for (i = 0; i < elements[0]; i++)
    if (tag == b[i].tag())
      return &b[i];
  // 第 0 层遍历完成之后，如果指定向下查找层数小于等于 0，就停止向下查找
  if (level <= 0)
    return NULL;
  // 如果指定向下查找层数大于 0，就继续在第 1 层里查找
  // Level 1
  // 根据 MD5 值进行 hash 后得到 Bucket 的编号，然后得到该 Bucket 所在指定层级的第一个数据元素
  b = cache_bucket(folded_md5, 1);
  // 遍历指定 Bucket 内所有的数据元素，找到就立即返回该元素
  for (i = 0; i < elements[1]; i++)
    if (tag == b[i].tag())
      return &b[i];
  // 第 1 层遍历完成之后，如果指定向下查找层数小于等于 1，就停止向下查找
  if (level <= 1)
    return NULL;
  // 如果指定向下查找层数大于 1，就继续在第 2 层里查找
  // Level 2
  // 根据 MD5 值进行 hash 后得到 Bucket 的编号，然后得到该 Bucket 所在指定层级的第一个数据元素
  b = cache_bucket(folded_md5, 2);
  // 遍历指定 Bucket 内所有的数据元素，找到就立即返回该元素
  for (i = 0; i < elements[2]; i++)
    if (tag == b[i].tag())
      return &b[i];
  // 所有层级都查找过之后，仍未找到，只能返回 NULL
  return NULL;
}
```

### void MultiCache<C>::delete_block(block)

删除指定的数据元素：

- 如果被删除的数据元素没有备份到下一层，那么直接将 Entry 设置为空白即可，
- 如果被删除的数据元素已经备份到下一层，那么需要将其在所有层级的备份也都删除掉。

将数据元素删除后，其占用的 Entry 空间可用于保存新的数据元素，但是其占用的 HEAP 空间不会立即释放，需要等到下一次 HeapGC 操作时，才会释放。

```
//
// This code is a bit of a mess and should probably be rewritten
//
template <class C>
inline void
MultiCache<C>::delete_block(C *b)
{
  // 如果要删除的数据元素已经备份到下一层
  if (b->backed) {
    // 获取要删除的数据元素所在的层级，
    unsigned int l = level_of_block(b);
    // 如果不是位于最底层
    if (l < levels - 1) {
      // 计算要删除的数据元素所在的 Bucket 的编号
      int bucket = (((char *)b - data) - level_offset[l]) / bucketsize[l];
      // 找到同一 Bucket 编号，下一层级的第一个数据元素
      C *x = (C *)(data + level_offset[l + 1] + bucket * bucketsize[l + 1]);
      // 遍历下一层级同一个 Bucket 内的数据元素，查找要删除数据元素的备份
      // 如果找到就递归调用 delete_block 方法进行删除
      for (C *y = x; y < x + elements[l + 1]; y++)
        if (b->tag() == y->tag())
          delete_block(y);
    }
  }
  // 将 Entry 的状态设置为空白
  b->set_empty();
}
```

## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)
- [MultiCache.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/MultiCache.cc)

