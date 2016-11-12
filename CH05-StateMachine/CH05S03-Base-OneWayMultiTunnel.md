# 基础组件：OneWayMultiTunnel

直接翻译 I_OneWayMultiTunnel.h 中的一段注释，如下：

OneWayMultiTunnel是连接一个源VC和多个目标VC的通用状态机，它与OneWayTunnel非常相似。
但是，它连接了多个目标VC，将源VC读取到的数据写入所有目的VC。
宏定义ONE_WAY_MULTI_TUNNEL_LIMIT，限定了目标VC的最大数量。

其它请参考 OneWayTunnel

## 定义

```
/** Maximum number which can be tunnelled too */
// 由于每一个目标VC都需要从MIOBuffer分配一个 IOBufferReader，而源VC也使用了一个IOBufferReader。
// 所以，这里定义的目标VC数量的最大值要比 IOBufferReader 的最大值少一个（被源VC使用）
// 在 I_IOBuffer.h 中定义了一个 MIOBuffer 可分配的 IOBufferReader 的最大值为 5
//     #define MAX_MIOBUFFER_READERS 5
// 所以这里只能定义为 4，如果需要更多的目标VC，需要同时修改 IOBufferReader 的最大值。
#define ONE_WAY_MULTI_TUNNEL_LIMIT 4

// 继承自 OneWayTunnel
struct OneWayMultiTunnel : public OneWayTunnel {
  //
  // Public Interface
  //

  // Use these to construct/destruct OneWayMultiTunnel objects

  /**
    Allocates a OneWayMultiTunnel object.
    静态方法，用来创建 OneWayMultiTunnel 对象

    @return new OneWayTunnel object.

  */
  static OneWayMultiTunnel *OneWayMultiTunnel_alloc();

  /**
    Deallocates a OneWayTunnel object.
    静态方法，用来释放 OneWayMultiTunnel 对象
  */
  static void OneWayMultiTunnel_free(OneWayMultiTunnel *);
  // 构造函数
  OneWayMultiTunnel();

  // Use One of the following init functions to start the tunnel.
  // init函数有多重类型，请根据需求选择恰当的init函数启动tunnel
  /**
    第一种 init() 方法，下面会有详细的介绍和分析
    This init function sets up the read (calls do_io_read) and the write
    (calls do_io_write).

    @param vcSource source VConnection. A do_io_read should not have
      been called on the vcSource. The tunnel calls do_io_read on this VC.
    @param vcTargets array of Target VConnections. A do_io_write should
      not have been called on any of the vcTargets. The tunnel calls
      do_io_write on these VCs.
    @param n_vcTargets size of vcTargets.
    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling
      back anybody.
    @param size_estimate size of the MIOBuffer to create for reading/
      writing to/from the VC's.
    @param nbytes number of bytes to transfer.
    @param asingle_buffer whether the same buffer should be used to read
      from vcSource and write to vcTarget. This should be set to true
      in most cases, unless the data needs be transformed.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target if true, the tunnel closes vcTarget at the end.
      If aCont is not specified, this should be set to true.
    @param manipulate_fn if specified, the tunnel calls this function
      with the input and the output buffer, whenever it gets new data
      in the input buffer. This function can transform the data in the
      input buffer.
    @param water_mark for the MIOBuffer used for reading.

  */
  void init(VConnection *vcSource, VConnection **vcTargets, int n_vcTargets, Continuation *aCont = NULL,
            int size_estimate = 0, // 0 == best guess
            int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, bool aclose_source = true, bool aclose_target = true,
            Transform_fn manipulate_fn = NULL, int water_mark = 0);

  /**
    第二种 init() 方法，下面会有详细的介绍和分析
    Use this init function if both the read and the write sides have
    already been setup. The tunnel assumes that the read VC and the
    write VCs are using the same buffer and frees that buffer when the
    transfer is done (either successful or unsuccessful).

    @param aCont continuation to call back when the tunnel finishes. If
      not specified, the tunnel deallocates itself without calling back
      anybody.
    @param SourceVio read VIO of the Source VC.
    @param TargetVios array of write VIOs of the Target VCs.
    @param n_vioTargets size of TargetVios array.
    @param aclose_source if true, the tunnel closes vcSource at the
      end. If aCont is not specified, this should be set to true.
    @param aclose_target ff true, the tunnel closes vcTarget at the
      end. If aCont is not specified, this should be set to true.

  */
  void init(Continuation *aCont, VIO *SourceVio, VIO **TargetVios, int n_vioTargets, bool aclose_source = true,
            bool aclose_target = true);

  //
  // Private
  //
  // 主状态处理函数
  int startEvent(int event, void *data);

  // 同时激活源VC和目标VC
  // 没有看到任何调用的示例，不清楚具体功能
  virtual void reenable_all();
  // 如果vio==NULL，关闭所有目标VIO及VC
  // 否则，只关闭指定的VIO及对应的VC
  virtual void close_target_vio(int result, VIO *vio = NULL);

  // 表示当前管理的目标VC的数量，默认为0
  int n_vioTargets;
  // 数组，保存了每一个目标VC对应的Write VIO
  VIO *vioTargets[ONE_WAY_MULTI_TUNNEL_LIMIT];
  // 设置为写入目标VC MIOBuffer的封装
  MIOBufferAccessor topOutBuffer;
  // 默认为false，在进入到init函数之前可能Source VIO就已经完成了（ntodo == 0）
  bool source_read_previously_completed;
};
```

