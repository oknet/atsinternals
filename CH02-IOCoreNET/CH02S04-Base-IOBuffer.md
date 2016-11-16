# 基础组件：IOBuffer

IOBuffer是ATS中基本的内存分配和管理系统，就像我们经常使用的malloc()。

在宏定义 DEFAULT_BUFFER_SIZES 中，ATS定义了有多少种尺寸的buf。buf的尺寸由以下宏来定义：

```
#define BUFFER_SIZE_INDEX_128 0
#define BUFFER_SIZE_INDEX_256 1
#define BUFFER_SIZE_INDEX_512 2
#define BUFFER_SIZE_INDEX_1K 3
#define BUFFER_SIZE_INDEX_2K 4
#define BUFFER_SIZE_INDEX_4K 5
#define BUFFER_SIZE_INDEX_8K 6
#define BUFFER_SIZE_INDEX_16K 7
#define BUFFER_SIZE_INDEX_32K 8
#define BUFFER_SIZE_INDEX_64K 9
#define BUFFER_SIZE_INDEX_128K 10
#define BUFFER_SIZE_INDEX_256K 11
#define BUFFER_SIZE_INDEX_512K 12
#define BUFFER_SIZE_INDEX_1M 13
#define BUFFER_SIZE_INDEX_2M 14
```

可以看到，最小128字节，最大2M字节，上面只是一个索引值，那么如何根据索引值计算出buf的字节数？

```
#define DEFAULT_BUFFER_BASE_SIZE 128
#define BUFFER_SIZE_FOR_INDEX(_i) (DEFAULT_BUFFER_BASE_SIZE * (1 << (_i)))
```
可以看到计算公式为: 128*2^idx ，通过宏BUFFER_SIZE_FOR_INDEX(_i)计算出索引值对应的字节数。

在ATS中，每种长度的buf都有它专用的内存池和分配器，通过一个全局数组ioBufAllocator[]来定义，DEFAULT_BUFFER_SIZES是这个数组的成员数量。

```
#define MAX_BUFFER_SIZE_INDEX 14
#define DEFAULT_BUFFER_SIZES (MAX_BUFFER_SIZE_INDEX + 1)
inkcoreapi extern Allocator ioBufAllocator[DEFAULT_BUFFER_SIZES];
```

## IOBuffer 系统的组成

内存管理层由IOBufferData和IOBufferBlock实现

  - 每一个IOBufferBlock内都有一个指向IOBufferData的指针
  - 每一个IOBufferBlock都有一个可以指向下一个IOBufferBlock的指针
  - 下面当我们提及IOBufferBlock的时候，通常是指整个IOBufferBlock的链表，而不是一个Block

数据操作层由IOBufferReader和MIOBuffer实现，这样不用去直接处理IOBufferBlock链表和IOBufferData结构。

  - IOBufferReader负责读取IOBufferBlock里的数据（消费者）
  - MIOBuffer则可以向IOBufferBlock中写入数据（生产者）
  - 同时MIOBuffer也可以包含多个IOBufferReader的指针
    - 这样可以记录一个内存数据，多个消费者的信息和消费情况

数据访问抽象层由MIOBufferAccessor实现

  - MIOBufferAccessor包含了对一个特定IOBufferBlock的读、写操作
    - 一个指向IOBufferReader的指针
    - 一个指向MIOBuffer的指针

```
      IOBufferReader  -----[entry.read]------+ 
        ||     ||                            V
      block   mbuf                   MIOBufferAccessor
        ||     ||                            |
        ||  MIOBuffer <----[mbuf.write]------+
        ||     ||
        ||  _writer
        ||     ||
      IOBufferBlock ===> IOBufferBlock ===> IOBufferBlock ===> NULL
           ||                 ||                 ||
         _data              _data              _data
           ||                 ||                 ||
      IOBufferData       IOBufferData       IOBufferData

|| or = means member reference
|  or - means method path
```

总结一下：

  - 当我们需要一个缓冲区存放数据的时候，就创建一个MIOBuffer实例
  - 然后就可以往MIOBuffer里面写数据了
    - MIOBuffer会自动创建IOBufferBlock和IOBufferData
  - 如果需要从缓冲区读取数据
    - 可以创建IOBufferReader
    - 或者直接通过MIOBuffer读取
  - 当调用do\_io\_read的时候
    - 需要传递MIOBuffer过去，因为要把读取到的数据写入IOBuffer
  - 当调用do\_io\_write的时候
    - 只需要传递IOBufferReader过去，因为要读取IOBuffer里面的数据发送出去
  - 当需要将IOBuffer传递给另一个方法/函数时，而又不确定对方的操作是读还是写
    - 可以传递MIOBufferAccessor实例
    - 例如：VIO就包含了MIOBufferAccessor的实例。

## 自动指针与内存管理

MIOBuffer里的成员```Ptr<IOBufferBlock> _writer;```，指向IOBufferBlock链表的第一个节点。

每次通过write方法向MIOBuffer里写入数据时，就会写入这个链表，当一个节点写满了，就会再增加一个节点。

当我们通过IOBufferReader消费数据时，对于已经消费掉的节点，Ptr指针自动调用该节点的free()方法释放该节点所占用的空间。

这里的free()方法就是用IOBufferBlock::free()，定义如下：

```
TS_INLINE void
IOBufferBlock::free()
{
  dealloc();
  THREAD_FREE(this, ioBlockAllocator, this_thread());
}

TS_INLINE void
IOBufferBlock::dealloc()
{
  clear();
}

TS_INLINE void
IOBufferBlock::clear()
{
  data = NULL;
  IOBufferBlock *p = next;
  while (p) {
    int r = p->refcount_dec();
    if (r)
      break;
    else {
      IOBufferBlock *n = p->next.m_ptr;
      p->next.m_ptr = NULL;
      p->free();
      p = n;
    }
  }
  next.m_ptr = NULL;
  _buf_end = _end = _start = NULL;
}
```

通过上面的源代码，可以清楚的看到调用流程为：free()-->dealloc()-->clear()-->free()-->...

很明显这是一个递归调用，在clear的实现中，为了避免重入，因此需要避免触发Ptr自动指针的重载操作。

在递归调用过程中，递归的深度受到Block链表长度的限制，同时也受到堆栈大小的限制，一旦超出堆栈空间，就会导致Stack Overflow错误？
PS：上面这段是对注释的理解和翻译，但是根据我后面对源代码的分析，这里并不是完全递归的。

因此当我们尝试将很多小的数据块，连接到Block链表时，应该考虑尽可能连接大的数据块，如果必须要连接一个小数据块，那么可以考虑通过数据复制的方式汇总多个小数据块的数据到一个大的数据块。这也是为什么在MIOBuffer里提供了两种write()方法的原因。
PS：参考I_IOBuffer.h的876行及913行。

