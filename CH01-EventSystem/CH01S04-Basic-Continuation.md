# 基础部件：Continuation（延续）

Continuation，延续式编程模式。

要理解ATS的源代码，就必须明白这玩意到底是个什么东西，我的理解就是：

- 这是一个容器，记住上一次干到哪儿了
- 这东西带着一个锁，避免两个人同时干一个活，产生冲突

这是一个非常古老的技术，因为ATS也是一个非常古老的系统，以至于后来微软在这个技术基础上提出了一个改进型：Co-Routine变成模式。
在ATS中大多数的的数据结构都继承自Continuation，例如：Action，Event，VConnection，各种SM等。

以下部分来自源代码中注释的简单翻译：

- Continuation 是一个通用类，可以用于实现事件驱动的状态机。
- 通过包括其他状态和方法，Continuation 可以将状态与控制流结合，通常这样的设计用于支持分阶段，事件驱动的控制流程。
- 一定程度上来讲Continuation是整个异步回调机制的多线程事件编程基础。
- 在ATS中，Continuation是一个最最基础的抽象结构，后续的所有高级结构，如Action Event VC等都封装Continuation数据结构。
- 在学术上，这种以Continuation为底层的设计（编程）模型叫做Continuation的编程模型（方法）
  - 这个技术相当古老
  - 后来微软围绕这个方案，改进出了coroutine的编程模型（方法）


## 定义／结构

延续(Continuation)是所有状态机接收事件通知的基类。

```
class Continuation: private force_VFPT_to_top
{
public:
    ContinuationHandler handler;
    Ptr<ProxyMutex> mutex;
    LINK(Continuation, link);

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

延续（Continuation）是一个轻型数据结构，它的实现只有一个用于回调的方法：

- handler：当前的Continuation处理函数。
   - 用户可以通过提供“ContinuationHandler”（成员函数handler的类型）来决定延续的行为。
      - 该函数在事件到达时，由handleEvent调用。
      - 这个函数可以通过“setHandler”方法来设置/改变。头文件内定义了两个宏：
         - SET_HANDLER(_h)
         - SET_CONTINUATION_HANDLER(_c, _h)
- mutex：Continuation的锁。 
- link：链接到其他Continuation的双向链表。 
- handleEvent：接收event的代码和数据，并交给当前的处理函数处理。 

延续可以被继承来添加额外的状态和方法。


## ProxyMutex 对象

鉴于事件系统的多线程特性，每个Continuation都带有一个对ProxyMutex对象的引用，以保护其状态，并确保原子操作。

因此，在创建任何一个Continuation对象或由其派生的对象时：

- 通常由使用EventSystem的客户来创建ProxyMutex对象
- 然后再创建Continuation对象时，将ProxyMutex对象作为参数传递给Continuation类的构造函数。

## TSAPI

Continuation在API中的结构叫TSCont，是插件开发中最常用到的抽象之一，是多数API要求的参数结构。


## 参考资料

- [I_Continuation.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Continuation.h)
