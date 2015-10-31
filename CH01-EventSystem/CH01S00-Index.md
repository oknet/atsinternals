# 简介


Apache Traffic Server 的核心是一个事件驱动的引擎（Event Driven Engine），该引擎通过事件（Event）与外部进行交互。在该引擎内有多个事件处理线程（EThread），这些线程由全局唯一实例（eventProcessor）统一管理。
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

