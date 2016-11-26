# 核心组件：NetHandler

专门处理网络IO相关的操作，从poll fd数组中获取需要处理的fd集合，然后根据状态（读、写）回调VConnection：

  - UnixNetVConnection
  - SSLNetVConnection
  - DNSConnection

另外还有一个特殊的：

  - ASYNC_SIGNAL
  - 用于打断 epoll_wait 的等待

在 NetHandler 中有八个队列：

  - Que: open_list
    - 保存当前EThread管理的所有VC
  - Dlist: cop_list
    - 在InactivityCop状态机中遍历open_list，把vc->thread==this_ethread()的vc放入这个cop_list
    - 然后对cop_list内的vc逐个检查超时情况
    - PS：我查阅了代码后，发现在open_list内的vc，不存在vc->thread不是this_ethread()的情况
  - Que: active_queue
    - 由上层状态机，如HttpSM等，通过add_to_active_queue将vc放入，用于实现inactive timeout
    - 在InactivityCop中，通过manage_active_queue来清理
    - 另外还提供了remove_from_active_queue的方法，从队列中删除vc
    - 注意：active_queue 和 keep_alive_queue 是互斥的，一个NetVC不可以同时出现在这两个队列中，这由add_(keep_alive|active)_queue保证
  - Que: keep_alive_queue
    - 由上层状态机，如HttpSM等，通过add_to_keep_alive_queue将vc放入，用于实现keep alive timeout
    - 在InactivityCop中，通过manage_keep_alive_queue来清理
    - 另外还提供了remove_from_keep_alive_queue的方法，从队列中删除vc
    - 对于 active_queue 和 keep_alive_queue，将在 InactivityCop 中详细分析
  - ASLLM: (read|write)_enable_list
    - 在执行异步reenable时，将vc直接放入原子队列
    - 在EThread A中要reenable一个VC，但是这个VC是由EThread B管理的，此时就属于异步reenable
  - QueM: (read|write)_ready_list
    - 在NetHandler::mainNetEvent执行时，会首先将(read|write)_enable_list全部取出，导入到(read|write)_ready_list
    - 另外在执行同步reenable时，也会将vc直接放入此队列
    - 在EThread A中要reenable一个VC，这个VC也是由EThread A管理的，此时就属于同步reenable

其中 (read|write)_enable_list 两个队列是原子队列，可以不加锁直接操作，其它六个队列都需要加锁才能操作

UnixNetVConnection 与 SSLNetVConnection 都可能在上述几个队列中流转：

![NetVC Routing Map](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-002.svg)

## 定义

