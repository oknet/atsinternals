# 基础部件：Continuation（延续）

在学术上，这种以 Continuation 为底层的设计（编程）模型叫做 Continuation 的编程模型（方法）。
这个技术相当古老，后来微软围绕这个方案，改进出了 Co-Routine 的编程模型（方法）。

那么什么是 Continuation？要理解 ATS 的源代码，就必须明白这到底是什么，我的理解：

- 它是一个记事本，使用一行信息记录了接下来要做的事情
- 记事本分配给一个工人，工人按照记事本上的记录开始干活
- 任何时候，工人都可以归还该记事本，只要把之前记录的信息划掉，然后把接下来要做的事情写上
- 在记事本归还后，会再次进行分配
- 有个特别的任务就是：撕毁这个记事本，那么记事本就不用归还了
- 它有一个锁，避免两个工人同时干一个活，产生冲突

Continuation 的派生类：

- 一般来说 Continuation 的派生类就是状态机
- 记事本附带了一个操作指南，里面的每一个步骤都有一个代号，与代号对应的操作流程
- 记事本上使用代号指示接下来要做的事情
- 工人通过代号，在操作指南中找到对应的操作流程进行操作

在一定程度上来讲 Continuation 是整个异步回调机制的多线程事件编程基础。

- 在 ATS 中，Continuation 是一个最最基础的抽象结构
- 后续的所有高级结构，如 Action，Event，VC 等都封装 Continuation 数据结构。

以下部分来自源代码中注释的简单翻译：

- Continuation 用来完成 IOCore EventSystem 向它的使用者传递事件的功能。
- Continuation 是一个通用类，可以用于实现事件驱动的状态机。
- 通过派生类，扩展出更多状态和方法，Continuation 可以将状态与控制流结合。通常这样的设计用于支持分阶段的，由事件驱动的控制流程。


## 定义／结构

延续（Continuation）是所有状态机接收事件通知的基类。

```
#define CONTINUATION_EVENT_NONE 0

#define CONTINUATION_DONE 0
#define CONTINUATION_CONT 1

typedef int (Continuation::*ContinuationHandler)(int event, void *data);

class Continuation: private force_VFPT_to_top
{
public:
  /**
    当前的 Continuation 处理函数

    在对 mutex 成员上锁之后，使用 SET_HANDLER 宏进行设置。
    不要直接对该成员进行操作。
  */
  ContinuationHandler handler;

  /**
    当前 Continuation 对象的锁

    通过构造函数完成初始化，不要直接对该成员进行操作。
  */
  Ptr<ProxyMutex> mutex;

  /**
    链接到其他Continuation的双向链表

    在需要创建一个队列用来保存 Continuation 对象时使用。
  */
  LINK(Continuation, link);

  /**
    接收来自 Event 回调的事件代码和事件数据

    接收事件代码和事件数据并且透传给 handler。
    回调 Continuation 的 Processor 负责对 Continuation 的 mutex 上锁。

    @param event 由 Processor 指定，在回调时传入的事件代码
    @param data 由 Processor 指定，与事件代码相关的数据
    @return 状态机和 Processor 指定的返回值（代码）

  */
  int handleEvent(int event = CONTINUATION_EVENT_NONE, void *data = 0) {
      return (this->*handler) (event, data);
  }

    Continuation(ProxyMutex * amutex = NULL);
};

inline Continuation::Continuation(ProxyMutex *amutex)
  : handler(NULL), mutex(amutex)
{
}

#define SET_HANDLER(_h) (handler = ((ContinuationHandler)_h))
#define SET_CONTINUATION_HANDLER(_c, _h) (_c->handler = ((ContinuationHandler)_h))
```

延续（Continuation）是一个轻型数据结构，

- 它的实现只有一个用于回调的方法```int handleEvent(int , void *)```
   - 该方法是面向 Processor 和 EThread 的一个接口。
   - 它可以被继承，以添加额外的状态和方法
- 可以通过提供```ContinuationHandler```（成员```handler```的类型）来决定延续的行为
   - 该函数在事件到达时，由 handleEvent 调用。
   - 可以通过以下方法来设置/改变（头文件内定义了两个宏）：
      - SET\_HANDLER(\_h)
      - SET\_CONTINUATION\_HANDLER(\_c, \_h)

在 ATS 中，我们不会直接创建 Continuation 类型的对象，而是创建其继承类的实例，因此也就不存在直接与之对应的 ClassAllocator 对象。


## ProxyMutex 对象

鉴于事件系统的多线程特性，每个 Continuation 都带有一个对 ProxyMutex 对象的引用，以保护其状态，并确保原子操作。

因此，在创建任何一个 Continuation 对象或由其派生的对象时：

- 通常由使用 EventSystem 的客户来创建 ProxyMutex 对象
- 然后在创建 Continuation 对象时，将 ProxyMutex 对象作为参数传递给 Continuation 类的构造函数。

## TSAPI

Continuation 在 Plugin API 中叫做 TSCont，是插件开发中最常用到的类型之一，是多数 API 要求的参数类型。


## 参考资料

- [I_Continuation.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/eventsystem/I_Continuation.h)