### 跨线程访问 IOBuffer

由于自动指针会在释放对象时将其归还到当前线程的freelist，而且这个释放操作是根据引用计数自动执行。
如果一个 IOBuffer 对象被跨线程操作，那么就很有可能会导致在线程A里分配的内存，在线程B里执行了释放操作，那么这块内存就被归还到线程B的freelist里。

如果线程A和线程B不在同一个NUMA节点，这样会导致内存访问出现一点点性能问题。

通常在ATS的代码里，总是由创建 IOBuffer 的状态机负责释放并回收它。所以，我们看到 MIOBuffer 被创建后，会立即（必须）创建一个 IOBufferReader，当线程
B需要访问该 MIOBuffer 时，为其分配一个 IOBufferReader，当线程B完成数据的处理后，再返回到线程A的状态机，再通过第一个创建的 IOBufferReader 对象，把MIOBuffer 里的数据消费掉，这样自动指针会在线程A里释放所有 IOBuffer 对象。此时，状态机A称之为主状态机，状态机B称之为从状态机，这也是ATS中多个状态机之间的典型关系，后创建的状态机总是作为附属。

但是有时候，我们需要在两个线程组之间传递数据，并且发起数据传递的状态机A会先于接收数据的状态机B消亡。状态机A创建了 MIOBuffer，并且把数据保存其中，然后传递给状态机B，这样状态机B就成了 MIOBuffer 的所有者。这样就不可避免的需要由状态机B来完成 MIOBuffer 的释放。这种情况在ATS中也是允许的，但是并不推荐。此时，状态机A与状态机B是一种平等的关系，这在ATS中是极少存在的情况。（目前，我阅读了一小部分的ATS代码里，还没有遇到这种模式。）


### IOBufferBlock链表释放过程的递归调用分析（真的会出现Stack Overflow吗？）

首先，假定我们拥有两个IOBufferReader分别为 LA 和 LF

  - 此时A，B，C，E，F的引用计数器都为1，只有D的引用计数器为2。
  - 对应的IOBufferBlock的链表关系如下图：

```
             +-----+------+     +-----+------+     +-----+------+     +-----+------+     +-----+------+
LA.block---->| A 1 | next |---->| B 1 | next |---->| C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+     +-----+------+     +-----+------+     +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

然后，我们调用IOBufferReader::clear()将LA完全释放掉：

  - LA.clear()中执行了block=NULL;

由于block的定义是：```Ptr<IOBufferBlock> block;```

  - 因此上面操作中的等号被Ptr重载
  - Ptr把NULL赋值给A.block
  - Ptr把A的引用计数减1，变成0
  - 当引用计数为0，Ptr会调用A.free()

```
LA.block---->NULL

             +-----+------+     +-----+------+     +-----+------+     +-----+------+     +-----+------+
             | A 0 | next |---->| B 1 | next |---->| C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+     +-----+------+     +-----+------+     +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

进入A.free()方法后，会首先由A.clear()方法进行操作

  - A.data=NULL; 释放IOBufferData
  - 由于A.next指向B，所以如果A被释放了，也需要对B的引用计数减1
  - 然后发现B的引用计数也变成0了，B也要释放
  - 在递归调用B.free()之前
     - 首先把C.m_ptr保存到n，这样不至于在释放B之后丢掉了C的地址
     - 然后把B.next指向NULL，就是断开与C的指向

```
LA.block---->NULL

             +-----+------+     +-----+------+     +-----+------+     +-----+------+     +-----+------+
             | A 0 | next |---->| B 0 | NULL |     | C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+     +-----+------+     +-----+------+     +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

进入B.free()方法后，会首先由B.clear()方法进行操作

  - B.data=NULL; 释放IOBufferData
  - 由于B.next已经被指向NULL，因此跳过while部分
  - 将B.next.m_ptr指向NULL
  - 从B.clear()方法返回B.free()，执行THREAD_FREE()释放B
  - 返回A.clear()继续执行

```
LA.block---->NULL

             +-----+------+                        +-----+------+     +-----+------+     +-----+------+
             | A 0 | next |---->????               | C 1 | next |---->| D 2 | next |---->| E 1 | NULL |
             +-----+------+                        +-----+------+     +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

重新返回到A.clear()

  - 把C的地址从n中取回来，将其赋值给p，因此p=C，然后继续while循环
  - 其实这里你可以认为A.next指向了C，因为A.clear()初次while循环时p=A.next=B
  - 然后将C的引用减1，也变成0了，B也要释放
  - 在递归调用C.free()之前
     - 首先把D.m_ptr保存到n，这样不至于在释放C之后丢掉了D的地址
     - 然后把C.next指向NULL，就是断开与D的指向

```
LA.block---->NULL

             +-----+------+                        +-----+------+     +-----+------+     +-----+------+
             | A 0 | next |---->????               | C 0 | NULL |     | D 2 | next |---->| E 1 | NULL |
             +-----+------+                        +-----+------+     +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

进入C.free()方法后，会首先由C.clear()方法进行操作

  - C.data=NULL; 释放IOBufferData
  - 由于C.next已经被指向NULL，因此跳过while部分
  - 将C.next.m_ptr指向NULL
  - 从C.clear()方法返回C.free()，执行THREAD_FREE()释放C
  - 返回A.clear()继续执行

```
LA.block---->NULL

             +-----+------+                                           +-----+------+     +-----+------+
             | A 0 | next |---->????                                  | D 2 | next |---->| E 1 | NULL |
             +-----+------+                                           +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

重新返回到A.clear()

  - 把D的地址从n中取回来，将其赋值给p，因此p=D，然后继续while循环
  - 然后将D的引用减1，发现D的引用计数没有变成0而是1，说明还有其它链表使用了C，因此不能继续释放C
  - 将A.next.m_ptr指向NULL
  - 从A.clear()方法返回A.free()，执行THREAD_FREE()释放A

```
LA.block---->NULL

                                                                      +-----+------+     +-----+------+
                                                                      | D 1 | next |---->| E 1 | NULL |
                                                                      +-----+------+     +-----+------+
                                                                         ^
             +-----+------+                                              |
LF.block---->| F 1 | next |----------------------------------------------+
             +-----+------+
```