```
//
// NetHandler
//
// A NetHandler handles the Network IO operations.  It maintains
// lists of operations at multiples of it's periodicity.
//
class NetHandler : public Continuation
{
public:
  Event *trigger_event;

  // 读、写Ready List的vc队列
  // 表示将UnixNetVConnection->read.ready_link链接起来形成队列，UnixNetVConnection->read是一个NetState类型
  QueM(UnixNetVConnection, NetState, read, ready_link) read_ready_list;
  // 表示将UnixNetVConnection->write.ready_link链接起来形成队列，UnixNetVConnection->write是一个NetState类型
  QueM(UnixNetVConnection, NetState, write, ready_link) write_ready_list;

  // InactivityCop的连接超时控制
  // 在以下方法中将vc加入open_list队列
  //     NetAccept::acceptFastEvent
  //     UnixNetVConnection::acceptEvent
  //     UnixNetVConnection::connectUp
  // 在close_UnixNetVConnection方法中：
  //     会将vc从open_list、cop_list、enable list和ready list中移除
  //     同时还会调用相关方法把vc从keep alive和active队列中移除
  // open_list保存了当前EThread打开的每一个UnixNetVConnection
  Que(UnixNetVConnection, link) open_list;
  // 在InactivityCop::check_inactivity方法（此方法每分钟运行一次）中：
  //     首先遍历open_list队列，将所有vc压入cop_list
  //     然后逐个从cop_list弹出vc，对超时的vc进行回调
  //     最后重新整理active和keep alive队列
  DList(UnixNetVConnection, cop_link) cop_list;

  // 读、写Enable List的vc队列，支持原子操作
  // 表示将UnixNetVConnection->read.enable_link链接起来形成队列，UnixNetVConnection->read是一个NetState类型
  ASLLM(UnixNetVConnection, NetState, read, enable_link) read_enable_list;
  // 表示将UnixNetVConnection->write.enable_link链接起来形成队列，UnixNetVConnection->write是一个NetState类型
  ASLLM(UnixNetVConnection, NetState, write, enable_link) write_enable_list;

  // keep alive 队列及其长度
  // 通过以下几个方法管理：
  //     NetHandler::add_to_keep_alive_queue()
  //     NetHandler::remove_from_keep_alive_queue()
  //     NetHandler::manager_keep_alive_queue()
  Que(UnixNetVConnection, keep_alive_queue_link) keep_alive_queue;
  uint32_t keep_alive_queue_size;
  // active 队列及其长度
  // 通过以下几个方法管理：
  //     NetHandler::add_to_active_queue()
  //     NetHandler::remove_from_active_queue()
  //     NetHandler::manager_active_queue()
  Que(UnixNetVConnection, active_queue_link) active_queue;
  uint32_t active_queue_size;

  // 用于限制keep alive和active队列的总大小：
  //     在 NetHandler::configure_per_thread() 中设置
  //     在 NetHandler::manage_keep_alive_queue() 中验证
  //     在 NetHandler::manage_active_queue() 中验证
  //     keep alive ＋ active 两个队列总长度不能超过max_connections_per_thread_in
  // 配置参数：proxy.config.net.max_connections_in
  // 每个EThread允许的最大连接数 ＝ max_connections_in / threads
  uint32_t max_connections_per_thread_in;
  // 配置参数：proxy.config.net.max_connections_active_in
  // 每个EThread允许的最大活动连接数 ＝ max_connections_active_in / threads
  uint32_t max_connections_active_per_thread_in;
  // 注意：上面在计算threads数量时，包含ET_NET和ET_SSL两种类型的EThread

  // 以下5个值由配置参数设置，用于管理active和keep alive队列
  // configuration settings for managing the active and keep-alive queues
  uint32_t max_connections_in;
  uint32_t max_connections_active_in;
  uint32_t inactive_threashold_in;
  uint32_t transaction_no_activity_timeout_in;
  uint32_t keep_alive_no_activity_timeout_in;

  // 可能已经被废弃
  time_t sec;
  int cycles;

  // 初始化函数，初始化配置变量等
  int startNetEvent(int event, Event *data);
  // 主处理函数，遍历poll fd数组，执行回调
  int mainNetEvent(int event, Event *data);
  // 无实际定义，已废弃
  int mainNetEventExt(int event, Event *data);
  // 将Enable List导入Ready List
  void process_enabled_list(NetHandler *);

  // 管理keep alive队列
  void manage_keep_alive_queue();
  // 管理active队列
  bool manage_active_queue();
  // 将VC添加到keep alive队列
  void add_to_keep_alive_queue(UnixNetVConnection *vc);
  // 从keep alive队列删除VC
  void remove_from_keep_alive_queue(UnixNetVConnection *vc);
  // 将VC添加到active队列
  bool add_to_active_queue(UnixNetVConnection *vc);
  // 从active队列删除VC
  void remove_from_active_queue(UnixNetVConnection *vc);

  // 设置 max_connections_per_thread_in 和 max_connections_active_per_thread_in
  void configure_per_thread();

  // 构造函数
  NetHandler();

private:
  // 强制关闭VC
  void _close_vc(UnixNetVConnection *vc, ink_hrtime now, int &handle_event, int &closed, int &total_idle_time,
                 int &total_idle_count);
};
```

## enable_list 和 ready_list

enable_list是在执行UnixNetVConnection::reenable()时，遇到无法取得nethandler mutex时，临时保存待激活VC的队列。

