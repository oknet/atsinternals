# 核心部件：ClassAllocator

ClassAllocator是一个C++的类模版，继承自Allocator类。


## 构成

ClassAllocator是一个C++的类模版，继承自Allocator类：

- 扩展了三个方法：alloc，free，free_bulk 输入指针为 Class * 类型，主要是为了适配构造函数
- 与Allocator的区别就是element_size = sizeof(Class)
- 该内存块通常一次性分配，定义为全局变量


## 设计思想

ATS中使用到的各种数据结构，特别是需要在EventSystem与各种Processor之间进行传递的数据结构，都需要分配内存。

对于ATS这种服务器系统，每一个会话进来的时候动态的分配内存，会话结束后释放内存，这种对内存的操作频率是非常高的。

而且在会话存续期间，对于各种类会有各种不同尺寸的内存空间被分配和释放，内存的碎片化也非常严重。

所有的数据结构都使用Allocator类来分配，不符合面向对象的设计习惯，因此ATS设计了ClassAllocator模版。

## 参考资料

- [Allocator.h](http://github.com/apache/trafficserver/tree/master/lib/ts/Allocator.h)

# Allocator

ATS的Allocator是ATS内部使用的一种内存分配管理技术，主要用来减少内存分配次数，从而提高性能。

## 构成

Allocator类是所有ClassAllocator类的基类：
- 它有三个方法
   - alloc_void 封装 ink_freelist_new
   - free_void 封装 ink_freelist_free
   - free_void_bulk 封装 ink_freelist_free_bulk
- 构造函数调用ink_freelist_init创建一整块内存，支持地址空间对齐
- 内存块有一个名字：name
- 内存块的尺寸由：element_size * chunk_size决定
- 该内存块由chunk_size个element_size字节的“小块内存”组成

## 参考资料

- [Allocator.h](http://github.com/apache/trafficserver/tree/master/lib/ts/Allocator.h)
- [ink_queue.h](http://github.com/apache/trafficserver/tree/master/lib/ts/ink_queue.h)
