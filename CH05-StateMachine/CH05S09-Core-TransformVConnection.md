# 核心组件：TransformVConnection

TransformVConnection 是用于数据流过滤和修改的核心组件，由于可能同时存在多个对数据流进行过滤或加工的组件，因此 TransformVConnection 的设计更像是一个流水线：

- 数据写入到 InputBuffer，
- 经过第一个 Transform Plugin(T1) 的加工后输出到 Buf1，
- 然后再经过第二个 Transform Plugin(T2) 的加工后输出到 Buf2，
- 然后再经过第三个 Transform Plugin(T3) 的加工后输出到 Buf3，
- 只要我们愿意，可以增加更多的加工流程，
- 最后经过 Terminus 处理后，将数据输出到 OutputBuffer

```
                +----+       +----+       +----+       +----------+
  -InputBuffer->| T1 |-Buf1->| T2 |-Buf2->| T3 |-Buf3->| Terminus |-OutputBuffer->
                +----+       +----+       +----+       +----------+
```

TransformVConnection 是：

- 由一节一节的管道连接起来的，进行单向数据传输的管道
  - 数据由 T1 传向 T2，再由 T2 传向 T3
  - 通道上的每一节都是一个 INKVConnInternal 对象，最后一节是 TransformTerminus 对象
     - INKVConnInternal 对象负责回调 Plugin 状态机
     - TransformTerminus 对象则内置一个状态机
  - 数据在每一节停留的时间受到该段 Plugin 状态机的控制
     - 该状态机可以收到数据后立即将数据转发出去
     - 也可以将所有数据都收集完成后，再一次性将数据转发出去
- 单向事件回调
  - 每一个 Transform Plugin 都会向其上一节的状态机回调事件，与数据传输方向刚好相反
  - 事件通常为 `VC_EVENT_WRITE_READY`，`VC_EVENT_WRITE_COMPLETE` 或者 `VC_EVENT_ERROR`
  - 每一节的状态机需要根据自身消费数据的情况向上游回调适当的事件
  - 但是对于 `VC_EVENT_ERROR` 事件，应该在收到后向上游透传

可以认为 TransformVConnection 就是把一节一节的管道连接起来之后，外面又套了一个壳/皮，让管道的连接更稳定。

另外，每一次 Transform Plugin 被回调时，应该首先检查其所在的 INKVConnInternal 是否已经被关闭，只有未关闭时，才可以对上游和下游进行操作，否则应该立即回收 Transform Plugin 的资源，然后返回。INKVConnInternal 对象将会自动销毁。

为了书写简便，以下使用 TVC 来表示 TransformVConnection。

## 定义

在 TVC 的链式通道上，最后一节是 TransformTerminus，它作为最后一个 Transform Plugin 的下游节点接收数据，并向最后一个 Transform Plugin 回调事件；同时还负责回调 HttpSM 和 HttpTunnel，向 HttpTunnel 传输数据。

由于 TVC 的链式通道的每一段都由一个 INKVConnInternal 和 Plugin 状态机构成，因此TransformTerminus 的定义与 INKVConnInternal 的定义非常相似：

```
class TransformTerminus : public VConnection
{
public:
  /* 构造函数：完成成员的初始化
   * 需要传入 TVC
   *   - Terminus 将会与 TVC 共享同一个 Mutex 对象
   *   - 成员 m_tvc 指向传入的 TVC 对象
   * 将以下成员初始化为 0
   *   - m_event_count 事件调度计数器
   *   - m_deletable 资源可被回收标志
   *   - m_closed 是否已经关闭标志
   *   - m_called_user 是否已经向 TVC 的使用者回调了 TRANSFORM_READ_READY 事件
   */
  TransformTerminus(TransformVConnection *tvc);

  // 主处理函数，相当于 Transform Plugin 的状态机
  int handle_event(int event, void *edata);

  // 实现 VConnection 的四个 do_io 方法 和 reenable 方法
  // 注意：这里没有实现 reenable_re 方法
  VIO *do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf);
  VIO *do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner = false);
  void do_io_close(int lerrno = -1);
  void do_io_shutdown(ShutdownHowTo_t howto);

  void reenable(VIO *vio);

public:
  // 成员变量
  TransformVConnection *m_tvc;
  // m_read_vio 相当于是 UnixNetVConnection::read.vio
  // m_read_vio 指向 Output Buffer 和对应的状态机
  VIO m_read_vio;
  // m_write_vio 相当于是 UnixNetVConnection::write.vio
  // m_write_vio 指向最后一个 Transform Plugin 输出的缓冲区 和 Plugin 状态机
  VIO m_write_vio;
  volatile int m_event_count;
  volatile int m_deletable;
  volatile int m_closed;
  int m_called_user;
};
```

