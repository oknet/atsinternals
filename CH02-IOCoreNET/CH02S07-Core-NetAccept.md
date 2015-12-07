# 核心组件：NetAccept

NetVConnection及其继承类，是用于实现FD与VIO的连接，那么VC是如何创建的？

- 可以作为客户端，通过connect创建一个与服务器的连接，这是另外的方法，后面我们再介绍
- 可以作为服务器，等客户端连接的时候，通过accept获得新连接，这就是NetAccept

NetAccept是NetProcessor的一个子系统，是NetProcessor服务端的引导器（Acceptor）。

在ATS的状态机设计中，每一个状态机（SM）的服务端实现，都包含了：

  - 一个引导器（Acceptor），用来接受客户端的连接，并且创建所需要类型的VConnection（SM）
  - 一个处理器（Processor），提供了大量的回调函数，用来处理每一个SM的状态（NetProcessor，HttpTransaction）

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
