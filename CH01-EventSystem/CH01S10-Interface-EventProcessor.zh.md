# 接口界面：EventProcessor

EventProcessor负责启动EThread，进行每一个EThread的初始化。

事件系统的主线程组(eventProcessor)，是事件系统的核心组成部分。

- 启动后，它负责创建和管理线程组，线程组中的线程则周期性的或者在指定的时间执行用户自定义的异步任务。
- 通过它提供的一组调度函数，你可以让一个线程来回调指定的延续。这些函数调用不会阻塞。
- 他们返回一个Event对象，并安排在 稍后 或 特定时间之后 或 尽快 或 以一定的时间间隔 回调延续。

唯一实例模式

- 每一个可执行的操作都是通过全局实例eventProcessor提供给EThread组
- 没有必要创建重复的EventProcessor实例，因此EventProcessor被设计为唯一实例模式
- 注意：EventProcessor的任何函数／方法都是不可重入的

线程组（事件类型）：

- 当EventProcessor开始时，第一组诞生的线程被分配了特定的id：ET_CALL
- 根据不同的状态机或协议的复杂性，你一定需要创建额外的线程和EventProcessor，你可以选择：
   - 通过spawn_thread创建一个独立的线程
      - 该线程是完全独立于线程组之外的
      - 在延续执行结束之前，它都不会退出
      - 该线程有自己的事件系统
   - 通过spawn_event_theads创建一个线程组
      - 你会得到一个id或Event Type
      - 你可以在将来通过该id或者Event Type在该组线程上调度延续

回调事件的代码：

- UNIX
   - 在所有的调度函数中，并未使用callback_event参数
   - 回调时，传递给延续处理函数的事件代码始终是EVENT_IMMEDIATE
- NT（相关代码在开源时已经被砍掉）
   - 传递给延续处理函数的事件代码的值是由callback_event参数提供。

事件分配策略：

- 事件由EventProcessor分配和释放
- 状态机可以访问返回的（returned），非周期性（non-recurring）事件，直到它被取消或该事件的回调完成
- 对于周期性（recurring）事件，该事件可以被访问，直到它被取消。
- 一旦事件完成或取消，eventProcessor将释放它。

调度方式：

- schedule_imm
   - 在特定的线程组上调度该延续，等待事件或超时。
   - 请求EventProcessor立即调度对于此延续的回调，该回调由指定的线程组（event type）里的线程处理。
- schedule_imm_signal
   - 与schedule_imm相似的功能，只是在最后还要向线程发送信号。
   - 这个函数其实是专为网络模块设计的
      - 一个线程可能一直阻塞在epoll_wait上，通过引入一个pipe或者eventfd，当调度一个线程执行某个event时，异步通知该线程从epoll_wait解放出来
      - 另外一个具有相同功能的函数 EThread::schedule_imm_signal          
- schedule_at
   - 请求EventProcessor在指定的时间（绝对时间）调度对于此延续的回调，该回调由指定的线程组（event type）里的线程处理。
- schedule_in
   - 请求EventProcessor在指定时间内（相对时间）调度对于此延续的回调，该回调由指定的线程组（event type）里的线程处理。
- schedule_every
   - 请求EventProcessor在每隔指定的时间调度对于此延续的回调，该回调由指定的线程组（event type）里的线程处理。
   - 如果间隔时间为负数，则表示此回调为随时执行，只要有机会，EventProcessor就会进行调度。
      - 目前此种类型的调度只有epoll_wait一种

## 定义

