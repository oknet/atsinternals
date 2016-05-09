# 权利保留

对本项目内容的任何转发和引用，请注明本项目地址：http://github.com/oknet/atsinternals

# Apache Traffic Server 源代码分析

本项目主要是我对Apache Traffic Server 6.0（以下简称ATS）源代码的分析结果。

是我对其内部实现，架构设计的个人认识，受限于个人能力，可能有理解、分析不到位的情况。

部分内容是直接将ATS源码中的注释直接翻译过来的，可能有些不通顺，可以对照源码理解阅读。

由于 ATS 6.0 的变化比较大，所以如果您在使用/参照的源代码不是6.0版本，请翻阅官方git仓库中的源代码变化，以帮助理解。

# 联系作者

如果对内容有不理解，或者认为有错误，可以在github直接留言给我。

# 源代码目录结构

ATS的源代码由多个部分构成，分别设立独立的目录以区分：

  - iocore
    - 实现各种底层的I/O交互
    - 包含了多个子系统（subsystem）
  - proxy

## IOCORE

在IOCore内包含了多个子系统，每个子系统使用独立的目录来区隔：

  - EventSystem
    - 提供基本的事件系统，用以实现延续（Continuation）编程模式的基础组件
  - Net
    - 提供基本的网络操作，TCP，UDP，OOB，Polling等网络I/O操作
  - DNS
    - 提供DNS解析服务
  - HostDB
    - 提供DNS解析结果的缓存／查询服务 
  - AIO
    - 提供磁盘I/O操作 
  - Cache
    - 提供缓存服务 
  - Cluster
    - 提供集群服务 

另外 Utils 目录是公共部分，不是一个子系统。

在每个子系统内，有两种类型的头文件前缀：

  - I_xxxx.h
    - 对外提供接口的定义，I表示Interface
    - 可以 include I_xxxx.h，但是不可以 include P_xxxx.h
  - P_xxxx.h
    - 在子系统内部使用的定义，P表示Private
    - 可以 include I_xxxx.h，也可以 include P_xxxx.h

### EventSystem子系统

主要包含Interface基础类型的定义：

  - Continuation
  - Action
  - Thread
  - Processor
  - ProxyMutex
  - IOBufferData
  - IOBufferBlock
  - IOBufferReader
  - MIOBuffer
  - MIOBufferAccessor
  - VIO

主要包含Interface扩展类型的定义

  - VConnection <- Continuation
  - Event <- Action
  - EThread <- Thread
  - EventProcessor <- Processor

可以看到EventSystem子系统基本上都是Interface类型的定义，这是因为它是延续（Continuation）编程模式的基础组件。

### Net子系统

主要包含Interface定义：

  - NetVCOptions
  - NetVConnection <- VConnection
  - NetProcessor <- Processor
  - SessionAccept <- Continuation
  - UDPConnection <- Continuation
  - UDPPacket
  - UDPNetProcessor <- Processor

主要包含Private基础定义：

  - PollDescriptor
  - PollCont <- Continuation
  - Connection
  - Server <- Connection
  - NetState

TCP 相关：

  - UnixNetVConnection <- NetVConnection
  - NetHandler <- Continuation
  - NetAccept <- Continuation
  - UnixNetProcessor <- NetProcessor

SSL 相关：

  - SSLNetVConnection <- UnixNetVConnection
  - SSLNetAccept <- NetAccept
  - SSLNetProcessor <- UnixNetProcessor
  - SSLSessionID
  - SSLNextProtocolAccept <- SessionAccept
  - SSLNextProtocolTrampoline <- Continuation
  - SSLNextProtocolSet

Socks 相关：

  - SocksEntry <- Continuation

UDP 相关：

  - UDPIOEvent <- Event
  - UDPConnectionInternal <- UDPConnection
  - UnixUDPConnection <- UDPConnectionInternal
  - UDPPacketInternal <- UDPPacket
  - PacketQueue
  - UDPQueue
  - UDPNetHandler <- Continuation

可以看到Net子系统已经开始区分Interface和Private类型的定义了，对于Private类型的定义，应该仅仅在Net子系统内部直接使用。

注意：由于ATS的代码由社区的许多开发者功能开发，因此在编写代码时可能打破了上面这种逻辑，所以在阅读代码时不必过于纠结。

#### NetVConnection的继承关系

早期，这非常的久远，作为ATS开源的前身，Inktomi Cache Server 是支持Windows操作系统的，从现在ATS的代码里仍然能看到一些被注释掉的代码，使用的一些类的名字以NT开头，而且可以找到与其对应的以Unix开头的类定义。

因此对于 NetVConnection 的继承类 UnixNetVConnection：

  - 可以大胆猜测存在 NTNetVConnection 也是 NetVConnection 的继承类
  - 然后我们看到，NetVConnection的定义是出现在I_NetVConnection.h中的，所以NetVConnection是对外提供的Interface
  - 而UnixNetVConnection的定义是出现在P_UnixNetVConnection.h中的，所以UnixNetVConnection则是只在Net子系统内使用
  - 然后再看所有头文件名称内包含Unix关键字的，都是以P\_开头的

因此我们可以得出下面的结论：

  - NetVConnection 是Net子系统对外部提供的Interface，所有可以供外部调用的方法都需要放在其中以虚方法来定义
  - UnixNetVConnection 和 NTNetVConnection 则在不同的平台，使用不同的实现方式，重新定义基类NetVConnection的虚方法
  - 以此来实现NetVConnection对多操作系统平台的支持
  - 而外部系统无需关心NetVConnection的具体底层实现是UnixNetVConnection还是NTNetVConnection


建立在对上述继承关系的理解之上，然后再来看一下SSLNetVConnection：

  - SSLNetVConnection是继承自UnixNetVConnection的
  - SSLNetVConnection的定义没有包含Unix或NT关键字
  - 实现SSLNetVConnection的 *.h 和 *.cc 文件名也没有包含Unix或NT关键字
  - 但是所有头文件名称内包含SSL关键字的，都是以P\_开头的

可以大胆猜测SSLNetVConnection的实现是通过宏定义来区分的：

  - 在Unix操作系统，SSLNetVConnection继承自UnixNetVConnection
  - 在WinNT操作系统，SSLNetVConnection继承自NTNetVConnection

但是后来NT部分的代码被砍掉了，所以现在看到的继承关系就是：

  - NetVConnection
    - UnixNetVConnection
      - SSLNetVConnection

但是在实现中，并未提供一个方法，让Net子系统的用户能够知道一个NetVConnection具体是什么类型，因此在proxy目录中看到大量的代码直接引用了UnixNetVConnection和SSLNetVConnection类型的动态转换来判断一个NetVConnection是什么类型。

由于SSL层的实现是建立在使用OpenSSL库的基础上，而且在实际的业务逻辑实现的时候，同样要设置一些SSL的属性，因此SSLNetVConnection需要提供一些Interface给Net子系统的用户，但是SSLNetVConnection的这些方法并没有在NetVConnection中进行定义，这里大概是因为SSLNetVConnection与NetVConnection中间还隔着UnixNetVConnection，有些不方便，而又有一些丑陋吧。

因此，Net子系统的用户在使用SSL相关的功能时，都需要将NetVConnection动态转换为SSLNetVConnection类型才可以进行调用，这些都打破了原有系统的良好设计。

注：上述分析由于包含大量猜测，只是本人个人理解。

