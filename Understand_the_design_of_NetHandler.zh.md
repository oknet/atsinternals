# Understand the design of NetHandler

根据 Sub-system 的设计规范，对 NetHander 的设计做一个解析。

首先对照三个基本元素：

- NetHandler（Handler）
- NetProcessor（Processor）
- NetVC（Task）

然后是三个处理流程：

- pre-Handler 有两个
    - 发起 TCP 连接
        - 状态函数 startEvent
        - 处理函数 connectUp
        - API 函数 NetProcessor::connect\_re
    - 接受 TCP 连接
        - 状态函数 acceptEvent
        - 处理函数 无
        - API 函数 NetProcessor::accept
- Handler 过程
	- 只有读、写任务
	- 不包含：接受一个新连接 或 发起一个新连接
	- NetHandler::mainNetEvent
- post-Handler
   - 无


## NetHandler

NetHandler 是处理 NetVC 读、写任务的一个 Sub-system，因此下面所列出的内容仅仅与读、写任务相关，可能会包含一点与连接建立相关的内容。

NetHandler 继承自 class Continuation，通过 Negative Event 周期性回调。其内部包含了：

- 保存 NetVC 的队列
    - 原子队列
        - Polling FDs
        - read\_enabled\_list
        - write\_enabled\_list
    - 本地队列
        - Polling result
        - read\_ready\_list
        - write\_ready\_list
    - Polling System 是 NetHandler 内部嵌套的一个 Sub-system，同样具有标准的三个基本元素。
- int NetHandler::startIO(NetVC \*p)
    - 将 NetVC 放入到 NetHandler 中，由 NetHandler 负责执行 NetVC 的读、写任务
- void NetHandler::stopIO(NetVC \*p)
    - 将 NetVC 从 NetHandler 中取出，解除 NetVC 与 NetHandler 之间的所有联系
- void NetHandler::process\_enabled\_list()
    - 分别遍历原子队列 read\_enabled\_list 和 write\_enabled\_list
    - 将其中需要进行读、写任务的 NetVC 放入对应的本地队列 read\_ready\_list 和 write\_ready\_list
- void NetHandler::process\_ready\_list()
    - 批量处理本地队列里的 NetVC 的读、写任务
    - 如果 NetVC 的状态为已经关闭，则通过 NetHandler::free\_netvc(netvc) 将其回收
- int NetHandler::mainNetEvent(int e, void \*data)
    - 主回调函数，在每一个 NetHandler 的运行周期，通过：
        - PollCont::pollEvent 获得 Polling result，转换为 NetVC 后，根据事件类型导入对应的本地队列
        - NetHandler::process\_enabled\_list() 将两个原子队列的 NetVC 分别导入对应的本地队列
        - NetHandler::process\_ready\_list() 批量处理 NetVC 的读、写任务
- void NetHandler::free_netvc(NetVC \*p)
    - 由 close_UnixNetVConnection() 实现。
    - 首先解除 NetVC 与 InactivityCop 的关联，
    - 然后通过 NetHandler::stopIO(NetVC *p) 解除 NetVC 与 NetHandler 的关联，
    - 最后通过 NetVC::free(t) 完成对 NetVC 的回收。
    - InactivityCop 是一个负责 NetVC 超时的子系统。

## NetProcessor

NetProcessor 提供了一部分 API，供 Net Sub-system 的用户使用：

- NetVC * NetProcessor::allocate_vc(t)
    - 用来创建一个 NetVC 对象并初始化
    - 返回：NetVC 对象

在新的 NetVC 创建后，由 accept 或 connect 状态机对 NetVC 进行初始化之后，就通过 NetHandler::startIO(netvc) 将 NetVC 交给 NetHandler 来执行读、写任务了。

NetProcessor 提供的，发起 TCP 连接的 API：

