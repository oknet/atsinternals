# 基础组件：TwoWayTunnel

OneWayTunnel 在设计之初就考虑到了数据不可能只是单向的流动，对于一对VC，会有数据从eastVC传输到westVC，也会有数据从westVC传输到eastVC。

通过把两个 OneWayTunnel 关联到一起，就非常简单的实现了TwoWayTunnel。

## 实现

假定我们当前在一个状态机里，该状态机已经得到一个成功建立的clientVC，同时该状态机已经发起了一个serverVC。

当这个状态机收到了针对此 serverVC 的 NET_EVENT_OPEN 事件时，就表示 serverVC 已经成功与服务器端建立了 TCP 连接。

此时我们要为 客户端 和 服务器端 的两个 VC 建立一个 TwoWayTunnel 以实现他们的互通。

首先，创建两个 OneWayTunnel

```
OneWayTunnel *c_to_s = OneWayTunnel::OneWayTunnel_alloc();
OneWayTunnel *s_to_c = OneWayTunnel::OneWayTunnel_alloc();

// 由于我们已经为 clientVC 做过 do_io_read 操作，已经申请了 MIOBuffer，对应的 IOBufferReader 为 reader
// 因此，调用第二种 init() 来初始化：
//   只对 serverVC 执行 do_io_write
//   传入的状态机为空，此时 mutex＝new()
c_to_s->init(clientVC, serverVC, NULL, clientVIO, reader);

// 然后，调用第一种init初始化：
//   对 serverVC 执行 do_io_read
//   对 clientVC 执行 do_io_write
//   将 c_to_s 的 mutex 传递进去，让两个 OneWayTunnel 共享同一个 mutex
s_to_c->init(serverVC, clientVC, NULL, 0 , c_to_s->mutex);

// 最后，关联两个Tunnel
OneWayTunnel::SetupTwoWayTunnel(c_to_s, s_to_c);
```

至此，这个 TwoWayTunnel 就建立起来了，clientVC 如果有数据可读，就会读取到内部的MIOBuffer里，然后在 serverVC 可写的时候发送出去。
同样的，当 serverVC 如果有数据可读，就会读取到内部的MIOBuffer里，然后在 clientVC 可写的时候发送出去。

可以完全看作是两个独立的 OneWayTunnel 在工作，但是由于两个 Tunnel 共享同一个 mutex，所以两个方向的数据传递总是交替进行。
实际上 clientVC 和 serverVC 在同一个 ET_NET 线程里，它们总是被先后回调，因此两个 OneWayTunnel 不会同时被回调。

双向通信的关键在于，当一个VC收到了EOS或者完成了既定的传输字节数，需要进行关闭时，它要同时考虑两个Tunnel的情况：

- 当clientVC收到了EOS，作为Tunnel c_to_s中的源VC时，需要查看serverVC是否已经将MIOBuffer的内容全部发送出去了
  - 如果还有数据需要发送，就激活serverVC，以完成剩余数据的发送
- 当clientVC收到了ERROR，作为Tunnel s_to_c中的目标VC时，需要通知Tunnel c_to_s，当前Tunnel即将关闭
  - Tunnel c_to_s收到通知后，会关闭clientVC的readVIO和serverVC的writeVIO，并释放 Tunnel，但是两个VC并不会关闭
  - 然后Tunnel s_to_c会关闭clientVC的writeVIO和serverVC的readVIO，关闭两个VC，并释放 Tunnel。

可以看到，TwoWayTunnel 是以目标VIO完成为依据来进行关闭的。

- 任何一个 OneWayTunnel 收到 WRITE_COMPLETE 事件，就直接关闭另一个 OneWayTunnel
  - 然后再关闭两个VC，最后关闭自己
- 但是收到 EOS 时，则会设置目标VC的VIO为完成剩余MIOBuffer内数据的发送
  - 然后等待目标VC完成数据发送，OneWayTunnel 收到 WRITE_COMPLETE 事件

在 TwoWayTunnel 中，一个方向关闭，另外一个方向也会被关闭，即使另外一个OneWayTunnel仍然在正常通信。
例如，通过shutdown进行半关闭，会导致一个OneWayTunnel的关闭，但是另外一个方向的OneWayTunnel仍然可以进行通信，此时也被迫关闭了。

## 成员

在 OneWayTunnel 中定义了成员：

- tunnel_peer
  - 指向关联的 OneWayTunnel
- free_vcs
  - 默认为 true，表示在关闭VIO之后，关闭VC
  - 但是当 OneWayTunnel 收到对端的关闭通知时，不会在关闭VIO之后继续关闭VC，而是由发起通知的一方完成VC的关闭

## 方法

在 startEvent 里判断来自对端 OneWayTunnel 的通知，在 WRITE_COMPLETE 之后发送通知给对端 OneWayTunnel：

