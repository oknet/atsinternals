# 核心部件：Thread & EThread

EThread继承自Thread，首先看一下Thread的定义：

## 定义

```
class Thread
{
public:
  /*-------------------------------------------*\
  | Common Interface                            |
  \*-------------------------------------------*/

  /**
    System-wide thread identifier. The thread identifier is represented
    by the platform independent type ink_thread and it is the system-wide
    value assigned to each thread. It is exposed as a convenience for
    processors and you should not modify it directly.

  */
  ink_thread tid;

  /**
    Thread lock to ensure atomic operations. The thread lock available
    to derived classes to ensure atomic operations and protect critical
    regions. Do not modify this member directly.

  */
  ProxyMutex *mutex;

  // PRIVATE
  void set_specific();
  Thread();
  virtual ~Thread();

  static ink_hrtime cur_time;
  // 静态变量，初始化为 init_thread_key()
  inkcoreapi static ink_thread_key thread_data_key;
  Ptr<ProxyMutex> mutex_ptr;

  // For THREAD_ALLOC
  ProxyAllocator eventAllocator;
  ProxyAllocator netVCAllocator;
  ProxyAllocator sslNetVCAllocator;
  ProxyAllocator httpClientSessionAllocator;
  ProxyAllocator httpServerSessionAllocator;
  ProxyAllocator hdrHeapAllocator;
  ProxyAllocator strHeapAllocator;
  ProxyAllocator cacheVConnectionAllocator;
  ProxyAllocator openDirEntryAllocator;
  ProxyAllocator ramCacheCLFUSEntryAllocator;
  ProxyAllocator ramCacheLRUEntryAllocator;
  ProxyAllocator evacuationBlockAllocator;
  ProxyAllocator ioDataAllocator;
  ProxyAllocator ioAllocator;
  ProxyAllocator ioBlockAllocator;

private:
  // prevent unauthorized copies (Not implemented)
  Thread(const Thread &);
  Thread &operator=(const Thread &);

public:
  ink_thread start(const char *name, size_t stacksize = DEFAULT_STACKSIZE, ThreadFunction f = NULL, void *a = NULL);

  virtual void
  execute()
  {
  }

  static ink_hrtime get_hrtime();
};

extern Thread *this_thread();
```

## 方法

### Thread::Thread()

  - 用于初始化 mutex
  - 并且上锁
  - 线程总是保持对其自身 mutex 的锁定
  - 这样凡是复用线程 mutex 对象，则相当于被线程永久锁定，成为只属于此线程的本地对象

```
Thread::Thread()
{
  mutex = new_ProxyMutex();
  mutex_ptr = mutex;
  MUTEX_TAKE_LOCK(mutex, (EThread *)this);
  mutex->nthread_holding = THREAD_MUTEX_THREAD_HOLDING;
}
```

### Thread::set_specific()

```
// source: iocore/eventsystem/P_Thread.h
TS_INLINE void
Thread::set_specific()
{
  把当前对象（this）关联到 key
  ink_thread_setspecific(Thread::thread_data_key, this);
}
```

## 关于 this_thread()

  - 返回当前 Thread 对象
  - 由于 EThread 继承自 Thread，因此
    - 在 EThread 中调用 this_thread() 返回的是 EThread 对象
    - 但是返回值的类型仍然是 Thread *
    - 为了提供正确的返回值类型，在 EThread 中又定义了 this_ethread() 方法

```
  // 下面是 I_Thread.h 中的一段注释
  The Thread class maintains a thread_key which registers *all*
  the threads in the system (that have been created using Thread or
  a derived class), using thread specific data calls.  Whenever, you
  call this_thread() you get a pointer to the Thread that is currently
  executing you.  Additionally, the EThread class (derived from Thread)
  maintains its own independent key. All (and only) the threads created
  in the Event Subsystem are registered with this key. Thus, whenever you
  call this_ethread() you get a pointer to EThread. If you happen to call
  this_ethread() from inside a thread which is not an EThread, you will
  get a NULL value (since that thread will not be  registered with the
  EThread key). This will hopefully make the use of this_ethread() safer.
  Note that an event created with EThread can also call this_thread(),
  in which case, it will get a pointer to Thread (rather than to EThread).
  
  // 翻译如下：
  Thread 类中包含一个 thread_key (成员 thread_data_key)，所有的线程都通过 线程特有数据 相关
  的调用，借助 thread_key 注册到系统中 (包含了所有创建出来的 Thread 类型和其继承类的实例)。
  
  任何时候，只要你调用 this_thread() 你就可以得到一个指向运行当前代码的 Thread 实例的指针。

  // 这句注释描述的机制没有在代码中体现，可能由于代码的删减，现在已经没有了 ---------
  Additionally, the EThread class (derived from Thread)
  maintains its own independent key.
  另外，EThread 类 (继承自 Thread 类) 维护了一个独立属于它自己的 thread_key.
  // --------------------------------------------------------------------------
  
  所有 EventSystem 中创建的线程都使用这个(同一个) thread_key 注册。
  
  // 这句注释描述的机制没有在代码中体现，可能由于代码的删减，现在已经没有了 ---------
  If you happen to call 
  this_ethread() from inside a thread which is not an EThread, you will
  get a NULL value (since that thread will not be  registered with the
  EThread key). This will hopefully make the use of this_ethread() safer.
  如果你从一个不是 EThread 的线程中调用 this_ethread()，那么你会得到 NULL。
  因为这个线程没有使用 EThread 的 key 来注册。
  我们希望可以让 this_ethread() 的使用更安全。
  // --------------------------------------------------------------------------

  需要注意的是，在 EThread 中可以调用 this_thread()，但是，得到的是一个指向 Thread 类型
  的指针（而不是 EThread 类型）。
  注：从当前的代码看，可以通过类型转换为 EThread，在 this_ethread() 里就是这么做的。

```

使用同一个 key，在不同的线程中，获得的数据是不同的，同样的，绑定的数据也只与当前线程关联。
因此，对 thread specific 相关函数的调用是与其所在的线程紧密相关的。

```
TS_INLINE Thread *
this_thread()
{
  通过 key 取回之前关联的对象
  return (Thread *)ink_thread_getspecific(Thread::thread_data_key);
}
```

### 关于 this_ethread()

```
// source: iocore/eventsystem/P_UnixEThread.h
TS_INLINE EThread *
this_ethread()
{
  直接调用 this_thread() 并进行类型转换
  return (EThread *)this_thread();
}
```

### Thread::start()

线程的启动

  - 在每个线程创建之前，会调用 set_specific() 方法绑定线程数据

