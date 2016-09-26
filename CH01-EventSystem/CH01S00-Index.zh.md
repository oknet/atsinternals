# 简介

事件驱动的引擎（Event Driven Engine）是 Apache Traffic Server 的核心组件之一。该引擎通过事件（Event）与外部进行交互。为了实现对事件的并行处理，在该引擎内有多个事件处理线程（EThread），这些线程由全局唯一实例（eventProcessor）统一创建和管理。

我们可以把这个引擎想象成汽车的发动机，参照下图，这就像是一个直列多缸的发动机，烧的不是汽油，而是事件（Event）。

```
+-------------------+
|   eventProcessor  |
+---+---+---+---+---+
| E | E | . | E | E |
| T | T | . | T | T |
| h | h | . | h | h |
| r | r | . | r | r |
| e | e | . | e | e |
| a | a | . | a | a |
| d | d | . | d | d |
+---+---+---+---+---+
```

## 事件系统（Event System）的组成

由以下三部分组成：

- 基础部件
   - ProxyMutex
   - Continuation
   - Lock
- 核心部件
   - 类型实例分配器（ClassAllocator）
      - 继承自 Allocator 基类
      - 用于在线程内为各种数据类型进行内存分配
   - 线程本地的内存分配器（ProxyAllocator）
      - freelist在线程内部的实例
   - 事件线程（EThread）
      - 继承自 Thread 基类
   - 事件（Event）
      - 继承自 Action 基类
- 对外接口
   - 事件处理机（eventProcessor）
      - 继承自 Processor 基类
      - 独立的唯一实例，创建了多个EThread线程

下面，我将先对事件系统的运行机制做一个大概的说明，然后我们会进入iocore/eventsystem目录，对照源代码，对每一个部分做详细的介绍和分析。
