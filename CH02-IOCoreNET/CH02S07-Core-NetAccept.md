# 核心组件：NetAccept

NetVConnection及其继承类，是用于实现FD与VIO的连接，那么VC是如何创建的？

- 可以作为客户端，通过connect创建一个与服务器的连接，这是另外的方法，后面我们再介绍
- 可以作为服务器，等客户端连接的时候，通过accept获得新连接，这就是NetAccept

NetAccept是NetProcessor的一个子系统，是NetProcessor服务端的引导器（Acceptor）。

在ATS的状态机设计中，每一个状态机（SM）的服务端实现，都包含了：

  - 一个引导器（Acceptor），用来接受客户端的连接，并且创建所需要类型的VConnection（SM）
  - 一个处理器（Processor），提供了大量的回调函数，用来处理每一个SM的状态（NetProcessor，HttpTransaction）

# 定义

```
//
// Default accept function
//   Accepts as many connections as possible, returning the number accepted
//   or -1 to stop accepting.
//
// 下面这两个typedef定义就是为了一个net_accept方法
// 所有的netaccept操作基本都会调用这个底层的net_accept方法
// 但是也有直接调用Server::accept()方法的情况
typedef int(AcceptFunction)(NetAccept *na, void *e, bool blockable);
typedef AcceptFunction *AcceptFunctionPtr;
AcceptFunction net_accept;

// TODO fix race between cancel accept and call back （这个注释是什么鬼？难道有个大坑没解决？）
// 创建一个 NetAcceptAction，继承自Action和RefCountObj
// 用于向NetAccept的调用者返回一个Action，在将来可以随时将此NetAccept操作cancel掉
// 为何要支持引用计数？没看懂...
struct NetAcceptAction : public Action, public RefCountObj {
  // 指向NetAccept::server成员的指针
  Server *server;

  // 提供此Action唯一的方法cancel
  void
  cancel(Continuation *cont = NULL)
  {
    Action::cancel(cont);
    // cancel操作，直接把listen fd给关闭了。感觉好野蛮！！！
    // 这样在NetAccept的操作中就会出现错误，遇到错误，NetAccept就终止运行了。
    server->close();
  }

  // 重载赋值操作符
  // 由于继承自Action和RefCountObj两个类，
  // 因此这里选择继承Action的操作符重载，来更改Continuation和mutex两个成员。
  Continuation *operator=(Continuation *acont) { return Action::operator=(acont); }

  // 析构函数
  // 只是输出一个debug信息
  ~NetAcceptAction() { Debug("net_accept", "NetAcceptAction dying\n"); }
};


//
// NetAccept
// Handles accepting connections.
//
// 为表述方便，调整了成员变量定义的顺序
struct NetAccept : public Continuation {
  ink_hrtime period;
  // 仅用于net_accept方法内部，详细见net_accept方法的内部分析
  void *alloc_cache;
  // 用来标记listen fd在pd数组中的下标值，但是好像被废弃了
  //     pd->pfd[ifd].df == server.fd
  int ifd;
  
  // 在创建NetAccept实例之后，要填充下面的参数值，然后再启动NetAccept
  // 定义了Server实例，其内保存有listen fd，ip_family，local_ip，local_port信息
  Server server;
  // 定义accept_fn指针函数
  // 如果由 acceptFastEvent 调用accept_fn，则accept_fn必须指向net_accept
  // 但是由 acceptEvent 调用accept_fn，则accept_fn可以指向net_accept，也可以指向用户自定义的方法
  AcceptFunctionPtr accept_fn;
  // True = 在调用do_listen()创建了listen fd之后，回调状态机，
  // 传递NET_EVENT_ACCEPT_(SUCCEED|FAILED)事件，以及指向NetAccept实例的data指针
  bool callback_on_open;
  // True = 这是一个ATS内部的WebServer实现，只是内部使用，回头会详细介绍这个backdoor部分
  bool backdoor;
  // 返回给调用者的Action类型，同时其内也保存了指向上层状态机的Continuation
  Ptr<NetAcceptAction> action_;
  
  // 在NetAccept运行中，如果得到了一个新的连接，
  // 将会使用下面这些设置来设置新连接的socket属性。
  int recv_bufsize;
  int send_bufsize;
  uint32_t sockopt_flags;
  uint32_t packet_mark;
  uint32_t packet_tos;

  // 这个参数决定新的连接将被放入哪种类型的EThread里来处理
  // 例如：ET_NET, ET_SSL, ...
  EventType etype;
  // 以上是需要在启动NetAccept之前，需要设置的参数

  // 这个成员貌似被废弃了
  UnixNetVConnection *epoll_vc; // only storage for epoll events
  // 在accept_thread=0的时候，用来把Listen FD添加到对应EThread的epoll fd中。（参考：NetAccept::init_accept_per_thread())
  // 但是在NetHandler::mainNetEvent()方法中，并未看到对epd->type==EVENTIO_NETACCEPT类型的处理
  //     看上去是废弃了，但是并未废弃！！！
  // 因为添加到epoll fd之后，一旦有新连接进入，
  //     会让epoll_wait从阻塞等待中返回，这样就可以尽快回调NetAccept状态机
  // 不过感觉这里没有必要使用一个类成员变量来处理，使用函数内的临时变量就可以了。
  EventIO ep;

  // 返回 etype 成员
  virtual EventType getEtype() const;
  
  // 返回全局变量 netProcessor
  virtual NetProcessor *getNetProcessor() const;

  // 创建一个 DEDICATED ETHREAD, 回调NetAccept时运行acceptLoopEvent()
  void init_accept_loop(const char *);
  // 为指定的 REGULAR ETHREAD 添加一个定时回调的 NetAccept 状态机，回调时运行acceptEvent()
  virtual void init_accept(EThread *t = NULL, bool isTransparent = false);
  // 为每一个 ET_NET 组里的 REGULAR ETHREAD 添加一个定时回调的 NetAccept 状态机
  // 如果accept_fn == net_accept，那么回调时运行acceptFastEvent()，否则运行acceptEvent()
  // 所有的 NetAccept 状态机共享同一个listen fd
  virtual void init_accept_per_thread(bool isTransparent);
  // 克隆当前的 NetAccept 状态机，目前只有 init_accept_per_thread() 使用
  virtual NetAccept *clone() const;
  // 如果server.fd已经存在，那么就对其进行必要的设置，使其符合 listen fd 的要求
  // 否则，创建一个设置好的 listen fd
  // 0 == success
  int do_listen(bool non_blocking, bool transparent = false);

  int do_blocking_accept(EThread *t);
  virtual int acceptEvent(int event, void *e);
  virtual int acceptFastEvent(int event, void *e);
  int acceptLoopEvent(int event, Event *e);
  void cancel();

  NetAccept();
  virtual ~NetAccept() { action_ = NULL; };
};
```