```
// source: iocore/eventsystem/Thread.cc
static void *
spawn_thread_internal(void *a) 
{
  thread_data_internal *p = (thread_data_internal *)a;

  // 将 Thread 对象 p->me 通过 key 设置为当前线程的特定数据
  p->me->set_specific();
  ink_set_thread_name(p->name);
  // 如果函数指针 f 不为空，则调用 f
  if (p->f)
    p->f(p->a);
  else
  // 否则执行线程的默认运行函数
    p->me->execute();
  // 返回后释放对象 a
  ats_free(a);
  return NULL;
}

ink_thread
Thread::start(const char *name, size_t stacksize, ThreadFunction f, void *a) 
{
  thread_data_internal *p = (thread_data_internal *)ats_malloc(sizeof(thread_data_internal));

  // 参数 f 为函数指针
  p->f = f;
  // 参数 a 为调用 f 时传入的参数
  p->a = a;
  // p->me 为当前 Thread 对象
  p->me = this;
  memset(p->name, 0, MAX_THREAD_NAME_LENGTH);
  ink_strlcpy(p->name, name, MAX_THREAD_NAME_LENGTH);
  // 创建线程时，传入 spawn_thread_internal 函数，这样线程启动会会立即调用该函数
  tid = ink_thread_create(spawn_thread_internal, (void *)p, 0, stacksize);

  return tid;
}
```

  - 在 Main.cc 里有一段很奇特的地方也调用了 set_specific
    - 但是之后就没看到任何使用 main_thread 的地方了
    - 从注释上看，这里是为了对 win_9xMe 的系统运行 ATS 提供的支持
    - 针对 start_HttpProxyServer() 的调用做的优化

```
// source: proxy/Main.cc
  // This call is required for win_9xMe
  // without this this_ethread() is failing when
  // start_HttpProxyServer is called from main thread
  Thread *main_thread = new EThread;
  main_thread->set_specific();   
```

## main() 是如何成为 [ET_NET 0] 的？

在 Main.cc 的最后执行了 this_ethread()->execute()

  - 原本 execute() 是在 spawn_thread_internal() 中被调用的
  - 而spawn_thread_internal()是线程创建之后才会调用的第一个函数
  - 但是为了节省一个线程的空间，ATS直接在main()中调用了 this_ethread()->execute() 开始执行
  - 但是在 spawn_thread_internal() 中需要调用 set_specific() 的过程去了那里？
  - 如果不调用 set_specific() 那么 this_ethread() 又怎么能返回 EThread 对象呢？

```
// source: iocore/eventsystem/UnixEventProcessor.cc
int
EventProcessor::start(int n_event_threads, size_t stacksize)
{
...
  int first_thread = 1;

  for (i = 0; i < n_event_threads; i++) {
    EThread *t = new EThread(REGULAR, i); 
    if (first_thread && !i) {
      // 如果 i 为 0 的时候，就会执行一次 set_specific 的动作
      ink_thread_setspecific(Thread::thread_data_key, t); 
      global_mutex = t->mutex;
      t->cur_time = ink_get_based_hrtime_internal();
    }   
    all_ethreads[i] = t;

    eventthread[ET_CALL][i] = t;
    t->set_event_type((EventType)ET_CALL);
  }
...
  // for 循环是从 1 开始的，跳过了 0
  for (i = first_thread; i < n_ethreads; i++) {
    snprintf(thr_name, MAX_THREAD_NAME_LENGTH, "[ET_NET %d]", i);
    // 也就是 all_ethreads[0]->start() 没有被调用
    ink_thread tid = all_ethreads[i]->start(thr_name, stacksize);
...
}
```

上面的代码，可以看出在 EventProcessor::start() 方法中对 all_ethreads[0] 做了特殊的处理

  - 为 all_ethreads[0] 调用 set_specific
  - 不启动 all_ethreads[0]

所以，在 main() 最后的那句 this_ethread()->execute() 就是启动了 [ET_NET 0] 的执行。

重新回顾一下这个流程：

  - 在 main() 的一开始首先调用了 eventProcessor.start()
    - 启动了 [ET_NET 1] ~ [ET_NET n] 的所有线程
    - 初始化 [ET_NET 0] 的数据，但不启动该线程
  - 然后 main() 执行其它的启动流程
  - 最后 main() 调用 this_ethread()->execute() 完成了 [ET_NET 0] 的启动

## 参考资料

- [I_Thread.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Thread.h)


# 核心部件：EThread

EThread 是 Thread 的具体实现。

在ATS中，只有EThread类基于Thread类实现。

EThread类是由Event System创建和管理的线程类型。

它是Event System调度事件的接口之一（另外两个是Event和EventProcessor类）。

## EThread的类型

```
source: iocore/eventsystem/I_EThread.h
enum ThreadType {
  REGULAR = 0,
  MONITOR,
  DEDICATED,
};
```

ATS可以创建两种类型的EThread线程：

- DEDICATED类型
   - 该类型的线程，在执行完某个延续后，就消亡了
   - 换句话说，这个类型只处理／执行一个Event，在创建这种类型的EThread时就要传入Event。
   - 或者是处理一个独立的任务，例如NetAccept的延续就是通过此种类型来执行的。
- REGULAR类型
   - 该类型的线程，在ATS启动后，就一直存在着，它的线程执行函数：execute()是一个死循环
   - 该类型的线程维护了一个事件池，当事件池不为空时，线程就从事件池中取出事件（Event），同时执行事件（Event）封装的延续（Continuation）
   - 当需要调度某个线程执行某个延续（Continuation）时，通过EventProcessor将一个延续（Continuation）封装成一个事件（Event）并将其加入到线程的事件池中
   - 这个类型是EventSystem的核心，它会处理很多的Event，而且可以从外部传入Event，然后让这个EThread处理／执行Event。
- MONITOR类型
   - 实际上还有第三种类型，但是MONITOR类型当前未使用到
   - 也许在Yahoo开源时被删除了

在创建一个EThread实例之前就要决定它的类型，创建之后就不能更改了。


## 事件池／队列的类型

为了处理事件，每个EThread实例实际有四个事件队列：

- 外部队列（EventQueueExternal被声明为EThread的成员，是一个保护队列）
   - 让EThread的调用者，能够追加事件到该特定的线程
   - 由于它同时可以被其它线程访问，所以对它的操作必须保持原子性
- 本地队列（外部队列的子队列）
   - 外部队列是满足原子操作的，可以批量导入本地队列
   - 本地队列只能被当前线程访问，不支持原子操作
- 内部队列（EventQueue被声明为EThread的成员，是一个优先级队列）
   - 在另一方面，专门用于由EThread处理在一定时间框架内的定时事件
   - 这些事件在内部被排队
   - 也可能包含来自外部队列的事件（本地队列内符合条件的事件会进入到内部队列）
- 隐性队列（NegativeQueue被声明为execute()函数的局部变量）
   - 由于是局部变量，不可被外部直接操作，所以称之为隐性队列
   - 通常只有epoll_wait的事件出现在这个队列里

## 调度界面

EThread提供了8个调度函数（其实是9个，但是有一个 _signal 除了IOCore内部使用，其它地方都用不到）。

