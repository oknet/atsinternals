# 基础组件：MultiCacheSync

`MultiCacheSync` 状态机是 MultiCache 数据库实现数据持久化的核心，由 `MultiCacheBase::sync_partitions` 方法创建并激活，周期性的调用该状态机以确保数据持续同步到磁盘，避免在系统意外 Crash 时丢失数据。但是由于两次调用之间，存在一定的时间间隔，因此仍然可能会有一小部分数据丢失的可能。

`MultiCacheSync` 状态机定义了三个状态函数：

- heapEvent：用于将 HEAP 区的数据同步到磁盘
- mcEvent：用于将数据区的数据同步到磁盘
- pauseEvent：用于在两次 mcEvent 调用之间进行延迟等待，避免磁盘 I/O 负载过高

```
void
MultiCacheBase::sync_partitions(Continuation *cont)
{
  // don't try to sync if we were not correctly initialized
  if (data && mapped_header) {
    // HEAP 当前工作半区的空间利用率未达到 80%（MULTI_CACHE_HEAP_HIGH_WATER 默认为 0.8）以上时，触发 MultiCacheSync 操作；
    if (heap_used[heap_halfspace] > halfspace_size() * MULTI_CACHE_HEAP_HIGH_WATER)
...
    else
      eventProcessor.schedule_imm(new MultiCacheSync(cont, this), ET_CALL);
  }
}
```

## 定义