这里的示例使用的链表比较小，但是仍然可以看到释放的过程是一个递归的过程：

  - 因为，在当前节点释放后，next节点的引用也需要减1
  - 所以，在释放当前节点的时候，首先看一下next节点是否存在
     - 如果存在，就对next节点的引用计数减1
     - 如果在减1之后，next节点的引用计数变成0了，那么就需要先释放next节点指向的元素。
  - 释放该元素的流程如下：
     - 首先，保存被释放元素的next节点到n，并将next节点指向NULL，这样就使得next节点成为一个孤节点。
     - 然后，通过free()释放next节点
     - 然后，将n作为next节点，循环做判断。
  - 所以对于一个block=A[1]->B[1]->C[1]->D[2]->E[1]->NULL的链表（括号内表示引用计数器的值），在free整个链表的时候的调用栈如下：

```
block=NULL
A[1] ==> A[0]
A[0]->free()
   A[0]->dealloc()
      A[0]->clear()
         p=A[0].next.m_ptr=B[1].m_ptr;
         while(p) {
         B[1] ==> B[0];
         n=B[0].next.m_ptr=C[1].m_ptr;
         B[0].next.m_ptr=NULL;
         B->free()
            B->dealloc()
               B->clear()
               p=B[0].next.m_ptr=NULL;
               B[0].next.m_ptr=NULL;
            THREAD_FREE(B);
         p=n=C[1].m_ptr;
         }
         while(p) {
         C[1] ==> C[0];

         n=C[0].next.m_ptr=D[2].m_ptr;
         C[0].next.m_ptr=NULL;
         C->free()
            C->dealloc()
               C->clear()
               p=C[0].next.m_ptr=NULL;
               C[0].next.m_ptr=NULL;
            THREAD_FREE(C);
         p=n=D[2].m_ptr;
         }
         while(p) {
         D[2] ==> D[1];
         break;
         }
         A[0].next.m_ptr=NULL;
   THREAD_FREE(A);
```

所以，如果F.next指向的是D，上面LA.block的释放过程就是B，C，A，元素D，E不会被释放。

在释放过程中，并不是完全递归释放的，第一个节点是在最后释放，剩余的节点是按链表顺序逐个释放，最深的调用栈是6层。

由于三个关键函数都使用了INLINE声明，因此实际的调用栈可能会更浅，那么在MIOBuffer的实际使用中，并不会出现IOBufferBlock链表过长，在释放链表时导致的Stack Overflow问题。

## 参考资料
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)

## 基础组件：IOBufferData

IOBufferData是最底层的内存单元，实现了内存的分配和释放，对引用进行计数。

  - 通过继承RefCountObj，从而支持引用计数器。
  - 为实现引用计数功能，被其它类包含时，总是定义为智能指针，如：Ptr<IOBufferData> data;
  - 成员变量 \_data 指向通过通过全局变量 ioBufAllocator[] 分配的内存空间。
  - AllocType枚举值定义了该对象管理的对象所采用的内存分配类型
    - NO_ALLOC
    - FAST_ALLOCATED
    - XMALLOCED
    - MEMALIGNED
    - DEFAULT_ALLOC
    - CONSTANT
  - 可以通过 new_IOBufferData() 创建一个实例
  - 可以通过 new_xmalloc_IOBufferData() 创建一个实例
  - 可以通过 new_constant_IOBufferData() 创建一个实例
  - 它的实例占用的内存空间由 ioDataAllocator 分配

### 定义

```
class IOBufferData : public RefCountObj
{
public:
  // 返回分配的内存大小
  int64_t block_size();

  // 释放之前由alloc分配，由该类管理的内存。
  void dealloc();

  // 根据size_index和type来分配内存，如果之前已经分配过，会先执行dealloc操作
  void alloc(int64_t size_index, AllocType type = DEFAULT_ALLOC);

  // 返回 _data 成员
  char *
  data()
  {
    return _data;
  }

  // 重载 char * 操作，返回 _data 成员
  operator char *() { return _data; }

  // 释放 IOBufferData 实例自身
  // 首先执行dealloc，然后释放IOBuffeData对象自身（通过 ioDataAllocator 回收内存资源）
  // 因此,执行该方法后就不能再使用和引用该对象了
  virtual void free();

  // 表示内存块的字节数，通过公式128*2^size_index 来得出
  int64_t _size_index;

  // 内存分配类型，AllocType枚举值
  // NO_ALLOC 表示当前未分配，用于延时分配。
  AllocType _mem_type;

  // 指向内存块的指针
  char *_data;

#ifdef TRACK_BUFFER_USER
  const char *_location;
#endif

  /**
    Constructor. Initializes state for a IOBufferData object. Do not use
    this method. Use one of the functions with the 'new_' prefix instead.

  */
  // 构造函数，初始化IOBufferData
  // 但是不要直接使用这个方法，需要时，请通过 new_IOBufferData 获得一个实例
  IOBufferData()
    : _size_index(BUFFER_SIZE_NOT_ALLOCATED), _mem_type(NO_ALLOC), _data(NULL)
#ifdef TRACK_BUFFER_USER
      ,
      _location(NULL)
#endif
  {
  }

private:
  // declaration only
  IOBufferData(const IOBufferData &);
  IOBufferData &operator=(const IOBufferData &);
};

// 声明一个全局分配此类型实例的 ClassAllocator
inkcoreapi extern ClassAllocator<IOBufferData> ioDataAllocator;
```

## 基础组件：IOBufferBlock

IOBufferBlock 用于链接多个IOBufferData，构成更大的存储单元，实现可伸缩的内存管理。

  - 是一个单链表，成员 智能指针 next 指向下一个IOBufferBlock
  - 成员 智能指针 data 是一个IOBufferData类型的内存块
  - 通过成员 \_start 描述了IOBufferData内存块中的数据起始位置
  - 通过成员 \_end 描述了IOBufferData内存块中的数据结束位置
  - 通过成员 \_buf\_end 描述了IOBufferData内存块的结束边界，空闲可用空间为_end到_end_buf之间的部分。
  - 因此一个IOBufferBlock是对IOBufferData使用情况的描述，同时它提供了数个操作IOBufferData的方法。
  - 将多个IOBufferBlock连接起来，可以将多个IOBufferData里的数据组合成更大的buf。
  - 通过全局变量 ioBlockAllocator 可以创建IOBufferBlock的实例
  - MIOBuffer 通过挂载一个 IOBufferBlock 结构，可以知道缓冲区内的使用情况（哪一部分正在使用以及哪一部分是可用的）。
  - 不能在buffer之间共享。（The IOBufferBlock is not sharable between buffers）\*\*
  - 可以通过 new_IOBufferBlock() 创建一个实例
  - 它的实例占用的内存空间由 ioBlockAllocator 分配

### 定义