NetAccept可以是一个独立的线程

  - 内部是一个死循环
  - 通过netProcessor.accept(AcceptorCont, fd, opt)创建的DEDICATED ETHREAD。
  - CONFIG proxy.config.accept_threads INT 1

也可以是一个周期性执行的Event封装的Continuation

  - 被EThread反复回调
  - 由于REGULAR ETHREAD也是一个死循环，因此实际上也是放在了一个大的死循环线程中执行。
  - CONFIG proxy.config.accept_threads INT 0

下面首先分析 DEDICATED ETHREAD 方式，然后再分析 Event 模式。

## netProcessor.accept(AcceptorCont, NO_FD, opt) 流程分析

创建一个NetAccept线程之前，需要首先创建一个引导器（Acceptor）的实例AcceptorCont，传递给netProcessor.accept()。

首先来看一下，如何创建一个 DEDICATED ETHREAD 来执行 NetAccept 的实例。

以下流程分析，简化了netProcessor.accept()的部分代码，以使得表达更清晰：

```
   NetAccept *na = createNetAccept();
   // Potentially upgrade to SSL. 
   // 由于NetProcessor是SSLNetProcessor的基类，所以这里要判断是否需要由NetVC升级为SSLVC。
   // 详细分析在SSLNetProcessor部分。
   upgradeEtype(upgraded_etype);

   // 创建了一个Action
   na->action_ = new NetAcceptAction();
   // 请注意这里 na->action_ 的值
   // 由于na->action_是Ptr<NetAcceptAction>类型
   //     NetAcceptAction继承自Action和RefCountObj类型，
   //     Action类型对赋值操作符进行了重载，
   //     NetAcceptAction类型对赋值操作符的重载定义为直接调用：Action的赋值操作符重载
   // 同时由于Ptr模版的定义的原因，
   //     这里使用了 *na->action，这样就只让NetAcceptAction的赋值操作符重载生效
   //     而Ptr模版定义的赋值操作符重载“没有”生效
   // 所以以下操作相当于：
   //     na->action_->continuation = AcceptorCont;
   //     na->action_->mutex = AcceptorCont->mutex;
   //     也就是只是修改了 na 的 continuation 和 mutex 两个成员
   *na->action_ = AcceptorCont;
   // 端口号8080是通过opt结构来获取
   // 创建独立Accept线程，参数[ACCEPT 0:8080]是线程名称
   na->init_accept_loop("[ACCEPT 0:8080]");
      // 在 init_accept_loop 内
      // 设置独立线程内运行的实例函数：NetAccept::acceptLoopEvent()
      // 这表示线程创建后，DEDICATED ETHREAD会直接回调这个方法
      SET_CONTINUATION_HANDLER(this, &NetAccept::acceptLoopEvent);
      // 调用eventProcessor来创建线程
      eventProcessor.spawn_thread(NACont, thr_name, stacksize)
         // 在spawn_thread内部
         Event *e = eventAllocator.alloc();
         // 使用前面的NetAccept Continuation实例来初始化Event
         e->init(NACont, 0, 0);
         // 创建一个新的EThread，类型为DEDICATED，用来执行这个Event
         all_dthreads[n_dthreads] = new EThread(DEDICATED, e);
         // 同时反向连接该EThread到Event
         e->ethread = all_dthreads[n_dthreads];
         // 设置mutex，Event，NACont的mutex都默认使用线程的mutex
         e->mutex = e->continuation->mutex = all_dthreads[n_dthreads]->mutex;
         n_dthreads++;
         // 启动线程
         e->ethread->start(thr_name, stacksize);
            // 下面是子线程执行的代码
            // 会调用eventProcessor的execute()方法，在execute内判断出此线程为DEDICATED类型
            // handleEvent函数指针指向NetAccept::acceptLoopEvent
            oneevent->continuation->handleEvent(EVENT_IMMEDIATE, oneevent);
         // 下面是主线程执行的代码
         // spawn_thread结束
         return
      // init_accept_loop结束
      return
   // accept结束
   return
```