以下方法把Event放入当前EThread线程的外部队列
- schedule_imm
- schedule_imm_signal
  - 专为网络模块设计
  - 一个线程可能一直阻塞在epoll_wait上，通过引入一个pipe或者eventfd，当调度一个线程执行某个event时，异步通知该线程从epoll_wait解放出来。
- schedule_at
- schedule_in
- schedule_every

上述5个方法最后都会调用schedule()的公共部分

以下方法功能与上面的相同，只是它们直接把Event放入内部队列
- schedule_imm_local
- schedule_at_local
- schedule_in_local
- schedule_every_local

上述4个方法最后都会调用schedule_local()的公共部分

## 定义

```
class EThread : public Thread
{
public:
  // 这里省略上面介绍了9个schedule_*方法，以及两个公共部分schedule()和schedule_local()
  schedule_*();

  InkRand generator;

private:
  // prevent unauthorized copies (Not implemented)
  EThread(const EThread &);
  EThread &operator=(const EThread &);

public:
  // 构造函数 & 析构函数 详细参考：iocore/eventsystem/UnixEThread.cc
  // 基本构造函数，初始化thread_private，同时成员tt初始化为REGULAR，成员id初始化为NO_ETHREAD_ID
  EThread();
  // 增加对ethreads_to_be_signalled的初始化，同时成员tt初始化为att的值，成员id初始化为anid的值
  EThread(ThreadType att, int anid);
  // 专用于DEDICATED类型的构造函数，成员tt初始化为att的值，oneevent初始化指向e
  EThread(ThreadType att, Event *e);
  // 析构函数，刷新single，并释放ethreads_to_be_signalled
  virtual ~EThread();

  // 已经被废弃，没有发现在ATS中有使用过
  Event *schedule_spawn(Continuation *cont);

  // 线程内部数据，用于保存如：统计系统的数组
  /** Block of memory to allocate thread specific data e.g. stat system arrays. */
  char thread_private[PER_THREAD_DATA];

  // 连接到Disk Processor以及与之配合的AIO队列
  /** Private Data for the Disk Processor. */
  DiskHandler *diskHandler;
  /** Private Data for AIO. */
  Que(Continuation, link) aio_ops;

  // 外部事件队列：保护队列
  ProtectedQueue EventQueueExternal;
  // 内部事件队列：优先级队列
  PriorityEventQueue EventQueue;

  // 当schedule操作时，如果设置signal为true则需要触发信号通知到执行该Event的EThread
  // 但是当无法获得目标EThread的锁，就不能立即发送信号，因此只能先将该线程挂在这里
  // 在空闲时刻进行通知。
  // 这个指针数组的长度不会超过线程总数，当达到线程总数的时候，会转换为直接映射表。
  EThread **ethreads_to_be_signalled;
  int n_ethreads_to_be_signalled;

  // 已经被废弃，估计是在init_accept_per_thread里用来保存accept事件，一旦不打算accept了，可以再取消掉
  // 想到可以使用的估计就是ftp data通道的accept了
  Event *accept_event[MAX_ACCEPT_EVENTS];
  int main_accept_index;

  // 从0开始的线程id，在EventProcessor::start中new EThread时设置
  int id;
  // 事件类型，如：ET_NET，ET_SSL等，通常用于设置该EThread仅处理指定类型的事件
  // 该值仅由is_event_type读取，set_event_type写入
  unsigned int event_types;
  bool is_event_type(EventType et);
  void set_event_type(EventType et);

  // Private Interface

  // 事件处理机主函数
  void execute();
  // 处理事件（Event），回调Cont->HandleEvent
  void process_event(Event *e, int calling_code);
  // 释放事件（Event），回收资源
  void free_event(Event *e);

  // 后面分析，用于高优先级Event的signal hook
  void (*signal_hook)(EThread *);
  // 与signal_hook相关，netProcessor内部用这个来实现唤醒和锁定
#if HAVE_EVENTFD
  int evfd;
#else
  int evpipe[2];
#endif
  EventIO *ep;

  // EThread类型：REGULAR，MONITOR，DEDICATED
  ThreadType tt;
  // 专门用于DEDICATED类型的EThread
  Event *oneevent; // For dedicated event thread

  // ???
  ServerSessionPool *server_session_pool;
};

// 该宏定义用来返回给定线程内，线程私有数据中特定offset位置为起点的指针
#define ETHREAD_GET_PTR(thread, offset) ((void *)((char *)(thread) + (offset)))

// 用于获取当前EThread实例的指针
extern EThread *this_ethread();

source: iocore/eventsystem/P_UnixEThread.h
TS_INLINE EThread *
this_ethread()
{
  return (EThread *)this_thread();
}

TS_INLINE void
EThread::free_event(Event *e)
{
  ink_assert(!e->in_the_priority_queue && !e->in_the_prot_queue);
  e->mutex = NULL;
  EVENT_FREE(e, eventAllocator, this);
}
```

## 关于 EThread::event_types

EventType实际上就是int类型，从ET_CALL(0)开始，最大值7，可以定义8种。EventType的定义如下：

```
source: iocore/eventsystem/I_Event.h
typedef int EventType;
const int ET_CALL = 0;
const int MAX_EVENT_TYPES = 8; // conservative, these are dynamically allocated
```

EThread的成员event_types是一个按位操作的状态值，由于EventType最多可定义8种，所以只有低8位被使用。

通过多次调用方法```void set_event_type(EventType et)```，可以让一个EThread同时处理多钟类型的event。

可以通过方法```bool is_event_type(EventType et)```，来判断该EThread是否支持处理指定类型的event。

```
source: iocore/eventsystem/UnixEThread.cc
bool
EThread::is_event_type(EventType et)
{
  return !!(event_types & (1 << (int)et));
}

void
EThread::set_event_type(EventType et)
{
  event_types |= (1 << (int)et);
}
```

## EThread::process_event() 分析

