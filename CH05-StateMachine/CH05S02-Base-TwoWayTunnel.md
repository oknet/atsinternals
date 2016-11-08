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

## 参考资料
- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