- Action * NetProcessor::connect_re(Continuation *cont, ...)
    - 创建一个 NetVC 对象，使用传入的 cont 初始化 NetVC::action 对象，使用传入的其它参数初始化 NetVC，如 NetVCOptions 等
    - 尝试对 NetHandler 上锁
    - 如果上锁成功了，就直接调用 NetVC::connectUp(t) 完成连接的任务，然后回调 cont
        - 返回：ACTION\_RESULT\_DONE
    - 如果上锁失败，则把 NetVC 放入有 NetHandler 运行的 EThread 里，尝试稍后回调 NetVC::startEvent() 再次对 NetHandler 上锁后调用 NetVC::connectUp(t)
        - 返回：NetVC 的 action 成员

NetProcessor 提供的，接受 TCP 连接的 API：

- Action * NetProcessor::accept(Continuation *cont, …)
    - 创建一个 NetAccept 对象，由 NetAccept 状态机执行 accept() 系统调用
    - 得到 socket fd 之后创建 NetVC
        - 复制 NetAccept 的 action 对象到 NetVC 的 action 对象
        - 设置 NetVC::acceptEvent 作为起始状态函数
    - 通过事件系统回调 NetVC::acceptEvent，再回调成功或者失败的事件给 NetVC 的 action 对象指向的状态机
    - 返回值：NetAccept 对象的 action 成员


## NetVC

NetVC 继承自 class VConnection，而 VConnection 继承自 class Continuation，是执行任务的主体，包含以下内容：

- Action action;
    - 一个 Action 对象，注意不是指针
    - 在 pre-Handle 任务完成时，将回调 Action 对象指向的状态机
    - 如该 Action 对象被取消，则不回调状态机，并调用 NetVC::free(t) 回收
- int closed;
    - 表示 NetVC 的读、写任务均已完成，可以由 NetHandler 将其回收
- int read.enabled 和 write.enabled
    - 表示是否要执行读、写任务
- read.vio.\_cont 和 write.vio.\_cont
    - 读、写任务完成时，将回调 \_cont
- read.vio.mutex 和 write.vio.mutex
    - 从对应的 \_cont->mutex 获取的 mutex 副本，用于保护 enabled 和 vio.\_cont 的数据读取
- EThread *thread;
    - 一个指向 EThread 对象的指针
    - 在 NetVC 对象分配时初始化, 表示此 NetVC 对象与该 EThread 绑定
    - 在 acceptEvent，startEvent 和 connectUp 中获取的 NetHandler 对象，就是运行在这个 EThread 里的
- NetHandler \*nh;
    - 一个指向 Handler 对象的指针
    - 当该指针不为空时，只有 NetHandler 可以释放该 NetVC 对象
    - 当该指针为空时，可以直接调用 NetVC::free(t) 释放该 NetVC 对象
- void NetVC::free(EThread \*t)
    - 用来回收 NetVC 对象占用的资源
- void NetVC::clear()
    - 用来释放和清理 NetVC 内的各成员变量
    - 主要用来支持 freelist
- VIO \* do\_io\_read or write(Continuation \*cont, int nbytes, ...)
    - 使用传入的其它参数重新设置必要的 NetVC 成员
    - 将当前 NetVC 放入 NetVC::nh 指向的 NetHandler 对象的原子队列
- void reenable(VIO \*vio)
    - 重新将当前 NetVC 对象放入 NetHandler 的队列
- void readDisable(NetHandler \*nh) 和 writeDisable(NetHandler \*nh)
    - 临时暂停读、写任务
- void set_enabled(VIO \*vio)
    - 恢复之前暂停的读、写任务
- void do\_io\_close(int alerrno)
    - NetVC 的读、写任务已经执行完成，NetVC::closed 被设置为 非零
    - 接下来将由 NetHandler 将 NetVC 对象回收

NetVC 比较特殊，它内部包含了两个任务：读 和 写，而且都是反复执行。


## InactivityCop

InactivityCop 也是一个 Handler，继承自 class Continuation，通过 Period Event 周期性定时回调。


Polling System 是嵌套在 NetHandler 内部的一个子系统。

