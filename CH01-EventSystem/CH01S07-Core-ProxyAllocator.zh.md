# 核心部件：ProxyAllocator

ProxyAllocator 是线程访问 ClassAllocator 的缓存，看名字就知道，这个类是访问 ClassAllocator 的一个 Proxy 实现。

这个类本身并不负责分配和释放内存，而只是维护了上层 ClassAllocator 的一个可用内存块的列表。

## 构成

```
struct ProxyAllocator {
  // 表示 freelist 链表内有多少元素的计数器
  int allocated;
  // 维护一个可用“小块内存”的链表
  void *freelist;

  ProxyAllocator() : allocated(0), freelist(0) {}
};
```

## 设计思想

ATS 是一个多线程的系统，Allocator 和 ClassAllocator 是全局资源，那么多线程访问全局资源，必然要加锁和解锁，这样效率就会比较低。

于是ATS设计了一个 ProxyAllocator，它是每一个 Thread 内部的数据结构，它将 Thread 对 ClassAllocator 全局资源的访问进行了一个Proxy 操作

- Thread 请求内存时调用 THREAD_ALLOC 方法（其实是一个宏定义），判断 ProxyAllocator 里是否有可用的内存块
   - 如果没有
      - 调用对应的 ClassAllocator 的 alloc 方法申请一块内存
      - 此操作将从大块的内存中取出一个未分配的小块内存
   - 如果有
      - 直接从 ProxyAllocator 里取一个，freelist 指向下一个可用内存块，计数器 allocated 执行递减操作
- Thread 释放内存时调用 THREAD_FREE 方法（其实是一个宏定义）
   - 直接把该内存地址放入 ProxyAllocator 里 freelist 链表里，计数器 allocated 执行递增操作
   - 这样下一次 Thread 请求内存时就可以直接从 freelist 链表上获得内存空间，而不需要通过全局资源进行内存分配
   - 注意：此时，这块内存并未被在 ClassAllocator 里标记为可分配状态

从以上过程我们看出来，ProxyAllocator 就是不断的将全局空间据为己有。

那么会出现一种比较恶劣的情况

- 某个 Thread 由于在执行特殊操作时，将大量的全局空间据为己有
- 导致了 Thread 间内存分配不均衡

对于这种情况，ATS 也做了处理

- 参数：thread\_freelist\_low\_watermark 和 thread\_freelist\_high\_watermark
- 在 THREAD_FREE 内部，会做一个判断：如果 ProxyAllocator 的freelist 持有的内存片超过了 High Watermark 值
   - 就调用 thread_freeup 方法
      - 从 freelist 的头部删除连续的数个节点，直到 freelist 只剩下 Low Watermark 个节点
      - 调用 ClassAllocator 的 free 或者 free_bulk 方法，将从 freelist 里删除的节点标记为可再次进行分配的全局内存块资源

总结一下

- 凡是直接操作 ProxyAllocator 的内部元素 freelist
   - 都是不需要加锁和解锁的，因为那是 Thread 内部数据
- 但是需要 ClassAllocator 介入的，例如：
   - 当 ProxyAllocator 的 freelist 指向 NULL 时
   - 当 allocated 大于 thread\_freelist\_high\_watermark 时
   - 都是需要加锁和解锁的
- 通过 ProxyAllocator
   - 减少了对 ClassAllocator 的直接访问


## freelist 链表的巧妙实现

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

我们前面说过，在释放内存资源的时候，会直接把该内存块放入 freelist 链表，也就是说这个内存块里的数据都是没用的。

在设计一个单链表的时候，我们通常需要定义一个指向下一个节点的指针，而 freelist 为了节省内存的使用，把每个内存块的前 8 个字节（64bits）用来存储```void *```类型的指针，该指针指向下一个内存块地址。

所以我们看到了非常奇怪的一句代码：

```
l.freelist = *(void **)l.freelist;

大约等价于

l.freelist = (void *)l.freelist[0];
```

拆开来逐步分析：

