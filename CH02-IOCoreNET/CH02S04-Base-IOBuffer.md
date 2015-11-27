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


