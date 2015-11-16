# 基础组件：PollCont

PollCont 是将 PollDescriptor 封装成以Continuation为基类的状态机，提供pollEvent()方法来获取事件列表。

为了便于分析，以下只列出了epoll ET模式相关的代码。

## 定义

```
source: iocore/net/P_UnixNet.h
struct PollCont : public Continuation {
  // 反向指回到持有此PollCont的NetHandler
  NetHandler *net_handler;
  // Poll描述符封装，主要描述符
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

ATS的设计把polling操作当作事件核心系统的一部分，把对polling操作返回的fd集合的处理当做网络核心系统的一部分。

- PollCont的mutex指向ethread->mutex
- NetHandler的mutex则是new_ProxyMutex()

因此，在传统的方式中polling完成后立即就对fd集合做处理的整合方式，在ATS中被拆分为了PollCont和NetHandler两个状态机，这两个状态机在EThread::execute()的每一轮循环中都先后被回调，这就保证每一次的polling结果都能被NetHandler处理。

- 但是最终只有UDP部分使用了这个逻辑，把polling操作（PollCont）和对fd结果集的处理（UDPNetHandler）分成两个部分。
- TCP部分使用的NetHandler则内部直接调用了epoll_wait实现了polling操作，没有使用PollCont::pollEvent来调用epoll_wait，但是仍然通过PollCont的成员PollDescriptor存取epoll fd

下面对于PollCont的理解和认识来自于UDP的实现部分（这部分官方计划砍掉，但是我们先看一下，其实这个设计理念非常赞）：

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
