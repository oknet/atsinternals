# 核心部件：Action & Event

在介绍Event之前，首先看一下Action:

- 当一个 状态机 通过某个 Processor方法 发起一个异步操作时, Processor 将返回一个 Action 类的指针。
- 通过一个指向 Action 对象的指针, 状态机 可以取消正在进行中的异步操作。
- 在取消之后, 发起该操作的 状态机 将不会接收到来自该异步操作的回调。
- 对于Event System中的Processor，还有整个IO核心库公开的方法/函数来说Action或其派生类是一种常见的返回类型。
- Action的取消者必须是该操作将要回调的 状态机，同时在取消的过程中需要持有 状态机 的锁。

## 定义／成员

Event继承自Action，首先看一下Action类

```
class Action
{
public:
    Continuation * continuation;
    Ptr<ProxyMutex> mutex;
    // 防止编译器缓存该变量的值, 在 64bits 平台, 对该值的 读取 或 设置 是原子的
    volatile int cancelled;

    // 可由继承类重写, 实现继承类中对应的处理
    // 作为 Action 对外部提供的唯一接口
    virtual void cancel(Continuation * c = NULL) {
    if (!cancelled)
        cancelled = true;
    }

    // 此方法总是直接对 Action 基类设置取消操作, 跳过继承类的取消流程
    // 在 ATS 代码内, 此方法为 Event 对象专用
    void cancel_action(Continuation * c = NULL) {
    if (!cancelled)
        cancelled = true;
    }

    // 重载赋值（＝）操作
    // 用于初始化 Action
    //   acont 为操作完成时回调的状态机
    //   mutex 为上述状态机的锁, 采用 Ptr<> 自动指针管理
    Continuation *operator =(Continuation * acont)
    {
        continuation = acont;
        if (acont)
            mutex = acont->mutex;
        else
            mutex = 0;
        return acont;
    }

    // 构造函数
    // 初始化continuation为NULL，cancelled为false
    Action():continuation(NULL), cancelled(false) {
    }

    virtual ~ Action() {
    }
};
```

## Processor 方法实现者:

在实现一个 Processor 的方法时, 必须确保:

- 在操作被取消之后，不会有事件发送给状态机。

## 返回一个Action:

Processor 方法通常是异步执行的，因此必须返回Action，这样状态机才能在任务完成前随时取消该任务。
   - 此时, 状态机总是先获得 Action,
   - 然后才会收到该任务的回调,
   - 在收到回调之前, 随时可以通过 Action 取消该任务。

由于某些Processor的方法是可以同步执行的(可重入的)，因此可能会出现先回调状态机, 再向状态机返回Action的情况。
此时返回Action是毫无意义的, 为了处理这种情况，返回特殊的几个值来代替Action对象，以指示状态机该动作已经完成。
   - ACTION_RESULT_DONE 该Processor已经完成了任务，并内嵌(同步)回调了状态机
   - ACTION_RESULT_INLINE 当前未使用
   - ACTION_RESULT_IO_ERROR 当前未使用

也许会出现这样一种更复杂的问题：
   - 当结果为ACTION_RESULT_DONE
   - 同时，状态机在同步回调中释放了自身
 
因此，状态机的实现者必须：
   - 同步回调时, 不要释放自身(不容易判断出回调的类型是同步还是异步)
或者,
   - 立即检查 Processor 方法返回的 Action
   - 如果该值为ACTION_RESULT_DONE，那么就不能对状态机的任何状态变量进行读或写。

无论使用哪种方式，都要对返回值进行检查(是否为ACTION_RESULT_DONE)，同时进行相应的处理。


## 分配/释放策略:

Action的分配和释放遵循以下策略：

- Action由执行它的Processor进行分配。
  - 通常 Processor 方法会创建一个Task状态机来异步执行某个特定任务
  - 而 Action 对象则是该Task状态机的一个成员对象
- 在Action完成或者被取消后，Processor有责任和义务来释放它。
  - 当 Task状态机 需要回调 状态机 时, 
    - 通过 Action 获得 mutex 并对其上锁
    - 然后检查 Action 的成员 cancelled
    - 如已经 cancelled, 则销毁 Task状态机
    - 否则回调 Action.continuation
- 当返回的Action已经完成，或者状态机对一个Action执行了取消操作,
  - 状态机就不可以再访问该Action。


## 参考资料

[I_Action.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Action.h)


# 核心部件：Event

Event类继承自Action类, 它是EventProcessor返回的专用Action类型，它作为调度操作的结果由EventProcessor返回。

