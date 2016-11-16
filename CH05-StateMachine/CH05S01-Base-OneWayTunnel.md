# 基础组件：OneWayTunnel

OneWayTunnel 是使用 IOCore 网络子系统的 API 实现的一个完整的状态机。对于理解和学习如何使用iocore系统进行开发，这是一个很好的例子。

OneWayTunnel 实现了一个简单的数据传输功能，它从一个VC读取数据然后将数据写入另外一个VC。

## 定义

```
//////////////////////////////////////////////////////////////////////////////
//
//      OneWayTunnel
//
//////////////////////////////////////////////////////////////////////////////

#define TUNNEL_TILL_DONE INT64_MAX

#define ONE_WAY_TUNNEL_CLOSE_ALL NULL

typedef void (*Transform_fn)(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);

/**
  A generic state machine that connects two virtual conections. A
  OneWayTunnel is a module that connects two virtual connections, a source
  vc and a target vc, and copies the data between source and target. Once
  the tunnel is started using the init() call, it handles all the events
  from the source and target and optionally calls a continuation back when
  its done. On success it calls back the continuation with VC_EVENT_EOS,
  and with VC_EVENT_ERROR on failure.

  If manipulate_fn is not NULL, then the tunnel acts as a filter,
  processing all data arriving from the source vc by the manipulate_fn
  function, before sending to the target vc. By default, the manipulate_fn
  is set to NULL, yielding the identity function. manipulate_fn takes
  a IOBuffer containing the data to be written into the target virtual
  connection which it may manipulate in any manner it sees fit.
*/
```

上述注释翻译如下：
OneWayTunnel是连接两个VConnection的通用状态机。它可以作为一个模块，连接源VC和目的VC，将源VC读取到的数据写入目的VC。
一旦通过init()启动Tunnel之后，它将接受所有来自源VC和目的VC的事件，可以在操作完成后有选择的回调指定的状态机：

- 如果成功，回调时传递VC_EVENT_EOS状态
- 如果失败，回调时传递VC_EVENT_ERROR状态

通过设置 manipulate_fn 函数指针（不为 NULL），可以将Tunnel变成一个过滤器（filter）

- 它将调用manipulate_fn来处理所有来自源VC的数据，然后发送给目的VC。
- Tunnel会传递给 manipulate_fn 一个包含将要写入目标VC数据的 IOBuffer。
- manipulate_fn 可以以任意的方式处理其中包含的数据。
- 不过 manipulate_fn 的默认值通常为 NULL。