```
source: iocore/eventsystem/UnixEThread.cc

void
EThread::process_event(Event *e, int calling_code)
{
  ink_assert((!e->in_the_prot_queue && !e->in_the_priority_queue));
  // 尝试获取锁
  MUTEX_TRY_LOCK_FOR(lock, e->mutex.m_ptr, this, e->continuation);
  // 判断是否拿到锁
  if (!lock.is_locked()) {
    // 如果没有拿到锁，把event重新丢回外部本地队列
    e->timeout_at = cur_time + DELAY_FOR_RETRY;
    EventQueueExternal.enqueue_local(e);
    // 返回到execute()
  } else {
    // 如果拿到了锁，首先判断当前event是否已经被取消
    if (e->cancelled) {
      // 如果被取消了，则释放event
      free_event(e);
      // 返回到execute()
      return;
    }
    // 准备回调Cont->handleEvent
    Continuation *c_temp = e->continuation;
    // 注意：在回调期间，Cont->Mutex是被当前EThread锁定的
    e->continuation->handleEvent(calling_code, e);
    ink_assert(!e->in_the_priority_queue);
    ink_assert(c_temp == e->continuation);
    // 提前释放锁
    MUTEX_RELEASE(lock);

    // 如果该event是周期性执行，通过schedule_every添加
    if (e->period) {
      // 不在保护队列，也不在优先级队列。（就是没有在外部队列，也不在内部队列）
      //     通常在调用process_event之前从队列中dequeue了，就应该不会在队列里了
      //     但是，在Cont->handleEvent可能会调用了其他操作导致event被重新放回队列
      if (!e->in_the_prot_queue && !e->in_the_priority_queue) {
        if (e->period < 0)
          // 小于零表示这是一个隐性队列内的event，不用对timeout_at的值进行重新计算
          e->timeout_at = e->period;
        else {
          // 对下一次执行时间timeout_at的值进行重新计算
          cur_time = get_hrtime();
          e->timeout_at = cur_time + e->period;
          if (e->timeout_at < cur_time)
            e->timeout_at = cur_time;
        }
        // 将重新设置好timeout_at的event放入外部本地队列
        EventQueueExternal.enqueue_local(e);
      }
    } else if (!e->in_the_prot_queue && !e->in_the_priority_queue)
      // 不是周期性event，也不在保护队列，也不在优先级队列。（就是没有在外部队列，也不在内部队列），则释放event
      free_event(e);
  }
}
```

## EThread::execute() REGULAR 和 DEDICATED 流程分析

EThread::execute() 由switch语句分成多个部分：

- 第一部分是 REGULAR 类型的处理
  - 它由一个无限循环内的代码，持续扫描/遍历多个队列，并回调Event内部Cont的handler
- 第二部分是 DEDICATED 类型的处理
  - 直接回调oneevent内部的Cont的handler，其实是一个简化版的process_event(oneevent, EVENT_IMMEDIATE)

下面是对execute()代码的注释和分析：

```
source: iocore/eventsystem/UnixEThread.cc
//
// void  EThread::execute()
//
// Execute loops forever on:
// Find the earliest event.
// Sleep until the event time or until an earlier event is inserted
// When its time for the event, try to get the appropriate continuation
// lock. If successful, call the continuation, otherwise put the event back
// into the queue.
//

void
EThread::execute()
{
  switch (tt) {
  case REGULAR: {
    // REGULAR 部分开始
    Event *e;
    Que(Event, link) NegativeQueue;
    ink_hrtime next_time = 0;

    // give priority to immediate events
    // 设计目的：优先处理立即执行的事件
    for (;;) {
      // execute all the available external events that have
      // already been dequeued
      // 设置cur_time，在每次调用process_event方法时，在状态机回调完成后，也可能会更新cur_time
      // 如果是周期性事件，在process_event内会重新把该事件添加到内部队列，此时就会更新cur_time
      cur_time = ink_get_based_hrtime_internal();
      // 遍历：外部本地队列
      // dequeue_local() 方法每次从外部本地队列取出一个事件
      while ((e = EventQueueExternal.dequeue_local())) {
        // 事件被异步取消时
        if (e->cancelled)
          free_event(e);
        // 立即执行的事件
        else if (!e->timeout_at) { // IMMEDIATE
          ink_assert(e->period == 0);
          // 通过process_event回调状态机
          process_event(e, e->callback_event);
        // 周期执行的事件
        } else if (e->timeout_at > 0) // INTERVAL
          // 放入内部队列
          EventQueue.enqueue(e, cur_time);
        // 负事件
        else { // NEGATIVE
          Event *p = NULL;
          Event *a = NegativeQueue.head;
          // 注意这里timeout_at的值是小于0的负数
          // 按照从大到小的顺序排列
          // 如果遇到相同的值，则后插入的事件在前面
          while (a && a->timeout_at > e->timeout_at) {
            p = a;
            a = a->link.next;
          }
          // 放入隐性队列（负队列）
          if (!a)
            NegativeQueue.enqueue(e);
          else
            NegativeQueue.insert(e, p);
        }
      }
      // 遍历：内部队列（优先级队列）
      bool done_one;
      do {
        done_one = false;
        // execute all the eligible internal events
        // check_ready方法将优先级队列内的多个子队列进行重排（reschedule）
        // 使每个子队列容纳的事件的执行时间符合每一个字队列的要求
        EventQueue.check_ready(cur_time, this);
        // dequeue_ready方法只操作0号子队列，这里面的事件需要在5ms内执行
        while ((e = EventQueue.dequeue_ready(cur_time))) {
          ink_assert(e);
          ink_assert(e->timeout_at > 0);
          if (e->cancelled)
            free_event(e);
          else {
            // 本次循环处理了一个事件，设置done_one为true，表示继续遍历内部队列
            done_one = true;
            process_event(e, e->callback_event);
          }
        }
        // 每次循环都会把0号子队列里面的事件全部处理完
        // 如果本次对0号子队列的遍历中至少处理了一个事件，那么
        //    这个处理过程是会花掉时间的，但是事件若是被取消则不会花掉多少时间，所以不记录在内；
        //    此时，1号子队列里面的事件可能已经到了需要执行的时间，
        //    或者，1号子队列里面的事件已经超时了，
        // 所以要通过do－while(done_one)循环来再次整理优先级队列，
        //    然后再次处理0号子队列，以保证事件的按时执行。
        // 如果本次循环一个事件都没有处理，
        //    那就说明在整理队列之后，0号子队列仍然是空队列，
        //    这表示内部队列中最近一个需要执行的事件在5ms之后
        //    那就不再需要对内部队列进行遍历，结束do－while(done_one)循环
        // 注意：此处或许有一个假设，那就是每个REGULAR ETHREAD的大循环，运行时间为5ms，
        //    如果一次循环的时长超过了5ms，内部队列的事件就可能会超时
      } while (done_one);
      // execute any negative (poll) events
      // 遍历：隐性队列（如果隐性队列不为空的话）
      if (NegativeQueue.head) {
        // 触发signals，向其它线程通知事件
        // 在flush_signals这里有可能阻塞
        if (n_ethreads_to_be_signalled)
          flush_signals(this);
        // dequeue all the external events and put them in a local
        // queue. If there are no external events available, don't
        // do a cond_timedwait.
        // 如果外部队列不为空，那么就一次性取出所有事件丢入外部本地队列
        // 外部队列是EThread从外部（跨EThread）获取事件的唯一队列，操作方式必须为原子操作
        if (!INK_ATOMICLIST_EMPTY(EventQueueExternal.al))
          EventQueueExternal.dequeue_timed(cur_time, next_time, false);
        // 再次遍历：外部本地队列
        // 目的1: 优先处理立即执行的事件
        // 目的2: 可能有需要执行的负事件
        while ((e = EventQueueExternal.dequeue_local())) {
          // 首先是立即执行的事件
          // 这里为何没有首先判断cancelled？
          if (!e->timeout_at)
            // 通过process_event回调状态机
            process_event(e, e->callback_event);
          else {
            // 事件被异步取消时
            if (e->cancelled)
              free_event(e);
            else {
              // If its a negative event, it must be a result of
              // a negative event, which has been turned into a
              // timed-event (because of a missed lock), executed
              // before the poll. So, it must
              // be executed in this round (because you can't have
              // more than one poll between two executions of a
              // negative event)
              if (e->timeout_at < 0) {
              // 负事件，按照timeout_at值，由大到小的顺序排序放入隐性队列
                Event *p = NULL;
                Event *a = NegativeQueue.head;
                while (a && a->timeout_at > e->timeout_at) {
                  p = a;
                  a = a->link.next;
                }
                if (!a)
                  NegativeQueue.enqueue(e);
                else
                  NegativeQueue.insert(e, p);
              // 不是负事件，就放入内部队列
              } else
                EventQueue.enqueue(e, cur_time);
            }
          }
        }
        // execute poll events
        // 执行隐性队列里的polling事件，每次取一个事件，通过process_event呼叫状态机，目前：
        // 对于TCP，状态机是NetHandler::mainNetEvent
        // 对于UDP，状态机时PollCont::pollEvent
        while ((e = NegativeQueue.dequeue()))
          process_event(e, EVENT_POLL);
        // 再次判断外部队列是否为空，如果外部队列不为空，那么就一次性取出所有事件丢入外部本地队列
        if (!INK_ATOMICLIST_EMPTY(EventQueueExternal.al))
          EventQueueExternal.dequeue_timed(cur_time, next_time, false);
        // 然后回到for循环开始，重新遍历外部本地队列
      // 隐性队列为空，没有负事件
      } else { // Means there are no negative events
        // 没有负事件，那就是只有周期性事件需要执行，那么此时就需要节省CPU资源避免空转
        // 通过内部队列里最早需要执行事件的时间距离当前时间的差值，得到可以休眠的时间
        next_time = EventQueue.earliest_timeout();
        ink_hrtime sleep_time = next_time - cur_time;

        // 如果该休眠时间超过了允许的最大休眠时间则使用最大允许值
        // 注意：这里最后得到的是一个绝对时间
        if (sleep_time > THREAD_MAX_HEARTBEAT_MSECONDS * HRTIME_MSECOND) {
          next_time = cur_time + THREAD_MAX_HEARTBEAT_MSECONDS * HRTIME_MSECOND;
        }
        // dequeue all the external events and put them in a local
        // queue. If there are no external events available, do a
        // cond_timedwait.
        // 因为flush_signals方法内有可能会阻塞，因此在dequeue_timed方法内传入的是next_time绝对时间
        // 如果flush_signals的阻塞导致已经错过了next_time的值，那么dequeue_timed就不会阻塞
        if (n_ethreads_to_be_signalled)
          flush_signals(this);
        // 首先阻塞等待，可以被其它线程的flush_signals踢出，或者到达next_time的绝对时间
        // 然后把外部队列的事件全部导入外部本地队列
        EventQueueExternal.dequeue_timed(cur_time, next_time, true);
      }
    }
  }

  // 下面是DEDICATED模式
  case DEDICATED: {
    // coverity[lock]
    MUTEX_TAKE_LOCK_FOR(oneevent->mutex, this, oneevent->continuation);
    oneevent->continuation->handleEvent(EVENT_IMMEDIATE, oneevent);
    MUTEX_UNTAKE_LOCK(oneevent->mutex, this);
    free_event(oneevent);
    break;
  }

  default:
    ink_assert(!"bad case value (execute)");
    break;
  } /* End switch */
  // coverity[missing_unlock]
}
```