```
class IOBufferBlock : public RefCountObj
{
public:
  // 返回指向底层数据块 IOBufferData 的指针
  char *
  buf()
  {
    return data->_data;
  }

  // 返回 _start 成员变量，指向正在使用数据区域的开始位置的指针
  char *
  start()
  {
    return _start;
  }

  // 返回 _end 成员变量，指向正在使用数据区域的结束位置的指针
  char *
  end()
  {
    return _end;
  }

  // 返回 _buf_end 成员变量，指向底层数据块结束位置的指针
  char *
  buf_end()
  {
    return _buf_end;
  }

  // 返回 _end - _start 的值，表示正在使用的数据区域的大小
  int64_t
  size()
  {
    return (int64_t)(_end - _start);
  }

  // 返回 _end - _start 的值，表示对于读操作可以读取的长度，与size()等价
  int64_t
  read_avail()
  {
    return (int64_t)(_end - _start);
  }

  // 返回 _buf_end - _end 的值，表示对于写操作可以继续写入的长度，表示可用的空间大小
  int64_t
  write_avail()
  {
    return (int64_t)(_buf_end - _end);
  }

  // 返回底层数据块的大小，调用 IOBufferData->block_size() 方法
  int64_t
  block_size()
  {
    return data->block_size();
  }

  // 减少正在使用的数据区域，_start -= len
  void consume(int64_t len);

  // 增加正在使用的数据区域，_end＋＝len
  // 但是，首先要用end()获取当前的数据区域的结尾，然后拷贝数据到结尾后，再调用此方法
  // 注意，不能超过 _buf_end，就是 len <= write_avail()
  void fill(int64_t len);

  // 重置正在使用的数据区域，_start＝_end＝buf()，_buf_end＝buf()＋block_size()
  // 重置之后，read_avail()＝＝0，write_avail()＝＝block_size()
  void reset();

  // 克隆IOBufferBlock，但是底层数据块IOBufferData不会被克隆，所以克隆出来的IOBufferBlock实例引用同一个底层数据块
  // 注意，克隆出来的IOBufferBlock实例的 write_avail()＝＝0，就是buf_end＝end
  IOBufferBlock *clone();

  // 清除底层数据快，注意与reset()的区别
  // 本操作通过 data＝NULL 只断开当前Block与底层数据块的连接，底层数据块是否被释放／回收，由其引用计数决定。
  // 由于IOBufferBlock是一个链表，因此要递归将 next 的引用计数减少，如果减为0时，还要调用 free() 释放 next 指向的Block
  // 最后，data＝_buf_end＝_end＝_start＝next＝NULL
  // 事实上可以把 clear 看做析构函数，除了不释放Block自身
  // 如果需要重新使用这个Block，可以通过 alloc() 重新分配底层数据块
  void clear();

  // 分配一块长度索引值为 i 的 buffer 给 data，并通过 reset() 方法初始化
  void alloc(int64_t i = default_large_iobuffer_size);

  // 直接调用 clear() 方法
  void dealloc();

  // 将IOBufferData与该Block实例建立引用关系
  // 可以通过 len 和 offset 指定仅引用Data的一部分
  // _start＝buf()＋offset，_end＝_start＋len，_buf_end＝buf()＋block_size()
  // 注：源代码有一个古老的bug，在写这篇笔记时被发现，TS-3754
  void set(IOBufferData *d, int64_t len = 0, int64_t offset = 0);
  // 通过内部调用建立一个没有立即分配内存块的IOBufferData实例
  // 然后将已经分配好的内存指针赋值给IOBufferData实例，其它与set相同
  void set_internal(void *b, int64_t len, int64_t asize_index);
  
  // 把当前数据复制到 b，然后调用 dealloc() 释放数据块，然后调用set_internal()
  // 最后让新数据块的size()与原数据块一致: _end = _start + old_size
  void realloc_set_internal(void *b, int64_t buf_size, int64_t asize_index);
  // 同：realloc_set_internal(b, buf_size, BUFFER_SIZE_NOT_ALLOCATED)
  void realloc(void *b, int64_t buf_size);
  // 通过 ioBufAllocator[i].alloc_void() 来分配一个缓冲区 b，然后调用realloc_set_internal()
  void realloc(int64_t i);
  
  // xmalloc分配模式，未做分析 **
  void realloc_xmalloc(void *b, int64_t buf_size);
  void realloc_xmalloc(int64_t buf_size);

  // 释放IOBufferBlock实例自身
  // 首先调用dealloc，然后通过 ioBlockAllocator 回收内存。
  virtual void free();

  // 指向数据区内可读取的第一个字节
  char *_start;
  // 指向数据区内可写入的第一个字节，也是最后可读取字节的下一个字节
  char *_end;
  // 指向整个数据区的最后的位置，边界，此位置已经不能写入数据了
  char *_buf_end;

#ifdef TRACK_BUFFER_USER
  const char *_location;
#endif

  // 指向IOBufferData类型的智能指针，上述_start, _end, _buf_end指针范围都落在其成员 _data 内
  // 若更改 data 的指向，则必须重新设置 _start, _end, _buf_end
  Ptr<IOBufferData> data;

  // 为了形成Block链表，next 是指向下一个Block的智能指针
  Ptr<IOBufferBlock> next;

  // 构造函数，初始化IOBufferBlock
  // 但是不要直接使用这个方法，需要时，请通过 new_IOBufferBlock 获得一个实例
  IOBufferBlock();

private:
  IOBufferBlock(const IOBufferBlock &);
  IOBufferBlock &operator=(const IOBufferBlock &);
};

// 声明一个全局分配此类型实例的 ClassAllocator
extern inkcoreapi ClassAllocator<IOBufferBlock> ioBlockAllocator;
```

## 基础组件：IOBufferReader

IOBufferReader

  - 不依赖MIOBuffer
    - 当多个读取者从同一个MIOBuffer中读取数据时，每一个读取者都需要标记自己从哪里开始读，总共读多少数据，当前读了多少
    - 此时不能直接修改MIOBuffer中的指针，而通过IOBufferReader来描述这些元素
  - 用于读取一组IOBufferBlock。
    - 通过MIOBuffer来创建IOBufferReader时，是直接从MIOBuffer中复制IOBufferBlock的成员
    - 因此实际上是对IOBufferBlock进行读取操作
  - IOBufferReader表示一个给定的缓冲区数据的消费者从哪儿开始读取数据。
  - 提供了一个统一的界面，可以轻松访问一组IOBufferBlock内包含的数据。
  - IOBufferReader 内部封装了自动移除数据块的判断逻辑。
    - consume 方法中block=block->next的操作会导致block的引用计数变化，通过自动指针Ptr实现对Block和Data的自动释放。
  - 简单的说：IOBufferReader将多个IOBufferBlock单链表作为一个大的缓冲区，提供了读取/消费这个缓冲区的一组方法。
  - 内部成员 智能指针 block 指向多个IOBufferBlock构成的单链表的第一个元素。
  - 内部成员 mbuf 指回到创建此IOBufferReader实例的MIOBuffer实例。

