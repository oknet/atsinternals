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
  
  //////////////////////
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
  //////////////////////

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
  // 如果accept_fn == net_accept，那么回调时：
  //     运行acceptFastEvent()，但是该方法不会调用accept_fn
  //     否则：运行acceptEvent()，该方法通过accept_fn接受新连接
  // 所有的 NetAccept 状态机共享同一个listen fd
  virtual void init_accept_per_thread(bool isTransparent);
  // 克隆当前的 NetAccept 状态机，目前只有 init_accept_per_thread() 使用
  // 通过 new NetAccept 创建新对象，然后通过成员拷贝初始化新对象的成员
  virtual NetAccept *clone() const;
  
  // 如果server.fd已经存在，那么就对其进行必要的设置，使其符合 listen fd 的要求
  // 否则，创建一个设置好的 listen fd
  // 此方法根据 callback_on_open 的设置，可能会对上层状态机进行回调
  // 注意：如果在上层状态机中设置了NetAccept的mutex，将会是无效的，
  //       因为do_listen在回调完成后，会将mutex设置为NULL
  //       这里比较特殊，因为此时该NetAccept状态机还未交付到任何REGULAR ETHREAD中
  // 返回 0 表示成功
  int do_listen(bool non_blocking, bool transparent = false);

  // 在 DEDICATED ETHREAD 中的主回调函数
  // 持续调用do_blocking_accept(t)，当其返回值小于0时，释放并析构自身，DEDICATED ETHREAD将会退出
  int acceptLoopEvent(int event, Event *e);
  // do_blocking_accept是acceptLoopEvent的一部分
  // 通过阻塞方式调用 accept 获得新连接，
  //     从全局内存池为新连接分配 vc 对象
  //     同时设置 vc->from_accept_thread = true，这样在释放 vc 占用的内存时，返回全局内存池
  // 返回 0：达到当前总连接限定时
  // 返回-1：当NetAccept被cancel时，分配 vc 对象内存失败时
  // 返回 1：当 accept_till_done==false，表示每次只接受一个新连接
  int do_blocking_accept(EThread *t);
  
  // 采用非阻塞方式调用 accept_fn 获得新连接：
  //     accept_fn 指向的默认方法 net_accept 是从全局内存空间分配 vc 内存的。
  // 首先尝试拿action_->mutex的锁，其实就是上层状态机的锁。
  //     如果情况特殊，action_->mutex 为NULL，那么就使用NetAccept自身的mutex
  // 拿到锁之后：
  //     如果该NetAccept没有取消，调用accept_fn()方法：
  //         大于 0：成功获取新连接，返回EventSystem等待下一次调用
  //         等于 0：目前没有连接，返回EventSystem等待下一次调用
  //         小于 0：
  //     若已被取消：取消回调此方法的Event，同时释放NetAccept自身
  // 如果没有拿到锁，返回EventSystem等待下一次调用
  // 通过该方法创建的VC状态机，第一个被回调的函数是acceptEvent，
  //     传入的是 EVENT_NONE（同步）或 EVENT_IMMEDIATE（异步）事件
  //     再由 acceptEvent 回调上层状态机，然后会设置状态机下一次回调的函数为mainEvent
  virtual int acceptEvent(int event, void *e);
  // 与acceptEvent不同：
  //     本方法不处理 backdoor 类型的连接，
  //     不调用accept_fn方法，而是使用socketManager.accept来获得新连接，
  //     是从线程内存空间分配 vc 内存的
  //     被取消时，会同时关闭 listen fd
  //     总是同步回调状态机，传递 NET_EVENT_ACCEPT 事件和 VC
  //     第一个被回调的函数是mainEvent，也就是该方法包含了acceptEvent内的工作
  // 对新连接设置 socket options 的时候，也是直接调用socketManager的相关方法进行设置：
  //     send buffer, recv buffer
  //     TCP_NODELAY, SO_KEEPALIVE, non blocking
  // 每次执行，会重复调用accept，创建多个新的 VC，
  //     直到返回EAGAIN，或出现错误，才会返回到EventSystem
  virtual int acceptFastEvent(int event, void *e);
  
  // 取消 NetAccept
  // 调用 action_->cancel(), 然后关闭listen fd
  void cancel();

  // 构造函数
  // 所有的成员都被初始化为0，false，NULL，ifd 为 －1
  NetAccept();
  
  // 析构函数
  // action_ 是自动指针，实际是释放该资源
  virtual ~NetAccept() { action_ = NULL; };
};

