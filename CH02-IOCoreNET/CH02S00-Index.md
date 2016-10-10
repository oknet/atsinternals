# 第二章 I/O核心：网络

根据对事件系统的了解，设计一个状态机，只需要接收来自事件系统的事件通知即可：

  - EVENT_IMMEDIATE
  - EVENT_INTERVAL

所以状态机总是被动的接收“通知”，但是我们知道，对于 Socket 来说，至少在 Linux 系统上并没有这种通知机制。

我们拥有的机制是 “polling”，需要应用程序主动发起 syscall 来判断哪些 Socket 有数据可读，哪些 Socket 是可以写入数据的。

这就需要设计一个把主动“polling”转换为事件的网络子系统（Net Sub-System），这个功能由 NetHandler 状态机来实现。

同时，在一个 “polling” 为基础的网络子系统中，新连接的进入由 NetAccept 状态机来实现，NetAccept 可以运行在两种模式下：

  - 阻塞模式
    - 在 DEDICATED EThread 中以阻塞方式调用 accept()，实现接受新连接
  - 非阻塞模式
    - 在 REGULAR EThread 中以非阻塞方式调用 accept()，实现接受新连接

另外，我们还需要对处于空闲状态的 Socket 连接进行超时管理，这个功能由 InactivityCop 状态机来实现。

为了实现上述功能，还有一些必要的基础部件和其它部件来配合，因此整个网络子系统（Net Sub-System）由以下部分组成：

  - 基础部件
    - 对 “polling” 的封装
      - EventIO（对 epoll/kqueue 的封装）
      - PollDescriptor（用来抽象 poll 句柄）
      - PollCont（周期性执行 polling 操作的状态机）
    - NetVConnection（用来连接 Socket，缓冲区 和 状态机）
      - 继承自 VConnection 基类
    - IOBuffer（用于存放数据的缓冲区）
    - VIO（用来描述 Socket 读写操作）
    - 对 “Socket操作” 的封装
      - Connection
      - Server
  - 核心部件
    - NetAccept（实现接受新连接的状态机）
    - NetHandler（实现从 “polling” 到事件转换的状态机）
    - InactivityCop（实现连接的超时控制）
    - Throttle（实现最大连接的限定）
    - UnixNetVConnection（NetVConnection在Unix操作系统上的具体实现）
      - 继承自 NetVConnection 基类
    - 实现协议类型的探测
      - ProtocolProbeSessionAccept
      - ProtocolProbeTrampoline
  - 对外接口
    - UnixNetProcessor
      - 继承自 NetProcessor 基类