看上去 IOBufferReader 是可以直接访问 IOBufferBlock 链表的，但是其内部成员 mbuf 又决定了它是为 MIOBuffer 设计的。

### 定义

```
class IOBufferReader
{
public:
  // 返回可供消费（读取）数据区域的开始位置（通过成员 start_offset 协助）
  // 返回 NULL 表示没有关联数据块
  char *start();

  // 返回可供消费（读取）数据在第一个数据块（IOBufferBlock）里的结束位置
  // 返回 NULL 表示没有关联数据块
  char *end();

  // 返回当前所有数据块里面剩余可消费（读取）的数据长度
  // 遍历所有的数据块，累加每一个数据块内的可用数据长度再减去代表已经消费数据长度的 start_offset
  int64_t read_avail();

  // 返回当前所有数据块里面剩余可消费（读取）的数据长度是否大于 size
  bool is_read_avail_more_than(int64_t size);

  // 返回当前所有数据块里面剩余可消费（读取）的数据块的数量
  // 随着数据的消费（读取）：
  //     成员 block 逐个指向链表中的下一个 IOBufferBlock，
  //     成员 start_offset 也会按照新的Block的信息重新设置
  int block_count();

  // 返回可供消费（读取）数据在第一个数据块里的长度
  int64_t block_read_avail();

  // 根据 start_offset 的值，跳过不需要的 block
  // start_offset 的值必须在 [ 0, block->size() ) 范围内
  void skip_empty_blocks();

  // 清除所有成员变量，IOBufferReader将不可用
  void clear();

  // 重置IOBufferReader的状态，成员 mbuf 和 accessor 不会被重置
  // 只初始化 block，start_offset, size_limit 三个成员
  void reset();

  // 记录消费 n 字节数据，n 必须小于 read_avail()
  // 在消费时，自动指针 block 会逐个指向 block->next
  // 本函数只是记录消费的状态，具体数据的读取操作，仍然要访问成员mbuf，或者block里的底层数据块来进行
  void consume(int64_t n);

  // 克隆当前实例，复制当前的状态和指向同样的IOBufferBlock，以及同样的start_offset值
  // 通过直接调用 mbuf->clone_reader(this) 来实现
  IOBufferReader *clone();

  // 释放当前实例，之后就不能再使用该实例了
  // 通过直接调用 mbuf->dealloc_reader(this) 来实现
  void dealloc();

  // 返回 block 成员
  IOBufferBlock *get_current_block();

  // 当前剩余可写入空间是否处于低限状态
  // 直接调用 mbuf->current_low_water() 实现
  // 判断规则如下：
  //     在不增加block的情况下，当前MIOBuffer类型的成员 mbuf 可写入的空间，
  //     如果低于water_mark值，则返回true，否则返回false
  // return mbuf->current_write_avail() <= mbuf->water_mark;
  bool current_low_water();

  // 当前剩余可写入空间是否处于低限状态
  // 直接调用 mbuf->low_water() 实现
  // 与 current_low_water() 相似，但是在返回ture时表示追加了一个空的block到 mbuf
  // return mbuf->write_avail() <= mbuf->water_mark;
  bool low_water();

  // 当前可读取数据是否高于低限
  // return read_avail() >= mbuf->water_mark;
  bool high_water();

  // 在 IOBufferBlock 链表上执行 memchr 的操作，但是在ATS中好像没有使用到这个方法
  // 返回 -1 表示没有找到 c，否则返回 c 首次在 IOBufferBlock 链表中出现的偏移值
  inkcoreapi int64_t memchr(char c, int64_t len = INT64_MAX, int64_t offset = 0);

  // 从当前的 IOBufferBlock 链表，复制 len 长度的数据到 buf，buf 必须事先分配好空间
  // 如果 len 超过了当前可读取的数据长度，则使用当前可读取数据的长度作为 len 值
  // 通过 consume() 消费已经读取完成的Block
  // 返回实际复制的数据长度
  inkcoreapi int64_t read(void *buf, int64_t len);

  // 从当前的 IOBufferBlock 链表的当前位置，偏移 offset 字节开始，复制 len 长度的数据到 buf
  // 但是与 read() 不同，该操作不执行 consume() 操作
  // 返回指针，指向 memcpy 向 buf 中写入数据的结尾；如果没有发生写操作，则等于 buf
  // 例如，当offset超过当前可读数据的最大值，返回值就等于 buf
  inkcoreapi char *memcpy(const void *buf, int64_t len = INT64_MAX, int64_t offset = 0);

  /**
    Subscript operator. Returns a reference to the character at the
    specified position. You must ensure that it is within an appropriate
    range.

    @param i positions beyond the current point of the reader. It must
      be less than the number of the bytes available to the reader.

    @return reference to the character in that position.

  */
  // 重载下标操作符 reader[i]，可以像使用数组一样使用IOBufferReader
  // 使用时需要注意不要让 i 超出界限，否则会抛出异常
  // 这里没有使用 const char 来声明返回值的类型，但是我们仍然应该只从 IOBufferReader 中读取数据，
  //     虽然没有硬性限制不可向其写入，但是实际上这里不应该使用 reader[i] = x; 的代码
  char &operator[](int64_t i);

  // 返回成员 mbuf
  MIOBuffer *
  writer() const
  {
    return mbuf;
  }
  // 当一个IOBufferReader已经与一个MIOBuffer关联之后，就会设置mbuf指向该MIOBuffer
  // 通过这个方法来判断当前的IOBufferReader是否已经与MIOBuffer关联
  // 返回与之关联的MIOBuffer指针，或NULL表示未关联
  MIOBuffer *
  allocated() const
  {
    return mbuf;
  }

  // 如果与 MIOBufferAccessor 关联，则指向 MIOBufferAccessor
  MIOBufferAccessor *accessor; // pointer back to the accessor

  // 指向分配此 IOBufferReader 的MIOBuffer
  MIOBuffer *mbuf;
  // 智能指针 block 在初始情况指向 MIOBuffer 里的 IOBufferBlock 的链表头
  // 随着 consume() 的使用，block 逐个指向 block->next
  Ptr<IOBufferBlock> block;

  // start_offset 用来标记当前可用数据与 block 成员的偏移位置
  // 每次 block＝block->next，start_offset都会重新计算
  // 通常情况下 start_offset 不会大于当前 block 的可用数据长度
  // 如果超过，则在 skip_empty_blocks() 中会跳过无用的 block 并修正该值
  int64_t start_offset;
  
  // 好像没什么用处，在初始化的时候、reset()时被设置为INT64_MAX
  // 其它对 size_limit 修改的地方都是判断不等于INT64_MAX的情况才做修改
  int64_t size_limit;

  // 构造函数
  IOBufferReader() : accessor(NULL), mbuf(NULL), start_offset(0), size_limit(INT64_MAX) {}
};
```