以下是简化的逻辑和逻辑图：

//for (;;)

- 外部本地队列
- 内部队列
- if 隐性队列有Event
  - 判断之前的schedule_*_signal调用，异步触发signal
  - if 外部队列有Event --> 为了避免下面的操作出现阻塞
    - 外部队列导入外部本地队列
  - 外部本地队列
  - 隐性队列
  - if 外部队列有Event --> 为了避免下面的操作出现阻塞
    - 外部队列导入外部本地队列
- else
  - 计算出内部队列最早需要处理的Event时间距离现在的时间差，作为sleep time
  - 判断是否超出最长sleep time，如超出，则使用最长sleep time值
  - 判断之前的schedule_*_signal调用，异步触发signal
  - 外部队列导入本地队列，如果外部队列没有Event，则最长等待sleep time指定的时间
  - 等待期间，可以使用schedule_*_signal来中断此等待

//end for(;;)

```
                                                                                     +-------------+
       +-----------------------------------------------------------------------------o    Sleep    |
       |                                                                             +------^------+
       |                                                                                    |
       |                     +---------+                                                    |
       |                     |  START  |                                                    |
       |                     +----o----+                                                    |
       |                          |                                                         |
       |                          |                                                         |
       |                          |                                                         |
       |                          |                                                         |
+------V------+            +------V---V--+                +--V-------V--+            +------o------+
|   External  o------------>   External  o---------------->    Event    o------------>    Flush    |
|    Event    |            |  EventQueue |                |             |            |             |
|    Queue    o############>   (local)   o################>    Queue    |            |   Signals   |
+------^------+            +--^---o---^--+                +--^---^---^--+            +------o------+
       |                      #   #   %                      %   #   %                      |
       |                      #   #   %                      %   #   %                      |
       |                      #   #   %                      %   #   %                      |
       |                      #   #   %  +----------------+  %   #   %  +----------------+  |
       |                      #   #   %  |   Timed Event  <%%%   #   %%%> Interval Event |  |
       |                      #   #   %  +--------o-------+      #      +--------o-------+  |
       |                      #   #   %           |              #               |          |
       |  +----------------+  #   #   %  +--------V-------+      #      +--------V-------+  |
       |  | process_event  O###   #   %  | process_event  |      #   ###o process_event  |  |
       |  +--------^-------+      #   %  +--------^-------+      #   #  +----------------+  |
       |           |              #   %           |              #   #                      |
       |  +--------o-------+      #   %  +--------o-------+      #   #                      |
       |  | Interval Event <%%%   #   %%%> Immediate Event<%%%   #   #                      |
       |  +----------------+  %   #      +----------------+  %   #   #                      |
       |                      %   #                          %   #   #                      |
       |                      %   #                          %   #   #                      |
       |                      %   #                          %   #   #                      |
       |                   +--V---V------+                +--V---o---V--+            +------V------+
       |                   |   Negative  <################o   External  <############o   External  |
       +-------------------o             |                |  EventQueue |            |    Event    |
                           |    Queue    <----------------o   (local)   <------------o    Queue    |
                           +-------------+                +-------------+            +-------------+

o--------->  execute route
o#########>  event route
<%%%%%%%%%>  call and return
```

从上图中，我们可以看出，根据隐性队列是否为空，REGULAR EThread又被分为两种类型

