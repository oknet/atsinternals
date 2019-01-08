# 接口：TransformProcessor

TransformVConnection 是一个核心组件，通过 TransformProcessor 提供了接口，同时通过 TransformVCChain 基类提供了流量数据统计的功能。

所有 TransformVConnection 的使用者，应该仅通过 TransformProcessor 和 TransformVCChain 提供的接口对 TransformVConnection 进行管理和访问。

## 定义

```
class TransformProcessor
{
public:
  // 用来启动 prefetch 系统（会有独立的篇幅来介绍 prefetch 系统）
  void start();

public:
  // 创建 TransformVConnection 管道
  VConnection *open(Continuation *cont, APIHook *hooks);
  /* 在 TVC 管道的输入端，插入一个仅转发数据，但是不对数据进行任何修改的内部 Transform Plugin
   *
   * 可在 records.config 里，通过设置以下 Debug Tag 可以激活该内部插件，用来实现调试信息的输出：
   *   - http_post_nullt
   *      - 在 HTTP Request 阶段，让 POST 传输的内容强制通过 null transform 进行过滤
   *   - http_nullt
   *      - 在 HTTP Response 阶段，让 响应的内容强制通过 null transform 进行过滤
   *
   * 该内部 Plugin 仅用于调试用途，与 plugins/
   */
  INKVConnInternal *null_transform(ProxyMutex *mutex);
  /* 在 TVC 管道的最后一个 Transform Plugin 之后，追加一个用于支持 Range 头的内部 Transform Plugin
   *
   * 此时 TransformTerminus 仍然是 TVC 管道的最后一节，range_transform 则固定为倒数第二节。
   * 该插件仅针对 Http Response 的内容进行过滤和处理。
   */
  INKVConnInternal *range_transform(ProxyMutex *mutex, RangeRecord *ranges, int,
                                    HTTPHdr *, const char *content_type,
                                    int content_type_len, int64_t content_length);
};

// TransformProcessor 是单例模式，跟 netProcessor 是一样的。
extern TransformProcessor transformProcessor;
```

在 TransformVConnection 的基类 TransformVCChain 里提供了一个 backlog 的接口，用于获得在 TVC 管道内的数据量，用来辅助进行流量控制。

```
/** A protocol class.
    This provides transform VC specific methods for external access
    without exposing internals or requiring extra includes.
*/
class TransformVCChain : public VConnection
{
protected:
  /// Required constructor
  TransformVCChain(ProxyMutex *m);

public:
  /** Compute the backlog.  This is the amount of data ready to read
      for each element of the chain.  If @a limit is non-negative then
      the method will return as soon as the computed backlog is at
      least that large. This provides for more efficient checking if
      the caller is interested only in whether the backlog is at least
      @a limit. The default is to accurately compute the backlog.
  */
  // 定义纯虚函数 backlog 接口
  // TransformVConnection::backlog 负责具体实现该接口的功能。
  virtual uint64_t backlog(uint64_t limit = UINT64_MAX ///< Maximum value of interest
                           ) = 0;
};
```

## 方法

### TransformVConnection::backlog

`TransformVConnection::backlog` 方法由 `HttpTunnelProducer::backlog` 调用，此时 TVC 作为 HttpTunnel 的 Producer 一端，用于实现流量控制。

```
uint64_t
TransformVConnection::backlog(uint64_t limit)
{
  // 初始化 backlog 计数器为 0，用于表示在 TVC 管道内总共存储了多少待传输的数据
  uint64_t b = 0; // backlog
  VConnection *raw_vc = m_transform;
  MIOBuffer *w;
  // 从第一个 Transform Plugin 开始，逐个遍历所有 Transform Plugin
  // 不包括 TVC 管道最后的 TransformTerminus
  while (raw_vc && raw_vc != &m_terminus) {
    INKVConnInternal *vc = static_cast<INKVConnInternal *>(raw_vc);
    // 对读缓冲区 MIOBuffer 的最大可读取数据长度进行累加
    // BUG? 在整个 TVC 的设计里，只有 TransformTerminus::do_io_read 可以被调用，
    // 而 TVC 管道里其它的小节，只有 INKVConnInternal::do_io_write 被调用，
    // 因此这里对整个 TVC 管道的 m_read_vio 遍历是没有意义的。
    // 这里我认为应该是对 m_write_vio 进行遍历。
    if (0 != (w = vc->m_read_vio.buffer.writer()))
      b += w->max_read_avail();
    // 如果 backlog 计数器达到限定值就返回 backlog 的值
    if (b >= limit)
      return b;
    // 继续下一个 Plugin
    raw_vc = vc->m_output_vc;
  }
  // 由于 TransformTerminus 的结构与 INKVConnInternal 有差异，
  // 因此，在最后单独对 TransformTerminus 的 读缓冲区 MIOBuffer 的最大可读取数据长度进行累加
  if (0 != (w = m_terminus.m_read_vio.buffer.writer()))
    b += w->max_read_avail();
  // 如果 backlog 计数器达到限定值就返回 backlog 的值
  if (b >= limit)
    return b;

  // 对 TransformTerminus 的 写缓冲区 MIOBuffer 的最大可读取数据长度进行累加
  IOBufferReader *r = m_terminus.m_write_vio.get_reader();
  if (r)
    b += r->read_avail();

  // 返回 backlog 的值
  return b;
}
```