TransformTerminus 作为 TVC 的成员（不是指针对象），与 TVC 同时创建和销毁。

```
// TVC 继承自 TransformVCChain，这个类主要用于实现流控，将在后面介绍
// 由于 TransformVCChain 继承自 VConnection，因此这里可以认为 TVC 也继承自 VConnection
class TransformVConnection : public TransformVCChain
{
public:
  // 构造函数
  TransformVConnection(Continuation *cont, APIHook *hooks);
  // 析构函数
  ~TransformVConnection();

  // 状态函数（无用途，不可被调用）
  int handle_event(int event, void *edata);

  // 实现 VConnection 的四个 do_io 方法 和 reenable 方法
  // 注意：这里没有实现 reenable_re 方法
  VIO *do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf);
  VIO *do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner = false);
  void do_io_close(int lerrno = -1);
  void do_io_shutdown(ShutdownHowTo_t howto);

  void reenable(VIO *vio);

  // 继承自 TransformVCChain 的方法
  /** Compute the backlog.
      @return The actual backlog, or a value at least @a limit.
  */
  virtual uint64_t backlog(uint64_t limit = UINT64_MAX);

public:
  // m_transform 指向第一个 Transform Plugin 的 INKVConnInternal 对象
  // 所有的 Transform Plugin 的 INKVConnInternal 对象在构造函数中被连接成一个链表
  // 就像是一节一节的管道连接起来的样子
  VConnection *m_transform;
  // m_cont 指向 TVC 的使用者，通常为 HttpSM 对象
  Continuation *m_cont;
  // 成员 TransformTerminus，作为 TVC 链表的最后一节
  TransformTerminus m_terminus;
  // TVC 是否已经关闭的标志
  volatile int m_closed;
};
```

## 方法

## TransformProcessor::open

在 HttpSM 中，通过调用 TransformProcessor::open 创建 TVC 数据过滤通道。

参数：

- cont
  - 使用者状态机对象
- hooks
  - 保存有全部已经注册到 HttpSM 里的 Transform Plugin 的 INKVConnInternal 链表

```
VConnection *
TransformProcessor::open(Continuation *cont, APIHook *hooks)
{
  if (hooks) {
    return new TransformVConnection(cont, hooks);
  } else {
    return NULL;
  }
}
```

## TransformVConnection 构造函数

```
TransformVConnection::TransformVConnection(Continuation *cont, APIHook *hooks)
// TVC 共享了 cont->mutex 的锁
  : TransformVCChain(cont->mutex),
// 初始化 m_cont 指向 TVC 的使用者
    m_cont(cont),
// 构造 TransformTerminus，传入 TVC 对象
    m_terminus(this),
// 重置 m_closed，表示 TVC 处于开启状态
    m_closed(0)
{
  INKVConnInternal *xform;

  SET_HANDLER(&TransformVConnection::handle_event);

  // 至少要有一个 Transform Plugin，否则只有一个 TransformTerminus 是无法工作的
  ink_assert(hooks != NULL);

  // 遍历所有的 Transform Plugin
  m_transform = hooks->m_cont;
  while (hooks->m_link.next) {
    // xform 保存当前的 INKVConnInternal 对象
    xform = (INKVConnInternal *)hooks->m_cont;
    // hooks 移动到下一个 INKVConnInternal 对象
    hooks = hooks->m_link.next;
    // 每一个 INKVConnInternal 对象内都有一个 m_output_vc 指针
    // INKVConnInternal::do_io_transform 的功能是将当前 INKVConnInternal 的 m_output_vc，
    // 指向下一个 INKVConnInternal 对象，这样就把所有的 INKVConnInternal 对象连接了起来。
    xform->do_io_transform(hooks->m_cont);
  }
  // 最后把 TransformTerminus 放入 TVC 链的尾部，TransformTerminus 是没有 m_output_vc 指针的。
  xform = (INKVConnInternal *)hooks->m_cont;
  xform->do_io_transform(&m_terminus);

  Debug("transform", "TransformVConnection create [0x%lx]", (long)this);
}
```

## TransformVConnection::do\_io\_write

TVC 是一个单向数据传输的管道，因此首先要向其写入数据。

```
VIO *
TransformVConnection::do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool /* owner ATS_UNUSED */)
{
  Debug("transform", "TransformVConnection do_io_write: 0x%lx [0x%lx]", (long)c, (long)this);

  return m_transform->do_io_write(c, nbytes, buf);
}
```

向 TVC 内写入数据，就是向第一个 Transform Plugin 的 INKVConnInternal 写入数据，因此这里直接透传调用了 `INKVConnInternal::do_io_write`。