```
class EventProcessor : public Processor
{
public:
  /**
    Spawn an additional thread for calling back the continuation. Spawns
    a dedicated thread (EThread) that calls back the continuation passed
    in as soon as possible.

    @param cont continuation that the spawn thread will call back
      immediately.
    @return event object representing the start of the thread.

  */
  // 创建一个包含Cont的立即执行的事件（Event）
  // 然后创建 DEDICATED 类型的EThread，并传入该事件（Event）
  // 然后启动EThread，并将事件（Event）返回调用者。
  // EThread启动后会立即调用Cont->handleEvent
  Event *spawn_thread(Continuation *cont, const char *thr_name, size_t stacksize = 0);

  /**
    Spawns a group of threads for an event type. Spawns the number of
    event threads passed in (n_threads) creating a thread group and
    returns the thread group id (or EventType). See the remarks section
    for Thread Groups.

    @return EventType or thread id for the new group of threads.

  */
  // 创建一个数量为 n_threads 的线程组
  // 线程组的id 取 n_thread_groups ，然后增加它的计数
  EventType spawn_event_threads(int n_threads, const char *et_name, size_t stacksize);


  // 此处省略上面已经介绍过的 5种schedule_*()方法
  // Event *schedule_*();

  ////////////////////////////////////////////
  // reschedule an already scheduled event. //
  // may be called directly or called by    //
  // schedule_xxx Event member functions.   //
  // The returned value may be different    //
  // from the argument e.                   //
  ////////////////////////////////////////////
  // 这些reschedule方法已经被废弃，整个ATS代码中没有看到任何调用和声明的部分
  Event *reschedule_imm(Event *e, int callback_event = EVENT_IMMEDIATE);
  Event *reschedule_at(Event *e, ink_hrtime atimeout_at, int callback_event = EVENT_INTERVAL);
  Event *reschedule_in(Event *e, ink_hrtime atimeout_in, int callback_event = EVENT_INTERVAL);
  Event *reschedule_every(Event *e, ink_hrtime aperiod, int callback_event = EVENT_INTERVAL);

  // 构造函数
  EventProcessor();

  /**
    Initializes the EventProcessor and its associated threads. Spawns the
    specified number of threads, initializes their state information and
    sets them running. It creates the initial thread group, represented
    by the event type ET_CALL.

    @return 0 if successful, and a negative value otherwise.

  */
  // 初始化唯一实例eventProcessor，并且启动ET_CALL/ET_NET线程组
  int start(int n_net_threads, size_t stacksize = DEFAULT_STACKSIZE);

  /**
    Stop the EventProcessor. Attempts to stop the EventProcessor and
    all of the threads in each of the thread groups.

  */
  // 停止唯一实例eventProcessor，但是该函数定义是空的
  virtual void shutdown();

  /**
    Allocates size bytes on the event threads. This function is thread
    safe.

    @param size bytes to be allocated.

  */
  // 从线程私有数据区分配空间，通过原子操作来分配，所以是线程安全的。
  off_t allocate(int size);

  /**
    An array of pointers to all of the EThreads handled by the
    EventProcessor. An array of pointers to all of the EThreads created
    throughout the existence of the EventProcessor instance.

  */
  // 每创建一个 REGULAR 类型的EThread都会保存指向其实例的指针。
  EThread *all_ethreads[MAX_EVENT_THREADS];

  /**
    An array of pointers, organized by thread group, to all of the
    EThreads handled by the EventProcessor. An array of pointers to all of
    the EThreads created throughout the existence of the EventProcessor
    instance. It is a two-dimensional array whose first dimension is the
    thread group id and the second the EThread pointers for that group.

  */
  // 每创建一个 REGULAR 类型的EThread都会保存指向其实例的指针。
  // 这是一个二维指针数组，按照EThread的功能分组归类。
  EThread *eventthread[MAX_EVENT_TYPES][MAX_THREADS_IN_EACH_TYPE];

  // 用于 assign_thread，被用来记录上一次分配线程的位置，以实现RoundRobin算法
  unsigned int next_thread_for_type[MAX_EVENT_TYPES];
  // 记录每一种 REGULAR 线程组的当前总数，例如：ET_CALL线程组里总共有多少个线程？
  int n_threads_for_type[MAX_EVENT_TYPES];

  /**
    Total number of threads controlled by this EventProcessor.  This is
    the count of all the EThreads spawn by this EventProcessor, excluding
    those created by spawn_thread

  */
  // 总共有多少个 REGULAR 类型的EThread，不包含 DEDICATED 类型的EThread。
  int n_ethreads;

  /**
    Total number of thread groups created so far. This is the count of
    all the thread groups (event types) created for this EventProcessor.

  */
  // 当前有多少组线程，例如：ET_CALL，ET_TASK，ET_SSL等
  int n_thread_groups;

private:
  // prevent unauthorized copies (Not implemented)
  EventProcessor(const EventProcessor &);
  EventProcessor &operator=(const EventProcessor &);

public:
  /*------------------------------------------------------*\
  | Unix & non NT Interface                                |
  \*------------------------------------------------------*/

  // schedule_*() 共享的底层实现
  Event *schedule(Event *e, EventType etype, bool fast_signal = false);
  // 根据RoundRobin算法从指定类型（etype）的线程组里选择一个EThread，返回其指针
  // 通常用来分配一个EThread，然后将事件（Event）传入
  EThread *assign_thread(EventType etype);

  // 管理 DEDICATED 类型的EThread专用数组
  // 每创建一个DEDICATED类型的EThread，则在此数组中保存指向其实例的指针
  // 同时增加 n_dthreads 的计数
  EThread *all_dthreads[MAX_EVENT_THREADS];
  int n_dthreads; // No. of dedicated threads

  // 记录线程私有数据区的分配情况，由allocate()采用原子操作来读写
  volatile int thread_data_used;
};

extern inkcoreapi class EventProcessor eventProcessor;
```