```
struct NetState {
  // 表示当前VIO是否是激活的状态
  volatile int enabled;
  // VIO 实例
  VIO vio;
  // 定义双向链表节点，可以把多个UnixNetVConnection连接起来，间接的把NetState也就联系了起来
  // ready_link有两个成员：UnixNetVConnection *prev和集成自SLINK的UnixNetVConnection *next
  Link<UnixNetVConnection> ready_link;
  // 定义单向链表节点，可以把多个UnixNetVConnection连接起来，间接的把NetState也就联系了起来
  // enable_link有一个成员：UnixNetVConnection *next
  SLink<UnixNetVConnection> enable_link;
  
  // 表示这个vc被放入了enable_list
  int in_enabled_list;
  
  // 表示这个vc被epoll_wait返回
  int triggered;

  // 构造函数
  // 这里使用VIO::NONE调用VIO的构造函数初始化vio成员
  NetState() : enabled(0), vio(VIO::NONE), in_enabled_list(0), triggered(0) {}
};

class UnixNetVConnection : public NetVConnection
{
...
  // 定义Read和Write两个网络状态，其内包含ready_link和enable_link两个链表节点。
  NetState read;
  NetState write;

  LINK(UnixNetVConnection, cop_link);

  // 开始定义链表操作类，并未定义实例，只是定义类型
  // 带有M的表示
  //    1. 只定义Class类型，而不声明实例（不以M结尾的，如LINK，在定义类型之后，同时声明一个实例）
  //    2. 表示定义的是一个操作内成员的链表类型，Class Name为：Link_Member_LinkName
  //    3. 通常定义方法为：XXXM(Class, Member, LinkName)
  //    4. 区别于不带M的定义方法为：XXX(Class, LinkName);   <-- 最后要有一个分号
  // 定义Class Link_read_ready_link : public Link<UnixNetVConnection>
  LINKM(UnixNetVConnection, read, ready_link)
  // 定义Class Link_read_enable_link : public SLink<UnixNetVConnection>
  SLINKM(UnixNetVConnection, read, enable_link)
  // 定义Class Link_write_ready_link : public Link<UnixNetVConnection>
  LINKM(UnixNetVConnection, write, ready_link)
  // 定义Class Link_write_enable_link : public SLink<UnixNetVConnection>
  SLINKM(UnixNetVConnection, write, enable_link)
  // 结束
  // 有些不太明白，为何这里只定义了类型，而不定义实例？

  LINK(UnixNetVConnection, keep_alive_queue_link);
  LINK(UnixNetVConnection, active_queue_link);
...
};

class NetHandler : public Continuation
{
...
  // 连接UnixNetVConnection类型，FIFO队列（enqueue/dequeue）也可以是FILO（push/pop），需要加锁之后才能操作
  QueM(UnixNetVConnection, NetState, read, ready_link) read_ready_list;
  // 展开为：Queue<UnixNetVConnection, UnixNetVConnection::Link_read_ready_link> read_ready_list;
  QueM(UnixNetVConnection, NetState, write, ready_link) write_ready_list;
  // 展开为：Queue<UnixNetVConnection, UnixNetVConnection::Link_write_ready_link> write_ready_list;
  // 进一步分析：
  // write_ready_list的push方法调用的是DLL<UnixNetVConnection, UnixNetVConnection::Link_write_ready_link>::push方法
  //     该link内head为空时，将调用next方法，获取待插入元素的next指针，将该指针指向head，然后将head指向新放入的元素。
  //     该link内head不为空时，则调用prev方法，获取head元素的prev指针，然后将该指针指向新元素，然后再把新元素的next指针指向head元素，然后head指向新元素。
  // 上面的next方法则是调用的UnixNetVConnection::Link_write_ready_link::next_link()
  //     返回的是 UnixNetVConnection实例->write.ready_link.next;
  // 上面的prev方法则是调用的UnixNetVConnection::Link_write_ready_link::prev_link()
  //     返回的是 UnixNetVConnection实例->write.ready_link.prev;
  // 其中 write 为 NetState 类型 ready_link 则是其 Link<UnixNetVConnection> 类型成员
...
  // 连接UnixNetVConnection类型，支持原子操作的LIFO单向链表，只在链表头部操作，直接操作无需加锁
  ASLLM(UnixNetVConnection, NetState, read, enable_link) read_enable_list;
  // 展开为：AtomicSLL<UnixNetVConnection, UnixNetVConnection::Link_read_enable_link> read_enable_list;
  ASLLM(UnixNetVConnection, NetState, write, enable_link) write_enable_list;
  // 展开为：AtomicSLL<UnixNetVConnection, UnixNetVConnection::Link_write_enable_link> write_enable_list;
...
};
```