//
// General case network connection accept code
//
// 这是一个通用的 accept_fn 函数
// 当listen fd是non blocking的时候，blockable要传入false；否则传入true
int
net_accept(NetAccept *na, void *ep, bool blockable)
{
  Event *e = (Event *)ep;
  int res = 0;
  int count = 0;
  int loop = accept_till_done;
  UnixNetVConnection *vc = NULL;

  // 非阻塞的IO要保证net_accept也是非阻塞的
  // 尝试对 na->action_->mutex (通常就是上层状态机的mutex) 加锁
  // 失败就要返回，避免阻塞
  if (!blockable)
    if (!MUTEX_TAKE_TRY_LOCK_FOR(na->action_->mutex, e->ethread, na->action_->continuation))
      return 0;
  // do-while for accepting all the connections
  // added by YTS Team, yamsat
  // 下面的代码主要是尝试在非阻塞IO的情况下，尽可能多的accept新连接
  //     通常accept返回 EAGAIN 就表示已经没有新连接了
  // 解释一下 alloc_cache 的功能：
  //     我觉得这个 alloc_cache 设计的比较渣，把我的理解说明如下：
  //     由于 na->server.accept(&vc->con) 要使用vc里的成员con，
  //     所以就要先申请/分配一个vc，但是一旦server.accept()失败了，就要释放该vc
  //     作者觉得这个方法是频繁调用的，因此就把vc保存到 alloc_cache 里，
  //     每次要申请/分配一个vc的时候，先用 alloc_cache 这个
  // 但是，在NetAccept析构的时候，没有释放这个 alloc_cache，因此存在内存泄漏的bug
  // 为什么没有在析构函数里面释放这个 alloc_cache 呢？
  //     因为要判断 vc 的类型，是 UnixNetVConnection，还是 SSLNetVConnection，还是...
  //     才能调用对应类型的Allocator.free()方法，才能释放掉内存
  //     但是，根本没有办法在NetAccept里进行判断，然并软...
  do {
    // 获得一个 VC
    vc = (UnixNetVConnection *)na->alloc_cache;
    if (!vc) {
      // 从线程分配池里分配一个 VC，需要注意的是：
      //     如果 na 是NetAccept的继承类，而getNetProcessor()在继承类内被改写，
      //     那么返回的就不一定是UnixNetVConnection类型，这里只是进行了一个类型转换。
      vc = (UnixNetVConnection *)na->getNetProcessor()->allocate_vc(e->ethread);
      // 这里还有一个潜在的bug，就是没有判断 VC 是否分配成功了，对于分配失败的情况没有处理
      NET_SUM_GLOBAL_DYN_STAT(net_connections_currently_open_stat, 1);
      vc->id = net_next_connection_number();
      na->alloc_cache = vc;
    }
    // 调用 na->server.accept() 方法获取新连接
    if ((res = na->server.accept(&vc->con)) < 0) {
      // 失败时进行判断
      // 如果是 EAGAIN 等表示没有新连接了，此时就跳转到Ldone，返回调用者本次调用创建的新连接的数量（count）
      if (res == -EAGAIN || res == -ECONNABORTED || res == -EPIPE)
        goto Ldone;
      // 其它出错情况，回调上层状态机，告知出现了错误
      // 注：如果被cancel的话，server.fd 会被关闭，并设置为NO_FD，会导致 accept 返回值小于 0
      if (na->server.fd != NO_FD && !na->action_->cancelled) {
        if (!blockable)
          na->action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        else {
          SCOPED_MUTEX_LOCK(lock, na->action_->mutex, e->ethread);
          na->action_->continuation->handleEvent(EVENT_ERROR, (void *)(intptr_t)res);
        }
      }
      // count 设置为错误码，跳转到Ldone，返回调用者这个错误码
      count = res;
      goto Ldone;
    }
    
    // 创建新连接的数量加 1
    count++;
    // 由于使用掉了 alloc_cache 保存的vc，因此将其置为 NULL
    na->alloc_cache = NULL;

    // 初始化 vc 的一些参数
    vc->submit_time = Thread::get_hrtime();
    ats_ip_copy(&vc->server_addr, &vc->con.addr);
    vc->mutex = new_ProxyMutex();
    vc->action_ = *na->action_;
    vc->set_is_transparent(na->server.f_inbound_transparent);
    vc->closed = 0;
    
    // 设置VC状态机的回调函数，注意这里，无论该VC是UnixNetVConnection还是SSLNetVConnection，
    // 该回调函数总是：UnixNetVConnection::acceptEvent
    SET_CONTINUATION_HANDLER(vc, (NetVConnHandler)&UnixNetVConnection::acceptEvent);

    // 如果当前EThread支持此类型VC的处理，就同步回调VC状态机（acceptEvent）
    // 例如：当ssl accept thread＝0的时候，ET_NET ＝ ET_SSL，
    //       那么SSLUnixNetVConnection也会放在 ET_NET 的线程里运行
    if (e->ethread->is_event_type(na->etype))
      vc->handleEvent(EVENT_NONE, e);
    // 如果不支持，就异步回调状态机，把VC状态机封装到一个Event里，放入可以处理它的 EThread 中
    else
      eventProcessor.schedule_imm(vc, na->etype);
  } while (loop);

Ldone:
  // 解锁，其实这里不用写，因为是离开有效域的时候自动解锁
  if (!blockable)
    MUTEX_UNTAKE_LOCK(na->action_->mutex, e->ethread);
  // 返回 count 值
  return count;
}