至此，完成了独立accept线程的创建。

## NetAccept::acceptLoopEvent开始运行，其内部逻辑及流程如下：

```
int NetAccept::acceptLoopEvent(int event, Event *e)
   // 事实上这里传入的Event *e就是eventProcessor.spawn_thread函数中alloc的Event
   // 而 t 也是eventProcessor.spawn_thread函数中new出来的EThread
   EThread *t = this_ethread();
   // 死循环调用do_blocking_accept
   while (do_blocking_accept(t) >= 0) ;
```

## NetAccept::do_blocking_accept(t)开始运行，被嵌套在while死循环里

```
int NetAccept::do_blocking_accept(EThread *t)
   // 请注意，这里我们又运行在NetAccept类的环境下了，所以action_表示的是AcceptorCont
   // 声明一个Connection类，用于接收accept方法返回的新连接句柄
   // 这里所说的accept方法不是glibc函数，而是ATS SocketManager类里的一个方法
   Connection con;
   do {
      res = server.accept(&con)
      // Use 'NULL' to Bypass thread allocator
      // 分配一个VC，注意这里也跟SSLProcessor有关系
      //    如果getNetProcessor返回的是sslNetProcessor，那么这里拿到的VC就是SSLVC
      //    否则就是一个NetVC
      // 由于allocate_vc传入的参数是NULL，因此是从全局内存空间分配的内存
      vc = (UnixNetVConnection *)this->getNetProcessor()->allocate_vc(NULL);
      // 初始化VC
      vc->con = con;
      vc->options.packet_mark = packet_mark;
      vc->options.packet_tos = packet_tos;
      vc->apply_options();
      // 此值为true，表示该VC是通过DEDICATED ETHREAD创建的
      // 若为false，表示该VC是通过Event方式创建的
      vc->from_accept_thread = true;
      // 为此VC分配一个全局唯一的ID，这个id是一个自增长的int类型
      // net_next_connection_number()方法保证该值在多线程环境中的原子自增
      vc->id = net_next_connection_number();
      // 复制IP结构数据
      ats_ip_copy(&vc->server_addr, &vc->con.addr);
      // 设置透明模式
      vc->set_is_transparent(server.f_inbound_transparent);
      // 给新VC创建Mutex
      vc->mutex = new_ProxyMutex();
      // 由于vc->action_是Action类型（不是指针，是一个实例成员），
      //     Action类型对赋值操作符进行了重载，
      // 所以以下操作相当于：
      //     vc->action_->continuation = action_;
      //     vc->action_->mutex = action_->mutex;
      // 请注意，由于this->action_是Ptr<NetAcceptAction>类型，
      // 所以这里等号右侧是 *action_ 而不是 action_，以避免引用计数被减少。
      //     （看Ptr<>重载操作的定义，不写 * 好像也是可以的？因为左侧不是Ptr<>类型？）
      // 这里，action_->mutex 被初始化为了 AcceptorCont->mutex，通常它也是new_ProxyMutex()创建的。
      vc->action_ = *action_;
      // 设置该VC的回调函数
      SET_CONTINUATION_HANDLER(vc, (NetVConnHandler)&UnixNetVConnection::acceptEvent);
      // 压入eventProcessor的外部队列
      // 并且产生信号，打断eventProcessor内部的sleep以及epoll_wait的阻塞，以便于及时处理新加入的连接。
      eventProcessor.schedule_imm_signal(vc, getEtype());
   } while (loop)
```