不同于Action的异步操作，Event是不可重入的。

  - EventProcessor 总是返回 Event 对象给状态机,
  - 然后, 状态机才会收到回调。
  - 不会像 Action类 的返回者, 可能存在同步回调 状态机 的情形。

除了能够取消事件（因为它是一个动作），你也可以在收到它的回调之后重新对它进行调度。

## 定义／成员

```
class Event : public Action 
{ 
public: 
    // 设置事件(Event)类型的方法
    void schedule_imm(int callback_event = EVENT_IMMEDIATE); 
    void schedule_at(ink_hrtime atimeout_at, int callback_event = EVENT_INTERVAL); 
    void schedule_in(ink_hrtime atimeout_in, int callback_event = EVENT_INTERVAL); 
    void schedule_every(ink_hrtime aperiod, int callback_event = EVENT_INTERVAL);

//  inherited from Action
    // Continuation * continuation;
    // Ptr<ProxyMutex> mutex;
    // volatile int cancelled;
    // virtual void cancel(Continuation * c = NULL);  

    // 处理此Event的ethread指针，在ethread处理此Event之前填充（就是在schedule时）。
    // 当一个 Event 由一个 EThread 管理后, 就无法在转交给其它 EThread 管理。
    EThread *ethread;

    // 状态及标志位
    unsigned int in_the_prot_queue:1; 
    unsigned int in_the_priority_queue:1; 
    unsigned int immediate:1; 
    unsigned int globally_allocated:1; 
    unsigned int in_heap:4; 

    // 向Cont->handler传递的事件(Event)类型
    int callback_event;

    // 组合构成四种事件(Event)类型
    ink_hrtime timeout_at; 
    ink_hrtime period;

    // 在回调Cont->handler时作为数据(Data)传递
    void *cookie;

    // 构造函数
    Event();

    // 初始化一个Event
    Event *init(Continuation * c, ink_hrtime atimeout_at = 0, ink_hrtime aperiod = 0);
    // 释放Event
    void free();

private:
    // use the fast allocators
    void *operator new(size_t size);

    // prevent unauthorized copies (Not implemented)
    Event(const Event &); 
    Event & operator =(const Event &);
public: 
    LINK(Event, link);

    virtual ~Event() {}
};

TS_INLINE Event *
Event::init(Continuation *c, ink_hrtime atimeout_at, ink_hrtime aperiod)
{
  continuation = c;
  timeout_at = atimeout_at;
  period = aperiod;
  immediate = !period && !atimeout_at;
  cancelled = false;
  return this;
}

TS_INLINE void
Event::free()
{
  mutex = NULL;
  eventAllocator.free(this);
}

TS_INLINE
Event::Event()
  : ethread(0), in_the_prot_queue(false), in_the_priority_queue(false), 
    immediate(false), globally_allocated(true), in_heap(false),
    timeout_at(0), period(0)
{
}

// Event 的内存分配不对空间进行bzero()操作, 因此在 Event::init() 方法中会初始化所有必要的值
#define EVENT_ALLOC(_a, _t) THREAD_ALLOC(_a, _t)
#define EVENT_FREE(_p, _a, _t) \
  _p->mutex = NULL;            \
  if (_p->globally_allocated)  \
    ::_a.free(_p);             \
  else                         \
  THREAD_FREE(_p, _a, _t)
```

## Event 方法

Event::init()

- 初始化一个Event
- 通常用来准备一个Event，然后选择一个新的EThread线程来处理这个事件时使用
- 接下来通常会调用EThread::schedule()方法

Event::schedule_*()

- 如果事件(Event)已经存在于EThread的```内部队列```，则先从队列中删除
- 然后直接向EThread的```本地队列```添加事件(Event)
- 因此此方法只能向当前EThread事件池添加事件(Event)，不能垮线程添加

对于Event::schedule_*()的使用，在源代码中有相关注释，翻译如下：

- 当通过任何Event类的调度函数重新调度一个事件时，状态机（SM）不可以在除了回调它的线程以外的线程中调用这些调度函数（好绕，其实就是状态机必须在回调它的线程里调用重新调度的函数），而且必须在调用之前获得该延续的锁。

## 事件(Event)的类型

ATS中的事件(Event)，被设计为以下四种类型：

- 立即执行
   - timeout_at=0，period=0
   - 通过schedule_imm设置callback_event=EVENT_IMMEDIATE
- 绝对定时执行
   - timeout_at>0，period=0
   - 通过schedule_at设置callback_event=EVENT_INTERVAL，可以理解为在xx时候执行
- 相对定时执行
   - timeout_at>0，period=0
   - 通过schedule_in设置callback_event=EVENT_INTERVAL，可以理解为在xx秒内执行