TVC 是一个管道，数据流从一端流入，然后再从另外一端流出，管道里会存有一部分数据，这部分数据会占用一些内存空间，backlog 就是为了统计这部分内存空间的占用，以避免当大量 TVC 管道建立时，消耗的内存不受控制的增长。

通过分析，我们了解到 HttpTunnel 也实现了 backlog 方法，这是因为 HttpTunnel 是一种支持多个输出的管道，对于流入的数据，HttpTunnel 会根据输出端的数量，将数据流同时发送给这些输出端，此时会由于输出端接收数据的速度不同，导致 HttpTunnel 需要缓存一些数据，这样也会导致对内存的占用。

当 TVC 存在时，HttpTunnel 的其中一个输出就是连接到 TVC 上的，然后 TVC 的输出会连接到第二个 HttpTunnel。

以下是 POST Transform 执行过程的示意图：

- 首先 HttpSM 创建第一个 HttpTunnel
  - Producer 端的 VC 是 ClientVC
  - 其中一个 Consumer端的 VC 是 TVC（HttpTunnel 可以有多个 Consumer 端）
  - 在 ClientVC 上执行 `do_io_read`，在 TVC 上执行 `do_io_write`
- POST 数据从 ClientVC 上读取到之后，写入 ProducerBuffer，同时回调 HttpTunnel Producer
- HttpTunnel Producer 将数据写入到 HttpTunnel Consumer
- HttpTunnel Consumer 内的 VC 就是 TVC，因此接收数据的缓冲区就是 TVC 的 InputBuffer
- 然后数据流逐个通过 TVC 管道上的每一个 Transform Plugin
- 最后到达 TransformTerminus，TransformTerminus 向 HttpSM 回调 TRANSFORM_READ_READY
- 然后 HttpSM 创建第二个 HttpTunnel
  - Producer 端的 VC 是 TVC
  - 其中一个 Consumer端的 VC 是 ServerVC
  - 在 TVC 上执行 `do_io_read`，在 ServerVC 上执行 `do_io_write`
- 经过 TVC 处理的数据从 TransformTerminus 上读取到之后，写入 OutputBuffer
- 事实上 ProducerBuffer 在执行 `do_io_read` 时传入到 TVC 作为 OutputBuffer
- 然后 HttpTunnel Consumer 从 ProducerBuffer 里消费数据，写入到 ServerVC

```

                  +----+       +----+       +----+       +----------+
              +-->| T1 |-Buf1->| T2 |-Buf2->| T3 |-Buf3->| Terminus |-->+
              |   +----+       +----+       +----+       +----------+   |
              |                                                         |
         InputBuffer                                               OutputBuffer
              |                                                         |
              |                                                         |
        +-----+------+                                            +-----+------+
        | HttpTunnel |                                            | HttpTunnel |
        |  Consumer  |                                            |  Producer  |
        +-----+------+                                            +-----+------+
              |                                                         |
              |                                                         |
        ProducerBuffer                                            ProducerBuffer
              |                                                         |
              |                                                         |
        +-----+------+                                            +-----+------+
        | HttpTunnel |                                            | HttpTunnel |
        |  Producer  |                                            |  Consumer  |
        +-----+------+                                            +-----+------+
              |                                                         |
              |                                                         |
           ClientVC                                                  ServerVC

```

在这个复杂的管道里，每一部分都可能临时保存一部分数据，通过 backlog 可以对这部分数据的长度进行统计，并控制这部分数据占用的内存。

## 参考资料

- [PluginVC.h](http://github.com/apache/trafficserver/tree/6.0.x/proxy/Transform.h)
- [PluginVC.cc](http://github.com/apache/trafficserver/tree/6.0.x/proxy/Transform.cc)