## 基础组件：MIOBuffer

MIOBuffer

  - 它是一个单一写（生产者），多重读（消费者）的内存缓冲区。
  - 是所有IOCore数据传输的中心。
  - 它是用于VConnection接收和发送操作的数据缓冲区。
  - 它指向一组IOBufferBlock，IOBufferBlock又指向包含实际数据的IOBufferData结构
  - MIOBuffer允许一个生产者和多个消费者。
  - 它写入（生产）数据的速度取决于最慢的那个数据读取（消费）者。
  - 它支持多个不同速度的读取（消费）者之间的自动流量控制。
  - IOBuffer内的数据是不可改变的，一旦写入就不能修改。
  - 在所有的读取（消费）完成后，一次性释放。
  - 由于数据（IOBufferData）可以在buffer之间共享
    - 多个IOBufferBlock就可能引用了同一个数据（IOBufferData），
    - 但是只有一个具有所有权，可以写入数据，如果允许修改IOBuffer内的数据，就会导致混乱。
  - 成员 IOBufferReader readers[MAX_MIOBUFFER_READERS] 定义了多重读（消费者），默认最大值5。
  - 可以通过 new\_MIOBuffer() 创建一个实例，free\_MIOBuffer(mio) 销毁一个实例。
  - 它的实例占用的内存空间由 ioAllocator 分配

### 定义