- 小循环：隐性队列为空
  - 每次循环从外部队列的本地队列开始
  - 然后是内部队列
  - 然后 flush_signals()
  - 最后阻塞，直到下一个定时事件 或 被 signal() 唤醒
  - 唤醒后，把外部队列的原子队列导入到外部本地队列
  - 开始下一个循环
- 大循环：隐性队列非空
  - 每次循环从外部队列的本地队列开始
  - 然后是内部队列
  - 然后 flush_signals()
  - 把外部队列的原子队列导入到外部本地队列
  - 再次处理外部队列的本地队列
  - 然后是隐性队列
  - 把外部队列的原子队列导入到外部本地队列
  - 开始下一个循环

在大循环里，第一次处理外部队列是为了内部队列，第二次处理外部队列是为了隐性队列，而且不会像小循环那样，在每一次循环的结束让EThread挂起。

由于：

  - 内部队列 和 隐性队列 都是 EThread 的私有成员
  - 这两个队列里的事件都需要从外部队列传入

为了能够及时处理最新的事件，每次在处理这两个队列之前都要首先检查一次外部队列，以确定能够及时的处理最新的事件。
 

外部队列 & 本地队列

- 外部队列为Free Lock队列，保证多个线程读写队列无须加锁，而且不会出现冲突
- 本地队列是外部队列的子队列，是一个普通的List结构赋予了Que的操作方法，如：pop，push，insert等
- 通常新Event直接加入外部队列，部分内部方法schedule_*_local()可以直接将Event加入外部队列的本地队列
- 但是Free Lock的操作仍然是有代价的，因此每个循环中，会一次性把外部队列的Event全部取出，放入外部队列的本地队列
- 接下来只需要对外部队列的本地队列进行遍历，判断Event的类型
  - 立即执行(通常为NetAccept Event)
     * 由schedule_imm()添加，timeout_at=0，period=0
     * 调用handler，然后Free(Event)
  - 定时执行(通常为Timeout Check)
     * 由schedule_at/in()添加，timeout_at>0，period=0
     * 简单理解at和in的不同：at是绝对时间，in是相对时间
     * 时间到达时，调用handler
     * 然后检查是否需要 "定期执行（可能为资源回收、缓冲区刷新等Event）"
     * 由schedule_every()添加，timeout_at=period>0
     * 如果需要定期执行则将Event放入外部队列的本地队列，否则Free(Event)
  - 随时执行(目前只有NetHandler Event)
     * 由schedule_every()添加，timeout_at=period<0
     * 在每个循环周期都会调用handler，然后再将Event放入外部队列的本地队列
     * handler对于TCP事件为NetHandler::mainNetEvent()

内部队列

- 为了提高Event处理的效率，特别是定时和定期Event
  - 在第一次遍历时，会将定时执行的Event拿出来，放入内部队列，内部队列是一个优先级队列
  - 在内部队列里，按照Event即将被执行的先后时间顺序排序
  - 根据距离执行时间还有多少ms，分成多个子队列（<5ms, 10, 20, 40, 80, 160, 320, 640, 1280, 2560, 5120）
  - 其中子队列0（<5ms）保存了所有需要在5ms以内需要执行的Event
  - 在一个独立遍历内部队列的过程之前会首选对内部队列进行重新排序和整理
  - 然后再遍历内部队列的子队列0，并执行Event内的handler
  - 执行完成后，由process_event重新放回外部本地队列，等待下一次循环。

隐性队列(负事件队列／负队列)

- 在向此队列添加事件时，都是通过schedule_every方法，时间参数是一个负数，这个负数在队列里被当作排序的key
- 值为-1的Event最先被执行，然后是执行值为-2的Event，...
- 如果该值相同，例如都是-2，那么按照加入队列的顺序，后加入的先执行，先加入的后执行。
- 执行完Event内的handler之后，由process_event重新放回外部本地队列，等待下一次循环。
- 通常放入此队列的都是Polling操作，如：
  - NetHandler::mainNetEvent
  - PollCont::pollEvent
- 上面两个handler都会调用epoll_wait，然后驱动netProcessor来完成网络数据的接收和发送

## signal_hook 介绍

在epoll_wait调用时，会有一个阻塞的超时等待时间，前面我们介绍Event System的时候，特别强调必须是完全无阻塞的。

但是epoll_wait中的阻塞又是一个可能会出现的情况，那么ATS是如何处理Event System中这种特殊的阻塞情况呢？

答案是```受控阻塞```:

- 阻塞的时间设置上限，由epoll_wait的超时参数决定
- 通过signal让epoll_wait在超时到达之前返回

通过man epoll_wait可以很容易得知：

- epoll_wait的timeout设置是以千分之一秒（ms／毫秒）为单位设置的
- 当此值大于0时有效，则epoll_wait不会立即返回
   - 而是等待timeout指定的时间，以等待可能的事件产生
   - 但是也可能没有等待到timeout的时间，而提前返回：
      - 出现了新的事件 
      - 被系统中断打断

系统中断是不可控的，但是新的事件呢？

- ATS通过eventfd()系统调用创建了一个evfd，并将这个fd封装到ep里，然后添加到了epoll fd里，关注evfd的READ事件。
- 如果让evfd变成可读，就会触发新事件，那么只需要提供一个方法让evfd可读，就可以产生新事件，这样epoll_wait就可以立即返回

ATS通过将evfd添加到fd集合中实现了这个受控阻塞，这就是受控阻塞的原理。

为了让这个设计更具有通用性，于是增加了signal_hook用于向evfd里写数据，这个signal_hook在iocore/net/UnixNet.cc里的initialize_thread_for_net中被初始化为net_signal_hook_function()，并且在初始化ep的时候，指定类型（ep->type）为EVENTIO_ASYNC_SIGNAL。

在NetHandler::mainNetEvent中可以看到专门对EVENTIO_ASYNC_SIGNAL类型的EventIO进行了处理，就是调用net_signal_hook_callback()来读取其中的数据，由于这里只是为了让epoll_wait从timeout wait状态中提前返回，所以读取到的数据没什么用。

就这样，ATS把epoll_wait的超时等待变成了可控等待。

因此，evfd，signal_hook 和 ep，这三个成员是一体的，是专门用来支持网络IO的受控阻塞。

实际上这个超时等待的默认值为10ms，也就是百分之一秒，ATS连这么短的时间都要打断去立即处理一个Event，可见ATS是为了达到实时处理的目的，真正实现schedule_imm_signal()的EVENT_IMMEDIATE的意义！

除了schedule_imm_signal()会调用到signal_hook，还有在UnixNetVConnection::reenable的时候也会调用到。

## Signal的异步通知

为什么会有 schedule_imm_signal 这样一个特殊的方法？

在schedule_*()中会通过```EventQueueExternal.enqueue(e, fast_signal＝true)```把event放入外部队列。