```
  +-----> EThread::execute() loop
  |                            |
  |                            V
  |        +-------------------+---------------+
  |        | InactivityCop::check_inactivity() |
  |        |                   |               |
  |        |                   |               |
  |        +-------------------+---------------+
  |                            |
  |                            V
  |        +-------------------+---------------+
  |        | NetHandler::mainNetEvent()        |
  |        |                   |               |
  |        |                   |               |
  |        +-------------------+---------------+
  |                            |
  |                            V
  +-----<----------------------+

```


首先对照三个基本元素：

- InactivityCop（Handler）
- 不明确（Processor）
- NetVC（Task）

InactivityCop 的设计太过于简化，以至于在基本元素的设计上没有形成独立的 Processor，而且 Task 与 NetHandler 共享，同为 NetVC。

然后是三个处理流程：

- pre-Handler
    - 无
- Handler
    - 对 NetVC 进行超时检测
    - 检查客户端连接数是否超出限制并触发超时事件
    - InactivityCop::check_inactivity
- post-Handler
    - 无

### 超时检测

为了实现超时检测，其内部包含了：

- 保存 NetVC 的队列
    - 外部队列
        - NetHandler::open\_list
        - 这不是一个原子队列。
    - 内部队列
        - NetHandler::cop\_list
    - 两个队列都没有定义在 InactivityCop 内，而是放在了 NetHandler 里面，由于 InactivityCop 与 NetHandler 共享同一个互斥锁，因此这样做没问题。
- Handler::startIO()
    - 该方法的定义不存在，但是其代码实际上在 NetVC::acceptEvent 和 NetVC::connectUp 中可以看到
    - ```nh->open_list.enqueue(netvc);```
    - 其功能就是直接把 netvc 放入 open_list 队列
- Handler::stopIO()
    - 该方法的定义不存在，但是其代码实际上在 close_UnixNetVConnection() 中可以看到
    - ```nh->open_list.remove(netvc); nh->cop_list.remove(netvc)```
    - 其功能就是直接把 netvc 从 open\_list 和 cop\_list 这两个队列中移除。
- InactivityCop::check\_inactivity(int e, void *data)
    - 该方法包含了对 open\_list 队列的遍历，并将 NetVC 导入到 cop\_list 的流程
    - 同时也包含了 Handler::process\_task() 的功能
- Handler::free_task()
    - 该方法在超时检测时不需要
    - 超时检测只是通知使用者达到了之前设定的时间


由于不需要额外分配任何资源，只需要将 NetVC 放入 open_list 中即可实现超时管理，因此没有对 Processor 进行定义，使用 NetVC 上的控制函数完成超时设置。

为了将超时控制整合到 NetVC 里，对 NetVC 进行了以下扩展：


- ink_hrtime next\_inactivity\_timeout\_at
    - 表示 Inactivity Timeout 检测是否开启
    - ink_hrtime inactivity\_timeout\_in 用于保存 Inactivity Timeout 的设置
- ink_hrtime next\_activity\_timeout\_at
    - 表示 Activity Timeout 检测是否开启
    - ink_hrtime active\_timeout\_in 用于保存 Activity Timeout 的设置
- Action action\_;
    - 复用了 pre-Handler 时的 Action 用于在超时发生时，回调状态机
- netActivity()
    - 相当于 Task::reDispatch()，在每次发生读、写操作后自动重新设置 next\_inactivity\_timeout\_at
- set\_inactivity\_timeout() 和 set\_active\_timeout()
    - 相当于 Task::dispatch()，分别用来：
    - 设置 inactivity\_timeout\_in 和 next\_inactivity\_timeout\_at
    - 设置 active\_timeout\_in 和 next\_activity\_timeout\_at
- cancel\_inactivity\_timeout() 和 cancel\_active\_timeout()
    - 相当于 Task::finised()，分别用来：
    - 清除 inactivity\_timeout\_in 和 next\_inactivity\_timeout\_at
    - 清除 active\_timeout\_in 和 next\_activity\_timeout\_at

### 客户端并发连接限制

为了在客户端并发连接过多时对其进行限制，以保证有足够的资源可以发起到源服务器的连接，其内部设计了：