在接受新连接之后，会创建一个新的VC，将此VC与UnixNetVConnection状态机（SM）绑定，之后将由Net处理机（Processor）来接手每一个状态的处理。

压入到外部队列后，就会进入eventProcessor的处理过程，将会直接调用UnixNetVConnection::acceptEvent函数。

那么如何在多个线程中选择一个来进行处理呢？（回顾一下eventProcessor的章节）

下面分析eventProcessor.schedule_imm_signal的代码：

```
TS_INLINE Event *
EventProcessor::schedule_imm_signal(Continuation *cont, EventType et, int callback_event, void *cookie)
{
  Event *e = eventAllocator.alloc();

  ink_assert(et < MAX_EVENT_TYPES);
#ifdef ENABLE_TIME_TRACE
  e->start_time = ink_get_hrtime();
#endif
  e->callback_event = callback_event;
  e->cookie = cookie;
  // 最终通过 schedule 完成压入外部队列的操作
  return schedule(e->init(cont, 0, 0), et, true);                                                                                             
}

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

// 通过轮询的方式，选择ethread
TS_INLINE EThread *
EventProcessor::assign_thread(EventType etype)
{
  int next;

  ink_assert(etype < MAX_EVENT_TYPES);
  if (n_threads_for_type[etype] > 1)
    next = next_thread_for_type[etype]++ % n_threads_for_type[etype];
  else
    next = 0;
  return (eventthread[etype][next]);
}
```

通过上面的分析，我们确认当新连接被Event封装之后，放入EThread线程的外部队列时，是通过对多个EThread采用轮询的负载算法进行选择的。

那么会在哪些EThread中轮训呢？由 getEtype() 的返回值决定：
  - 该值对于UnixNetVConnection类型为ET_NET
  - 该值对于SSLNetVConnection类型为ET_SSL

而AcceptorCont则是我们自定义的更高级别状态机的引导器，UnixNetVConnection::acceptEvent将会调用vc->action_内的回调函数

这样AcceptorCont则可以初始化更高级别的状态机。


## 引导器（Acceptor）

引导器（Acceptor）是一个简单的类，里面通常只有一两个方法，引导器主要负责：

- 创建状态机（SM）的实例
- 创建成功后，把vc上的回调函数设置到状态机（SM）对应的处理机（Processor）里的回调函数上

由于引导器负责创建状态机（SM），因此引导器（Acceptor）创建后，

- 通常不会被释放
- 除非遇到了严重的错误，此时ATS可能也会崩溃

而处理机则负责释放、销毁状态机。

多重引导器

- 当TCP链接建立后，需要判定接下来创建的上层状态机（SM）有多重类型时
- 需要读取一个用户的请求，根据请求的类型来判断需要创建哪一种上层状态机（SM）时
- 此时AcceptorCont则指向了第二重引导器，由其对数据简单分析后，再创建对应的上层状态机（SM）

## NO_FD 与 opt

未完待续...