```
//
// Syncs MulitCache
//
struct MultiCacheSync;
typedef int (MultiCacheSync::*MCacheSyncHandler)(int, void *);

struct MultiCacheSync : public Continuation {
  int partition;
  MultiCacheBase *mc;
  Continuation *cont;
  int before_used;

  // 由 Event System 回调，回调事件为 EVENT_IMMEDIATE
  int
  heapEvent(int event, Event *e)
  {
    // 开始 heap 同步之前，先记录当前 heap 空间使用到的位置，因为在此位置之前的 HEAP 空间都已经同步到磁盘了，
    // 这样在后面的 mcEvent 中处理 UnsunkPtr 的时候，就可以将数据存储于 before_used 之前的 UnsunkPtr 都更新到 Entry 里。
    if (!partition) {
      // BUG：严格来说此处应该先做 MultiCacheHeader 的快照，然后从快照里获取 before_used 的值
      before_used = mc->heap_used[mc->heap_halfspace];
      mc->header_snap = *(MultiCacheHeader *)mc;
    }
    // 逐个分区操作，首次进入 partition 为 0
    if (partition < MULTI_CACHE_PARTITIONS) {
      // sync_heap 强制将 HEAP 区平均分为 64 份，分 64 次完成 HEAP 区的同步
      mc->sync_heap(partition++);
      // 此处使用了 schedule_imm 为连续执行，因此对分区 0 的互斥锁是连续上锁
      e->schedule_imm();
      return EVENT_CONT;
    }
    // 如 partition == 64 则表示已经完成 HEAP 区的同步操作
    // 将之前保存的快照存入 mapped_header，并同步到磁盘
    *mc->mapped_header = mc->header_snap;
    ink_assert(!ats_msync((char *)mc->mapped_header, STORE_BLOCK_SIZE, (char *)mc->mapped_header + STORE_BLOCK_SIZE, MS_SYNC));
    // 将 partition 重置为 0，准备进行数据库的同步操作
    partition = 0;
    // 回调函数切换为 mcEvent，由于状态机的互斥锁一直设定为与分区 0 共享，因此直接调用 mcEvent 方法
    SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::mcEvent);
    return mcEvent(event, e);
  }

  // 首次调用从 heapEvent 进入，传入 EVENT_IMMEDIATE 事件，
  // 后续由 Event System 回调，回调事件为 EVENT_IMMEDIATE
  int
  mcEvent(int event, Event *e)
  {
    (void)event;
    // 如 partition == 64 则表示已经完成数据区的同步操作
    if (partition >= MULTI_CACHE_PARTITIONS) {
      // 向状态机回调 MULTI_CACHE_EVENT_SYNC 事件
      cont->handleEvent(MULTI_CACHE_EVENT_SYNC, 0);
      Debug("multicache", "MultiCacheSync done (%d, %d)", mc->heap_used[0], mc->heap_used[1]);
      // 释放 MultiCacheSync 状态机
      delete this;
      return EVENT_DONE;
    }
    // 逐个分区遍历，首次进入 partition 为 0
    // 调用 fixup_heap_offsets 方法，处理 UnsunkPtr
    mc->fixup_heap_offsets(partition, before_used);
    // 同步当前分区到磁盘
    mc->sync_partition(partition);
    // 继续下一个分区
    partition++;
    // 切换到 pauseEvent 状态函数，将 mutex 设置为当前线程的互斥锁
    mutex = e->ethread->mutex;
    SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::pauseEvent);
    // BUG：联合 HostDBSyncer 的代码，数据同步间隔时间的计算步骤，应该不符合 HostDBSyner 要求
    e->schedule_in(MAX(MC_SYNC_MIN_PAUSE_TIME, HRTIME_SECONDS(hostdb_sync_frequency - 5) / MULTI_CACHE_PARTITIONS));
    return EVENT_CONT;
  }

  // 由 Event System 回调，回调事件为 EVENT_INTERVAL
  // 与 mcEvent 状态函数间隔运行。
  // 建议：pauseEvent 状态函数实际上可以使用 schedule_in 调用替代，目前的设计过于复杂。可参考 MultiCacheHeapGC 的简洁设计。
  int
  pauseEvent(int event, Event *e)
  {
    (void)event;
    (void)e;
    // 将 MultiCacheSync 状态机的互斥锁与下一个分区的互斥锁共享，
    if (partition < MULTI_CACHE_PARTITIONS)
      mutex = mc->locks[partition];
    else
    // 如果所有的分区都处理完毕，那么就将 MultiCacheSync 状态机的互斥锁与回调的状态机的互斥锁共享
      mutex = cont->mutex;
    // 切换回 mcEvent 状态函数
    SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::mcEvent);
    e->schedule_imm();
    return EVENT_CONT;
  }

  // 构造函数
  MultiCacheSync(Continuation *acont, MultiCacheBase *amc)
    : Continuation(amc->locks[0]), partition(0), mc(amc), cont(acont), before_used(0)
  {
    // 这里初始化状态机的互斥锁与第一个分区共享
    // BUG：heapEvent 负责同步 HEAP 区，但是 HEAP 区是不需要上锁就可以进行访问的，
    // 而且 HEAP 区也不是每个分区有独立的区域，是所有分区的数据元素共享的，
    // 可以看到首先会重复回调 64 次 heapEvent，而且在此期间一直对分区 0 的互斥锁上锁，
    // 这样对分区 0 的数据元素的访问是有一定影响的，个人认为，这里设置为 acont->mutex 会比较合理。
    mutex = mc->locks[partition];
    SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::heapEvent);
  }
};
```

## 方法

### UnsunkPtrRegistry *MultiCacheBase::fixup\_heap\_offsets(int partition, int before_used, ...)

`fixup_heap_offsets` 方法用于遍历指定分区的 UnsunkPtr，将数据位置在 `before_used` 之前的 UnsunkPtr 回收，将其保存的 offset 数据存储到 Entry 里。如果给定 UnsunkPtrRegistry 对象内的所有 UnsunkPtr 都被回收了，并且是连表上的最后一个 UnsunkPtrRegistry 对象，那么就释放它，但是不能释放每个分区的第一个 UnsunkPtrRegistry 对象。如果当前 UnsunkPtrRegistry 对象被释放就返回 NULL，如果没有被释放，就返回当前 UnsunkPtrRegistry 对象的地址。

参数：