如果VC不会跨线程操作，那么NetHandler以下成员就可以满足要求：

  - ready_list 保存所有epoll_wait返回的vc
  - 遍历 ready_list 就可以处理所有网络事件
  - reenable 时只需要把vc添加到ready_list中即可

在多线程的设计中，跨线程的资源访问是很难避免的，但是跨线程的加锁操作也会比较慢，而且还要考虑拿不到锁的时候不能阻塞EThread执行的异步情况

  - NetHandler 实现了 enable_list 这两个分别用于读、写操作的原子队列
  - 这样就可以跨线程直接向 enbale_list 添加 vc
  - 然后 NetHandler 再从 enable_list 把 vc 转入 ready_list
 
但是，我们注意到 NetHandler 的mutex竟然没有继承自EThread，而是通过 new_ProxyMutex 创建了一个新的。

  - 结果就是，EThread 回调 NetHandler 时可能拿不到锁，那就会在process_event中安排重新调度
  - 好处是，另外一个线程如果想访问 NetHandler 是可以拿到锁的

问题：

  - 如果其它线程可以拿到NetHandler的锁，那么为何还要设计原子队列？
  - 如果有原子队列，为什么NetHandler的mutex不是继承自EThread呢？


## 主函数 NetHandler::mainNetEvent(int event, Event *e) 流程分析：


首先，调用 ```process_enabled_list(this)``` 将enable_list转存到ready_list。在函数前的注释中明确了这一步操作是要把在其它线程中enabled的VC移动到ready_list中。

mainNetEvent()是NetHandler状态机的回调函数，而每个线程内会有一个NetHandler实例，在回调时会锁定状态机NetHandler的mutex，只有本线程内运行的状态机才可以操作其内部数据。

当我们在状态机（SM）中需要处理一个关联的VC，但是这个VC在另外一个线程中管理时，那么就牵扯到跨线程的操作，此时由于无法直接操作ready_list，因此ATS IOCore实现了一个中间队列enable_list。

由于enable_list是支持原子操作的，可以在不用加锁的情况下跨线程操作，只需要在每次处理ready_list时，首先把当前的enable_list的内容导入就可以了。

```
//
// Move VC's enabled on a different thread to the ready list
//
void
NetHandler::process_enabled_list(NetHandler *nh)
{
  UnixNetVConnection *vc = NULL;

  SListM(UnixNetVConnection, NetState, read, enable_link) rq(nh->read_enable_list.popall());
  while ((vc = rq.pop())) {
    vc->ep.modify(EVENTIO_READ);
    vc->ep.refresh(EVENTIO_READ);
    vc->read.in_enabled_list = 0;
    if ((vc->read.enabled && vc->read.triggered) || vc->closed)
      nh->read_ready_list.in_or_enqueue(vc);
  }

  SListM(UnixNetVConnection, NetState, write, enable_link) wq(nh->write_enable_list.popall());
  while ((vc = wq.pop())) {
    vc->ep.modify(EVENTIO_WRITE);
    vc->ep.refresh(EVENTIO_WRITE);
    vc->write.in_enabled_list = 0;
    if ((vc->write.enabled && vc->write.triggered) || vc->closed)
      nh->write_ready_list.in_or_enqueue(vc);
  }
}
```