## 方法

### 第一种 init() 方法

```
void init(VConnection *vcSource, VConnection **vcTargets, int n_vcTargets, 
         Continuation *aCont = NULL, int size_estimate = 0, // 0 == best guess
         int64_t nbytes = TUNNEL_TILL_DONE, bool asingle_buffer = true, 
         bool aclose_source = true, bool aclose_target = true,
         Transform_fn manipulate_fn = NULL, int water_mark = 0);
```
init函数，将在源VC上设置read（调用do_io_read），在所有目标VC上设置write（调用do_io_write）。

相对于第一种 OneWayTunnel::init()

- 增加了n_vcTargets用于指定vcTargets的数量
- 砍掉了aMutex，因为不需要为了实现 TwoWayTunnel 而强制指定同一个mutex
- 因此，会优先选用aCont的mutex，如果没有指定aCont，那么就new_ProxyMutex()

需要注意的是，当asingle_buffer设置为false时：

- 源VC独立使用一个buf
- 所有的目标VC是共享同一个buf的，目标VC使用的IOBufferReader都是从该buf分配的

### 第二种 init() 方法

```
void init(Continuation *aCont, VIO *SourceVio, VIO **TargetVios, int n_vioTargets, 
         bool aclose_source = true, bool aclose_target = true);
```
init函数，表示源VC的读和目的VC的写都已经设置完成，如果没有aCont传入，则会new一个mutex出来使用，Tunnel针对源VC的读和目的VC的写，调用者需要保证，源VC和所有的目的VC都使用同一个buf（如果不是同一个buf，目前的代码会抛出异常），并在完成后释放buf。

相对于第三种 OneWayTunnel::init()

- 增加了n_vcTargets用于指定vcTargets的数量。

这种情况下，有可能之前在源VC上的读操作已经完成了，因此在初始化过程中：
```
source_read_previously_completed = (SourceVio->ntodo() == 0);
```

通常在 SourceVC 上收到 READ_COMPLETE 事件后会关闭 SourceVC

- 然后每一个 TargetVC 收到 WRITE_COMPLETE 事件后会关闭 TargetVC
- 最后一个 TargetVC 关闭后，会自动关闭整个 OneWayMultiTunnel

但是，当 source_read_previously_completed 为 true 时，startEvent 不会在SourceVC上接收到VC_EVENT_READ_COMPLETE事件

- 但是每一个 TargetVC 仍然会收到 WRITE_COMPLETE 事件，然后后关闭 TargetVC
- 最后一个 TargetVC 关闭后，只剩下 SourceVC
- 此时判断 source_read_previously_completed 为 true，则直接关闭 SourceVC
- 最后关闭整个 OneWayMultiTunnel

## startEvent 流程分析

在init调用之后，Tunnel的回调函数被设置为OneWayMultiTunnel::startEvent，其内部逻辑如下：