- 保存 NetVC 的队列
    - 外部队列1
        - NetHandler::active_queue
    - 当 NetVC 正处于以下状态时，被 HttpSM 放入此队列
        - 正在等待来自客户端发送的 HTTP 请求
        - 正在等待 ATS 送回 HTTP 响应
    - 内部队列1
        - 无，直接在 NetHandler::manage_active_queue() 中遍历 active_queue
    - 外部队列2
        - NetHandler::keep_alive_queue
    - 当 NetVC 正处于以下状态时，被 HttpSM 放入此队列
        - ATS 已经将 HTTP 响应完整的发送给了客户端
        - 客户端与 ATS 之间的连接支持 Http Keep-alive 并且连接未被任何一方关闭
    - 内部队列2
        - 无，直接在 NetHandler::manage_keep_alive_queue() 中遍历 keep_alive_queue
- NetHandler::add\_to\_active\_queue() 和 NetHandler::add\_to\_keep\_alive\_queue()
    - 相当于 Handler::startIO
    - 其功能就是直接把 netvc 放入 active\_queue 或 keep\_alive\_queue 队列
- NetHandler::remove\_from\_active\_queue() 和 NetHandler::remove\_from\_keep\_alive\_queue()
    - 相当于 Handler::stopIO()
    - 其功能就是直接把 netvc 从 active\_queue 或 keep\_alive\_queue 队列移除
- NetHandler::manage_active_queue 和 NetHandler::manage_keep_alive_queue
    - 遍历 active\_queue 或 keep\_alive\_queue 队列
    - 优先向 keep_alive_queue 中的 NetVC 回调超时事件
    - 然后再向 active\_queue 中的 NetVC 回调超时事件
- NetHandler::_close_vc()
    - 相当于 Handler::free\_task(Task *p)
    - 其功能主要是向 NetVC 回调超时事件，但是 NetVC 并不一定会关闭，这取决于状态机对于超时事件的反应

同样没有对 Processor 进行定义，为了状态机调用的便利，在 NetVC 上直接封装了：

- NetVC::add\_to\_active\_queue() 和 NetVC::add\_to\_keep\_alive\_queue()
    - 相当于 Task::dispatch()
    - 都是直接调用 NetHandler 上定义的方法
- NetVC::remove\_from\_active\_queue() 和 NetVC::remove\_from\_keep\_alive\_queue()
    - 相当于 Task::finished()
    - 都是直接调用 NetHandler 上定义的方法

这样状态机可以直接通过 NetVC 上的 API 函数完成操作，而不需要 Processor 上的 API 函数。

## Polling System

Polling System 是嵌套在 NetHandler 内部的一个子系统。

```
  +-----> EThread::execute() loop
  |                            |
  |                            V
  |        +-------------------+---------------+
  |        | InactivityCop::check_inactivity() |
  |        |                   |               |
  |        |                   |               |
  |        +-------------------+---------------+
  |                            |
  |                            V
  |        +-------------------+---------------+
  |        | NetHandler::mainNetEvent()        |
  |        |                   |               |
  |        |                   V               |
  |        |         +---------+-------------+ |
  |        |         | PollCont::pollEvent() | |
  |        |         +---------+-------------+ |
  |        |                   |               |
  |        +-------------------+---------------+
  |                            |
  |                            V
  +-----<----------------------+

```

首先对照三个基本元素：

- PollCont + sys poll（Handler）
- NetHandler（Processor）
- EventIO（Task）

然后是三个处理流程：

- pre-Handler
    - 无
- Handler 过程
    - PollCont::pollEvent
- post-Handler
    - 无

由于 Polling System 跟操作系统直接联系在一起，下面将以 Linux epoll 为例进行分析。

### PollCont 和 Sys Poll

PollCont 和 sys poll 共同组成了 Handler：

- 保存 EventIO 的队列
    - 原子队列
        - epoll fd
        - 由 epoll\_create() 创建，可通过 epoll\_ctl(EPOLL\_ADD, ...) 向原子队列中添加元素
    - 本地队列
        - pollDescriptor->ePoll\_Triggered\_Events 数组
        - 由 epoll\_wait 对 epoll fd 执行类似 ```popall()``` 的操作后导入
