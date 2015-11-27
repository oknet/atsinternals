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
                +------[read]------> MIOBufferAccessor ------[write]-----+
                |                                                        V
          IOBufferReader <------------------[read]------------------> MIOBuffer
                |                                                        |
          IOBufferBlock ---- IOBufferBlock ---- IOBufferBlock ---- IOBufferBlock ----> NULL
                |                  |                  |                  |
          IOBufferData       IOBufferData       IOBufferData       IOBufferData
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

### 方法

- block_size 返回分配的内存大小
- dealloc 释放之前由alloc分配，由该类管理的内存。
- alloc 根据size_index和type来分配内存，如果之前已经分配过，会先执行dealloc操作
- data 返回_data成员
- free 首先执行dealloc，然后释放该对象。因此执行该方法后就不能再使用和引用该对象了。

### 成员变量

- \_size_index 表示内存块的字节数，通过公式128*2^size_index 来得出。
- _mem_type 内存分配类型，AllocType枚举值
- _data 指向内存块的指针

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

### 方法

- buf 返回指向底层数据块的指针
- start 返回指向正在使用数据区域的开始位置的指针
- end 返回指向正在使用数据区域的结束位置的指针
- buf_end 返回指向底层数据块结束位置的指针
- size 返回正在使用的数据区域的大小， end - start
- read_avail 对于读操作可以读取的长度，与size等价
- write_avail 对于写操作可以继续写入的长度，表示可用的空间大小，buf_end - end
- block_size 底层数据块的大小，IOBufferData->block_size()
- consume 减少正在使用的数据区域，start＋＝len
- fill 增加正在使用的数据区域，end＋＝len，但是首先要用end()获取当前的数据区域的结尾，然后拷贝数据到结尾后，再调用此方法
- reset 重置正在使用的数据区域，start＝end＝buf，之后read_avail＝＝0, buf_end＝buf＋block_size
- clone 克隆IOBufferBlock，但是底层数据块不会被克隆，所以克隆出来的实例引用同一个底层数据块，并且write_avail＝＝0, 就是buf_end＝end
- clear 重置data＝buf_end＝end＝start＝NULL，并且对next指向的下一个Block做引用计数减少操作
- alloc 分配一块长度为i的buffer给data
- dealloc 同clear
- set 将IOBufferData与该Block实例建立引用关系，可以通过len和offset指定仅使用部分Data。start＝buf＋offset，end＝start＋len，buf_end＝start＋block_size
- set_internal 通过内部调用建立一个没有分配内存块的IOBufferData实例，然后将已经分配好的内存指针赋值给IOBufferData实例，其它与set相同
- free 首先调用dealloc，然后再释放该实例占用的内存，注意此Block引用的底层IOBufferData不会受到影响

### 成员变量

- 
- 
-


## 基础组件：IOBufferReader

IOBufferReader

  - 不依赖MIOBuffer。
  - 用于读取一组IOBufferBlock。
  - IOBufferReader表示一个给定的缓冲区数据的消费者从哪儿开始读取数据。
  - 提供了一个统一的界面，可以轻松访问一组IOBufferBlock内包含的数据。
  - IOBufferReader 内部封装了自动移除数据块的判断逻辑。
    - consume 方法中block=block->next的操作会导致block的引用计数变化，通过自动指针Ptr实现对Block和Data的自动释放。
  - 简单的说：IOBufferReader将多个IOBufferBlock单链表作为一个大的缓冲区，提供了读取/消费这个缓冲区的一组方法。
  - 内部成员 智能指针 block 指向多个IOBufferBlock构成的单链表的第一个元素。
  - 内部成员 mbuf 指回到创建此IOBufferReader实例的MIOBuffer实例。

### 方法

- 
- 
- 


### 成员变量

- 
- 
-

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

### 方法

- inkcoreapi int64_t write(const void *rbuf, int64_t nbytes);
   - 从rbuf复制nbytes字节到当前的IOBufferBlock
   - 如果当前的IOBufferBlock写满了，就再追加一个到末尾，继续完成nbytes的写入
- inkcoreapi int64_t write(IOBufferReader *r, int64_t len = INT64_MAX, int64_t offset = 0);
   - 将r中包含的可用数据，通过clone方法进行复制，然后追加到当前的IOBufferBlock链表。
   - 需要注意的是clone方法得到的IOBufferBlock是直接引用了IOBufferData，通过\_start和\_end来指定IOBufferData中被引用的部分数据，即使只有1个字节被引用，同时它不可以被写入任何数据，因此如果继续追加数据时，会创建新的Block，并追加到链表尾部。
   - 此方法会遍历r中IOBufferBlock链表中包含的所有可读取的Block，并逐个clone。
   - 由于在释放IOBufferBlock链表时，是一个递归调用的过程，这样会导致调用栈的叠加，如果IOBufferBlock链表过长的话，会导致调用栈溢出，因此在使用write方法时，应当尽可能避免使用IOBufferReader作为源，写入较小字节的方法。因为这样会导致IOBufferBlock链表不合理的增长。当预期写入连续多个长度较小的数据时，可以考虑使用第一种write方法，以减少IOBufferBlock链表的长度。
- 

### 成员变量

- water_mark
   - 它可以用于，当你需要读取到一定数量的数据或者可用数据低于某个数量时才去执行某个操作
   - 例如当```可读取的数据量```和```可写入的空间```同时小于此值时，才会追加新的Block到MIOBuffer。
   - 例如我们可以设置当buf里的可用数据低于water_mark值时，才从磁盘读取数据，这样可以降低磁盘I/O，不然每次读取的数据量很小，非常浪费磁盘I/O的调度。
   - water_mark用来表示这个最小的数量
   - 默认值为0
   - 
- 
-

## 基础组件：MIOBufferAccessor

MIOBufferAccessor

  - IOBuffer 读（消费者）、写（生产者）的封装
  - 它将MIOBuffer和IOBufferReader封装在一起

### 方法

- 
- 
- 

### 成员变量

- 
- 
-