可以看出，只有VC的NetState属性 enabled 和 triggered 设置为true，或者VC的属性 closed 设置true才会被加入ready_list队列。

与NetState分为读（read）和写（write）两个状态对应的，enable_list和ready_list也都是分成读、写两个队列，因此这儿就一下出现了4个队列。

然后，设置poll_timeout的值，用于epoll_wait的等待时间，并调用epoll_wait

```
  if (likely(!read_ready_list.empty() || !write_ready_list.empty() || !read_enable_list.empty() || !write_enable_list.empty()))
    poll_timeout = 0; // poll immediately returns -- we have triggered stuff to process right now
  else
    poll_timeout = net_config_poll_timeout;

  PollDescriptor *pd = get_PollDescriptor(trigger_event->ethread);
  UnixNetVConnection *vc = NULL;
#if TS_USE_EPOLL
  pd->result = epoll_wait(pd->epoll_fd, pd->ePoll_Triggered_Events, POLL_DESCRIPTOR_SIZE, poll_timeout);
  NetDebug("iocore_net_main_poll", "[NetHandler::mainNetEvent] epoll_wait(%d,%d), result=%d", pd->epoll_fd, poll_timeout,
           pd->result);
```

可以看出ATS真是精细到极致，如果ready_list不为空，就把poll_timeout设置为0，这样可以最快的速度处理ready_list内的连接。

接着，是对epoll_wait返回的数据进行处理。详细分析参见下面代码中的注释。
```
  vc = NULL;
  for (int x = 0; x < pd->result; x++) {
    epd = (EventIO *)get_ev_data(pd, x);
    // 首先处理普通的VC读写事件
    if (epd->type == EVENTIO_READWRITE_VC) {
      vc = epd->data.vc;
      // 如果是读事件，或者有错误
      if (get_ev_events(pd, x) & (EVENTIO_READ | EVENTIO_ERROR)) {
        // 设置 VC的read NetState属性 triggered 为true
        vc->read.triggered = 1;
        if (!read_ready_list.in(vc))
          // 如果不在ready_list队列，就放入队列
          read_ready_list.enqueue(vc);
        else if (get_ev_events(pd, x) & EVENTIO_ERROR) {
          // 如果在ready_list队列，但是这次状态出现了ERROR，输出一个DEBUG信息
          // check for unhandled epoll events that should be handled
          Debug("iocore_net_main", "Unhandled epoll event on read: 0x%04x read.enabled=%d closed=%d read.netready_queue=%d",
                get_ev_events(pd, x), vc->read.enabled, vc->closed, read_ready_list.in(vc));
        }
      }
      vc = epd->data.vc;  // 这行多余？？？？？？
      // 如果是写事件，或者有错误
      if (get_ev_events(pd, x) & (EVENTIO_WRITE | EVENTIO_ERROR)) {
        // 设置 VC的write NetState属性 triggered 为true
        vc->write.triggered = 1;
        if (!write_ready_list.in(vc))
          // 如果不在ready_list队列，就放入队列
          write_ready_list.enqueue(vc);
        else if (get_ev_events(pd, x) & EVENTIO_ERROR) {
          // 如果在ready_list队列，但是这次状态出现了ERROR，输出一个DEBUG信息
          // check for unhandled epoll events that should be handled
          Debug("iocore_net_main", "Unhandled epoll event on write: 0x%04x write.enabled=%d closed=%d write.netready_queue=%d",
                get_ev_events(pd, x), vc->write.enabled, vc->closed, write_ready_list.in(vc));
        }
      // 如果不是写事件也没有错误的情况。这儿不是所有的读事件都会触发了？？？？？？
      // 下面的 if 语句有bug，已提交官方并修复：TS-3969
      // 该bug只是影响debug时的输出，并不对ATS的工作流程产生影响
      } else if (!(get_ev_events(pd, x) & EVENTIO_ERROR)) {
        // 就输出一个DEBUG信息
        Debug("iocore_net_main", "Unhandled epoll event: 0x%04x", get_ev_events(pd, x));
      }
    // 然后处理DNS连接事件
    } else if (epd->type == EVENTIO_DNS_CONNECTION) {
      if (epd->data.dnscon != NULL) {
        epd->data.dnscon->trigger(); // Make sure the DNSHandler for this con knows we triggered
#if defined(USE_EDGE_TRIGGER)
        epd->refresh(EVENTIO_READ);
#endif
      }
    }
    // 然后处理异步信号
    else if (epd->type == EVENTIO_ASYNC_SIGNAL) {
      net_signal_hook_callback(trigger_event->ethread);
    }
    ev_next_event(pd, x);
  }

  // 所有的epoll事件都处理完了，最后将事件清零
  pd->result = 0;
```

