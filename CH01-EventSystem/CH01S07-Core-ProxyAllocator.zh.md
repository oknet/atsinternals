# 核心部件：ProxyAllocator

ProxyAllocator是线程访问ClassAllocator的缓存，看名字就知道，这个类是访问ClassAllocator的一个Proxy实现。

这个类本身并不负责分配和释放内存，而只是维护了上层ClassAllocator的一个可用内存块的列表。

## 构成

ProxyAllocator
- 该结构维护一个可用“小块内存”的链表 freelist
- 和一个表示该链表内有多少元素的计数器 allocated


## 设计思想

ATS是一个多线程的系统，Allocator和ClassAllocator是全局数据，那么多线程访问全局数据，必然要加锁和解锁，这样效率就会比较低。

于是ATS设计了一个ProxyAllocator，它是每一个Thread内部的数据结构，它将Thread对ClassAllocator全局数据的访问进行了一个Proxy操作

- Thread请求内存时调用THREAD_ALLOC方法（其实是一个宏定义），判断ProxyAllocator里是否有可用的内存块
   - 如果没有
      - 调用对应的ClassAllocator的alloc方法申请一块内存
      - 此操作将从大块的内存中取出一个未分配的小块内存
   - 如果有
      - 直接从ProxyAllocator里拿一个，freelist指向下一个可用内存块，allocated--
- Thread释放内存时调用THREAD_FREE方法（其实是一个宏定义）
   - 直接把该内存地址放入ProxyAllocator里freelist里，allocated++
   - 注意：此时，这块内存并未被在ClassAllocator里标记为可分配状态

从以上过程我们看出来，ProxyAllocator就是不断的将全局空间据为己有。

那么会出现一种比较恶劣的情况

- 某个Thread由于在执行特殊操作时，将大量的全局空间据为己有
- 导致了Thread间内存分配不均衡

对于这种情况，ATS也做了处理

- 参数：thread_freelist_low_watermark 和 thread_freelist_high_watermark
- 在THREAD_FREE内部，会做一个判断：如果ProxyAllocator的freelist持有的内存片超过了High Watermark值
   - 就调用thread_freeup方法
      - 从freelist的头部删除连续的数个节点，直到freelist只剩下Low Watermark个节点
      - 调用ClassAllocator的free或者free_bulk，将从freelist里删除的节点标记为可分配内存块

总结一下

- 凡是直接操作ProxyAllocator的内部元素freelist
   - 都是不需要加锁和解锁的，因为那是Thread内部数据
- 但是需要ClassAllocator介入的
   - 都是需要加锁和解锁的
   - 当ProxyAllocator的freelist指向NULL时
   - 当allocated大于thread_freelist_high_watermark时
- 通过ProxyAllocator
   - Thread在访问全局内存池的资源时，可以有较少的资源锁冲突


## ProxyAllocator里freelist链表的巧妙实现

它的定义很简单

```
struct ProxyAllocator {
  int allocated;
  void *freelist;

  ProxyAllocator() : allocated(0), freelist(0) {}
};
```

那么只是通过一个freelist如何实现一个链表操作的呢？请参考如下代码：

```
  if (l.freelist) {                                                                                                
    void *v = (void *)l.freelist;
    l.freelist = *(void **)l.freelist;
    --(l.allocated);
    return v;
  }
```

```
#define THREAD_FREE(_p, _a, _t)                            \
  do {                                                     \
    *(char **)_p = (char *)_t->_a.freelist;                \                                                       
    _t->_a.freelist = _p;                                  \
    _t->_a.allocated++;                                    \
    if (_t->_a.allocated > thread_freelist_high_watermark) \
      thread_freeup(::_a, _t->_a);                         \
  } while (0)
```

我们前面说过，freelist这个链表里只保存可用内存块

- 在这个链表里的内存空间里的数据都是没用的

为了节省内存的使用，freelist将链表内下一个内存块的指针地址存储在了freelist[0]的位置

所以我们看到了非常奇怪的一句代码：

```
l.freelist = *(void **)l.freelist;

大约等价于

l.freelist = (void *)l.freelist[0];
```

```
*(char **)_p = (char *)_t->_a.freelist;

大约等价于

(char *)_p[0] = (char *)_t->_a.freelist;
```

## 参考资料
- [I_ProxyAllocator.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_ProxyAllocator.h)