```
TS_INLINE Event *
EventProcessor::schedule(Event *e, EventType etype, bool fast_signal)
{
  ink_assert(etype < MAX_EVENT_TYPES);
  // assign_thread(etype) 负责从线程列表中返回指定etype类型的ethread线程指针
  e->ethread = assign_thread(etype);
  // 如果Cont有自己的mutex则Event继承Cont的mutex，否则Event和Cont都要继承ethread的mutex
  if (e->continuation->mutex)
    e->mutex = e->continuation->mutex;
  else
    e->mutex = e->continuation->mutex = e->ethread->mutex;

  // 将Event压入所分配的ethread线程的外部队列
  e->ethread->EventQueueExternal.enqueue(e, fast_signal);
  return e;
}
```

在阅读enqueue()方法的代码时，请一定看一下上面的schedule()方法，同时找一个大脑最清醒的时刻，否则很容易晕...

EventQueueExternal 是一个 ProtectedQueue，那么它的enqueue方法如下：

```
source:iocore/eventsystem/ProtectedQueue.cc
void
ProtectedQueue::enqueue(Event *e, bool fast_signal)
{
  ink_assert(!e->in_the_prot_queue && !e->in_the_priority_queue);
  // doEThread 为任务发起方所在线程，runEThread 为任务执行方所在线程，通过 Event 对象传递将任务描述（状态机）
  // 通常由 doEThread 创建一个Event对象，但是该对象的成员 ethread 成员为 NULL，
  // 然后，再通过eventProcessor.schedule_*()为该 Event 分配一个runEThread，然后调用此方法。
  // 由于 doEThread 可能跟 RunEThread 在同一的线程池中，因此 doEThread 可能与 runEThread 相同。
  // 所以，这里需要考虑 e->ethread 可能会等于 doEThread 的情况，而 doEThread 则为 this_ethread()。
  EThread *e_ethread = e->ethread;
  e->in_the_prot_queue = 1;
  // ink_atimiclist_push 执行原子操作将e压入到al的头部，返回值为压入之前的头部
  // 因此was_empty为true表示压入之前al是空的
  bool was_empty = (ink_atomiclist_push(&al, e) == NULL);

  if (was_empty) {
    // 如果保护队列压入新event之前为空
    // inserting_thread 为发起插入操作的队列
    // 例如：从DEDICATED ACCEPT THREAD插入event到ET_NET，
    //     那么inserting_thread就是DEDICATED ACCEPT THREAD
    EThread *inserting_thread = this_ethread();
    // queue e->ethread in the list of threads to be signalled
    // inserting_thread == 0 means it is not a regular EThread
    // 如果 doEThread 与 runEThread 为同一的 EThread 那么这里不需要进行特殊处理，
    //     此时，发起插入操作的 EThread 就是将要处理该 Event 的 EThread，这叫做内部插入。
    // 如果 doEThread 与 runEThread 不同，才需要进行下面的处理流程，
    //     此时，发起插入操作的 EThread 不是将要处理该 Event 的 EThread，简单说就是从线程外部插入
    //     这个event是由当前EThread(inserting_thread / doEThread)创建，要插入到另外一个EThread（e_ethread / runEThread）
    //     调用本方法的方式必定是通过e_ethread->ExternalEventQueue.enqueue()的方式
    if (inserting_thread != e_ethread) {
      // 如果发起插入操作的ethread不是REGULAR类型。
      // 例如：DEDICATED类型（该类型的ethreads_to_be_signalled为NULL）
      if (!inserting_thread || !inserting_thread->ethreads_to_be_signalled) {
        // 由于doEThread没有ethreads_to_be_signalled，无法实现延迟通知机制，只能采用阻塞方式向runEThread发出通知。
        // 发出通知后，阻塞在ProtectedQueue::dequeue_timed里的ink_cond_timedwait会立即返回。
        // 下面的signal就是直接通知e->ethread持有的xxxQueue
        signal();
        if (fast_signal) {
          // 如果需要立即触发signal，那么尝试通知产生此Event的EThread
          if (e_ethread->signal_hook)
            // 调用产生event的EThread的signal_hook方法，实现异步通知
            // 目前在 NetHandler 里通过signal_hook可以让 epoll_wait 的阻塞等待中断并返回
            e_ethread->signal_hook(e_ethread);
        }
      } else {
      // 如果当前EThread是REGULAR类型，而且ethreads_to_be_signalled不为空（就是支持signal的队列化）
#ifdef EAGER_SIGNALLING
        // 此处宏定义开关的含义：更及时的发送signal。（这样做好不好？见后面的分析）
        // 由于已经把event插入队列中，因此就要向持有此event的队列发送信号
        // 而这个队列必然是e->ethread->EventQueueExternal
        // 这里应该可以简写为 if (try_signal())，因为当前调用的enqueue()方法就是通过：
        //     e->ethread->EventQueueExternal.enqueue() 发起的。
        // Try to signal now and avoid deferred posting.
        if (e_ethread->EventQueueExternal.try_signal())
          return;
        // 压入signal队列之前尝试直接发起signal一次，
        //     如果成功就直接返回，如果失败则继续压入signal队列。
#endif
        if (fast_signal) {
          // 如果需要立即触发signal
          if (e_ethread->signal_hook)
            // 调用持有event的EThread的signal_hook方法，实现异步通知
            e_ethread->signal_hook(e_ethread);
        }
        // signal队列操作，向发起插入操作的EThread内的signal队列添加待通知列表
        int &t = inserting_thread->n_ethreads_to_be_signalled;
        EThread **sig_e = inserting_thread->ethreads_to_be_signalled;
        if ((t + 1) >= eventProcessor.n_ethreads) {
          // 如果增加到队列之后，就会超出或者达到队列的上限
          // we have run out of room
          if ((t + 1) == eventProcessor.n_ethreads) {
            // 当增加到队列之后，就会出现队列满的时候，将此队列转换为直接映射的数组，就是利用EThread::id作为数组下标
            // convert to direct map, put each ethread (sig_e[i]) into
            // the direct map loation: sig_e[sig_e[i]->id]
            // 遍历当前队列内的成员，算法如下：
            // 1. 取出一个成员cur
            // 2. 保存sig_e[cur->id]为next
            // 3. 如果cur＝＝next则说明此成员的顺序符合映射表的要求，跳到第5步
            // 4. 把cur放到sig_e[cur->id]，然后把之前保存的next赋值给cur，跳到第2步
            // 处理在2～4步循环中，遇到了sig_e[cur->id]为NULL的情况，
            //   此时sig_e[i]已经被移走到争取的位置了，但是老位置的指针没有被清空。
            // 5. 判断sig_e[i]->id是否为i，是否满足映射表的要求，如果不满足就把sig_e[i]指向NULL（0）
            // 6. 选择下一个成员，跳转到第1步 
            for (int i = 0; i < t; i++) {
              EThread *cur = sig_e[i]; // put this ethread
              while (cur) {
                EThread *next = sig_e[cur->id]; // into this location
                if (next == cur)
                  break;
                sig_e[cur->id] = cur;
                cur = next;
              }
              // if not overwritten
              if (sig_e[i] && sig_e[i]->id != i)
                sig_e[i] = 0;
            }
            t++;
          }
          // 此时signal队列必然已经是映射表的模式了，因此按照映射表的方式插入等待被通知的EThread到列表中
          // we have a direct map, insert this EThread
          sig_e[e_ethread->id] = e_ethread;
        } else
          // 队列插入模式
          // insert into vector
          sig_e[t++] = e_ethread;
      }
    }
  }
}
```