接下来对ready_list进行处理，先处理read，再处理write。以下以 epoll ET 模式的处理代码为例：
```
#if defined(USE_EDGE_TRIGGER)
  // 遍历read_ready_list队列
  while ((vc = read_ready_list.dequeue())) {
    if (vc->closed)
      // 在close_UnixNetVConnection()中会将vc从read_ready_list和write_ready_list里同时删除，
      // 所以不用担心在下面处理write_ready_list时重复处理vc->closed的情况。
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->read.enabled && vc->read.triggered)
      // 如果enabled和triggered同时为true则调用net_read_io进行读取操作，
      // 在net_read_io()中会呼叫Continuation并传递READ_READY，READ_COMPLETE等事件。
      // 注意：
      //     在net_read_io()中可能会将vc重新放回read_ready_list队列尾部。
      //     如果处理不当，可能导致这个while循环会运行很久，迟迟不会处理写操作，
      //     而且不会返回到EventSystem，将会导致这个ET_NET线程阻塞。
      vc->net_read_io(this, trigger_event->ethread);
    else if (!vc->read.enabled) {
      // 如果enabled为false，那么就从read_ready_list中删除这个vc。此时triggered通常为true。
      // 为什么会出现这种情况？？？？？？
      //     通常将enabled为false时，同时会调用remove将其从队列中移除。
      //     通常能够被epoll_wait的vc，enabled必定也被设置为true了。
      read_ready_list.remove(vc);
    }   
  }
  while ((vc = write_ready_list.dequeue())) {
    if (vc->closed)
      // 在close_UnixNetVConnection()中会将vc从read_ready_list和write_ready_list里同时删除
      close_UnixNetVConnection(vc, trigger_event->ethread);
    else if (vc->write.enabled && vc->write.triggered)
      // 如果enabled和triggered同时为true则调用write_to_net进行写操作，
      // 在write_to_net()中会呼叫Continuation并传递WRITE_READY，WRITE_COMPLETE等事件。
      // 注意：
      //     在write_to_net()中可能会将vc重新放回write_ready_list队列尾部。
      //     如果处理不当，可能导致这个while循环会运行很久，迟迟不会返回到EventSystem，
      //     将会导致这个ET_NET线程阻塞。
      write_to_net(this, vc, trigger_event->ethread);
    else if (!vc->write.enabled) {
      // 如果enabled为false，那么就从write_ready_list中删除这个vc。此时triggered通常为true。
      // 为什么会出现这种情况？？？？？？
      //     通常将enabled为false时，同时会调用remove将其从队列中移除。
      //     通常能够被epoll_wait的vc，enabled必定也被设置为true了。
      write_ready_list.remove(vc);
    }
#else
```

## net_read_io 与 write_to_net

在net_read_io方法里，通过判断MIOBuffer的状态和调用readv()函数的返回值来执行不同的处理过程：
- 如果MIOBuffer满了，那么会忽略本次读事件，并且将此VC的READ VIO禁止掉，除非通过vio->reenable()恢复，否则此VC上不会再次触发读事件。
- 如果MIOBuffer有可以容纳新数据的空间，那么就会调用readv()读取数据，其返回值：
   - 等于-EAGAIN 或 -ENOTCONN，从read ready队列删除VC，但是仍然会让epoll等待此VC上的EVENTIO_READ事件。返回
   - 等于0 或 -ECONNRESET，从read ready队列删除VC，触发VC_EVENT_EOS事件，返回
   - 大于0 并且 vio.ntodo() 小于等于0，触发VC_EVENT_READ_COMPLETE，返回
   - 大于0 并且 vio.ntodo() 大于0，触发VC_EVENT_READ_READY事件，然后调用read_reschedule将此VC重新放回read_ready_list队列，返回

