# 基础组件：NetVConnection



# 基础组件：VConnection

VConnection是所有提供IO功能的Connection类的基类，自身无法实现功能，必须通过派生类实现具体功能。

VConnection类是由Processor返回的一种单向或双向数据管道的抽象描述。

在某种意义上来说，与文件描述符的功能相似。

根本上来说，VConnection是定义Stream IO处理方法的基类。

它也是一个被Processor回调的Continuation。

VConnection定义了一个可以串联的虚拟管道，可以让ATS在网络、状态机、磁盘间形成流式数据通信，是上层通信机制的很关键一环。

## 结构

```

class VConnection : public Continuation
{
public:
  virtual ~VConnection();

  virtual VIO *do_io_read(Continuation *c = NULL, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0) = 0;

  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false) = 0;

  virtual void do_io_close(int lerrno = -1) = 0;

  virtual void do_io_shutdown(ShutdownHowTo_t howto) = 0;

  VConnection(ProxyMutex *aMutex);

// Private
  // Set continuation on a given vio. The public interface
  // is through VIO::set_continuation()
  virtual void set_continuation(VIO *vio, Continuation *cont);

  // Reenable a given vio.  The public interface is through VIO::reenable
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  virtual bool
  get_data(int id, void *data)

  virtual bool
  set_data(int id, void *data)

public:
  int lerrno;
};

```

### 通用Event Code

VC_EVENT_INACTIVITY_TIMEOUT

  - 表示在一定时间内: 没有从连接读取到数据 / 没有向连接写入数据
  - 对于读操作可能是buffer为空，对于写操作可能是连接的缓冲区满了

VC_EVENT_ACTIVITY_TIMEOUT

  - 读/写操作超时

VC_EVENT_ERROR

  - 在读取/写入过程中发生了错误


## 方法

### do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf)

准备接收来自 VConnection 数据

- 状态机（SM）调用它来从 VConnection 读取数据。
- 处理机（Processor）实现这个读取功能时，首先获取锁，然后将新的数据放入buf，回调Continuation，然后释放锁。
- 例如：让状态机处理以特殊字符标记事务结束数据传输协议。

可能的Event Code（在状态机回调 Continuation 时，VConn可能会使用这些值作为Event Code）

- VC_EVENT_READ_READY
  - 数据已经添加到buffer，或者buffer满了
- VC_EVENT_READ_COMPLETE
  - nbytes字节的数据已经被读取到buffer了
- VC_EVENT_EOS
  - Stream的读取端关闭了连接

参数

- c
  - 在读操作完成后，将要回调的Continuation，同时会传递Event Code过去
- nbytes
  - 想要读取的字节数，如果不确定要读取多少，必须设定为INT64_MAX
- buf
  - 读取到的数据会放入这里

返回值

- 代表调度IO操作的VIO类型

注意

- 当c＝NULL，nbytes＝0，buf＝NULL时表示取消之前设置的do_io_read操作。
- 当没有数据可读，或者buf写满，会导致VIO被disable，需要调用reenable才会继续读取数据。

### do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *buf, bool owner)

准备向 VConnection 写入数据

- 状态机（SM）调用它来向 VConnection 写入数据。
- 处理机（Processor）实现这个读取功能时，首先获取锁，然后将来自buf的数据写入，回调Continuation，然后释放锁。

可能的Event Code（在状态机回调 Continuation 时，VConn可能会使用这些值作为Event Code）

- VC_EVENT_WRITE_READY
  - 来自buffer的数据已经写入，或者buffer为空
- VC_EVENT_WRITE_COMPLETE
  - 来自buffer的nbytes字节的数据已经被写入

参数

- c
  - 在读操作完成后，将要回调的Continuation，同时会传递Event Code过去
- nbytes
  - 想要写入的字节数，如果不确定要写入多少，必须设定为INT64_MAX
- buf
  - 数据源，将从这里读取数据，然后写入
- owner
  - 未使用，默认false，如果设置为true，可能会引发assert ??

返回值

- 代表调度IO操作的VIO类型

注意

- 当c＝NULL，nbytes＝0，buf＝NULL时表示取消之前设置的do_io_write操作。
- 当写数据失败，或者buf空了，会导致VIO被disable，需要调用reenable才会继续写数据。

### reenable & reenable_re

激活VIO

- 让之前设置的VIO恢复运行，EventSystem将会呼叫处理机（Processor）。

### do_io_close

表示不再需要这个 VConnection 了 

- 当状态机使用完一个VConnection，必须调用这个函数，表示可以进行回收。
- 在调用了close之后
  - VConnection和底层的Processor都不能再发送任何与此VConnection相关联的Event给状态机。
  - 状态机也不能够访问该VConnection和从其获得/返回的VIO。

参数

- lerrno
  - 用于表示这是一个正常的关闭还是一个异常的关闭
  - 两种关闭的区别由底层的VConnection类型来决定

### do_io_shutdown

终结VConnection的单向或者双向传输

  - 调用此方法后，与之相关方向的I/O操作都将被禁止
  - Processor 不可以发送任何与之相关的EVENT（甚至是Timeout Event）给状态机
  - 状态机也不可以使用来自被Shutdown方向的VIO
  - 即使双向传输都被Shutdown，当状态机需要回收该VConnection时，仍然必须通过调用do_io_close来完成回收操作。

可能的Howto值
  - IO_SHUTDOWN_READ ＝ 0
    - 关闭读取端，表示该VConn不应该再产生Read Event
  - IO_SHUTDOWN_WRITE ＝ 1
    - 关闭写入端，表示该VConn不应该再产生Write Event
  - IO_SHUTDOWN_READWRITE = 2
    - 双向关闭，表示该VConn不应该再产生Read和Write Event

### get_data & set_data

这个功能被用于状态机，从/向一个VConnection传送信息，而不破坏VConnection的抽象

  - 其行为取决于哪种VConnection的类型被使用
  - 在派生类中存取自定义的数据结构

参数

- id
  - 为枚举类型TSApiDataType
- data
  - 对应的数据
- 返回值
  - 成功返回True

```
  /** Used in VConnection::get_data(). */
  enum TSApiDataType {
    TS_API_DATA_READ_VIO = VCONNECTION_API_DATA_BASE,
    TS_API_DATA_WRITE_VIO,
    TS_API_DATA_OUTPUT_VC,
    TS_API_DATA_CLOSED,
    TS_API_DATA_LAST ///< Used by other classes to extend the enum values.
  };
```

## 参考资料
- [I_VConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VConnection.h)
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
