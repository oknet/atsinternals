# 基础组件：PollCont

虽然 Polling Sub-System 的抽象没有那么清晰，但是其仍然包含基本的三要素：

|  EventSystem   |    epoll   |  Polling SubSystem  |
|:--------------:|:----------:|:-------------------:|
|      Event     |  socket fd |       EventIO       |
|     EThread    | epoll_wait |      PollCont       |
| EventProcessor |  epoll_ctl |       EventIO       |

PollCont 作为 Polling Sub-System 中的状态机，通过周期性调用 epoll_wait() 实现了对 socket fd 的事件获取：

- PollDescriptor
  - 可以理解为是一个包含了 socket fd 的队列
  - 通过 epoll_wait 可以从这个队列里获得成员
- EventIO(1)
  - 可以看作对 socket fd 上读／写请求的封装
- PollCont
  - 从 PollDescriptor 这个“队列”里获取成员（数据）
  - 回调关联到 epoll fd 的状态机（NetHandler）并传递获取的成员（数据）
  - 可能是为了便于代码的实现，NetHandler 被关联到了 PollCont
- EventIO(2)
  - 可以看作 PollProcessor
  - 负责管理 PollDescriptor “队列” 中的成员
  - 通过几个方法实现了“队列”的增、删、改操作

可以看到 EventIO 同时承担了 Processor 的功能，这是因为 EventIO 在作为 Processor 时总是操作当前线程的 PollDescriptor，这不同于 EventProcessor 是作用于整个线程池的。因此，这里将 EventIO 的 Processor 功能：

  - start() / stop()
  - modify() / refresh()
  - close()

合并到 EventIO 内，这有点像 Event 类自身也提供了 schedule_\*() 的方法一样。

由于 epoll / kqueue 是 O(1) 复杂度的，非常的高效，所以：

- 我们没有必要在一个线程里设置多个 epoll fd
- 每个线程只需要配置一个 PollDescriptor 对象即可

PollCont 通过调用 epoll_wait 得到了结果集：

- 通常应该遍历该结果集
- 对每一个结果（EventIO）回调它的状态机
- 状态机的功能就是把 EventIO 中包含的 NetVC 放入到 NetHandler 的队列里
- 等到 NetHandler 被 EventSystem 驱动时，再遍历它自己的队列

可以看到，逐个把 EventIO 放入 NetHandler 的队列的过程其实可以批量完成：

- ATS 的设计是让 NetHandler 直接访问结果集
- 这就免去了从 结果集 到 NetHandler 队列的数据复制过程

所以，每一个处理网络任务的线程里只需要一个 PollDescriptor 和与之对应的 PollCont 和 NetHandler：

- ATS 把 PollCont 的结果集放在了 PollDescriptor 里
- 由 NetHandler 直接访问 PollDescriptor 里的结果集
- 从而省去了通过回调传递结果集的过程

虽然，从代码上看不到 PollCont 对 NetHandler 的回调，但是 NetHandler 的确是 PollCont 的上层状态机，只是 Polling Sub-System 在系统中的定义相对模糊一点，在 UDP 的实现部分，Polling Sub-System 则定义的相对清晰一些。

为了便于分析，以下只列出了epoll ET模式相关的代码。

## 定义

