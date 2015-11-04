# 核心部件：Action & Event

在介绍Event之前，首先看一下Action，由于在写这段内容的时候，我对Action的理解也不是十分透彻。

因此，下面的内容来自对Action类注释的翻译，翻译的不好，见谅；等我对Action理解透彻了再回来用我自己的语言重构这一个部分。

Action是将要被某个Processor执行的操作，Action类是对该操作的抽象表示。

在异步处理环境中，用于表示一个可撤销的Continuation，是Continuation类与Event类之间的桥梁。

对一个Action对象的引用可以让你在该Action完成前取消正在进行的异步操作。

- 这意味着，对于该操作指定的延续将不会被回调。

对于由Event System中的Processor，还有整个IO核心库公开的方法/函数来说Action或其派生类是典型的返回类型。

Action的取消者必须是该任务将要回调的状态机，同时在取消的过程中需要持有状态机的锁。

## 定义／成员

Event继承自Action，首先看一下Action类

```
class Action
{
public:
    Continuation * continuation;
    Ptr<ProxyMutex> mutex;
    volatile int cancelled;

    virtual void cancel(Continuation * c = NULL) {
    if (!cancelled)
        cancelled = true;
    }

    void cancel_action(Continuation * c = NULL) {
    if (!cancelled)
        cancelled = true;
    }

    // 重载赋值（＝）操作
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

## Processor 实现者:

你必须确保，在操作被取消之后，不会有事件发送给状态机。

## 返回一个Action:

Processor的函数是异步的，必须返回Action，这样才能在任务完成前呼叫状态机来取消任务。
由于某些Processor的函数是可重入的，他们可以在创建Action的调用返回之前回调状态机。
为了处理这种情况，特殊值代替Action被返回，以指示状态机该动作已经完成。
   - ACTION_RESULT_DONE 该Processor已经完成了任务，并内嵌回调了状态机
   - ACTION_RESULT_INLINE 当前未使用
   - ACTION_RESULT_IO_ERROR 当前未使用

也许会出现这样一种更复杂的问题：
   - 当结果为ACTION_RESULT_DONE
   - 同时，状态机在可重入的回调中释放了自身
 
因此，状态机的实现者必须：
   - 对没有在可重入回调中释放自身的状态机实现一个策略
   - 另外, 在创建异步任务时立即检查返回值
      - 如果该值为ACTION_RESULT_DONE，那么就不能对任何状态变量进行读写。

无论使用哪种方法，都要对返回值进行检查，同时进行相应的处理。


## 分配/释放策略:

Action的分配和释放遵循以下策略：

- Action由执行它的Processor进行分配。
- 在Action完成或者被取消后，Processor有责任和义务来释放它。
- 当返回的Action已经完成，或者已经取消，状态机就不可以再访问该Action。


## 参考资料
[I_Action.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Action.h)


# 核心部件：Event

Event类定义了一种EventProcessor返回的Action类型，它作为调度一个操作的结果由EventProcessor返回。

不同于Action的异步操作，Event是不可重入的。

除了能够取消事件（因为它是一个动作），你也可以在收到它之后重新进行调度。

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

    // 处理此Event的ethread指针，在ethread处理此Event之前填充（就是在schedule时）
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

    // ???
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


## 参考资料

- [I_Event.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Event.h)
- [P_UnixEvent.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/P_UnixEvent.h)
- [UnixEvent.cc](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/UnixEvent.cc)