```
INKVConnInternal::do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner)
{
  ink_assert(!owner);
  // 将包含有写入数据的 MIOBuffer 保存到 m_write_vio 对象
  m_write_vio.buffer.reader_for(buf);
  m_write_vio.op = VIO::WRITE;
  m_write_vio.set_continuation(c);
  m_write_vio.nbytes = nbytes;
  m_write_vio.ndone = 0;
  m_write_vio.vc_server = this;

  // 如果有可读数据，就通过回调方式通知 Transform Plugin 状态机
  if (m_write_vio.buffer.reader()->read_avail() > 0) {
    // 由于在接下来的 schedule_imm 操作中，并没有保存返回的 Event 对象，
    // 那么也就无法在 INKVConnInternal 对象关闭时，取消那些还未来得及回调的 Event 对象。
    // 这里通过一个原子计数器，记录下来还有多少个将会回调的 Event 在事件系统里，
    // 当这个计数器为 0 的时候，那么就可以安全的释放 INKVConnInternal 对象了。
    // 如果这个计数器不为 0，那么就继续等待，直到最后一个 Event 回调完成。
    // 这个设计可能效率不是很高，但是回头想想 PluginVC 里那四个 Event 的设计也挺复杂的。
    if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
      ink_assert(!"not reached");
    }
    // 为了避免对 HttpTunnel 或 上游 Transform Plugin 的阻塞，通过事件系统对 Transform Plugin 进行回调
    eventProcessor.schedule_imm(this, ET_NET);
  }

  // 向 HttpTunnel 或 上游 Transform Plugin 返回 VIO
  return &m_write_vio;
}
```

当第一个 Transform Plugin 向它下游的第二个 Transform Plugin 写入数据时，同样也是调用了 `INKVConnInternal::do_io_write` 方法。

## INKVConnInternal::reenable

INKVConnInternal 被 `do_io_write` 设置回调后，就会通过事件系统回调 Transform Plugin 的状态机，`m_write_vio` 指向的数据缓冲区会被 Transform Plugin 消费，如果 VIO 描述的数据传输未完成，Transform Plugin 会向上游发送 `VC_EVENT_WRITE_READY` 事件，然后上游状态机会重新填充缓冲区，通过调用 `do_io_write` 返回的 VIO 对象的 `reenable()` 方法，通知 Transform Plugin 的状态机继续消费数据。

而 `VIO::reenable()` 方法将会调用 `INKVConnInternal::reenable()`：

```
INKVConnInternal::reenable(VIO * /* vio ATS_UNUSED */)
{
  if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
    ink_assert(!"not reached");
  }
  eventProcessor.schedule_imm(this, ET_NET);
}
```

而 `TransformVConnection::reenable()` 方法则永远不会调用，也不应该被调用。


## TransformTerminus::do\_io\_write

当最后一个 Transform Plugin 向 TransformTerminus 写入数据时，调用的则是 `TransformTerminus::do_io_write` 方法。

```
VIO *
TransformTerminus::do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner)
{
  // In the process of eliminating 'owner' mode so asserting against it
  ink_assert(!owner);
  // 将包含有写入数据的 MIOBuffer 保存到 m_write_vio 对象
  m_write_vio.buffer.reader_for(buf);
  m_write_vio.op = VIO::WRITE;
  m_write_vio.set_continuation(c);
  m_write_vio.nbytes = nbytes;
  m_write_vio.ndone = 0;
  m_write_vio.vc_server = this;

  // 这里与 INKVConnInternal::do_io_write 几乎一样，只是少了对缓冲区可读数据的判断
  if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
    ink_assert(!"not reached");
  }
  Debug("transform", "[TransformTerminus::do_io_write] event_count %d", m_event_count);

  // TransformTerminus::handle_event 将会自动处理 m_write_vio
  eventProcessor.schedule_imm(this, ET_NET);

  // 向最后一个 Transform Plugin 返回 VIO
  return &m_write_vio;
}
```

## TransformTerminus::reenable

与 INKVConnInternal 一样，最后一个上游状态机填充数据后，会调用 `TransformTerminus::reenable` 通知 TransformTerminus 处理数据。