需要注意的是最后的VC_EVENT_READ_READY事件，将VC重新放回read_ready_list队列时，那么在mainNetEvent的while循环中，又会重新调用net_read_io方法，只要read_ready_list队列里的VC持续有数据可读，那么就会一直在这个while循环。从而不会进入到write_ready_list的while循环中。

在ATS中的表现就是，ATS只接收数据，而不转发数据出去，除非接收到了所有的数据才会开始向外发送数据。在网络速度比较快的情况下这种现象会比较明显，网速不好的情况，由于不能保证数据持续到达，因此readv可能很快就会遇到-EAGAIN错误，所以很少产生这个问题。特别是在实现Tunnel的时候，必须考虑到这种情况的出现。

在write_to_net方法里，通过判断MIOBuffer的状态和调用load_buffer_and_write()函数的返回值来执行不同的处理过程：

  - 如果vio.ntodo==0
    - 那么会忽略本次写事件，并且将此VC的WRITE VIO禁止掉，除非通过vio->reenable()恢复，否则此VC上不会再次触发写事件，返回。
  - 如果本次输出不会完成VIO（就是不会触发VC_EVENT_WRITE_COMPLETE），同时MIOBuffer还有空间可以写入数据：
    - 先触发一次VC_EVENT_WRITE_READY事件，让状态机尝试将MIOBuffer填满。
  - 如果MIOBuffer内仍然无数据，将此VC的WRITE VIO禁止掉，返回。
  - 如果MIOBuffer有数据供输出，那么就会调用load_buffer_and_write()写数据，其返回值：
    - 等于 -EAGAIN 或 -ENOTCONN，根据need的值进行如下设定之后返回：
      - need & EVENTIO_WRITE == true，从write ready队列删除VC，但是仍然会让epoll等待此VC上的EVENTIO_WRITE事件。
      - need & EVENTIO_READ == true，从read ready队列删除VC，但是仍然会让epoll等待此VC上的EVENTIO_READ事件。
    - 等于0 或 -ECONNRESET，触发VC_EVENT_EOS事件，返回
    - 大于0 并且 vio.ntodo() 小于等于0，触发VC_EVENT_WRITE_COMPLETE，返回
    - 大于0 并且 vio.ntodo() 大于0，
      - 如果之前触发过VC_EVENT_WRITE_READY事件，并且状态机设置了WBE EVENT，则触发WBE EVENT，返回。
      - 如果之前没有触发过VC_EVENT_WRITE_READY事件，则触发VC_EVENT_WRITE_READY事件，然后调用write_reschedule将此VC重新放回 write_ready_list队列，返回
  - 最后
    - 在写操作之后，再次判断MIOBuffer为空，则将此VC的WRITE VIO禁止掉，返回。
    - 根据need的值，把vc重新放回ready list队列
      - 如果放回write ready list则仍然会被while循环调用，
      - 如果放回read ready list则会在下一次EventSystem的调用时被处理。
      - 猜测此处是为了SSL快速完成握手操作而设置。**

这里就不存在read_from_net的问题，因为write操作比较容易遇到EAGAIN的情况时。但是不排除在高速网络环境，如万兆网卡，可能也会出现与read_from_net相同的问题。

重新让epoll等待事件则会在下一次EventSystem调用NetHandler::mainNetEvent时将此VC放入ready_list队列，并且triggered为true，但是enabled的情况就不好说了。

## 为什么要分成read ready和write ready两个队列