- int partition：遍历指定分区的 UnsunkPtr
- int before_used：给定偏移地址之前的 HEAP 区数据已经同步到磁盘
- UnsunkPtrRegistry *r：处理给定 UnsunkPtrRegistry 对象内的 UnsunkPtr，默认为 NULL，表示取指定分区上的第一个 UnsunkPtrRegistry 对象
- int base：由于该方法支持递归调用处理 UnsunkPtrRegistry 链表，用于指明链表上每个 UnsunkPtrRegistry 对象内第一个 UnsunkPtr 的绝对序号，默认为 0

```
UnsunkPtrRegistry *
MultiCacheBase::fixup_heap_offsets(int partition, int before_used, UnsunkPtrRegistry *r, int base)
{
  // 如果 r 为 NULL，则表示从指定分区的第一个 UnsunkPtrRegistry 开始
  if (!r)
    r = &unsunk[partition];
  bool found = 0;
  // 遍历当前 UnsunkPtrRegistry 上的所有 UnsunkPtr，共有 r->n 个 UnsunkPtr
  // 目前最多有 8 个（MULTI_CACHE_UNSUNK_PTR_BLOCK_SIZE）
  for (int i = 0; i < r->n; i++) {
    // 取一个 UnsunkPtr 对象
    UnsunkPtr &p = r->ptrs[i];
    // 判断 offset 是否有效（offset == 0 是无效数据）
    if (p.offset) {
      Debug("multicache", "fixup p.offset %d offset %d %" PRId64 " part %d", p.offset, *p.poffset,
            (int64_t)((char *)p.poffset - data), partition);
      // 确认当前的 UnsunkPtr 与该 Entry 的关联是正确的
      // Entry 内 int 类型成员应该小于 0，表示 UnsunkPtr 的编号，通过公式：- ( 编号 + 1 ) 转换后存入 *poffset
      if (*p.poffset == -(i + base) - 1) {
        // 如果 offset 与当前的 HEAP 半区不匹配，就触发 assert 并将数据清除
        // 因为 UnsunkPtr 在内存中，每次 HeapGC 操作都能及时更新数据，因此不会存在 offset 与当前 HEAP 半区不匹配的情况
        if (halfspace_of(p.offset) != heap_halfspace) {
          ink_assert(0);
          *p.poffset = 0;
        } else {
          // offset 合法性验证通过
          // 判断 offset 指向的 HEAP 空间是否在 before_used 之前
          // BUG：p.offset 值的取值范围是 [8, heap_size)，而 before_used 的取值范围是 [8, heap_size/2)
          // 这里没有考虑 before_used 位于第二个 HEAP 半区的情况
          if (p.offset < before_used) {
            // 如果在 before_used 之前，则将 offset 值存入 Entry 中
            // 因为 Entry 中的 offset 值为 0 表示无效数据，因此所有 Entry 中存储的 offset 值都做 +1 操作，待使用时再做 -1 操作。
            *p.poffset = p.offset + 1;
            // 确认 Entry 中的 offset 值不为 0
            ink_assert(*p.poffset);
          } else
            // 如果在 before_used 则维持 UnsunkPtr 不变，继续处理下一个
            continue;
        }
      } else {
        // 如果当前的 UnsunkPtr 与该 Entry 的关联是错误的，输出 debug 信息
        Debug("multicache", "not found %" PRId64 " i %d base %d *p.poffset = %d", (int64_t)((char *)p.poffset - data), i, base,
              *p.poffset);
      }
      // 当前 UnsunkPtr 的数据已经被存入 Entry，
      // 因此可以将其数据清除，
      p.offset = 0;
      // 然后回收，放入 next_free 链表
      p.poffset = (int *)r->next_free;
      r->next_free = &p;
      // 只要找到任意一个有效的 UnsunkPtr，就设置 found 标志为 true
      found = true;
    }
  }
  // 遍历完当前 UnsunkPtrRegistry 后，继续遍历 UnsunkPtrRegistry 链表上的下一个 UnsunkPtrRegistry 对象
  if (r->next) {
    // 每个 UnsunkPtrRegistry 对象内保存 8 （MULTI_CACHE_UNSUNK_PTR_BLOCK_SIZE）个 UnsunkPtr
    int s = MULTI_CACHE_UNSUNK_PTR_BLOCK_SIZE(totalelements);
    // 以递归方式处理下一个 UnsunkPtrRegistry 对象
    r->next = fixup_heap_offsets(partition, before_used, r->next, base + s);
  }
  // 如果当前 UnsunkPtrRegistry 是链表上的最后一个，但是不是第一个（唯一一个），
  // 同时也从未在该 UnsunkPtrRegistry 上回收 UnsunkPtr 对象，
  // 那么就释放它，并返回 NULL
  if (!r->next && !found && r != &unsunk[partition]) {
    // found 为 false，表示当前 UnsunkPtrRegistry 所有的 UnsunkPtr 对象都是无效数据，
    // 因此当 UnsunkPtrRegistry 内的所有 UnsunkPtr 对象都被释放后，也不会立即释放，
    // 只有当第二次遍历 UnsunkPtrRegistry 内的所有 UnsunkPtr 对象都是空数据后，才会释放它。
    delete r;
    return NULL;
  }
  return r;
}
```