构造函数对EventProcessor进行基本的初始化，都是置0操作。

```
TS_INLINE
EventProcessor::EventProcessor() : n_ethreads(0), n_thread_groups(0), n_dthreads(0), thread_data_used(0)
{
  memset(all_ethreads, 0, sizeof(all_ethreads));
  memset(all_dthreads, 0, sizeof(all_dthreads));
  memset(n_threads_for_type, 0, sizeof(n_threads_for_type));
  memset(next_thread_for_type, 0, sizeof(next_thread_for_type));
}
```

## 创建 DEDICATED 类型的EThread

EventProcessor 提供了专门用于创建 DEDICATED 类型的EThread的方法：

```
Event *
EventProcessor::spawn_thread(Continuation *cont, const char *thr_name, size_t stacksize)
{
  ink_release_assert(n_dthreads < MAX_EVENT_THREADS);
  Event *e = eventAllocator.alloc();

  // 创建一个包含Cont的立即执行的事件（Event）
  e->init(cont, 0, 0);
  // 然后创建 DEDICATED 类型的EThread，并传入该事件（Event）：oneevent＝e
  all_dthreads[n_dthreads] = new EThread(DEDICATED, e);
  e->ethread = all_dthreads[n_dthreads];
  e->mutex = e->continuation->mutex = all_dthreads[n_dthreads]->mutex;
  n_dthreads++;
  // 启动EThread，
  e->ethread->start(thr_name, stacksize);
  // 在EThread的分析中，可以看到 DEDICATED 类型的EThread启动后会立即调用Cont->handleEvent
  // 最后将事件（Event）返回调用者。
  return e;
}
```

可以通过全局唯一实例eventProcessor.spawn_thread()调用此方法，通常创建 DEDICATED 类型的EThread返回的事件（Event）没什么用，不需要保存。

## 创建 REGULAR 类型的EThread

EventProcessor 提供了专门用于创建 REGULAR 类型的EThread组的方法，此方法一次创建一批线程来实现并发处理某种功能的事件（Event）：

```
EventType
EventProcessor::spawn_event_threads(int n_threads, const char *et_name, size_t stacksize)
{
  char thr_name[MAX_THREAD_NAME_LENGTH];
  EventType new_thread_group_id;
  int i;

  ink_release_assert(n_threads > 0);
  ink_release_assert((n_ethreads + n_threads) <= MAX_EVENT_THREADS);
  ink_release_assert(n_thread_groups < MAX_EVENT_TYPES);

  // 分配线程组ID
  new_thread_group_id = (EventType)n_thread_groups;

  // 通过 for 循环逐个创建 REGULAR 类型的EThread实例
  for (i = 0; i < n_threads; i++) {
    // 创建EThread实例，并初始化 tt=REGULAR, id=n_ethreads+i
    EThread *t = new EThread(REGULAR, n_ethreads + i);
    // 保存EThread实例的指针到数组中，统一管理
    all_ethreads[n_ethreads + i] = t;
    eventthread[new_thread_group_id][i] = t;
    // 设置EThread可以处理的事件（Event）类型
    t->set_event_type(new_thread_group_id);
  }

  // 设置该线程组的线程数，注意
  n_threads_for_type[new_thread_group_id] = n_threads;
  // 通过 for 循环逐个线程组内的线程
  for (i = 0; i < n_threads; i++) {
    snprintf(thr_name, MAX_THREAD_NAME_LENGTH, "[%s %d]", et_name, i);
    eventthread[new_thread_group_id][i]->start(thr_name, stacksize);
  }

  // 创建完毕，累加线程组计数
  n_thread_groups++;
  // 累加总线程数
  n_ethreads += n_threads;
  Debug("iocore_thread", "Created thread group '%s' id %d with %d threads", et_name, new_thread_group_id, n_threads);

  // 返回线程组ID
  return new_thread_group_id;
}
```