```
  switch (event) {
  case ONE_WAY_TUNNEL_EVENT_PEER_CLOSE:
    /* This event is sent out by our peer */
    ink_assert(tunnel_peer);
    tunnel_peer = NULL;
    free_vcs    = false;
    goto Ldone;

...

  Ldone:
  case VC_EVENT_WRITE_COMPLETE:
    if (tunnel_peer) {
      // inform the peer:
      tunnel_peer->startEvent(ONE_WAY_TUNNEL_EVENT_PEER_CLOSE, data);
    }
    close_source_vio(result);
    close_target_vio(result);
    connection_closed(result);
    break;
```


在关闭VIO之后，根据free_vcs的值，判断是否继续关闭VC
```
void
OneWayTunnel::close_source_vio(int result)
{
  if (vioSource) {
    if (last_connection() || !single_buffer) {
      free_MIOBuffer(vioSource->buffer.writer());
      vioSource->buffer.clear();
    }
    if (close_source && free_vcs) {
      vioSource->vc_server->do_io_close(result ? lerrno : -1);
    }
    vioSource = NULL;
    n_connections--;
  }
}

void
OneWayTunnel::close_target_vio(int result, VIO *vio)
{ 
  (void)vio;
  if (vioTarget) {
    if (last_connection() || !single_buffer) {
      free_MIOBuffer(vioTarget->buffer.writer());
      vioTarget->buffer.clear();
    } 
    if (close_target && free_vcs) {
      vioTarget->vc_server->do_io_close(result ? lerrno : -1);
    }
    vioTarget = NULL;
    n_connections--;
  }
}
```

通过 SetupTwoWayTunnel 完成两个 OneWayTunnel 的关联：

```
void
OneWayTunnel::SetupTwoWayTunnel(OneWayTunnel *east, OneWayTunnel *west)
{
  // make sure the both use the same mutex
  ink_assert(east->mutex == west->mutex);

  east->tunnel_peer = west;
  west->tunnel_peer = east;
}
```
## 状态机的设计模式

我们看到 TwoWayTunnel 是由两个 OneWayTunnel 组成，它们共享一个 mutex，而且它们之间还设计了专门进行协同的事件。

每一个 OneWayTunnel 管理两个VC，但是对于其中任何一个VC来说：

- OneWayTunnel 要么负责接收数据，要么负责发送数据
- 但是不会既接收又发送数据

也就是说 OneWayTunnel 只管理这两个VC的一半功能。

在 TwoWayTunnel 中：

- 一个VC的 数据接收 被一个OneWayTunnel管理时，
- 这个VC的 数据发送 则被另外一个OneWayTunnel管理

TwoWayTunnel 的设计实现了对两个VC的完全管理，但是 TwoWayTunnel 并不是一个状态机，它是通过两个 OneWayTunnel 之间的通知机制，以及在两个 OneWayTunnel 中共享同一个 mutex 来实现了两个 OneWayTunnel 的协同。

但是我们从代码的实现上也看到了 TwoWayTunnel 里存在的一个问题，就是不支持“TCP的半关闭”，当其中一个VC发起了shutdown，那么TwoWayTunnel就要同时关闭两个VC，所以我们看到 Tunnel 状态机能做的事情非常的简单，使用它来实现单纯的 TCP代理 及其 附属功能 是完全没有问题的，但是如果需要实现基于TCP通信的高级协议处理，Tunnel 状态机则会很难实现。

因此，TCP代理和流处理才是 Tunnel 状态机的强项。

同时，我们看到双 MIOBuffer 的功能还未在代码里实现，通过双MIOBuffer我们可以

- 实现对Tunnel内数据流的修改
- 在Tunnel里实现流量控制等功能

但是当我们需要实现复杂的状态机设计时，我们会让一个状态机同时管理一个VC的数据接收和发送，例如：

- 接收一个请求，发现请求是不合法的，我们可以直接发送错误信息
- 但是当请求是合法内容时，可以通过Tunnel状态机完成数据流的传递

此时我们就需要先建立一个能够与ClientVC进行交互的状态机，然后再把VC的控制权转交给Tunnel状态机，Tunnel状态机处理完成后，再返回到交互状态机。

这种嵌套的设计，在ATS中使用主－从关系来表述：

- 与ClientVC进行交互的状态机被叫做“主状态机”（Master SM）
- 与ServerVC进行交互的状态机被叫做“从状态机”（Slave SM）
- 把ClientVC和ServerVC连接起来的Tunnel状态机也被叫做“从状态机”（Slave SM）

一个“主状态机”（Master SM）可以关联多个“从状态机”（Slave SM），当需要执行特定子任务时：

- 主状态机创建从状态机，并调用从状态机的构造函数把VC的控制权移交给从状态机，
- 当从状态机完成任务之后，回调主状态机，由主状态机重新获得VC的控制权，并销毁从状态机。

在后面介绍 HttpSM 状态机时，会看到其内包含了 Tunnel 状态机的设计，在ATS中普遍存在这种大状态机里面嵌套小状态机的设计。

## 参考资料
- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