```
struct OneWayTunnel : public Continuation {
  //
  //  Public Interface
  //

  //  Copy nbytes from vcSource to vcTarget.  When done, call
  //  aCont back with either VC_EVENT_EOS (on success) or
  //  VC_EVENT_ERROR (on error)
  //

  // Use these to construct/destruct OneWayTunnel objects

  /**
    Allocates a OneWayTunnel object.
    静态方法，用来创建 OneWayTunnel 对象

    @return new OneWayTunnel object.

  */
  static OneWayTunnel *OneWayTunnel_alloc();

  /** Deallocates a OneWayTunnel object. 
    静态方法，用来释放 OneWayTunnel 对象
  */
  static void OneWayTunnel_free(OneWayTunnel *);

  // 设置 TwoWayTunnel，将两个 OneWayTunnel 关联起来
  static void SetupTwoWayTunnel(OneWayTunnel *east, OneWayTunnel *west);
  // 构造函数
  OneWayTunnel();
  // 析构函数
  virtual ~OneWayTunnel();

  // Use One of the following init functions to start the tunnel.
  // init函数有多重类型，请根据需求选择恰当的init函数启动tunnel
  /**
    第一种 init() 方法，下面会有详细的介绍和分析
    This init function sets up the read (calls do_io_read) and the write
    (calls do_io_write).

    @param vcSource source VConnection. A do_io_read should not have
      been called on the vcSource. The tunnel calls do_io_read on this VC.
    @param vcTarget target VConnection. A do_io_write should not have
      been called on the vcTarget. The tunnel calls do_io_write on this VC.
    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling back
      anybody. Otherwise, its the callee's responsibility to deallocate
      the tunnel with OneWayTunnel_free.
    @param size_estimate size of the MIOBuffer to create for
      reading/writing to/from the VC's.
    @param aMutex lock that this tunnel will run under. If aCont is
      specified, the Continuation's lock is used instead of aMutex.
    @param nbytes number of bytes to transfer.
    @param asingle_buffer whether the same buffer should be used to read
      from vcSource and write to vcTarget. This should be set to true in
      most cases, unless the data needs be transformed.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.
    @param manipulate_fn if specified, the tunnel calls this function
      with the input and the output buffer, whenever it gets new data
      in the input buffer. This function can transform the data in the
      input buffer
    @param water_mark watermark for the MIOBuffer used for reading.

  */
  void init(VConnection *vcSource, VConnection *vcTarget, Continuation *aCont = NULL, int size_estimate = 0, // 0 = best guess
            ProxyMutex *aMutex = NULL, int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, bool aclose_source = true,
            bool aclose_target = true, Transform_fn manipulate_fn = NULL, int water_mark = 0);

  /**
    第二种 init() 方法，下面会有详细的介绍和分析
    This init function sets up only the write side. It assumes that the
    read VConnection has already been setup.

    @param vcSource source VConnection. Prior to calling this
      init function, a do_io_read should have been called on this
      VConnection. The tunnel uses the same MIOBuffer and frees
      that buffer when the transfer is done (either successful or
      unsuccessful).
    @param vcTarget target VConnection. A do_io_write should not have
      been called on the vcTarget. The tunnel calls do_io_write on
      this VC.
    @param aCont The Continuation to call back when the tunnel
      finishes. If not specified, the tunnel deallocates itself without
      calling back anybody.
    @param SourceVio VIO of the vcSource.
    @param reader IOBufferReader that reads from the vcSource. This
      reader is provided to vcTarget.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.
  */
  void init(VConnection *vcSource, VConnection *vcTarget, Continuation *aCont, VIO *SourceVio, IOBufferReader *reader,
            bool aclose_source = true, bool aclose_target = true);

  /**
    第三种 init() 方法，下面会有详细的介绍和分析
    Use this init function if both the read and the write sides have
    already been setup. The tunnel assumes that the read VC and the
    write VC are using the same buffer and frees that buffer
    when the transfer is done (either successful or unsuccessful)
    @param aCont The Continuation to call back when the tunnel finishes. If
    not specified, the tunnel deallocates itself without calling back
    anybody.

    @param SourceVio read VIO of the Source VC.
    @param TargetVio write VIO of the Target VC.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.

    */
  void init(Continuation *aCont, VIO *SourceVio, VIO *TargetVio, bool aclose_source = true, bool aclose_target = true);

  //
  // Private
  //
  // 这种方法目前不被支持，该函数中存在assert
  OneWayTunnel(Continuation *aCont, Transform_fn manipulate_fn = NULL, bool aclose_source = false, bool aclose_target = false);

  // 主状态处理函数
  int startEvent(int event, void *data);

  // 将数据从 in_buf 传送到 out_buf
  // 由于只支持single buffer，所以实际上这个函数没有任何意义
  virtual void transform(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);

  /** Result is -1 for any error. */
  // 关闭源VC，释放MIOBuffer
  void close_source_vio(int result);

  // 关闭目标VC，释放MIOBuffer
  virtual void close_target_vio(int result, VIO *vio = ONE_WAY_TUNNEL_CLOSE_ALL);

  // 关闭 Tunnel
  // 直接释放 Tunnel 对象，或回调指定状态机，由状态机释放 Tunnel 对象
  void connection_closed(int result);

  // 同时激活源VC和目标VC
  // 没有看到任何调用的示例，不清楚具体功能
  virtual void reenable_all();

  // true 表示当前tunnel只剩下最后一个VC通道了。
  bool last_connection();

  // 保存源VC的vio
  VIO *vioSource;
  // 保存目标VC的vio
  VIO *vioTarget;
  Continuation *cont;
  // 用于实现在源buf和目标buf之间传递数据，但是目前由于只支持single buffer模式，所以没有任何意义
  Transform_fn manipulate_fn;
  // 当前Tunnel持有的通道（VC）数量，可能的值为：0，1，2
  int n_connections;
  // 来自出现错误的 VC 上的错误代码
  int lerrno;

  // 源buf与目标buf是否为同一个buf，目前只能设置为true，否则会抛出异常
  bool single_buffer;
  // 初始化时传入，表示在完成tunnel时，是否关闭源VC和目标VC
  bool close_source;
  bool close_target;
  // 为true表示一直到源VC关闭（EOS）才结束Tunnel
  bool tunnel_till_done;

  /** Non-NULL when this is one side of a two way tunnel. */
  // 在TwowayTunnel中介绍
  OneWayTunnel *tunnel_peer;
  bool free_vcs;

private:
  OneWayTunnel(const OneWayTunnel &);
  OneWayTunnel &operator=(const OneWayTunnel &);
};
```

