# 核心部件：ClassAllocator

ClassAllocator 是一个 C++ 的类模版，继承自 Allocator 类：

- 扩展了三个方法：alloc()，free()，free_bulk()
   - 输入指针为 Class * 类型，主要是为了适配构造函数
- 与 Allocator 的区别就是 element_size = sizeof(Class)

ATS 中使用到的各种数据结构，特别是需要在 EventSystem 与 各种状态机 之间进行传递的数据结构，都需要分配内存。

对于 ATS 这种服务器系统，每一个会话进来的时候动态的分配内存，会话结束后释放内存，这种对内存的操作频率是非常高的。

而且在会话存续期间，对于各种对象会有各种不同尺寸的内存空间被分配和释放，内存的碎片化也非常严重。

所有的数据结构都使用 Allocator 类来分配，不符合面向对象的设计习惯，因此ATS设计了 ClassAllocator 模版。

使用 ClassAllocator 模板为指定的数据结构类型（class C）定义一个全局实例，该实例内部有一个 proto 结构体：

```
template <class C> class ClassAllocator : public Allocator
{
...
  struct {
    C typeObject;
    int64_t space_holder;
  } proto;
  uint16_r  proxy_index;
};
```

其中 ```proto.typeObject``` 是一个 ```class C``` 类型的成员。

ClassAllocator 的构造函数负责从操作系统分配到的大块内存，例如每个内存块 16 MB，然后按照```proto.typeObject``` 对象的尺寸切成小块。每一个大块内存将按照 ```sizeof(C)``` 为最小单位，分割为 ```chunk_size``` 个小单元，这些小单元被放入一个 **free pool** (```InkFreeList *fl```) 暂存。

然后，通过 ClassAllocator::alloc() 从 **free pool** 获取一个小块内存单元，ClassAllocator::free() 则用来归还小块内存单元。

这样就减少了直接通过操作系统进行内存内存的次数，而且把相同类型的对象连续存放，减少了内存的碎片化。

## 方法

在 alloc() 时，通过直接将 ```proto.typeObject``` 对象复制（memcpy）到新的内存区域完成对象的创建：

```
  /** Allocates objects of the templated type. */
  C * 
  alloc()
  {
    void *ptr = ink_freelist_new(this->fl);

    memcpy(ptr, (void *)&this->proto.typeObject, sizeof(C));
    return (C *)ptr;
  }
``` 

在 free() 时，则直接将内存块归还至大块内存：

```
  /**
    Deallocates objects of the templated type.

    @param ptr pointer to be freed.
  */
  void
  free(C *ptr)
  {
    ink_freelist_free(this->fl, ptr);
  }
```

因此，通过 ClassAllocator 创建和销毁对象时，构造函数和析构函数都不会被执行。

但是，对于 ```proto.typeObject``` 成员，构造函数和析构函数仍然有效。

## 使用

使用 ClassAllocator 模板为一个特定类型的对象创建内存分配池的时候，该特定对象：

- 构造函数内不得对指针类型的成员进行赋值
   - 例如：不可以 new 一个新的对象，不可 malloc 一块内存，等
   - 如果该指针指向全局共享的对象则是允许的
   - 通常会定义一个 init() 方法，在对象创建之后，再对指针类型的成员进行赋值
- 析构函数通常为空
   - 通常该对象内会定义一个 clear() 方法，用来释放资源
   - 另外再定义一个 destroy() 方法，该方法先调用 clear()，然后调用 ClassAllocator::free(this) 完成资源回收 
- 如包含子对象
   - 其子对象的构造函数和析构函数也要遵循上述要求

在 [TS-4007](https://issues.apache.org/jira/browse/TS-4007) 之后

- 我们可以为 ```class C``` 定义析构函数，但是 ```proto.typeObject``` 的析构函数不会被调用
- 同时 ```proto.typeObject``` 的构造函数仍然会被调用

```
commit e4cb305315e4969748c9dc8414fd96c8d36b4091
Author: Can Selcik <cselcik@linkedin.com>
Date:   Mon Nov 9 22:29:37 2015 -0800

    TS-4007: ClassAllocator: don't attempt to destruct the prototype object when ClassAllocator goes out of scope

diff --git a/lib/ts/Allocator.h b/lib/ts/Allocator.h
index 3b489a7..a21ff90 100644
--- a/lib/ts/Allocator.h
+++ b/lib/ts/Allocator.h
@@ -40,6 +40,7 @@
 #ifndef _Allocator_h_
 #define _Allocator_h_
 
+#include <new>
 #include <stdlib.h>
 #include "ts/ink_queue.h"
 #include "ts/ink_defs.h"
@@ -193,11 +194,12 @@ public:
   */
   ClassAllocator(const char *name, unsigned int chunk_size = 128, unsigned int alignment = 16)
   {
+    ::new ((void*)&proto.typeObject) C();
     ink_freelist_init(&this->fl, name, RND16(sizeof(C)), chunk_size, RND16(alignment));
   }
 
   struct {
-    C typeObject;
+    uint8_t typeObject[sizeof(C)];
     int64_t space_holder;
   } proto;
 };
```


### 定义一个全局实例

```
ClassAllocatror<NameOfClass> nameofclassAllocator("nameofclassAllocator", 256)
```
第一个参数是内存块的名字，第二个参数 256 表示 Allocator 创建大块内存时，每个大块内存可以存放 256 个 NameOfClass 对象。

### 为 NameOfClass 类型的对象分配一个实例

```
NameOfClass *obj = nameofclassAllocator.alloc();
obj->init(...);
```
alloc() 返回的对象，其构造函数已经执行过，再次调用 init() 是为了初始化指针类型的成员。

### 回收内存

```
obj->clear();
nameofclassAllocator.free(obj);
obj = NULL;
```
使用 clear() 方法，主要是为了释放指针类型的成员指向的内存对象。

或者调用 destroy() 方法：

```
obj->destroy();
obj = NULL;
```

## 参考资料

- [Allocator.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/Allocator.h)

# Allocator

Allocator 是 ATS 内部使用的一种内存分配管理技术，主要用来减少内存分配次数，从而提高性能。

另外，通过把相同尺寸的内存块连续存放，也减少了内存的碎片化。

## 构成

Allocator 类是所有 ClassAllocator 类的基类：

- free pool
   - 内存块有一个名字，在构造函数中设置
   - 至少包含一个内存单元
   - 内存单元的尺寸由：element\_size * chunk\_size 决定
   - 该内存单元由 chunk\_size 个 element\_size 字节的“小块内存”组成
   - chunk\_size 默认为 128 个
- 它有三个方法
   - alloc\_void 调用 ink\_freelist\_new 从 free pool 获得一个固定尺寸的内存块
   - free\_void 调用 ink\_freelist\_free 将一个内存块归还给 free pool
   - free\_void\_bulk 调用 ink\_freelist\_free\_bulk 将一组内存块归还给 free pool
- 构造函数调用 ink\_freelist\_init 创建一整块内存，支持地址空间对齐


## 参考资料

- [Allocator.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/Allocator.h)
- [ink_queue.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/ink_queue.h)