上面的 BUG 可以使用以下补丁修复：

```
diff --git a/iocore/hostdb/MultiCache.cc b/iocore/hostdb/MultiCache.cc
index 734114d2d70..cf24c4ea5cf 100644
--- a/iocore/hostdb/MultiCache.cc
+++ b/iocore/hostdb/MultiCache.cc
@@ -1080,7 +1080,7 @@ MultiCacheBase::fixup_heap_offsets(int partition, int before_used, UnsunkPtrRegi
           ink_assert(0);
           *p.poffset = 0;
         } else {
-          if (p.offset < before_used) {
+          if (p.offset < before_used + heap_halfspace * halfspace_size()) {
             *p.poffset = p.offset + 1;
             ink_assert(*p.poffset);
           } else
```

你也可以修改 `MultiCacheSync::heapEvent` 中保存 `before_used` 的那一行代码。


## 修正数据同步时间间隔

以下补丁是修正数据同步间隔时间的，在 MultiCacheHeapGC 和 MultiCacheSync 都存在同样的问题。[下载 diff 文件](https://github.com/oknet/trafficserver/commit/ab867acbdc0640fce066e0adc4f0ca2dd7048560.diff)

以下补丁同时还对 MultiCacheSync 进行了重构，去掉了 pauseEvent，简化了处理逻辑。

```
diff --git a/iocore/hostdb/HostDB.cc b/iocore/hostdb/HostDB.cc
index 7081b4658bb..b228c4441f1 100644
--- a/iocore/hostdb/HostDB.cc
+++ b/iocore/hostdb/HostDB.cc
@@ -383,7 +383,7 @@ HostDBSyncer::sync_event(int, void *)
 {
   SET_HANDLER(&HostDBSyncer::wait_event);
   start_time = Thread::get_hrtime();
-  hostDBProcessor.cache()->sync_partitions(this);
+  hostDBProcessor.cache()->sync_partitions(this, hostdb_sync_frequency - 5);
   return EVENT_DONE;
 }
 
@@ -393,10 +393,16 @@ HostDBSyncer::wait_event(int, void *)
   ink_hrtime next_sync = HRTIME_SECONDS(hostdb_sync_frequency) - (Thread::get_hrtime() - start_time);
 
   SET_HANDLER(&HostDBSyncer::sync_event);
-  if (next_sync > HRTIME_MSECONDS(100))
+  if (next_sync > HRTIME_MSECONDS(100)) {
     eventProcessor.schedule_in(this, next_sync, ET_TASK);
-  else
+  } else {
+    if (next_sync < 0) {
+      Warning("The HostDB did not sync to disk in time (delayed %" PRId64
+              " secs), please increase the sync time by proxy.config.cache.hostdb.sync_frequency",
+              ink_hrtime_to_sec(-next_sync));
+    }
     eventProcessor.schedule_imm(this, ET_TASK);
+  }
   return EVENT_DONE;
 }
 
diff --git a/iocore/hostdb/MultiCache.cc b/iocore/hostdb/MultiCache.cc
index 8bec81f0a8a..734114d2d70 100644
--- a/iocore/hostdb/MultiCache.cc
+++ b/iocore/hostdb/MultiCache.cc
@@ -988,67 +988,74 @@ struct MultiCacheSync;
 typedef int (MultiCacheSync::*MCacheSyncHandler)(int, void *);
 
 struct MultiCacheSync : public Continuation {
+  int frequency;
   int partition;
   MultiCacheBase *mc;
   Continuation *cont;
   int before_used;
+  ink_hrtime time_per_partition;
+  ink_hrtime time_start;
 
   int
   heapEvent(int event, Event *e)
   {
     if (!partition) {
-      before_used     = mc->heap_used[mc->heap_halfspace];
+      // Make a snapshot for MultiCache header
       mc->header_snap = *(MultiCacheHeader *)mc;
+      before_used     = mc->header_snap.heap_used[mc->header_snap.heap_halfspace];
     }
     if (partition < MULTI_CACHE_PARTITIONS) {
       mc->sync_heap(partition++);
       e->schedule_imm();
       return EVENT_CONT;
     }
+    // Sync the snapshot to disk
     *mc->mapped_header = mc->header_snap;
     ink_assert(!ats_msync((char *)mc->mapped_header, STORE_BLOCK_SIZE, (char *)mc->mapped_header + STORE_BLOCK_SIZE, MS_SYNC));
+
     partition = 0;
     SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::mcEvent);
-    return mcEvent(event, e);
+
+    ink_hrtime now = Thread::get_hrtime();
+    Debug("multicache", "MultiCacheSync heapEvent done (elapsed: %" PRId64 " secs)", ink_hrtime_to_sec(now - time_start));
+    time_per_partition = (HRTIME_SECONDS(frequency) - (now - time_start)) / MULTI_CACHE_PARTITIONS;
+    time_start         = now;
+    mutex              = mc->locks[partition];
+    e->schedule_imm();
+    return EVENT_CONT;
   }
 
   int
   mcEvent(int event, Event *e)
   {
     (void)event;
-    if (partition >= MULTI_CACHE_PARTITIONS) {
-      cont->handleEvent(MULTI_CACHE_EVENT_SYNC, 0);
-      Debug("multicache", "MultiCacheSync done (%d, %d)", mc->heap_used[0], mc->heap_used[1]);
-      delete this;
-      return EVENT_DONE;
-    }
     mc->fixup_heap_offsets(partition, before_used);
     mc->sync_partition(partition);
     partition++;
-    mutex = e->ethread->mutex;
-    SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::pauseEvent);
-    e->schedule_in(MAX(MC_SYNC_MIN_PAUSE_TIME, HRTIME_SECONDS(hostdb_sync_frequency - 5) / MULTI_CACHE_PARTITIONS));
-    return EVENT_CONT;
-  }
 
-  int
-  pauseEvent(int event, Event *e)
-  {
-    (void)event;
-    (void)e;
-    if (partition < MULTI_CACHE_PARTITIONS)
-      mutex = mc->locks[partition];
-    else
-      mutex = cont->mutex;
-    SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::mcEvent);
-    e->schedule_imm();
-    return EVENT_CONT;
+    if (partition < MULTI_CACHE_PARTITIONS) {
+      ink_hrtime time_rest = time_start + time_per_partition * partition - Thread::get_hrtime();
+      mutex                = mc->locks[partition];
+      e->schedule_in(time_rest > MC_SYNC_MIN_PAUSE_TIME ? time_rest : MC_SYNC_MIN_PAUSE_TIME);
+      return EVENT_CONT;
+    } else {
+      eventProcessor.schedule_imm(cont, ET_TASK, MULTI_CACHE_EVENT_SYNC, nullptr);
+      Debug("multicache", "MultiCacheSync done (%d, %d), half = %d", mc->heap_used[0], mc->heap_used[1], mc->heap_halfspace);
+      delete this;
+      return EVENT_DONE;
+    }
   }
 
-  MultiCacheSync(Continuation *acont, MultiCacheBase *amc)
-    : Continuation(amc->locks[0]), partition(0), mc(amc), cont(acont), before_used(0)
+  MultiCacheSync(Continuation *acont, MultiCacheBase *amc, int afreq)
+    : Continuation(amc->locks[0]),
+      frequency(afreq),
+      partition(0),
+      mc(amc),
+      cont(acont),
+      before_used(0),
+      time_per_partition(0),
+      time_start(Thread::get_hrtime())
   {
-    mutex = mc->locks[partition];
     SET_HANDLER((MCacheSyncHandler)&MultiCacheSync::heapEvent);
   }
 };
@@ -1113,73 +1120,79 @@ struct MultiCacheHeapGC : public Continuation {
   int partition;
   int n_offsets;
   OffsetTable *offset_table;
+  ink_hrtime time_per_partition;
+  ink_hrtime time_start;
 
   int
   startEvent(int event, Event *e)
   {
     (void)event;
-    if (partition < MULTI_CACHE_PARTITIONS) {
-      // copy heap data
 
-      char *before = mc->heap + mc->heap_halfspace * mc->halfspace_size() + mc->heap_used[mc->heap_halfspace];
-      mc->copy_heap(partition, this);
-      char *after = mc->heap + mc->heap_halfspace * mc->halfspace_size() + mc->heap_used[mc->heap_halfspace];
+    // copy heap data
+    char *before = mc->heap + mc->heap_halfspace * mc->halfspace_size() + mc->heap_used[mc->heap_halfspace];
+    mc->copy_heap(partition, this);
+    char *after = mc->heap + mc->heap_halfspace * mc->halfspace_size() + mc->heap_used[mc->heap_halfspace];
 
-      // sync new heap data and header (used)
+    // sync new heap data and header (used)
+    if (after - before > 0) {
+      ink_assert(!ats_msync(before, after - before, mc->heap + mc->heap_size, MS_SYNC));
+      ink_assert(!ats_msync((char *)mc->mapped_header, STORE_BLOCK_SIZE, (char *)mc->mapped_header + STORE_BLOCK_SIZE, MS_SYNC));
+    }
 
-      if (after - before > 0) {
-        ink_assert(!ats_msync(before, after - before, mc->heap + mc->heap_size, MS_SYNC));
-        ink_assert(!ats_msync((char *)mc->mapped_header, STORE_BLOCK_SIZE, (char *)mc->mapped_header + STORE_BLOCK_SIZE, MS_SYNC));
-      }
-      // update table to point to new entries
-
-      for (int i = 0; i < n_offsets; i++) {
-        int *i1, i2;
-        // BAD CODE GENERATION ON THE ALPHA
-        //*(offset_table[i].poffset) = offset_table[i].new_offset + 1;
-        i1  = offset_table[i].poffset;
-        i2  = offset_table[i].new_offset + 1;
-        *i1 = i2;
-      } 
-      n_offsets = 0;
-      mc->sync_partition(partition);
-      partition++;
-      if (partition < MULTI_CACHE_PARTITIONS)
-        mutex = mc->locks[partition];
-      else
-        mutex = cont->mutex;
-      e->schedule_in(MAX(MC_SYNC_MIN_PAUSE_TIME, HRTIME_SECONDS(hostdb_sync_frequency - 5) / MULTI_CACHE_PARTITIONS));
+    // update table to point to new entries
+    for (int i = 0; i < n_offsets; i++) {
+      int *i1, i2;
+      // BAD CODE GENERATION ON THE ALPHA
+      //*(offset_table[i].poffset) = offset_table[i].new_offset + 1;
+      i1  = offset_table[i].poffset;
+      i2  = offset_table[i].new_offset + 1;
+      *i1 = i2;
+    }
+    n_offsets = 0;
+    mc->sync_partition(partition);
+    partition++;
+
+    if (partition < MULTI_CACHE_PARTITIONS) {
+      ink_hrtime time_rest = time_start + time_per_partition * partition - Thread::get_hrtime();
+      mutex                = mc->locks[partition];
+      e->schedule_in(time_rest > MC_SYNC_MIN_PAUSE_TIME ? time_rest : MC_SYNC_MIN_PAUSE_TIME);
       return EVENT_CONT;
+    } else {
+      mc->heap_used[mc->heap_halfspace ? 0 : 1] = 8; // skip 0
+      eventProcessor.schedule_imm(cont, ET_TASK, MULTI_CACHE_EVENT_SYNC, nullptr);
+      Debug("multicache", "MultiCacheHeapGC done");
+      delete this;
+      return EVENT_DONE;
     }
-    mc->heap_used[mc->heap_halfspace ? 0 : 1] = 8; // skip 0
-    cont->handleEvent(MULTI_CACHE_EVENT_SYNC, 0);
-    Debug("multicache", "MultiCacheHeapGC done");
-    delete this;
-    return EVENT_DONE;
   }
 
-  MultiCacheHeapGC(Continuation *acont, MultiCacheBase *amc)
-    : Continuation(amc->locks[0]), cont(acont), mc(amc), partition(0), n_offsets(0)
+  MultiCacheHeapGC(Continuation *acont, MultiCacheBase *amc, int afreq)
+    : Continuation(amc->locks[0]),
+      cont(acont),
+      mc(amc),
+      partition(0),
+      n_offsets(0),
+      time_per_partition(HRTIME_SECONDS(afreq) / MULTI_CACHE_PARTITIONS),
+      time_start(Thread::get_hrtime())
   {
     SET_HANDLER((MCacheHeapGCHandler)&MultiCacheHeapGC::startEvent);
     offset_table = (OffsetTable *)ats_malloc(sizeof(OffsetTable) *
                                              ((mc->totalelements / MULTI_CACHE_PARTITIONS) + mc->elements[mc->levels - 1] * 3 + 1));
     // flip halfspaces
-    mutex              = mc->locks[partition];
     mc->heap_halfspace = mc->heap_halfspace ? 0 : 1;
   }
   ~MultiCacheHeapGC() { ats_free(offset_table); }
 };
 
 void
-MultiCacheBase::sync_partitions(Continuation *cont)
+MultiCacheBase::sync_partitions(Continuation *cont, int frequency)
 {
   // don't try to sync if we were not correctly initialized
   if (data && mapped_header) {
     if (heap_used[heap_halfspace] > halfspace_size() * MULTI_CACHE_HEAP_HIGH_WATER)
-      eventProcessor.schedule_imm(new MultiCacheHeapGC(cont, this), ET_TASK);
+      eventProcessor.schedule_imm(new MultiCacheHeapGC(cont, this, frequency), ET_TASK);
     else
-      eventProcessor.schedule_imm(new MultiCacheSync(cont, this), ET_TASK);
+      eventProcessor.schedule_imm(new MultiCacheSync(cont, this, frequency), ET_TASK);
   }
 }
 
diff --git a/iocore/hostdb/P_MultiCache.h b/iocore/hostdb/P_MultiCache.h
index dfc76caa50d..7db26903b81 100644
--- a/iocore/hostdb/P_MultiCache.h
+++ b/iocore/hostdb/P_MultiCache.h
@@ -346,7 +346,7 @@ struct MultiCacheBase : public MultiCacheHeader {
   int sync_heap(int part); // part varies between 0 and MULTI_CACHE_PARTITIONS
   int sync_header();
   int sync_partition(int partition);
-  void sync_partitions(Continuation *cont);
+  void sync_partitions(Continuation *cont, int frequency);
 
   MultiCacheBase();
   virtual ~MultiCacheBase() { reset(); }
```

## 参考资料

- [P_MultiCache.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_MultiCache.h)
- [MultiCache.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/MultiCache.cc)