```
void
TransformTerminus::reenable(VIO *vio)
{
  ink_assert((vio == &m_read_vio) || (vio == &m_write_vio));

  // 原子计数器，与 INKVConnInternal 里的功能一样
  // 如果当前不存在对 TransformTerminus 进行回调的 Event，就对 TransformTerminus 进行一次调度，
  if (m_event_count == 0) {
    if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
      ink_assert(!"not reached");
    }
    Debug("transform", "[TransformTerminus::reenable] event_count %d", m_event_count);
    // TransformTerminus::handle_event 将会自动处理 m_write_vio 和/或 m_read_vio
    eventProcessor.schedule_imm(this, ET_NET);
  } else {
    // 否则就不需要对 TransformTerminus 进行重复的调度。
    // 这里的实现要比 INKVConnInternal 的效率高。
    Debug("transform", "[TransformTerminus::reenable] skipping due to "
                       "pending events");
  }
}
```

## TransformVConnection::do\_io\_read

当数据被写入到最后一节管道 TransformTerminus 之后，`TransformTerminus::handle_event` 将会向 HttpSM 回调 `TRANSFORM_READ_READY` 事件，同时传递数据总长度（`m_write_vio.nbytes`）。然后 HttpSM 就会创建 HttpTunnel 从 TVC 的尾端 TransformTerminus 读取数据了。

由于 `TransformTerminus::handle_event` 承担了多个功能，这里先跳过它，放在最后进行分析。

```
VIO *
TransformVConnection::do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf)
{
  Debug("transform", "TransformVConnection do_io_read: 0x%lx [0x%lx]", (long)c, (long)this);

  return m_terminus.do_io_read(c, nbytes, buf);
}
```

## TransformTerminus::do\_io\_read

由于 TVC 总是以 TransformTerminus 作为尾端，因此这里直接透传调用了 `TransformTerminus::do_io_read`。

```
TransformTerminus::do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf)
{
  // 将准备接收数据的 MIOBuffer 保存到 m_read_vio 对象
  m_read_vio.buffer.writer_for(buf);
  m_read_vio.op = VIO::READ;
  m_read_vio.set_continuation(c);
  m_read_vio.nbytes = nbytes;
  m_read_vio.ndone = 0;
  m_read_vio.vc_server = this;

  // 这里与 TransformTerminus::do_io_write 一样
  if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
    ink_assert(!"not reached");
  }
  Debug("transform", "[TransformTerminus::do_io_read] event_count %d", m_event_count);

  // TransformTerminus::handle_event 将会自动处理 m_read_vio
  eventProcessor.schedule_imm(this, ET_NET);

  // 向 HttpTunnel 返回 VIO
  return &m_read_vio;
}
```

## TransformVConnection::do\_io\_close

TransformVConnection 管道中间的各个小节，总是被动接收来自上游的数据，不会主动调用上游的 `do_io_read` 方法；其总是向上游发送 `VC_EVENT_WRITE_READY` 事件，让上游向下游主动传输数据。

因此，当 TVC 关闭的时候，也是由上游调用下游的 `do_io_close`。

```
void
TransformVConnection::do_io_close(int error)
{
  Debug("transform", "TransformVConnection do_io_close: %d [0x%lx]", error, (long)this);

  // BUG: 这里没有判断 m_closed 的值
  // 当 do_io_close 存在重入的时候会出现 double do_io_close() 导致的 crash
  // 参考：
  //   - [Pull Request #2907](https://github.com/apache/trafficserver/pull/2907)
  //     - 这个 PR 通过检查 m_closed 的值防止重复调用 INKVConnInternal::do_io_close
  //     - 同时在 m_output_vc->do_io_close 执行后，将 m_output_vc 设置为 NULL
  //   - [Pull Request #1765](https://github.com/apache/trafficserver/pull/1765)
  //     - 注意：这个提交被回退了
  //     - 在这个 PR 里可以看到对 do_io_close 的分析，确认会出现 TVC::do_io_close 的重复调用
  if (error != -1) {
    m_closed = TS_VC_CLOSE_ABORT;
  } else {
    m_closed = TS_VC_CLOSE_NORMAL;
  }

  m_transform->do_io_close(error);
}
```

可以看到 TVC 的 `do_io_close` 操作透传给了 TVC 管道上第一个小节的 `INKVConnInternal::do_io_close`。

```
void
INKVConnInternal::do_io_close(int error)
{
  if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
    ink_assert(!"not reached");
  }

  INK_WRITE_MEMORY_BARRIER;

  if (error != -1) {
    lerrno = error;
    m_closed = TS_VC_CLOSE_ABORT;
  } else {
    m_closed = TS_VC_CLOSE_NORMAL;
  }

  m_read_vio.op = VIO::NONE;
  m_read_vio.buffer.clear();

  m_write_vio.op = VIO::NONE;
  m_write_vio.buffer.clear();

  // 继续将 do_io_close 操作向 TVC 管道的下一节传递
  if (m_output_vc) {
    m_output_vc->do_io_close(error);
    // BUG: 调用 do_io_close 之后，没有把 m_output_vc 设置为 NULL
    // 根据 do_io_close 的设计要求，只要调用了 do_io_close 之后，就不能再次访问该 VConnection 对象
    // 参考：
    //   - [Pull Request #2907](https://github.com/apache/trafficserver/pull/2907)
  }

  // 回调 Transform Plugin 的状态机
  // 该状态机应该首先通过 TSVConnClosedGet() 检查 INKVConnInternal 对象是否已经关闭
  eventProcessor.schedule_imm(this, ET_NET);
}
```