可以通过全局唯一实例eventProcessor.spawn_event_threads()调用此方法，通常创建 REGULAR 类型的EThread返回的是线程组的ID，需要保存备用。

## 第一个EThread线程组ET_CALL/ET_NET的启动（eventProcessor启动流程）

在系统启动时，有一个全局eventProcessor，作为Event System的唯一实例，最早启动，并开始运行

- eventProcessor内维护了一组类型为ET_CALL，名称为[ET_NET n]的线程
- 在ATS中，类型为ET_CALL或ET_NET的线程组是同一个组
- 每个线程内部都在执行EThread::execute()的REGULAR部分
- 每个线程都设置了CPU affinity（CPU绑定）
- 但是[ET_NET 0]是由main()函数在最后执行thread->execute()变成的
- 上面这个过程是通过eventProcessor.start(...)实现的

```
int
EventProcessor::start(int n_event_threads, size_t stacksize)
{
  char thr_name[MAX_THREAD_NAME_LENGTH];
  int i;

  // do some sanity checking.
  // 通过静态变量started，防止eventProcessor.start被重入
  static int started = 0;
  ink_release_assert(!started);
  ink_release_assert(n_event_threads > 0 && n_event_threads <= MAX_EVENT_THREADS);
  started = 1;

  // 因为是最早创建和启动的线程，所以初始化：
  //     当前线程数量为n_event_threads
  //     当前线程组的数量为1
  n_ethreads = n_event_threads;
  n_thread_groups = 1;

  // 创建EThread实例，但是注意 i==0 时的特殊处理是为了main()在最后可以变成[ET_NET 0]
  for (i = 0; i < n_event_threads; i++) {
    EThread *t = new EThread(REGULAR, i);
    if (i == 0) {
      ink_thread_setspecific(Thread::thread_data_key, t);
      global_mutex = t->mutex;
      t->cur_time = ink_get_based_hrtime_internal();
    }
    all_ethreads[i] = t;

    eventthread[ET_CALL][i] = t;
    t->set_event_type((EventType)ET_CALL);
  }
  // 设置ET_CALL/ET_NET类型的线程数量
  n_threads_for_type[ET_CALL] = n_event_threads;

  // 下面的过程是利用HWLOC库来实现CPU affinity
#if TS_USE_HWLOC
  // 首先判断要支持的CPU affinity的类型
  int affinity = 1;
  REC_ReadConfigInteger(affinity, "proxy.config.exec_thread.affinity");
  hwloc_obj_t obj;
  hwloc_obj_type_t obj_type;
  int obj_count = 0;
  char *obj_name;

  switch (affinity) {
  case 4: // assign threads to logical processing units
// Older versions of libhwloc (eg. Ubuntu 10.04) don't have HWLOC_OBJ_PU.
#if HAVE_HWLOC_OBJ_PU
    obj_type = HWLOC_OBJ_PU;
    obj_name = (char *)"Logical Processor";
    break;
#endif
  case 3: // assign threads to real cores
    obj_type = HWLOC_OBJ_CORE;
    obj_name = (char *)"Core";
    break;
  case 1: // assign threads to NUMA nodes (often 1:1 with sockets)
    obj_type = HWLOC_OBJ_NODE;
    obj_name = (char *)"NUMA Node";
    if (hwloc_get_nbobjs_by_type(ink_get_topology(), obj_type) > 0) {
      break;
    }
  case 2: // assign threads to sockets
    obj_type = HWLOC_OBJ_SOCKET;
    obj_name = (char *)"Socket";
    break;
  default: // assign threads to the machine as a whole (a level below SYSTEM)
    obj_type = HWLOC_OBJ_MACHINE;
    obj_name = (char *)"Machine";
  }

  // 根据CPU AF的类型，判断有多少个逻辑CPU
  obj_count = hwloc_get_nbobjs_by_type(ink_get_topology(), obj_type);
  Debug("iocore_thread", "Affinity: %d %ss: %d PU: %d", affinity, obj_name, obj_count, ink_number_of_processors());

#endif
  // 通过 for 循环启动每一个EThread
  for (i = 0; i < n_ethreads; i++) {
    ink_thread tid;
    // 这里同样对[ET_NET 0]做了特殊处理
    if (i > 0) {
      snprintf(thr_name, MAX_THREAD_NAME_LENGTH, "[ET_NET %d]", i);
      tid = all_ethreads[i]->start(thr_name, stacksize);
    } else {
      tid = ink_thread_self();
    }
    // 然后设置CPU AF特性
#if TS_USE_HWLOC
    if (obj_count > 0) {
      obj = hwloc_get_obj_by_type(ink_get_topology(), obj_type, i % obj_count);
#if HWLOC_API_VERSION >= 0x00010100
      int cpu_mask_len = hwloc_bitmap_snprintf(NULL, 0, obj->cpuset) + 1;
      char *cpu_mask = (char *)alloca(cpu_mask_len);
      hwloc_bitmap_snprintf(cpu_mask, cpu_mask_len, obj->cpuset);
      Debug("iocore_thread", "EThread: %d %s: %d CPU Mask: %s\n", i, obj_name, obj->logical_index, cpu_mask);
#else
      Debug("iocore_thread", "EThread: %d %s: %d\n", i, obj_name, obj->logical_index);
#endif // HWLOC_API_VERSION
      hwloc_set_thread_cpubind(ink_get_topology(), tid, obj->cpuset, HWLOC_CPUBIND_STRICT);
    } else {
      Warning("hwloc returned an unexpected value -- CPU affinity disabled");
    }
#else
    // 如果没有HWLOC库，那就只好啥都不做了
    (void)tid;
#endif // TS_USE_HWLOC
  }

  Debug("iocore_thread", "Created event thread group id %d with %d threads", ET_CALL, n_event_threads);

  // 返回 0，准确的说是返回ET_CALL
  // 其实就是创建的线程组的ID，因为是第一组，所以是0，跟spawn_event_threads()是一样的
  return 0;
}
```