- Handler::dispatch()
    - PollCont 没有定义此方法
    - 可以使用 epoll\_ctl(EPOLL\_CTL\_ADD, ...) 替代
- Handler::release()
    - PollCont 没有定义此方法
    - 可以使用 epoll_ctl(EPOLL\_CTL\_DEL, ...) 替代
- PollCont::pollEvent(int e, void *data)
    - 被周期性运行的 NetHandler::mainNetEvent() 回调
    - 向 NetHandler 提供整个本地队列里所有的 EventIO
    - 自身不处理这些 EventIO 对象
- Handler::free_task()
    - 由于 epoll\_wait 返回的结果存放在 pollDescriptor->ePoll_Triggered_Events 数组里，
    - 下一次调用 epoll_wait 时，会覆盖上一次的内容，因此无需释放和清理

### NetHandler 作为 Processor

由于 Polling System 是嵌套在 NetHandler 内部的子系统，因此 NetHandler 是唯一合法可以访问这个子系统的入口，因此 NetHandler 将作为 Polling System 的 Processor，为此提供以下 API 函数：

- NetHandler::startIO(NetVC *p)
    - 相当于 Processor::dispatch 
    - 将 NetVC 的成员 EventIO 对象放入 Polling System
- NetHandler::stopIO(NetVC *p)
    - 相当于 Handler::release()，由于 PollCont 与 NetHandler 共享同一个 mutex，为了简化设计，将PollCont::stopIO 改为 NetHandler::stopIO
    - 将 NetVC 的成员 EventIO 对象从 Polling System 中移除
- Processor::allocate_task(t)
    - 无
    - 由于 EventIO 大多数情况下以 NetVC 的成员形式定义，不需要独立的创建
    - 极少情况下通过 new EventIO 方式创建
    - 因此在 polling system 的 Processor 里不提供 allocate 的 API 函数

### EventIO

EventIO 提供了适配多种 VConnection 类型的能力，为了简化分析，我们以 NetVC 为例。

- EventLoop event_loop;
    - 为 nullptr 时表示任务已经完成
    - 但是 EventIO 没有独立的回收方法，因此任务完成时并不会立即回收，而是作为 NetVC 的成员，随着 NetVC 一起被回收。
- get_PollCont(t)
    - 相当于 Task 对象内的 Handler *h;
    - 每个 EThread 里只有一个 NetHandler 和 一个 PollCont 状态机
- Continuation \*c
    - 指向 NetVC 对象的指针，用来将 EventIO 转换为 NetVC 后，由 NetHandler 统一处理
- int type
    - 只有 type == EVENTIO\_READWRITE\_VC 时，上面的 c 才指向 NetVC 对象
- int events
    - 用来向 polling system 描述需要执行的任务
    - 按位进行设置，可以描述为：读任务、写任务、读和写的复合任务等
- Ptr\<ProxyMutex\> cont_mutex;
    - 无
    - 由于总是以 NetVC 的成员的方式而存在，因此不需要
- 初始化
    - 通过构造函数完成
    - event_loop = nullptr; type = 0; data.c = nullptr;
- start(EventLoop l, int fd, Continuation \*c, int event)
    - 原本应该定义在 PollCont 上，实现 Handler::dispatch(Task *p) 的功能
    - 这里为了简化设计，将其定义在 EventIO 里
- modify(int event)
    - 当 event > 0 时，相当于 Task::resume()
    - 当 event < 0 时，相当于 Task::pause()
- refresh()
    - 相当于 Task::reDispatch()
- stop()
    - 相当于 Task::finished()
- close()
    - 原本应该定义在 PollCont 上，实现 Handler::free_task(Task *p) 的功能
    - 但是 EventIO::free() 没有定义，而且 Event 对象不由 PollCont 来释放，
    - 因此，定义了这个特殊的 close() 方法，当 EventIO 为 NetVC 服务时，此方法不会被使用。