可以看到 `do_io_close` 是以同步递归的方式从第一节管道一直传递到最后一节管道，一瞬间，管道上的每一节都关闭了，TransformTerminus 是最后关闭的一节。

## TransformTerminus::do\_io\_close

```
void
TransformTerminus::do_io_close(int error)
{
  if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
    ink_assert(!"not reached");
  }

  INK_WRITE_MEMORY_BARRIER;

  // BUG：同样未检测 m_closed 是否已经关闭
  if (error != -1) {
    lerrno = error;
    m_closed = TS_VC_CLOSE_ABORT;
  } else {
    m_closed = TS_VC_CLOSE_NORMAL;
  }

  m_read_vio.op = VIO::NONE;
  m_read_vio.buffer.clear();

  m_write_vio.op = VIO::NONE;
  m_write_vio.buffer.clear();

  // TransformTerminus::handle_event 将会自动判断出 TVC 的入口和出口都已经关闭
  eventProcessor.schedule_imm(this, ET_NET);
}
```

## TransformVConnection::do\_io\_shutdown

最后还剩下一种 do\_io 操作，就是 `do_io_shutdown`。

```
void
TransformVConnection::do_io_shutdown(ShutdownHowTo_t howto)
{
  ink_assert(howto == IO_SHUTDOWN_WRITE);

  Debug("transform", "TransformVConnection do_io_shutdown: %d [0x%lx]", howto, (long)this);

  m_transform->do_io_shutdown(howto);
}
```

TVC 仍然是将 `do_io_shutdown` 透传给了 INKVConnInternal。

```
void
INKVConnInternal::do_io_shutdown(ShutdownHowTo_t howto)
{
  // shutdown 操作对于 INKVConnInternal 来说只是清理对应方向的缓冲区
  if ((howto == IO_SHUTDOWN_READ) || (howto == IO_SHUTDOWN_READWRITE)) {
    m_read_vio.op = VIO::NONE;
    m_read_vio.buffer.clear();
  }

  if ((howto == IO_SHUTDOWN_WRITE) || (howto == IO_SHUTDOWN_READWRITE)) {
    m_write_vio.op = VIO::NONE;
    m_write_vio.buffer.clear();
  }

  // 原子计数器
  if (ink_atomic_increment((int *)&m_event_count, 1) < 0) {
    ink_assert(!"not reached");
  }

  // 回调 Transform Plugin 的状态机
  eventProcessor.schedule_imm(this, ET_NET);
}
```

`do_io_shutdown` 操作只是上游告知下游，已经没有更多数据了，作为下游可以继续将缓冲区内剩余的数据消费完，以及向下游的下游传递这些数据，甚至可以自己产生数据发送给下游。

`do_io_shutdown` 操作与 `do_io_close` 不同，不是一次性传递到 TVC 管道的每一节。

## TransformTerminus::do\_io\_shutdown

作为 TVC 管道最后一节的 TransformTerminus 也提供了 `do_io_shutdown` 方法，但是该方法没有触发对 `TransformTerminus::handle_event` 的回调，只有等到 TVC 关闭的时候，才会发现。

```
void
TransformTerminus::do_io_shutdown(ShutdownHowTo_t howto)
{
  if ((howto == IO_SHUTDOWN_READ) || (howto == IO_SHUTDOWN_READWRITE)) {
    m_read_vio.op = VIO::NONE;
    m_read_vio.buffer.clear();
  }

  if ((howto == IO_SHUTDOWN_WRITE) || (howto == IO_SHUTDOWN_READWRITE)) {
    m_write_vio.op = VIO::NONE;
    m_write_vio.buffer.clear();
  }

  // BUG? 没有触发对 TransformTerminus::handle_event 的回调
}
```

## TransformTerminus::handle_event

通过上面的分析，可以知道事件系统将在方法被调用后，回调 `TransformTerminus::handle_event`：

- `TransformVConnection::do_io_close`
  - `TransformTerminus::do_io_close`
- `TransformVConnection::do_io_write`
  - `TransformTerminus::do_io_write`
- `TransformVConnection::do_io_read`
  - `TransformTerminus::do_io_read`