- freelist 声明为 ```void *``` 类型，因此 freelist 指向第一个内存块的地址
- 强制转换为 ```void **``` 类型，此时 freelist 指向的地址是按照顺序一个个排列的 ```void *``` 类型，在 64bits 的系统上 ```sizeof(void *) == 8```
   - 也就是第一个内存块内的空间被按照 8 个字节一片，切割开了
   - 实际上只有第一个分片存储了有用的数据
- 因此 ```*(void **)freelist``` 就是取出第一个内存块的前 8 个字节，再把这 8 个字节当作地址转换为指针，而这个指针就指向第二个内存块


```
*(char **)_p = (char *)_t->_a.freelist;

大约等价于

(char *)_p[0] = (char *)_t->_a.freelist;
```

freelist 实际上是一个堆栈：

- 申请对象内存，实际上就是从链表头部删除一个节点
- 释放对象内存，实际上就是向链表头部增加一个节点
- 典型的先进后出（LIFO）队列
- freelist 总是指向队列的头部

## 方法

```
// 分配对象内存
#define THREAD_ALLOC(_a, _t) thread_alloc(::_a, _t->_a)
#define THREAD_ALLOC_INIT(_a, _t) thread_alloc_init(::_a, _t->_a)
// 释放对象内存
#define THREAD_FREE(_p, _a, _t)                            \
  do {                                                     \
    *(char **)_p = (char *)_t->_a.freelist;                \
    _t->_a.freelist = _p;                                  \
    _t->_a.allocated++;                                    \
    if (_t->_a.allocated > thread_freelist_high_watermark) \
      thread_freeup(::_a, _t->_a);                         \
  } while (0)
```

### THREAD\_ALLOC

从 ProxyAllocator 上分配内存，但是该对象没有被初始化。

```
template <class C>
inline C *
thread_alloc(ClassAllocator<C> &a, ProxyAllocator &l)
{
  if (l.freelist) {
    C *v = (C *)l.freelist;
    l.freelist = *(C **)l.freelist;
    --(l.allocated);
    // 作为一个 C++ 的对象，在其头部有一个虚函数映射表，但是 freelist 把每个
    // 内存块的前 8 个字节修改为指向下一个内存块的指针，这样就把虚函数表给破坏了。
    // 这里利用 ClassAllocator 内保存的原始对象的前 8 个字节来修复虚函数表。
    // 好处：不需要复制整个原始对象的内存空间，节省内存复制的开销。
    // 坏处：该对象没有完成初始化，相当于构造函数没有执行。
    *(void **)v = *(void **)&a.proto.typeObject;
    return v;
  }
  return a.alloc();
}
```

在特殊情况下，对象成员在使用时会全部重新设置一次，也就没有必要运行构造函数进行初始化，例如：

```
#define EVENT_ALLOC(_a, _t) THREAD_ALLOC(_a, _t)
#define EVENT_FREE(_p, _a, _t) \
  _p->mutex = NULL;            \
  if (_p->globally_allocated)  \
    ::_a.free(_p);             \
  else                         \
  THREAD_FREE(_p, _a, _t)
```
接下来的章节中会介绍 Event 对象，Event 对象的成员数量非常的少，而且在使用时，几乎所有的成员都要按需设置，因此没有必要进行初始化。

### THREAD\_ALLOC\_INIT

从 ProxyAllocator 上分配内存，该对象已经完成了初始化。

```
template <class C>
inline C *
thread_alloc_init(ClassAllocator<C> &a, ProxyAllocator &l)
{
  if (l.freelist) {
    C *v = (C *)l.freelist;
    l.freelist = *(C **)l.freelist;
    --(l.allocated);
    // 对比 thread_alloc()，完整复制了整个原始对象的内存空间
    memcpy((void *)v, (void *)&a.proto.typeObject, sizeof(C));
    return v;
  }
  return a.alloc();
}
```

大多数通过 ProxyAllocator 进行内存分配的对象，都是通过该方法完成的。

## 参考资料

- [I_ProxyAllocator.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_ProxyAllocator.h)