## 方法

### 第一种 init() 方法

```
void init(VConnection *vcSource, VConnection *vcTarget, 
         Continuation *aCont = NULL, int size_estimate = 0, // 0 = best guess
         ProxyMutex *aMutex = NULL, 
         int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, 
         bool aclose_source = true, bool aclose_target = true, 
         Transform_fn manipulate_fn = NULL, int water_mark = 0);
```

init函数，将在源VC上设置read（调用do_io_read），在目标VC上设置write（调用do_io_write），自动创建一个MIOBuffer在内部使用，并在完成后释放buf。参数说明如下：

- vcSource
   - 源VC
   - Tunnel将会在这个VC上调用do_io_read，所以，如果之前在这个VC上调用过do_io_read，可能会失效，或者出现其它问题。
- vcTarget
   - 目标VC
   - Tunnel将会在这个VC上调用do_io_write，所以，如果之前在这个VC上调用过do_io_write，可能会失效，或者出现其它问题。
- aCont = NULL
   - 当Tunnel完成时，将会回调这个Continuation。
   - 如果未指定（NULL），Tunnel会直接释放自身。
   - 否则，回调函数应该调用OneWayTunnel_free()来完成Tunnel的释放。
- size_estimate = 0
   - MIOBuffer的长度，它作为在VC之间读写数据的缓冲区。
   - 设置为0时，表示自适应，在init内设置为default_large_iobuffer_size。
- aMutex = NULL
   - Tunnel运行的锁。
   - 如果指定了aCont，那么aMutex＝aCont->mutex。
- nbytes = TUNNEL_TILL_DONE = INT64_MAX
   - 被传送的字节数。
- asingle_buffer = true
   - 从源VC读取数据使用的buf，是否与写入目标VC的buf为同一个。
   - 大多数情况下为同一个（true），除非manipulate_fn不为NULL时，才可以设为false
   - 当设置为false时，init方法中会额外创建一个MIOBuffer，同时在 manipulate_fn 内需要进行数据的搬移
   - 如果未设置manipulate_fn而将此值设置为false，将抛出异常，因为transfer_data的实现不完整，官方屏蔽了这个函数。
- aclose_source = true
   - 在Tunnel完成时，是否关闭源VC，默认关闭（true）
- aclose_target = true
   - 在Tunnel完成时，是否关闭目的VC，默认关闭（true）