- 最后一个 Transform Plugin 或 HttpTunnel 调用 VIO::reenable
  - `TransformTerminus::reenable`

因此，在实现 `handle_event` 时需要考虑以下五种情况：

- TVC 关闭
- 有数据需要接收
- 通知 HttpSM 来接收数据
- 有数据需要发送
- 异常处理

下面是代码解析：

```
int
TransformTerminus::handle_event(int event, void * /* edata ATS_UNUSED */)
{
  int val;

  // 作为 TVC 管道的最后一节，handle_event 集成了 Transform Plugin 的功能，
  // 因此与 Transform Plugin 一样，在进入到 handle_event 时，第一步就要检测 TransformTerminus 是否已经关闭
  // 如果 TransformTerminus 与 TVC 都已经关闭，那么就设置 m_deletable 为 true 表示可以进行资源回收
  m_deletable = ((m_closed != 0) && (m_tvc->m_closed != 0));

  // 每次通过事件系统调度 TransformTerminus 之前原子递增，成功回调之后原子递减
  // val 为递减之前的值
  val = ink_atomic_increment((int *)&m_event_count, -1);

  Debug("transform", "[TransformTerminus::handle_event] event_count %d", m_event_count);

  if (val <= 0) {
    ink_assert(!"not reached");
  }

  // 如果递减之前的值为 1，那么表示这是最后一个对 TransformTerminus 进行回调的事件，
  // 对于已经完全关闭的 TVC 来说，将来不会再有任何 Event 会回调 TransformTerminus，
  // 那么，就真的可以释放 TVC 所占用的资源了
  m_deletable = m_deletable && (val == 1);

  if (m_closed != 0 && m_tvc->m_closed != 0) {
    // 优化：应该把对 m_deletable 的计算放在这里
    // 对于没有关闭的情况下，计算 m_deletable 是没有意义的
    if (m_deletable) {
      Debug("transform", "TransformVConnection destroy [0x%lx]", (long)m_tvc);
      // 如果满足条件，就释放 TVC 占用的内存
      delete m_tvc;
      return 0;
    }
    // 不满足条件，通常是还有 Event 会回调 TransformTerminus，因此什么也不做，继续等待就可以了

  // 最后一个 Transform Plugin 向 TransformTerminus 写入了数据
  } else if (m_write_vio.op == VIO::WRITE) {
    // 但是还没有任何状态机想要从 TVC 读取数据
    if (m_read_vio.op == VIO::NONE) {
      // 如果还没有通知 TVC 的使用者："TVC 已经有数据可以被读取了"
      if (!m_called_user) {
        Debug("transform", "TransformVConnection calling user: %d %d [0x%lx] [0x%lx]", m_event_count, event, (long)m_tvc,
              (long)m_tvc->m_cont);

        // 向 TVC 的使用者 HttpSM 发送 TRANSFORM_READ_READY 事件通知，
        // 同时标记 TVC 已经向使用者发送了数据准备好的通知。
        m_called_user = 1;
        // It is our belief this is safe to pass a reference, i.e. its scope
        // and locking ought to be safe across the lifetime of the continuation.
        m_tvc->m_cont->handleEvent(TRANSFORM_READ_READY, (void *)&m_write_vio.nbytes);
      }
      // 如果已经向使用者发送了通知，那么就静静的等待使用者调用 TVC::do_io_read

    // 使用者已经发起了读取数据的操作，
    // 那么就准备从 m_write_vio 缓冲区把数据搬到 m_read_vio 缓冲区
    } else {
      int64_t towrite;

      // 对 m_write_vio 和 m_read_vio 的状态机上锁
      MUTEX_TRY_LOCK(trylock1, m_write_vio.mutex, this_ethread());
      if (!trylock1.is_locked()) {
        RETRY();
      }

      // 此处上锁通常会成功，
      // 因为：
      //   - TVC 与其使用者共享一把锁
      //   - TVC 的使用者通常为 HttpSM
      //   - TVC 的读取者通常为 HttpTunnel
      //   - HttpTunnel 与 HttpSM 共享一把锁
      //
      // 所以在回调进入到 handle_event 之后，这里就已经被当前线程上锁了
      MUTEX_TRY_LOCK(trylock2, m_read_vio.mutex, this_ethread());
      if (!trylock2.is_locked()) {
        RETRY();
      }
      // BUG? 上锁完成之后，没有判断锁是否发生改变（按照目前 TVC 的应用代码来看，不会发生这种改变）
      // 只能说明这里的代码质量有欠缺，目前不会造成异常

      // 在上锁后，重新判断 m_closed
      // 如果 TransformTerminus 关闭，则立即返回
      // 等待 do_io_close 触发的对 TransformTerminus::handle_event 的再次回调，以处理 TVC 关闭的状况
      if (m_closed != 0) {
        return 0;
      }

      // 最后一个 Transform Plugin 是否在完成上锁前对 TransformTerminus 执行了 do_io_shutdown 或 do_io_close
      // 这个操作是否可以在对 m_write_vio.mutex 上锁之后立即进行判定？可以减少一次对 m_read_vio.mutex 上锁的尝试
      // 一端调用 shutdown 是否应该向另外一端回调 EOS 事件？
      if (m_write_vio.op == VIO::NONE) {
        return 0;
      }
      // 为何没有判断 m_read_vio.op ?

      // 计算出 VIO 里计划要传输的数据
      // 可能是一个特定的值，也可能是 INT64_MAX
      towrite = m_write_vio.ntodo();
      if (towrite > 0) {
        // 修正 towrite 为缓冲区内可传输的数据长度
        if (towrite > m_write_vio.get_reader()->read_avail()) {
          towrite = m_write_vio.get_reader()->read_avail();
        }
        // 如果超过了计划要接收的数据长度，则修正为计划要接收的数据长度
        if (towrite > m_read_vio.ntodo()) {
          towrite = m_read_vio.ntodo();
        }

        // 经过上述修正后，如果要传输的数据长度仍然大于 0 则从 m_write_vio 向 m_read_vio 传输数据，
        // 更新 m_read_vio.ndone 和 m_write_vio.ndone 标记已经完成传输的数据量
        if (towrite > 0) {
          m_read_vio.get_writer()->write(m_write_vio.get_reader(), towrite);
          m_read_vio.ndone += towrite;

          m_write_vio.get_reader()->consume(towrite);
          m_write_vio.ndone += towrite;
        }
      }

      // 根据 m_write_vio.ntodo() 判断应该向最后一个 Transform Plugin 回调哪种事件
      if (m_write_vio.ntodo() > 0) {
        if (towrite > 0) {
          // 如果本次发生了数据传输，并且仍然有继续传输数据的计划，
          // 则回调 VC_EVENT_WRITE_READY，让 Plugin 向缓冲区内填充数据
          m_write_vio._cont->handleEvent(VC_EVENT_WRITE_READY, &m_write_vio);
        }
        // 如果本次没有发生数据传输，
        // 此时通常是由于 m_read_vio 的缓冲区已经满了，无法再接收数据导致，
        // 因此，即使仍然有继续传输数据的计划，也不再重复回调 Plugin
      } else {
        // 如果未来没有数据传输计划，则回调 VC_EVENT_WRITE_COMPLETE
        m_write_vio._cont->handleEvent(VC_EVENT_WRITE_COMPLETE, &m_write_vio);
      }

      // 回调可能导致 TVC 的关闭，这里重复进行检查
      // BUG? 判断条件是否应该改为 || ?
      // We could have closed on the write callback
      if (m_closed != 0 && m_tvc->m_closed != 0) {
        return 0;
      }

      // 根据 m_read_vio.ntodo() 判断应该向 HttpTunnel 回调哪种事件
      if (m_read_vio.ntodo() > 0) {
        if (m_write_vio.ntodo() <= 0) {
          // 如果计划读取更多数据，但是没有后续的数据传输计划，
          // 则回调 EOS，表示 TVC 数据传输工作完成了，可以进行关闭了
          m_read_vio._cont->handleEvent(VC_EVENT_EOS, &m_read_vio);
        } else if (towrite > 0) {
          // 如果计划读取更多数据，后续仍然有数据传输计划，并且本次也发生了数据传输，
          // 则回调 VC_EVENT_READ_READY，让 HttpTunnel 继续接收数据
          m_read_vio._cont->handleEvent(VC_EVENT_READ_READY, &m_read_vio);
        }
        // 如果计划读取更多数据，后续也有数据传输计划，但是本次没有发生数据传输，
        // 此时通常由于 m_read_vio 的缓冲区已经满了，无法再接收数据导致，
        // 因此，即使仍然有继续传输数据的计划，也不再重复回调 HttpTunnel
      } else {
        // 如果没有计划读取更多数据，则回调 VC_EVENT_READ_COMPLETE
        m_read_vio._cont->handleEvent(VC_EVENT_READ_COMPLETE, &m_read_vio);
      }
    }
  } else {
    // 最后一个 Transform Plugin 没有向 TransformTerminus 写入数据
    // 此时 TVC 可能没有关闭
    // 也有可能处于半关闭状态：
    //   - TransformVConnection 未关闭，TransformTerminus 已经关闭
    //   - Transform Plugin 可能调用了 TransformTerminus::do_io_close
    //   - 我个人认为这种情况是不应该发生的，稍后解析

    // 下面的代码尝试检查 TVC 是否处于半关闭的状态，
    // 如果 TransformTerminus 已经关闭，则向 TVC 的使用者 或 m_read_vio 回调相应的事件

    // 未确定回调对象就提前上锁？
    // 对 m_read_vio 的状态机上锁，以便于接下来的回调
    MUTEX_TRY_LOCK(trylock2, m_read_vio.mutex, this_ethread());
    if (!trylock2.is_locked()) {
      RETRY();
    }

    // TVC 处于半关闭的状态（TransformTerminus 已经关闭）
    if (m_closed != 0) {
      // The terminus was closed, but the enclosing transform
      // vconnection wasn't. If the terminus was aborted then we
      // call the read_vio cont back with VC_EVENT_ERROR. If it
      // was closed normally then we call it back with
      // VC_EVENT_EOS. If a read operation hasn't been initiated
      // yet and we haven't called the user back then we call
      // the user back instead of the read_vio cont (which won't
      // exist).
      // 确认 TVC 处于半关闭状态（TransformVConnection 没有关闭）
      if (m_tvc->m_closed == 0) {
        // 如果调用 TransformTerminus::do_io_close 时传入的 error != -1 表示异常关闭，
        // 就将回调的事件设置为 VC_EVENT_ERROR，
        // 否则就将回调的事件设置为 VC_EVENT_EOS。
        int ev = (m_closed == TS_VC_CLOSE_ABORT) ? VC_EVENT_ERROR : VC_EVENT_EOS;

        if (!m_called_user) {
          // 如果还未向 TVC 的使用者发送过 TRANSFORM_READ_READY 事件通知，
          // 那么就向其发送 VC_EVENT_ERROR 或 VC_EVENT_EOS 事件
          // BUG: HttpSM 用于接收 TRANSFORM_READ_READY 回调的是：
          //   - HttpSM::state_request_wait_for_transform_read()
          //   - HttpSM::state_response_wait_for_transform_read()
          // 对于非 TRANSFORM_READ_READY 的事件，都会透传给：
          //   - HttpSM::state_common_wait_for_transform_read()
          // 而该回调函数是不能处理 VC_EVENT_EOS 事件的。
          m_called_user = 1;
          m_tvc->m_cont->handleEvent(ev, NULL);
        } else {
          // 已经向 TVC 的使用者发送过 TRANSFORM_READ_READY 事件通知，
          // 这里假定 TVC 的使用者收到了 TRANSFORM_READ_READY 事件后就立即调用了 TVC::do_io_read
          ink_assert(m_read_vio._cont != NULL);
          // 向 m_read_vio 发送 VC_EVENT_ERROR 或 VC_EVENT_EOS 事件
          // 用于接收这两个事件的是 HttpTunnel::producer_handler()
          m_read_vio._cont->handleEvent(ev, &m_read_vio);
        }
      }
      // 如果 TVC 完全关闭，则直接返回，等待下一次回调进行处理
      return 0;
    }
  }

  return 0;
}
```

