# 概述

EventSystem 和 Net Sub-System 作为ATS的基础架构，

  - 通过 NetAccept 实现了接受来自 Client 连接的能力
  - 通过 NetProcessor 实现了发起连接到 Server 的能力
  - 通过 NetHandler 实现了读写数据的能力

那么，如何支持各种通信协议呢？

在 TCP 通信中，需要首先判断出在 TCP 通信里采用哪一种协议，然后才能选择对应协议的状态机来处理这个 TCP 通信。

在前面的章节，分别介绍了两个重要的组件：

  - SSLNextProtocolTrampoline
    - 通过 NPN / ALPN 协议来获得 SSL 通道内即将进行的通信协议的类型
  - ProtocolProbeTrampoline
    - 通过读取 Client 发送来的第一个数据内容，进行分析后，得到即将进行的通信协议的类型

注：有一些协议（例如：SMTP协议），无法通过ProtocolProbeTrampoline进行协议类型的判断。

  - Client端连接到Server之后，Client不会发送任何信息，
  - 而是Server端首先要发送内容给Client端。

通过上面的两个组件，就可以知道接下来要进行通信的协议类型了，然后按照以下的流程进行：

  - 把 netvc 传递给对应该协议的 XXXSessionAccept 类实例（如：HttpSessionAccept）
  - XXXSessionAccept 类实例创建 XXXClientSession 类实例（如：HttpClientSession）
  - XXXClientSession 类实例创建 XXXSM 类实例（如：HttpSM），并将 netvc 传递给 XXXSM
  - XXXSM 类实例发起到 OServer 的连接，并创建对应的 netvc
  - XXXSM 类实例创建 XXXServerSession 类实例（如：HttpServerSession），并传入新建的 netvc

上面多种类对象，都将共享 Client 端 netvc 的 mutex：

  - XXXClientSession
  - XXXSM
  - Server 端 netvc
  - XXXServerSession

在完成一次通信之后：

  - XXXSM 会将 XXXServerSession 剥离（release / deattach）
    - 会根据 OServer 是否支持连接复用的情况，来决定是否同时关闭与 OServer 连接的 netvc
  - 然后 XXXSM 会将 netvc 传递回 XXXClientSession，然后将自己删除
  - 然后 XXXClientSession 会等待下一个请求的到来，或者是连接的关闭

接下来将会对 XXXSessionAccept 和 XXXClientSession 进行详细的介绍

  - 对于 XXXServerSession 的介绍，将会留到 SessionPool 的部分再一起详细的说明。
  - 对于 XXXSM 的介绍，将会有单独的章节进行分析。