再来看signal和try_signal

- signal
   - 首先以阻塞方式获得锁
   - 然后触发cond_signal
   - 最后释放锁
- try_signal是signal的非阻塞版本
   - 尝试获得锁，如果获得锁，就跟signal是一样的，然后返回1，表示成功执行signal操作
   - 如果没有获得锁，就返回0

如果try_signal拿到了锁，成功发送了通知，则表示：

- 当前没有处于epoll_wait()等特殊阻塞场景，
- 不需要通过signal_hook()来向特殊组件发送通知。

相反，则表示：

- 目标EThread当前没有处于 cond_wait() 等待中
- 目标EThread当前正在执行Event标记的任务，此时可能会有特殊组件产生的阻塞
- 需要通过signal_hook()来打断特殊组件的阻塞

相关代码如下：

```
source: iocore/eventsystem/P_ProtectedQueue.h
TS_INLINE void
ProtectedQueue::signal()
{
  // Need to get the lock before you can signal the thread
  ink_mutex_acquire(&lock);
  ink_cond_signal(&might_have_data);
  ink_mutex_release(&lock);
}

TS_INLINE int 
ProtectedQueue::try_signal()
{
  // Need to get the lock before you can signal the thread
  if (ink_mutex_try_acquire(&lock)) {
    ink_cond_signal(&might_have_data);
    ink_mutex_release(&lock);
    return 1;
  } else {
    return 0;
  }
}
```

当通知被放入signal队列中时，会在EThread::execute()的REGULAR模式中进行判断，如果发现signal队列有元素，就会调用flush_signals(this)进行通知，相关代码如下：

```
void
flush_signals(EThread *thr)
{
  ink_assert(this_ethread() == thr);
  int n = thr->n_ethreads_to_be_signalled;
  if (n > eventProcessor.n_ethreads)
    n = eventProcessor.n_ethreads; // MAX
  int i;

// Since the lock is only there to prevent a race in ink_cond_timedwait
// the lock is taken only for a short time, thus it is unlikely that
// this code has any effect.
#ifdef EAGER_SIGNALLING
  // 首先通过try_signal，以非阻塞的方式完成一部分signal操作。
  for (i = 0; i < n; i++) {
    // Try to signal as many threads as possible without blocking.
    if (thr->ethreads_to_be_signalled[i]) {
      if (thr->ethreads_to_be_signalled[i]->EventQueueExternal.try_signal())
        thr->ethreads_to_be_signalled[i] = 0;
    }
  }
#endif
  // 然后再使用signal()，以阻塞方式完成剩余的signal操作。
  // 如果一个EThread的Negative Queue为空，那么在没有事件需要执行的时候，
  //     需要通过ProtectedQueue::timed_wait()让EThread挂起一段时间，
  //     在此期间，如果需要该EThread立即处理一个Event，就可以通过signal()唤醒它
  // 如果一个EThread的Negative Queue不为空，那么就存在需要随时执行的 Negative Event
  //     此时，该EThread永远都不需要挂起，因为它一有机会就需要不断的执行 Negative Event 指定的任务
  // 所以，
  //     对于存在 Negative Event 的 EThread 通常只需要调用 signal_hook() 就可以了，
  //     对于不存在 Negative Event 的 EThread 则需要同时考虑“在状态机中存在阻塞”或者“EThread被挂起”的情况。
  // 例如：
  //     NetHandler 是一个由 Negative Event 驱动的状态机，它调用 epoll_wait 时会存在一个阻塞的情况，
  //     因此，NetHandler 注册了signal_hook 函数来实现中止 epoll_wait 的阻塞并返回到EventSystem的功能
  for (i = 0; i < n; i++) {
    if (thr->ethreads_to_be_signalled[i]) {
      thr->ethreads_to_be_signalled[i]->EventQueueExternal.signal();
      if (thr->ethreads_to_be_signalled[i]->signal_hook)
        thr->ethreads_to_be_signalled[i]->signal_hook(thr->ethreads_to_be_signalled[i]);
      thr->ethreads_to_be_signalled[i] = 0;
    }   
  }
  // 队列长度清零
  // 由于队列长度在达到最大值时会转换为映射表，而且没有逆向转换的逻辑，因此每次要完全处理表内所有的元素
  thr->n_ethreads_to_be_signalled = 0;
}
```

## 关于 EAGER_SIGNALLING

在 [ProtectedQueue.cc](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/ProtectedQueue.cc) 中对此进行了描述：

```
 36 // The protected queue is designed to delay signaling of threads
 37 // until some amount of work has been completed on the current thread
 38 // in order to prevent excess context switches.
 39 //
 40 // Defining EAGER_SIGNALLING disables this behavior and causes                                                                      
 41 // threads to be made runnable immediately.
 42 //
 43 // #define EAGER_SIGNALLING
```

翻译如下：

  - 保护队列被设计为：
    - 当前线程已经完成了一定量的工作时，才通知线程
    - 采用延迟通知的方式，可以阻止/减少过度的上下文切换
  - 但是可以定义宏 EAGER_SIGNALLING 来关闭上述行为，让通知立即发出

解释一下：

  - 如果需要为特定 Event 立即发起通知，则会立即唤醒目标线程
    - 目标线程被唤醒后 CPU 需要加载它休眠之前的寄存器状态，由此导致上下文切入
    - 处理完该指定 Event 之后，目标线程再次进入修庙，由此导致上下问切出
    - 如果每处理一个 Event 都要切入、切出一次，那么上下文切换次数会非常的惊人
  - 如果采用延迟通知，
    - 在每一个EThread::execute()的循环中，
      - 将需要通知的目标线程保存在当前线程的一个列表内
      - 循环结束时，遍历该列表，一次性通知所有需要唤醒的线程
    - 此时，在一个目标线程里可能存在多个 Event 需要运行
    - 同样的一次上下文切入操作，则可以处理多个 Event 之后再切出
    - 这样就减少了上下文切换的次数

## 参考资料

![EventQueue - EThread - Signals](https://cdn.rawgit.com/oknet/atsinternals/master/CH01-EventSystem/CH01-EventSystem-001.svg)

- [I_EThread.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_EThread.h)
- [P_UnixEThread.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_UnixEThread.h)
- [ProtectedQueue.cc](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/ProtectedQueue.cc)
- [EventProcessor](CH01S10-Interface-EventProcessor.zh.md)
- [Event](CH01S08-Core-Event.zh.md)