- 定期/周期执行
   - timeout_at=period>0
   - 通过schedule_every设置callback_event=EVENT_INTERVAL

另外针对隐性队列，还有一种特殊类型：

- 随时执行
   - timeout_at=period<0
   - 通过schedule_every设置callback_event=EVENT_INTERVAL
   - 调用Cont->handler时会固定传送EVENT_POLL事件
   - 对于TCP连接，此种类型的事件(Event)由NetHandler::startNetEvent添加
      - Cont->handler对于TCP事件为NetHandler::mainNetEvent()

### Time values:

任务调度函数使用了一个类型为ink_hrtime用来指定超时或周期的时间参数。
这是由libts支持的纳秒值，你应该使用在ink_hrtime.h中定义的时间函数和宏。

超时参数对于schedule_at和schedule_in之间的差别在于：
- 在前者，它是绝对时间，未来的某个预定的时刻
- 在后者，它是相对于当前时间（通过ink_get_hrtime得到）的一个量


### 取消Event的规则

与下面这些对于Action的规则是一样的

- Event的取消者必须是该任务将要回调的状态机，同时在取消的过程中需要持有状态机的锁
- 任何在该状态机持有的对于该Event对象（例如：指针）的引用，在取消操作之后都不得继续使用


### Event Codes:

- 在事件完成后，状态机SM使用延续处理函数（Cont->handleEvent）传递进来的Event Codes来区分Event类型和处理相应的数据参数。
- 定义Event Code时，状态机的实现者应该小心处理，因为它们对其它状态机产生影响。
- 由于这个原因，Event Code通常统一进行分配。

通常在调用Cont->handleEvent时传递的Event Code有如下几种：

```
#define EVENT_NONE CONTINUATION_EVENT_NONE // 0
#define EVENT_IMMEDIATE 1
#define EVENT_INTERVAL 2
#define EVENT_ERROR 3
#define EVENT_CALL 4 // used internally in state machines
#define EVENT_POLL 5 // negative event; activated on poll or epoll
```

通常Cont->handleEvent也会返回一个Event Callback Code

```
#define EVENT_DONE CONTINUATION_DONE // 0
#define EVENT_CONT CONTINUATION_CONT // 1
#define EVENT_RETURN 5
#define EVENT_RESTART 6
#define EVENT_RESTART_DELAYED 7
```

PS：但是在EThread::execute()中没有对Cont->handleEvent的返回值进行判断。

EVENT_DONE 通常表示该 Event 已经成功完成了回调操作, 该Event接下来应该被释放。(参照: ACTION_RESULT_DONE)
EVENT_CONT 通常表示该 Event 没有完成回调操作, 还需要保留以进行下一次回调的尝试。

## 使用

### 创建一个Event实例，有两种方式

- 全局分配
   - Event *e = ::eventAllocator.alloc();
   - 默认情况都是通过全局方式分配的，因为分配内存时还不确认要交给哪一个线程来处理。
   - 构造函数初始化globally_allocated(true)
   - 这样就需要全局锁
- 本地分配
   - Event *e = EVENT_ALLOC(eventAllocator, this);
   - 如果预先知道这个Event一定会交给当前线程来处理，那么就可以采用本地分配的方法
   - 调用EThread::schedule\_*\_local()方法时，会修改globally\_allocated＝false
   - 不会影响全局锁，效率更高

### 放入EventSystem

- 根据轮询规则选择下一个线程，然后将Event放入选择的线程
   - eventProcessor.schedule(e->init(cont, timeout, period));
   - EThread::schedule(e->init(cont, timeout, period));
- 放入当前线程
   - e->schedule_*();
   - this_ethread()->schedule_*_local(e);
      - 只能在e->ethread==this_ethread的时候使用

### 释放Event

- 全局分配
   - eventAllocator.free(e);
   - ::eventAllocator.free(e);
   - 在ATS的代码里能看到上面两种写法，我理解两种是一个意思，因为只有一个全局的eventAllocator
- 自动判断
   - EVENT_FREE(e, eventAllocator, t);
   - 根据e->globally_allocated来判断

### 重新调度Event

状态机在收到来自 EThread 的回调后, void \*data 指向触发此次回调的 Event 对象。
简单的进行类型转换后, 可以调用 e->schedule_\*() 将此 Event 重新放入当前线程。
在重新调度后, Event 的类型将会被 schedule_\*() 方法重新设置。


## 参考资料

- [I_Event.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Event.h)
- [P_UnixEvent.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_UnixEvent.h)
- [UnixEvent.cc](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/UnixEvent.cc)