- 随后netProcessor启动，针对每一个服务端口创建NetAccept独立线程
  - 独立线程由execute()的DEDICATED部分实现

- NetAccept独立线程为每一个接收到的sockfd创建一个netVC
  - 首先将该netVC的handler设置为acceptEvent
  - 然后将netVC封装为立即执行的Event类型放入eventProcessor的外部队列


## 线程间的事件（Event）调度

当通过eventProcessor.schedule_*()方法将事件（Event）放入线程组的时候，如何决定由具体由哪个线程（EThread）来处理呢？

这个机制实现的比较简单，每次创建事件（Event）时会调用assign_thread()来获取一个指向EThread的指针，然后就由这个线程（EThread）来处理该事件（Event），具体实现如下：

```
TS_INLINE EThread *
EventProcessor::assign_thread(EventType etype)
{
  int next;

  ink_assert(etype < MAX_EVENT_TYPES);
  // 借助 next_thread_for_type[]数组 记住上一次分配的线程
  // 然后选择下一个相邻的线程
  if (n_threads_for_type[etype] > 1)
    next = next_thread_for_type[etype]++ % n_threads_for_type[etype];
  else
    next = 0;
  // 通过 eventthread[etype][]数组 得到下一个EThread的指针
  return (eventthread[etype][next]);
}
```

对 next_thread_for_type[] 数组的操作没有加锁，由此可能会导致竞争问题，但是不会导致程序崩溃，由于并不是非常严格的要求线程间的任务分配完全均衡，因此这里就不做过多的要求。

## 线程私有数据空间的分配管理

在每一个EThread内部都有一个thread_private数组，用来存储线程的私有数据。

但是要保证每一次分配，所有EThread线程都要同步，如果进入到每一个EThread内部进行分配，就要调用多次分配过程，eventProcessor中提供了一个方法用来进行分配这个空间。

所有EThread的thread_private区域，都是一样大小的，那么只需要为每一次分配指定一个偏移量（offset），每个EThread内部操作的时候，通过这个偏移量访问thread_private区域，就可以保证一次分配，所有EThread都进行了分配。为了保证内存访问效率，该方法在分配时还进行内存对齐。

