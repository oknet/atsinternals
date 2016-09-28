# 状态机

本章开始介绍的状态机实际上是特指连接了ClientVC和ServerVC的状态机。

前面介绍过的“蹦床”、“ClientSession”，实际上已经是一个状态机了，能够处理一些事件，但是这些状态机只能够处理来自一个VConnection的事件。

本章将开始介绍用于实现代理功能的状态机，该状态机将会：

  - 连接多个VConnection
  - 接收来自这些VConnection的事件
  - 在这些VConnection之间传递数据

对于两个 VConnection 之间的数据通信，可以分成：

  - 单向（OneWayTunnel）
    - 从一个VC读取数据然后通过另一个VC发送数据
  - 双向（TwoWayTunnel）
    - 从一个VC读取数据然后通过另一个VC发送数据
    - 同时相反方向的数据传递
    - 相当于把两个单向组合在一起

对于多个 VConnection 之间的数据通信，ATS可以实现：

  - 单向多出（OneWayMultiTunnel）
    - 从一个VC读取数据，然后向多个VC同时发送数据
    - 而且可以兼顾多个目标VC对数据消费速度的快慢不同
 
对于单纯实现数据传递来说，上面所描述的三种状态机已经足够了，但是有的时候我们要做一些更细致的设计来满足业务需要，例如：

  - 对接收到的数据进行验证后，才发送给目标VC
    - 如果验证失败，还要通知来源VC数据有错误，需要重新发送
    - 如果验证成功，要通知来源VC数据正在处理，请等待
  - 对目标VC返回的数据进行拆分，因为可能返回多个结果
    - 把拆分后的结果分别发给对应的来源VC

这个时候就不是简单的只实现数据传递工作，所以一个能够满足业务需求的状态机的架构应该是这样的：

```
                +----------+------------------------------+----------+
                |          |                              |          |
--------------> |          +===>>=== OneWayTunnel ===>>===+          | -------------->
                |          |                              |          |
                | ClientVC |         StateMachine         | ServerVC |
                |          |                              |          |
<-------------- |          +===<<=== OneWayTunnel ===<<===+          | <--------------
                |          |                              |          |
                +----------+------------------------------+----------+
```

实现业务功能的状态机对OneWayTunnel进行控制：

  - 在需要解析业务的时候，由状态机负责管理ClientVC和ServerVC
  - 在需要直接传递数据的时候，状态机把管理权交给OneWayTunnel，等数据传递完成后，再立即交还给状态机
  - Tunnel就像是状态机的一个助手，可以完成一些简单轻松的工作

那么，作为状态机是如何同时接收并处理来自两个VC的事件呢？

  - OneWayTunnel
    - ClientVC 与 ServerVC 共享同一个IOBuffer
    - 因此谁先拿到OneWayTunnel状态机的锁，谁就可以操作这个IOBuffer
    - ClientVC负责向IOBuffer写入数据，ServerVC总是从IOBuffer消费数据
    - 任何一方如果拿不到锁就等下一次
    - 如果IOBuffer写满了，就通知ServerVC赶紧消费
    - 如果IOBuffer读空了，就通知ClientVC赶紧生产
    - ClientVC收到了 EOS 事件，仍然需要把IOBuffer里剩余的数据写入ServerVC
    - ServerVC收到了 EOS 事件，整个Tunnel就终止了
  - TwoWayTunnel
    - 把两个OneWayTunnel组合在一起
    - 其中一个OneWayTunnel终止，那么也要通知另外一个OneWayTunnel终止。
  - OneWayMultiTunnel
    - 基本与OneWayTunnel一样
    - TargetVC收到 EOS 事件时，只会将一个TargetVC关闭
    - 只有全部TargetVC都关闭了，整个Tunnel才会终止
    - ClientVC收到 EOS 事件时，仍然需要将IOBuffer里剩余的数据写入所有存活的TargetVC

可以看到对于Tunnel来说，由于所有的VC都回调同一个状态机，共享状态机内的同一个IOBuffer，所以整个流程不会出现问题，即使ClientVC与ServerVC不在同一个线程里。

对于实现复杂业务的状态机，最关键的一点就是在收到来自ClientVC的回调时，如何安全的操作ServerVC

  - 调用 server_vc->reenable() 是安全的
  - 其它任何操作都需要对 server_vc->mutex 上锁

同样的，在收到ServerVC的回调时，操作ClientVC也要遵循以上原则。