```
#define MAX_MIOBUFFER_READERS 5

class MIOBuffer
{
public:
  // MIOBuffer 的写操作部分

  // 将当前正在使用的数据区域扩大 len 字节（增加了可读数据量，减少了可写空间）
  // 直接对IOBufferBlock链表 _writer 成员进行操作，如果当前剩余的可写空间不足，会自动追加空的 block
  void fill(int64_t len);

  // 向IOBufferBlock链表 _writer 成员追加 block
  // Block *b必须是当前MIOBuffer可写，其它MIOBuffer不可写
  void append_block(IOBufferBlock *b);

  // 向IOBufferBlock链表 _writer 成员追加指定大小的空 block
  void append_block(int64_t asize_index);

  // 向IOBufferBlock链表 _writer 成员追加当前MIOBuffer默认大小的空 block
  // 直接调用了 append_block(size_index)
  void add_block();

  /**
    Adds by reference len bytes of data pointed to by b to the end
    of the buffer.  b MUST be a pointer to the beginning of  block
    allocated from the ats_xmalloc() routine. The data will be deallocated
    by the buffer once all readers on the buffer have consumed it.

  */
  void append_xmalloced(void *b, int64_t len);

  /**
    Adds by reference len bytes of data pointed to by b to the end of the
    buffer. b MUST be a pointer to the beginning of  block allocated from
    ioBufAllocator of the corresponding index for fast_size_index. The
    data will be deallocated by the buffer once all readers on the buffer
    have consumed it.

  */
  void append_fast_allocated(void *b, int64_t len, int64_t fast_size_index);

  // 将 rbuf 内，长度为 nbytes 字节的数据，写入 IOBufferBlock 成员 _writer
  // 返回值为实际发生的写入字节数
  // 不检测 watermark 和 size_limit，但是剩余可写空间不足时会追加 block。
  // 如果需要流量控制，调用者需要自己实现，本方法没有实现流量控制功能
  // PS：write 是多态定义，下面还有一种 write 的定义
  inkcoreapi int64_t write(const void *rbuf, int64_t nbytes);

#ifdef WRITE_AND_TRANSFER
  // 基本与下面的 write() 相同
  // 但是把最后一个克隆出来的 block 的写入权限从 IOBufferReader *r 引用的MIOBuffer转到了当前MIOBuffer
  // PS: 每一次追加到 block 链表，_writer 成员会指向新添加的这个 block，因为 _writer 成员总是指向第一个可写入的 block
  inkcoreapi int64_t write_and_transfer_left_over_space(IOBufferReader *r, int64_t len = INT64_MAX, int64_t offset = 0);
#endif

  // 跳过 IOBufferReader 开始的 offset 字节的数据后，通过 clone() 复制当前 block，
  // 通过 append_block() 将新的 block 追加到 IOBufferBlock 成员 _writer，
  // 直到完成长度为 len 的数据（或剩余全部数据／INT64_MAX）的处理。
  // 返回值为实际发生的写入字节数
  // 以下为源代码中的注释直接翻译：
  // 不检测 watermark 和 size_limit，但是剩余可写空间不足时会追加 block。
  // 如果需要流量控制，调用者需要自己实现，本方法没有实现流量控制功能
  // 即使一个 block 中的可用数据只有 1 个字节，也会克隆整个 block，
  // 因此，当在两个 MIOBuffer 之间传输数据时需要小心处理，
  // 尤其是源MIOBuffer的数据是从网络上读取得到时，因为接收的数据可能都是很小的数据块。
  // 由于 Block 的释放是一个递归过程，当有太多的 Block 在链表中时，如果释放可能导致调用栈溢出（这里应该有错误，见上面的分析）
  // 在使用 write() 方法时，建议由调用者来进行流量控制，避免链表内的 block 太多，另外要控制使用克隆方式传送数据的最小字节数。
  // 如果遇到大量小数据块的传送，建议使用第一种write()方法复制数据，而不是使用克隆 block 的方式。
  // PS: 不会修改 IOBufferReader *r 的数据消费情况。
  inkcoreapi int64_t write(IOBufferReader *r, int64_t len = INT64_MAX, int64_t offset = 0);

  // 把 IOBufferReader *r 里面的 block 链表转移到当前MIOBuffer
  // IOBufferReader *r 原来的 block 链接则会被清空
  // 返回所有被转移的数据长度
  // 基本上可以认为这是一个concat操作
  int64_t remove_append(IOBufferReader *);

  // 返回第一个可写的 block
  // 通常 _writer 指向第一个可写的 block，
  // 但是极特殊情况，当前 block 被写满了，那么就是 block->next 是第一个可写的 block
  // 如果返回 NULL 表示没有可写的 block
  IOBufferBlock *
  first_write_block()
  {
    if (_writer) {
      if (_writer->next && !_writer->write_avail())
        return _writer->next;
      ink_assert(!_writer->next || !_writer->next->read_avail());
      return _writer;
    } else
      return NULL;
  }

  // 返回第一个可写的 block 的 buf()
  char *
  buf()
  {
    IOBufferBlock *b = first_write_block();
    return b ? b->buf() : 0;
  }
  // 返回第一个可写的 block 的 buf_end()
  char *
  buf_end()
  {
    return first_write_block()->buf_end();
  }
  // 返回第一个可写的 block 的 start()
  char *
  start()
  {
    return first_write_block()->start();
  }
  // 返回第一个可写的 block 的 end()
  char *
  end()
  {
    return first_write_block()->end();
  }

  // 返回第一个可写的 block 的剩余可写空间
  int64_t block_write_avail();

  // 返回所有 block 的剩余可写空间
  // 此操作不会追加空的 block 到链表尾部
  int64_t current_write_avail();

  // 返回所有 block 的剩余可写空间
  // 如果：可读数据小于 watermark值 并且 剩余可写空间小于等于 watermark 值
  //     那么自动追加空的 block 到链表尾部
  // 返回值也包含了新追加的 block 的空间
  int64_t write_avail();

  // 返回该MIOBuffer申请 block 时使用的缺省尺寸
  // 该值在创建MIOBuffer时传入由构造函数初始化，保存在成员 size_index 中。
  int64_t block_size();

  // 同block_size，貌似被废弃了，没有见到代码中有使用
  int64_t
  total_size()
  {
    return block_size();
  }

  // 可读取数据长度超过 watermark 值时返回 true
  bool
  high_water()
  {
    return max_read_avail() > water_mark;
  }

  // 可写空间小于等于 water_mark 值时返回 true
  // 此方法通过 write_avail 来获得当前剩余可写空间的大小，因此可能会追加空的 block
  bool
  low_water()
  {
    return write_avail() <= water_mark;
  }

  // 可写空间小于等于 water_mark 值时返回 true
  // 此方法不会追加空的 block
  bool
  current_low_water()
  {
    return current_write_avail() <= water_mark;
  }
  
  // 根据size的值选择一个最小的值，用来设置成员 size_index
  // 例如：
  //     当size＝128时，选择size_index为0，此时计算出来的block size＝128
  //     当size＝129时，选择size_index为1，此时计算出来的block size＝256
  void set_size_index(int64_t size);


  // MIOBuffer 的读操作部分
  
  // 从 readers[] 成员分配一个可用的IOBufferReader，并且设置该Reader的accessor成员指向anAccessor
  // 一个Reader必须与MIOBuffer关联，如果一个MIOBufferAccessor与MIOBuffer关联，
  //     那么与MIOBuffer关联的IOBufferReader也必须与MIOBufferAccessor关联
  IOBufferReader *alloc_accessor(MIOBufferAccessor *anAccessor);

  // 从 readers[] 成员分配一个可用的IOBufferReader
  // 与alloc_accessor不同，本方法将accessor成员指向NULL
  // 当一个MIOBuffer创建后，必须创建至少一个IOBufferReader。
  // 如果在MIOBuffer中填充了数据之后才创建IOBufferReader，那么IOBufferReader可能无法从数据的开头进行读取。
  // PS: 由于使用了自动指针Ptr，而 _writer 又总是指向第一个可写的block，
  //     那么随着 _writer 指针的移动，最早创建的block就会失去它的引用者，就会被Ptr自动释放
  //     所以在创建MIOBuffer之后，首先就要创建一个IOBufferReader，
  //     以保证最早创建的block被引用，这样就不会被Ptr自动释放了
  IOBufferReader *alloc_reader();

  // 从 readers[] 成员分配一个可用的IOBufferReader
  // 然后从 r 的成员 block, start_offset, size_limit 复制到刚分配的IOBufferReader
  // 但是需要注意，r 必须是readers[] 成员中的一个，否则对block 的访问会出现问题
  IOBufferReader *clone_reader(IOBufferReader *r);

  // 释放IOBufferReader，但是 e 必须是readers[] 成员中的一个
  // 如果 e 关联了 accessor，那么还会调用 e->accessor->clear()，那么 accessor 就无法对MIOBuffer进行读、写操作了
  // 然后调用 e->clear()，在clear中必须设置mbuf＝NULL
  void dealloc_reader(IOBufferReader *e);

  // 逐个释放 readers[] 的每一个成员
  void dealloc_all_readers();


  // MIOBuffer 的设置／初始化部分
  
  // 使用一个预分配的数据块初始化MIOBuffer
  // 并且将已经分配的IOBufferReader也指向此数据块
  void set(void *b, int64_t len);
  void set_xmalloced(void *b, int64_t len);
  // 使用指定的size_index值创建一个空内存块，初始化MIOBuffer
  // 并且将已经分配的IOBufferReader也指向此空内存块
  // 同时会使用新的size_index值更新MIOBuffer成员size_index
  void alloc(int64_t i = default_large_iobuffer_size);
  void alloc_xmalloc(int64_t buf_size);
  // 追加 IOBufferBlock *b 到 _writer->next，然后让 _writer 指向 b
  // 注意，这个操作会导致 _writer->next 的引用计数减少
  // 如果 b->next 不为 NULL，则遍历 _writer，直到 _writer->next == NULL
  void append_block_internal(IOBufferBlock *b);
  // 向当前数据块写入以 \0 或 \n 结尾的字符串 s，最大长度 len，写入内容包括 \0 或 \n
  // 返回0，如果当前数据块的剩余可写空间不足
  // 返回-1，如果在len长度内仍然没有找到 \0 或 \n
  // 返回实际写入的长度，当成功时
  // PS：len 包含了 \0 或 \n 的1个字节在内
  int64_t puts(char *buf, int64_t len);


  // 以下仅限内部使用

  // 判断是否为空MIOBuffer，就是没有block
  bool
  empty()
  {
    return !_writer;
  }
  int64_t max_read_avail();

  // 遍历 readers[] 成员内每一个Reader的block数量，找到最大值
  int max_block_count();
  
  // 根据 water_mark 值，判断是否需要追加block
  // 如果需要则追加
  // 相关逻辑请参阅：理解 water_mark
  void check_add_block();

  // 获得第一个可写入的 block
  IOBufferBlock *get_current_block();

  // 分别调用 block 和 所有 reader 的 reset() 重置MIOBuffer
  void
  reset()
  {
    if (_writer) {
      _writer->reset();
    }
    for (int j = 0; j < MAX_MIOBUFFER_READERS; j++)
      if (readers[j].allocated()) {
        readers[j].reset();
      }
  }

  // 初始化已分配的 reader
  // 将 reader 的 block 指向 _writer 引用的 IOBufferBlock 链表
  void
  init_readers()
  {
    for (int j = 0; j < MAX_MIOBUFFER_READERS; j++)
      if (readers[j].allocated() && !readers[j].block)
        readers[j].block = _writer;
  }

  // 释放 block 和 所有 reader
  void
  dealloc()
  {
    _writer = NULL;
    dealloc_all_readers();
  }

  // 清除所有成员的状态，首先调用dealoc()，
  // 然后设置 size_index 为 BUFFER_SIZE_NOT_ALLOCATED，water_mark = 0
  void
  clear()
  {
    dealloc();
    size_index = BUFFER_SIZE_NOT_ALLOCATED;
    water_mark = 0;
  }

  // 为当前 block 重新分配一个更大的内存块
  void
  realloc(int64_t i)
  {
    _writer->realloc(i);
  }
  
  // 为当前 block 重新分配一个更大的内存块，
  // 然后将 b 指向的内存块的内容复制到新分配的内存块
  void
  realloc(void *b, int64_t buf_size)
  {
    _writer->realloc(b, buf_size);
  }
  void
  realloc_xmalloc(void *b, int64_t buf_size)
  {
    _writer->realloc_xmalloc(b, buf_size);
  }
  void
  realloc_xmalloc(int64_t buf_size)
  {
    _writer->realloc_xmalloc(buf_size);
  }

  // 记录当前缺省内存块大小的索引值
  int64_t size_index;

  // 决定何时停止写和读
  // 例如: 需要至少读取多少字节，才触发上层状态机
  //       需要还有多少剩余空间，才分配新的block，等
  // PS：需要通过上层状态机配合实现
  int64_t water_mark;

  // 指向第一个可写入数据的IOBufferBlock
  Ptr<IOBufferBlock> _writer;
  // 保存可以读取此MIOBuffer的IOBufferReader，默认值最大为 5 个
  IOBufferReader readers[MAX_MIOBUFFER_READERS];

#ifdef TRACK_BUFFER_USER
  const char *_location;
#endif

  // 构造函数
  // 使用预分配的内存块初始化MIOBuffer，并设置water_mark值，size_index值设置为未分配状态
  MIOBuffer(void *b, int64_t bufsize, int64_t aWater_mark);
  // 只设置size_index值，但是不立即分配底层block
  // 写入数据时，自动根据size_index值分配block
  MIOBuffer(int64_t default_size_index);
  // size_index值设置为未分配状态
  MIOBuffer();
  
  // 析构函数
  // 其定义同dealloc()，释放 block 和 所有 reader
  ~MIOBuffer();
};
```