```
TS_INLINE off_t
EventProcessor::allocate(int size)
{
  // offsetof(EThread, thread_private) 方法返回thread_private成员在class EThread中的相对偏移量
  // 通过INK_ALIGN进行向上对齐
  static off_t start = INK_ALIGN(offsetof(EThread, thread_private), 16);
  // 因为内存对齐就会浪费掉thread_private开头的一小部分空间，计算出这个空间的大小
  static off_t loss = start - offsetof(EThread, thread_private);
  // 然后再使用INK_ALIGN对即将分配的空间尺寸size进行向上对齐
  size = INK_ALIGN(size, 16); // 16 byte alignment

  // 接下来是进行原子操作，分配空间
  // 使用thread_data_used来记录已经被分配的内存空间
  int old;
  do {
    old = thread_data_used;
    if (old + loss + size > PER_THREAD_DATA)
      // 没有空间可以分配时，返回 -1
      return -1;
  } while (!ink_atomic_cas(&thread_data_used, old, old + size));

  // 分配成功，返回偏移量，此时：
  //   thread_data_used 已经指向未分配区域的第一个地址
  //   start 是静态变量，永远指向对齐后的 thread_private 第一个地址
  //   old 则保存了上一次的 thread_data_used 的值
  return (off_t)(old + start);
}
```

偏移地址有了，如何在这个位置存取数据呢？

- 可以通过宏定义 ETHREAD_GET_PTR(thread, offset) 获得一个指针。
- 然后再对这个指针所在的地址进行初始化操作
- 初始化时一定要保证与申请这块内存区域时相同的数据类型的尺寸

下面以 NetHandler 实例在线程内空间分配的过程为例，来解释这个过程：

```
UnixNetProcessor::start(int, size_t)
{
...
  // 通过allocate分配内存，获得偏移量
  netHandler_offset = eventProcessor.allocate(sizeof(NetHandler));
...
}

P_UnixNet.h::get_NetHandler(EThread *t)

// 封装了专门获取NetHandler的操作
static inline NetHandler *
get_NetHandler(EThread *t)
{
  // 使用宏定义构造指针
  return (NetHandler *)ETHREAD_GET_PTR(t, unix_netProcessor.netHandler_offset);                                                           
}

UnixNet.cc::initialize_thread_for_net
void
initialize_thread_for_net(EThread *thread)
{
  // 这里是一个特殊的new的用法：new ( Buffer ) ClassName(Parameters)，表示：
  //   在指定地址：由get_NetHandler(thread)返回
  //   然后以 NetHandler() 方式调用构造函数，完成初始化
  //   ink_dummy_for_new 在I_EThread.h中定义为一个空的class
  new ((ink_dummy_for_new *)get_NetHandler(thread)) NetHandler();
...
}
// 任何时候需要访问，都可以：
NetHandler *nh = get_NetHandler(ethread);
```

在线程私有数据区存放的数据，通常是需要一次性分配，然后就不会释放的，例如：

- NetHandler
- PollCont
- udpNetHandler
- PollCont(udpNetInternal)
- RecRawStat

## 参考资料
[I_EventProcessor.h](http://github.com/opensource/trafficserver/tree/master/iocore/eventsystem/I_EventProcessor.h)

# 基类 Processor

EventProcessor 继承自 Processor，在ATS的IO Core中，针对每一种功能，都定义了Processor，这些 Processor 都是继承自 Processor 基类，如：

- tasksProcessor
- netProcessor
- sslProcessor
- dnsProcessor
- cacheProcessor
- clusterProcessor
- hostDBProcessor
- UDPNetProcessor(udpNet)

## 定义

```
class Processor
{
public:
  virtual ~Processor();

  /**
    Returns a Thread appropriate for the processor.

    Returns a new instance of a Thread or Thread derived class of
    a thread which is the thread class for the processor.

    @param thread_index reserved for future use.

  */
  virtual Thread *create_thread(int thread_index);

  /**
    Returns the number of threads required for this processor. If
    the number is not defined or not used, it is equal to 0.

  */
  virtual int get_thread_count();

  /**
    This function attemps to stop the processor. Please refer to
    the documentation on each processor to determine if it is
    supported.

  */
  virtual void
  shutdown()
  {
  }

  /**
    Starts execution of the processor.

    Attempts to start the number of threads specified for the
    processor, initializes their states and sets them running. On
    failure it returns a negative value.

    @param number_of_threads Positive value indicating the number of
        threads to spawn for the processor.
    @param stacksize The thread stack size to use for this processor.

  */
  virtual int
  start(int number_of_threads, size_t stacksize = DEFAULT_STACKSIZE)
  {
    (void)number_of_threads;
    (void)stacksize;
    return 0;
  }

protected:
  Processor();

private:
  // prevent unauthorized copies (Not implemented)
  Processor(const Processor &);
  Processor &operator=(const Processor &);
};
```

## 参考资料
[I_Processor.h](http://github.com/opensource/trafficserver/tree/master/iocore/eventsystem/I_Processor.h)