```
当 VC_EVENT_READ_READY
    // 由于只在源VC执行了do_io_read，所以只有源VC有数据可读时，才会触发此事件
    // 调用transform将数据由源VIO转移到临时VIO：topOutBuffer
    transform(vioSource->buffer, topOutBuffer);
    // 激活所有目标VC，使其完成写入操作
    for (int i = 0; i < n_vioTargets; i++)
        if (vioTargets[i])
            vioTargets[i]->reenable();
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
    // 如果不是传输到连接关闭才算任务完成的Tunnel类型，
    // 就需要判断传入的VIO是否还有数据没有完成，如果有，那就是Tunnel出现了错误
    if (!tunnel_till_done && vio->ntodo())
        goto Lerror;
    // 判断是哪一端关闭了连接，如果是源VC，那么把buf内的数据写完，然后按照VC_EVENT_READ_COMPLETE来处理
    if (vio == vioSource) {
        transform(vioSource->buffer, topOutBuffer);
        goto Lread_complete;
    // 如果是目的VC关闭了，那么就按照VC_EVENT_WRITE_COMPLETE来处理
    } else goto Lwrite_complete;

Lread_complete:
当 VC_EVENT_READ_COMPLETE
    // set write nbytes to the current buffer size
    // 处理每一个目标VC
    for (int i = 0; i < n_vioTargets; i++)
        if (vioTargets[i]) {
            // 计算实际的目标VC需要完成的写入字节数 ＝ 前面已经完成的 ＋ 缓冲区读到的新数据
            vioTarget->nbytes = vioTarget->ndone + vioTarget->buffer.reader()->read_avail();
            // 如果还没完成，激活目标VC完成写操作
            vioTarget->reenable();
        }
    // 由于读取操作完成，因此关闭源VC的VIO
    // 传递给close_source_vio参数0表示传递给do_io_close的lerrno参数为-1（代表正常关闭）
    close_source_vio(0);
    ret = VC_EVENT_DONE;

Lwrite_complete:
当 case VC_EVENT_WRITE_COMPLETE
    // 关闭该目标VC的VIO
    close_target_vio(0, (VIO *)data);
    // 判断剩余的通道数量
    //     如果剩余0个通道
    //     或者剩余通道数为1同时source_read_previously_completed为true
    // 备注：当source_read_previously_completed为true时，不会触发VC_EVENT_READ_COMPLETE事件，
    //      因此，源VC无法主动关闭，这里剩余的一个通道就是源VC的读通道。
    //      对比OneWayTunnel，则是以VC_EVENT_WRITE_COMPLETE作为Tunnel完成的唯一标识，
    //      收到VC_EVENT_WRITE_COMPLETE，就表示Tunnel完成，此时直接关闭源VC和目标VC的VIO，以及连接。
    if ((n_connections == 0) || (n_connections == 1 && source_read_previously_completed))
    // 就表示Tunnel整个都结束/完成了
        goto Ldone;
    // 否则，判断源VC可用，就激活源VC，使其读取数据
    // 这种情况下，源VC的VIO应该已经关闭了
    // 如果源VC没有关闭，激活源VC只是为了让源VC触发VC_EVENT_EOS，
    // 触发VC_EVENT_EOS后，又会激活所有可用的目标VC执行写操作。
    else if (vioSource)
      vioSource->reenable();

Lerror:
当 出现错误、超时、或者Tunnel完成后
    result = -1;
Ldone:
    // 需要根据init的设置，关闭源VC，目标VC
    close_source_vio(result);
    close_target_vio(result);
    // 负责释放Tunnel，如果在init时传入了aCont，则负责回调aCont，aCont将负责释放Tunnel
    connection_closed(result);

```

由于在源VC的MIOBuffer与目标VC的MIOBuffer中传递数据时，仍然调用的是继承自 OneWayTunnel 的 transform() 函数，因此 OneWayMultiTunnel 同样也仅支持 single_buffer == true 的情况。

## 适用场景

在 Cache 服务器的设计中，我们从源站拿到数据之后，如果这份数据需要同时发送给客户端和保存到磁盘时，就产生了对 OneWayMultiTunnel 的需求。

- 此时 ServerVC 作为源VC，而 ClientVC 和 CacheVC 作为目标VC
- OneWayMultiTunnel 从 ServerVC 读取数据
- 然后向 ClientVC 和 CacheVC 发送数据
- 当 ClientVC 接收了全部数据的同时，CacheVC 也接收了全部数据并保存到了磁盘

在实现代理服务的合并回源功能时，也同样会需要使用 OneWayMultiTunnel

- Client A 和 Client B 都发起了对 Server 的访问
- Client A 先发起了该请求，代理服务器将请求转发给 Server
- 在 Server 产生响应之前，Client B 的请求到达代理服务器，代理服务器判断可以与 Client A 的请求合并
- 这样以 ServerVC 作为源VC，Client A 和 Client B 作为目标 VC 创建 OneWayMultiTunnel
- Server VC 回应的数据会同时到达 Client A 和 Client B

Tunnel 状态机提供了非常基础的数据流转发功能，在事务信息处理完成后，大多数的状态机都会借助 Tunnel 状态机完成大量数据转发的工作。

## 参考资料

- [I_OneWayMultiTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayMultiTunnel.h)
- [I_OneWayTunnel.h](http://github.com/apache/trafficserver/tree/master/iocore/utils/I_OneWayTunnel.h)