### 方法

详细描述一下两种 write 方法，避免混乱(多数内容来自源码注释的直接翻译)

  - inkcoreapi int64_t write(const void *rbuf, int64_t nbytes);
    - 从rbuf复制nbytes字节到当前的IOBufferBlock
    - 如果当前的IOBufferBlock写满了，就再追加一个到末尾，继续完成nbytes的写入
  - inkcoreapi int64_t write(IOBufferReader *r, int64_t len = INT64_MAX, int64_t offset = 0);
    - 将r中包含的可用数据，通过clone方法进行复制，然后追加到当前的IOBufferBlock链表。
    - 需要注意的是：
      - clone方法得到的IOBufferBlock是直接引用了IOBufferData，通过\_start和\_end来指定IOBufferData中被引用的部分数据
      - 即使只有1个字节被引用，也会clone一个完整的IOBufferBlock实例，同时它不可以被写入任何数据。
      - 因此如果继续追加数据时，会创建新的Block，并追加到链表尾部。
    - 此方法会遍历r中IOBufferBlock链表中包含的所有可读取的Block，并逐个clone，但是并不会修改 r 的读取状态。
    - 由于在释放IOBufferBlock链表时，是一个递归调用的过程，
      - 这样会导致调用栈的叠加，如果IOBufferBlock链表过长的话，会导致调用栈溢出**
    - 因此在使用write方法时，
      - 在写入大量较小字节的数据块时，应当尽可能避免使用IOBufferReader作为源，因为这样会导致IOBufferBlock链表不合理的增长。
      - 当预期写入连续多个长度较小的数据时，可以考虑使用第一种write方法，以减少IOBufferBlock链表的长度。

### 理解 water_mark

它可以用于

  - 当你需要读取到一定数量的数据或者可用数据低于某个数量时才去执行某个操作
  - 例如当```可读取的数据量```和```可写入的空间```同时小于此值时，才会追加新的Block到MIOBuffer。参考：check_add_block()
  - 例如我们可以设置当buf里的可用数据低于water_mark值时，才从磁盘读取数据，这样可以降低磁盘I/O，不然每次读取的数据量很小，非常浪费磁盘I/O的调度。
  - 默认值为0

## 基础组件：MIOBufferAccessor

MIOBufferAccessor

  - IOBuffer 读（消费者）、写（生产者）的封装
  - 它将MIOBuffer和IOBufferReader封装在一起

### 定义

```
struct MIOBufferAccessor {
  // 返回 IOBufferReader
  IOBufferReader *
  reader()
  {
    return entry;
  }

  // 返回 MIOBuffer
  MIOBuffer *
  writer()
  {
    return mbuf;
  }

  // 返回 MIOBuffer 创建 block 时的缺省尺寸（只读）
  int64_t
  block_size() const
  {
    return mbuf->block_size();
  }

  // 同 block_size()
  int64_t
  total_size() const
  {
    return block_size();
  }

  // 用IOBufferReader *abuf来初始化当前accessor的读、写
  void reader_for(IOBufferReader *abuf);
  // 用MIOBuffer *abuf来初始化当前accessor的读、写
  void reader_for(MIOBuffer *abuf);
  // 用MIOBuffer *abuf来初始化当前accessor的写
  // 当前accessor不可读
  void writer_for(MIOBuffer *abuf);

  // 清除与 MIOBuffer 和 IOBufferReader 的关联
  void
  clear()
  {
    mbuf = NULL;
    entry = NULL;
  }

  // 构造函数
  // 功能同clear()
  MIOBufferAccessor()
    :
#ifdef DEBUG
      name(NULL),
#endif
      mbuf(NULL), entry(NULL)
  {
  }

  // 析构函数
  // 空函数
  ~MIOBufferAccessor();

#ifdef DEBUG
  const char *name;
#endif

private:
  MIOBufferAccessor(const MIOBufferAccessor &);
  MIOBufferAccessor &operator=(const MIOBufferAccessor &);

  // 指向用于写操作的MIOBuffer
  MIOBuffer *mbuf;
  // 指向用于读操作的IOBufferReader
  IOBufferReader *entry;
};
```

## 参考资料
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
- [P_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_IOBuffer.h)
- [IOBuffer.cc]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/IOBuffer.cc)