- manipulate_fn = NULL
   - 不为空时，每当Tunnel获得新数据时，都会调用此函数，并以输入buf和输出buf作为参数。
   - 可用于在Tunnel传输中对数据buf中的数据进行修改后发送给目的VC
   - 定义：void (*Transform_fn)(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf);
- water_mark = 0
   - 用于设置源VC使用的MIOBuffer的watermark
   - 更多信息可参考 IOBuffer 章节

### 第二种 init() 方法

```
void init(VConnection *vcSource, VConnection *vcTarget, 
          Continuation *aCont, 
          VIO *SourceVio, IOBufferReader *reader,
          bool aclose_source = true, bool aclose_target = true);
```

init函数，只在目标VC上设置write（调用do_io_write），需要确保源VC已经完成read设置（调用do_io_read），如果没有aCont传入，则会new一个mutex出来使用，Tunnel针对源VC的读和目的VC的写使用同一个buf，就是传入的 reader，并在完成后释放buf。参数说明如下：

- SourceVio
   - 源VC的VIO
   - 通过调用 SourceVio->set_continuation(this) 切换状态机
- reader
   - 从源VC的MIOBuffer读取数据的IOBufferReader，它将传递给目的VC作为数据的输入源

注意：虽然这里没有为源VC设置read（do_io_read），但是调用了 SourceVio->set_continuation(this)，这样当源VC有数据进来的时候，就会调用OneWayTunnel::startEvent来处理读取到的数据了。使用这种方式的调用可以沿用之前的IOBuffer，特别是之前的IOBuffer内已经有数据的情况下。

### 第三种 init() 方法

```
void init(Continuation *aCont, VIO *SourceVio, VIO *TargetVio, 
          bool aclose_source = true, bool aclose_target = true);
```
- SourceVio
   - 源VC的VIO
   - 通过调用 SourceVio->set_continuation(this) 切换状态机
- TargetVio
   - 目标VC的VIO
   - 通过调用 TargetVio->set_continuation(this) 切换状态机

这样调用init函数，表示源VC的读和目的VC的写都已经设置完成，如果没有aCont传入，则会new一个mutex出来使用，Tunnel针对源VC的读和目的VC的写，调用者需要保证，源VC和目的VC都使用同一个buf（如果不是同一个buf，目前的代码会抛出异常），并在完成后释放buf。

### 缺省manipulate_fn的内部实现：transfer_data

在代码中有一个不完整的内部 manipulate_fn 实现，可以用来参考。

注意：这个transfer_data函数内有一个assert，标记为“功能未完全实现”，因此不可以调用。

由于在OneWayTunnel::transform里判定源buf与目标buf的地址不同时，会调用transfer_data函数。
所以，可以认定 OneWayTunnel 不支持 single_buffer = false 的情况。

```
inline void
transfer_data(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf)
{
  ink_release_assert(!"Not Implemented.");

  int64_t n = in_buf.reader()->read_avail();
  int64_t o = out_buf.writer()->write_avail();

  if (n > o)
    n = o;
  if (!n)
    return;
  memcpy(in_buf.reader()->start(), out_buf.writer()->end(), n);
  in_buf.reader()->consume(n);
  out_buf.writer()->fill(n);
}

void
OneWayTunnel::transform(MIOBufferAccessor &in_buf, MIOBufferAccessor &out_buf)
{
  if (manipulate_fn)
    manipulate_fn(in_buf, out_buf);
  // 这里调用writer()方法获取两个buf的MIOBuffer缓冲区的指针
  // 如果相等，说明asingle_buffer为true
  // 如果不相等，就需要调用transfor_data实现两个buffer之间的数据copy
  // 但是transfer_data的实现可能有问题，官方屏蔽了它一旦调用就会导致异常抛出。
  else if (in_buf.writer() != out_buf.writer())
    transfer_data(in_buf, out_buf);
}
```

### connection_closed 函数

BUG：调用回调函数时，传递的第二个参数cont，应该是this，这样才能把Tunnel的实例传入回调函数内，然后由回调函数调用OneWayTunnel_free释放Tunnel。