## 为什么 TVC 不能够处于半关闭状态？

TVC 会进入到半关闭状态，是因为 TVC 管道中间的某一节向其下游调用了 `do_io_close` 操作，此节之后的部分全都被关闭了，也包括最后一节 TransformTerminus。

那么这么做了之后，有什么后果？

### TVC 管道可能会破裂

如果向下游发起 `do_io_close` 操作的这一个 Plugin 自身也关闭退出了，那么其上游是无法知道下游已经关闭了的事实。
而且其上游的 INKVConnInternal 内的 `m_output_vc` 仍然指向了这个已经关闭，并且内存空间可能已经被回收的地址。最后在 `TVC::do_io_close` 调用时，终究会将 `do_io_close` 传递到这一节的 INKVConnInternal，而此时 `m_output_vc` 已经指向了一个非法的内存地址，必然会导致进程崩溃。

如果这个 Plugin 自身一直保持不关闭，那么仍然可以保持 TVC 管道不破裂，此时该 Plugin 要持续接收来自上游的数据，但是不再向下游输出数据（下游已经被关闭了）。

但是这么做的意义何在呢？不如使用 `do_io_shutdown` 替代 `do_io_close` 来的稳妥。


## 参考资料

- [PluginVC.h](http://github.com/apache/trafficserver/tree/6.0.x/proxy/TransformInternal.h)
- [PluginVC.cc](http://github.com/apache/trafficserver/tree/6.0.x/proxy/Transform.cc)
- [Pull Request #2907](https://github.com/apache/trafficserver/pull/2907)