```

NetAccept可以是一个独立的线程（DEDICATED ETHREAD模式）

  - 内部是一个死循环
  - 通过netProcessor.accept(AcceptorCont, fd, opt)创建的DEDICATED ETHREAD。
  - CONFIG proxy.config.accept_threads INT 1

也可以是一个被周期性执行的Event封装的状态机，虽然只有一个状态（REGULAR ETHREAD模式）

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

那么会在哪些EThread中轮询呢？由 getEtype() 的返回值决定：

  - 该值对于UnixNetVConnection类型为ET_NET
  - 该值对于SSLNetVConnection类型为ET_SSL

而AcceptorCont则是我们自定义的更高级别状态机的引导器，UnixNetVConnection::acceptEvent将会调用vc->action_内的回调函数

这样AcceptorCont则可以初始化更高级别的状态机。

## DEDICATED ETHREAD 模式总结

在netProcessor.accept()中

  - 通过createNetAccept()创建了NetAccept的实例
  - 通过init_accept_loop()创建DEDICATED ETHREAD

DEDICATED ETHREAD

  - init_accept_loop 创建DEDICATED ETHREAD
    - acceptLoopEvent
      - do_blocking_accept
        - server.accept

就渊源不断的接受并创建新的连接和 VC，最后调用 schedule_imm_signal(vc, getEtype())，将 VC 放入EventSystem对应类型的REGULAR ETHREAD线程组

REGULAR ETHREAD

  - External Local Queue
    - UnixNetVConnection::acceptEvent
      - action_->cont->handleEvent(NET_EVENT_ACCEPT, vc)
  - Event Queue
    - InactivityCop::check_inactivity
      - vc->handleEvent(EVENT_IMMEDIATE, e) ==> UnixNetVConnection::mainEvent(EVENT_IMMEDIATE, e)

上面看到 acceptEvent 会向上层状态机回调，传送 NET_EVENT_ACCEPT 事件，之后 VC 状态机的回调函数就切换到 mainEvent，与 Inactivity 状态机配合实现超时控制。

## REGULAR ETHREAD 模式介绍

通过 NetAccept::init_accept_per_thread 方法为指定的线程组，如ET_NET，创建NetAccept状态机

  - init_accept_per_thread
    - do_listen
    - clone
    - schedule_every(na, period, etype)

这里设定回调函数通常为acceptFastEvent，回调时传入的Event为etype，如果 accept_fn 使用了用户自定义的函数，那么回调函数为acceptEvent。

由于这里使用了thread->schedule_every，而且period小于0，因此事件直接放入了Negative Queue。

REGULAR ETHREAD

  - Negative Queue
    - acceptFastEvent(etype, na)
      - socketManager.accept
      - nh->open_list.enqueue(vc)
      - nh->read_ready_list.enqueue(vc)
      - action_->cont->handlerEvent(NET_EVENT_ACCEPT, vc)

调用 acceptFastEvent 就相当于包含了acceptEvent的处理部分，但是这里可能存在bug，因为操作 nh（NetHandler）队列时，并未得到NetHandler的mutex锁。

  - Negative Queue
    - acceptEvent(etype, na)
      - accept_fn ==> net_accept 默认情况
        - na->server.accept
        - 同步 vc->handleEvent(EVENT_NONE, e) ==> UnixNetVConnection::acceptEvent(EVENT_NONE, e)
          - action_->cont->handleEvent(NET_EVENT_ACCEPT, vc)
        - 异步 eventProcessor.schedule_imm(vc, na->etype)
  - External Local Queue
    - UnixNetVConnection::acceptEvent(EVENT_IMMEDIATE, e)
      - action_->cont->handleEvent(NET_EVENT_ACCEPT, vc)

如果使用的是 acceptEvent 回调函数，仍然走 acceptEvent，由于在 acceptEvent 内对 NetHandler 进行了加锁的判断和上锁失败的重新调度，因此就不存在 acceptFastEvent 的bug。

在对上层状态机进行 NET_EVENT_ACCEPT 回调时，支持 同步 和 异步 两种方式。

## 创建单独的 NetAccept 状态机

有时候，我们不需要为线程组里的每一个线程都创建一个 NetAccept 状态机，那么应该怎么做呢？

如果你留意的话，上面的介绍中，还有一个NetAccept的方法没有介绍过：init_accept()

当我们需要在指定的REGULAR ETHREAD内创建一个NetAccept状态机时，只要首先new一个指定类型的NetAccept实例。

例如：想接受新连接后进行SSL的处理，那就：

  - sslna = new SSLNetAccept
  - sslna->init_accept(thread, isTransparent)

需要注意的是，这里的 thread 一定要与 na->etype 是对应类型的，或者你可以偷懒，使用 NULL 代替 thread，此时会根据 etype 值找到一个适合的thread（assign_thread）。

此时会生成一个周期性运行的事件，将NetAccept状态机放入线程中运行，回调函数被设置为NetAccept::acceptEvent。

如果只想接受一个连接，那么在收到 NET_EVENT_ACCEPT 事件后，就可以调用 sslna->cancel() 来取消状态机了。

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