这是iocore核心的通信层，要保证每一次调用能够快速返回，以便与EventSystem能够及时处理新的Event。

  - 所以EThread回调NetHandler状态机的频率非常的高（只要EThread空闲就调用）
  - 这样就可以让每次epoll_wait返回的VC数量比较少，可以迅速处理完读、写两类事件的回调

在这里，read是生产者，将输入的数据放入缓存中，write是消费者，负责消费缓存中的数据。

为了达到高性能，那么生产者会被设计为竭尽全力填满缓冲区，消费者会竭尽全力消费缓冲区的数据。所以：

  - 一旦缓冲区有空闲空间，就要调用生产者来填满缓冲区
  - 一旦缓冲区有数据，就要调用消费者来消费缓冲区的数据

当存在多个生产者和消费者时，批量处理生产者的申请，完成生产之后，再批量处理消费者的申请，完成消费，返回到EventSystem，如此算是一个循环。这样的批量设计需要一个场地（内存）摆放所有生产者生产出来的数据，然后再由消费者集中消费，那么场地（内存）的占用就会比较多。

ATS这种设计是否有问题呢？

  - 如果批量生产，那么生产的数据就需要暂时堆放在一处（需要内存空间来存储）
    - 然后再批量消费，腾空堆放处（释放内存）
    - 那么就会看到ATS对内存的使用应该是有一个波峰和波谷
  - 有时候要接收一个大文件，就会持续生产
    - ATS没有限制生产的次数，导致消费过程被延迟

如果边生产边消费呢？

  - 在一个线程里没法并发处理两个队列，所以只能让生产者和消费者轮流进场生产和消费。
  - 由于在生产的过程中，同时也在消费，所以需要的场地就没有之前那么大。

但是这种方式是否有问题？

  - 如果有一对生产者和消费者特别投脾气
    - 一个生产，一个消费
    - 再生产，再消费
    - 直到生产者不再生产数据
  - 结果就是场地的占用时间变长了，如果场地是按照时间租用的，那么成本也很高。
  - 如果第一个生产者生产的内容是A，而第一个消费者想要的内容是B
    - 消费者就会离开，不再回来，除非有生产者叫它回来
    - 那么消费效率就降低了
  - 结果就是最后一个消费者离开了，仍然还有很多数据堆在场地里，没人要。

前面提到过，这个nethandler里处理读写操作占用的时间不能太久，否则EventSystem就没法及时处理新的Event

  - 如果是边生产边消费的模型，在nethandler里处理的时间就完全不受控制了，
  - 所以，这不符合EventSystem对于IO子系统的要求。

所以，在ATS的IO处理中，分为读、写两个阶段：

  - 首先读取数据到内存
  - 然后处理数据
  - 然后写数据
  - 最后回收临时的内存空间

这样的逻辑最简单和清晰。

由于ATS的IO核心没有限制一个VC上的读、写触发的次数，为了避免出现循环处理IO读、写的情况，在READ READY和WRITE READY的处理函数中应该：

  - READ READY 处理函数
    - 应只从读缓冲区（场地）复制有限的内容到写缓冲区（场地）
    - 实现对读取流量的控制，同时控制数据对内存的占用
  - WRITE READY 处理函数
    - 要确保以下两个条件不能同时满足
      - 写缓冲区（场地）为空
      - 读缓冲区（场地）满
    - 一旦同时满足，就会导致VC被憋死，因为：
      - 写缓冲区为空，VIO Write就会自动被禁止
      - 读缓冲一满，VIO Read酒会呗自动禁止
    - 通常读、写会共用同一个缓冲区（例如：Single Buffer Tunnel）
      - 上述两个条件永远无法同时满足
    - 但是，在一些特殊的设计时，会使用分开的两个缓冲区
      - 就必须特别留意，不要让上面的两个条件同时满足

## 参考资料

![NetHandler - NetVC - VIO - UpperSM](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-001.svg)

- [P_UnixNet.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
- [P_UnixPollDescriptor.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixPollDescriptor.h)
- [UnixNetVConnection.cc]
(http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNetVConnection.cc)
- [UnixNet.cc]
(http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)