```
source: iocore/net/P_UnixNet.h
struct PollCont : public Continuation {
  // 指向此PollCont的上层状态机NetHandler
  // 可以理解为 Event 中的 Continuation 成员
  //     由于一个 PollCont 中的所有 EventIO 的上层状态机都是同一个 NetHandler
  //     因此，把这个 NetHandler 的对象直接放在了 PollCont 中
  NetHandler *net_handler;
  // Poll描述符封装，主要描述符，用来保存 epoll fd 等
  PollDescriptor *pollDescriptor;
  // Poll描述符分装，但是这个主要用于UDP在EAGAIN错误时的子状态机 UDPReadContinuation
  // 在这个状态机中只使用了pfd[]和nfds
  PollDescriptor *nextPollDescriptor;

  // 由构造函数初始化，在pollEvent()方法内使用
  // 但是如果net_handler不为空，则会在每次调用pollEvent时：
  //    按一定规则重置为 0 或者 net_config_poll_timeout
  int poll_timeout;

  // 构造函数
  // 第一种构造函数用于UDPPollCont
  // 此时成员net_handler==NULL，那么pollEvent就不会重置poll_timeout
  // 在ATS所有代码中，pt参数没有显示传入过，因此总是使用默认值
  PollCont(ProxyMutex *m, int pt = net_config_poll_timeout);
  // 第二种构造函数用于PollCont
  // 此时成员net_handler被初始化为nh，
  //     但是在PollCont的使用中，并未引用过PollCont内的net_handler成员
  //     从pollEvent()的设计来看，原本应该是在NetHandler::mainNetEvent中调用pollEvent()
  //     但是抽象设计好像不太成功，在NetHandler::mainNetEvent重新实现了一遍pollEvent()的逻辑
  // 在ATS所有代码中，pt参数没有显示传入过，因此总是使用默认值
  PollCont(ProxyMutex *m, NetHandler *nh, int pt = net_config_poll_timeout);

  // 析构函数
  ~PollCont();

  // 事件处理函数
  int pollEvent(int event, Event *e);
};
```

## 使用

ATS的设计把 polling 操作当作事件核心系统的一部分，把对polling操作返回的fd集合的处理当做网络核心系统的一部分。

- PollCont的mutex指向ethread->mutex
- NetHandler的mutex则是new_ProxyMutex()

因此当 NetHandler 被 EventSystem 驱动的时候，可以安全的访问 PollCont 和 PollDescriptor。

在传统的I/O复用系统的实现中，polling 操作完成后，立即就会对 fd 集合进行遍历和处理。

- 在ATS中使用 PollCont 实现 polling 操作
- 然后使用 NetHandler 完成了对 fd结果集 的遍历和处理

这两个状态机在EThread::execute()的每一轮循环中都先后被回调，这就保证每一次的 polling 结果都能被NetHandler处理：

- 目前只有 UDP 部分使用了这个逻辑，把 polling 操作（PollCont）和对 fd结果集 的处理（UDPNetHandler）分成了两个部分。
- 在 TCP 部分使用的 NetHandler 则在内部直接调用了 epoll_wait 实现了 polling 操作
  - 代码逻辑仍然跟 PollCont 是一样的，感觉是把 PollCont 内嵌到了 NetHandler 里
  - 没有使用 PollCont::pollEvent 来完成 polling 操作
  - 但是仍然通过 PollCont 的成员 PollDescriptor 存取 epoll fd 及 fd结果集

下面对于PollCont的理解和认识来自于UDP的实现部分（这部分官方计划砍掉，但是我们先简单看一下，其实这个设计理念非常赞）：

- 在UDP初始化时
  - PollCont和UDPNetHandler先后被丢入EventSystem的隐性队列，优先级都是：-9
  - 因此每一轮循环都是UDPNetHandler::mainNetEvent先执行，然后才是PollCont::pollEvent
  - 这样的结果就是pollEvent返回的fd集合需要到下一次循环才会被处理
  - 但是每轮循环的时间非常的短，所以基本没有什么问题
- UDPNetHandler::mainNetEvent()只负责遍历PollDescriptor->ePoll_Triggered_Events[]数组
  - 判断vc类型，判断事件类型
  - 判断对应的vio是否被激活
  - 回调vio关联的状态机
- PollCont::pollEvent()只负责调用epoll_wait
  - 把从epoll_wait得到的fd结果集放入PollDescriptor->ePoll_Triggered_Events[]数组里
  - 更新PollDescriptor->result为epoll_wait的返回值

（可以在看UDP部分的时候，再回来对照着看一下）

## 参考资料
- [P_UnixNet.h]
(http://github.com/apache/trafficserver/tree/master/iocore/net/P_UnixNet.h)