```
void
OneWayTunnel::connection_closed(int result)
{
  if (cont) {
#ifdef TEST
    cout << "OneWayTunnel::connection_closed() ... calling cont" << endl;
#endif
    cont->handleEvent(result ? VC_EVENT_ERROR : VC_EVENT_EOS, cont); // 最后的 cont 应该为 this
  } else {
    OneWayTunnel_free(this);
  }
}

```

### startEvent 流程分析

在init调用之后，Tunnel的回调函数被设置为OneWayTunnel::startEvent，其内部逻辑如下：

```
当 VC_EVENT_READ_READY
    // 由于只在源VC执行了do_io_read，所以只有源VC有数据可读时，才会触发此事件
    transform(vioSource->buffer, vioTarget->buffer);
    // 激活目标VC，使其完成写入操作
    vioTarget->reenable();
    ret = VC_EVENT_CONT;

当 VC_EVENT_WRITE_READY
    // 由于只在目的VC执行了do_io_write，所以只有目的VC缓冲区可写时，才会触发此事件
    // 但是由于是Tunnel，当目的VC可写时，说明目标VC的VIO有空闲空间，
    //     此时就要激活源VC来提供数据，以灌满目的VC的写缓冲区
    // 所以判断源VC可用，就激活源VC，使其读取数据
    if (vioSource) vioSource->reenable();
    ret = VC_EVENT_CONT;

当 VC_EVENT_EOS
    // 通常表示连接关闭
    // 判断是哪一端关闭了连接，如果是源VC，那么把buf内的数据写完，然后按照VC_EVENT_READ_COMPLETE来处理
    if (vio == vioSource) {
        transform(vioSource->buffer, vioTarget->buffer);
        goto Lread_complete;
    // 如果是目的VC关闭了，那么就啥都不用管，就当Tunnel整个都结束/完成了
    } else goto Ldone;

Lread_complete:
当 VC_EVENT_READ_COMPLETE
    // set write nbytes to the current buffer size
    // 计算实际的目标VC需要完成的写入字节数 ＝ 前面已经完成的 ＋ 缓冲区读到的新数据
    vioTarget->nbytes = vioTarget->ndone + vioTarget->buffer.reader()->read_avail();
    // 如果目标VC的数据也同时写完了，那就整个Tunnel也结束/完成了
    if (vioTarget->nbytes == vioTarget->ndone)
        goto Ldone;
    // 如果还没完成，激活目标VC完成写操作
    vioTarget->reenable();
    // 如果没有使用SetupTwoWayTunnel设置关联Tunnel，那么可以先关闭源VC
    // 传递给close_source_vio参数0表示传递给do_io_close的lerrno参数为-1（代表正常关闭）
    if (!tunnel_peer) close_source_vio(0);

Lerror:
当 VC_EVENT_ERROR
    lerrno = ((VIO *)data)->vc_server->lerrno;
当 VC_EVENT_INACTIVITY_TIMEOUT:
当 VC_EVENT_ACTIVE_TIMEOUT:
    result = -1;
    // 这里把 ERROR 和 TIMEOUT 都归为错误，并认为 Tunne 处理完成
Ldone:
当 VC_EVENT_WRITE_COMPLETE
    // 这时，表示Tunnel整个都结束/完成了
    // 如果使用SetupTwoWayTunnel设置了关联Tunnel，那么还要向关联Tunnel发送事件，通知其关闭
    if (tunnel_peer) tunnel_peer->startEvent(ONE_WAY_TUNNEL_EVENT_PEER_CLOSE, data);
    // 需要根据init的设置，关闭源VC，目标VC
    close_source_vio(result);
    close_target_vio(result);
    // 负责释放Tunnel，如果在init时传入了aCont，则负责回调aCont，aCont将负责释放Tunnel
    connection_closed(result);

```

## 参考资料

- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
