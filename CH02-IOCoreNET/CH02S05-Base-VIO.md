# 基础组件：VIO

VIO用于描述一个正在进行的IO操作。

它是由VConnections里的do_io_read和do_io_write方法返回。通过VIO，状态机可监控操作的进展，在数据到达时可以激活操作。

## 定义

```
class VIO 
{
public:
  ~VIO() {}

  /** Interface for the VConnection that owns this handle. */
  Continuation *get_continuation();
  void set_continuation(Continuation *cont);

  void done();

  int64_t ntodo();

  /////////////////////
  // buffer settings //
  /////////////////////
  void set_writer(MIOBuffer *writer);
  void set_reader(IOBufferReader *reader);
  MIOBuffer *get_writer();
  IOBufferReader *get_reader();

  inkcoreapi void reenable();

  inkcoreapi void reenable_re();

  VIO(int aop);
  VIO();

public:
  Continuation *_cont;

  int64_t nbytes;

  int64_t ndone;

  int op;

  MIOBufferAccessor buffer;

  VConnection *vc_server;

  Ptr<ProxyMutex> mutex;
};

```

## 方法

get_continuation()

  - 获得与此VIO关联的Continuation
  - 返回成员：_cont

set_continuation(Continuation *cont)

  - 设置与此VIO关联的Continuation（通常为状态机SM），同时继承Cont的mutex。
  - 通过调用：vc_server->set_continuation(this, cont) 通知到与VIO关联的VConnection。
     - 这个VConnection::set_continuation()方法，随着NT的支持在Yahoo开源时被取消了，也就被同时废弃了。
  - 设置之后，当此VIO上发生事件时，将会回调新Cont的handler，并传递Event。
  - 如果cont==NULL，那么会同时清除VIO的mutex，也会传递给vc_server

set_writer(MIOBuffer *writer)

  - 调用：buffer.writer_for(writer);

set_reader(IOBufferReader *reader)

  - 调用：buffer.reader_for(reader);

get_writer()

  - 返回：buffer.writer();

get_reader()

  - 返回：buffer.reader();

done()

  - 将VIO设置为完成，此VIO操作将会进入disable状态
  - 设置nbytes ＝ ndone + buffer.reader()->read_avail()
  - 如果buffer.reader()为空，则设置nbytes ＝ ndone
  - 将不会触发EVENT_READ_COMPLETE或者EVENT_WRITE_COMPLETE事件**

ntodo()

  - 查看此VIO还有多少工作没有完成
  - 返回 nbytes - ndone

reenable() & reenable_re()

  - 调用：vc_server->reenable(this) 或 vc_server->reenable_re(this)
  - 状态机SM通过它来激活一个I/O操作。
  - 告知VConnection有更多的数据等待被处理，但是首先要尝试继续之前的操作
  - 当无法进行后续操作时，I/O操作将休眠，并等待被再次唤醒
     - 对于读取操作，这意味着buffer满了。
     - 对于写入操作，这意味着buffer空了。
  - 当被唤醒之后，后续操作仍然无法进行时，这次唤醒将被忽略，不会有新的Event生成
     - 这表示下次被唤醒时，传入的Event仍然是上一次设置的Event
  - 应当避免非必要的唤醒，这会浪费CPU资源，并且降低系统的吞吐效率
  - 对于reenable和reenable_re的区别，需要参考VConnection的继承类中的定义。

## 成员变量

_cont

  - 该指针用来保存调用这个VConnection，并且传递了Event进来的Continuation。
  - 通常此Cont为状态机SM

nbytes

  - 表示需要处理的总字节数

ndone

  - 表示已经处理完成的字节数
  - 操作该值时必须取得锁

op

  - 表示这个VIO的操作类型

```
  enum {
    NONE = 0,
    READ,
    WRITE,
    CLOSE,
    ABORT,
    SHUTDOWN_READ,
    SHUTDOWN_WRITE,
    SHUTDOWN_READWRITE,
    SEEK,
    PREAD,
    PWRITE,
    STAT,
  };
```

buffer

  - 如果op为写操作，包含一个指向IOBufferReader的指针
  - 如果op为读操作，包含一个指向MIOBuffer的指针

vc_server

  - 这是指回VIO对应VConnection的一个反向指针，用于reenable内部。

mutex

  - 对于状态机的mutex的引用
  - 即使状态机已经关闭VConnection并且进行了回收，Processor仍然可以安全的锁定该操作

## 理解 VIO

状态机SM需要发起IO操作时，通过创建VIO来告知IOCore，IOCore在执行了物理IO操作后再通过VIO回调（通知）状态机SM。

所以VIO中包含了三个元素：BufferAccessor，VConnection，Cont（SM）

数据流在BufferAccessor与VConnection之间流动，消息流在IOCore与Cont（SM）之间传递。

需要注意的是VIO里的BufferAccessor是指向MIOBuffer的操作者，MIOBuffer是由Cont（SM）创建的。

另外IOCore回调Cont（SM）时，是通过VIO内保存的\_cont指针进行的，不是通过vc\_server里的Cont。

```
+-------+                 +--------------------+    +-----------------------------------------+
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |  |           |  |                    |    |  |           |                          |
|       ===> vc_server ==========> read  ==============> MIOBuffer |                          |
|       |  |           |  |                    |    |  |           |                          |
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |                 |  vc->read.vio._cont----->handleEvent(READ_READY, &vc->read.vio)   |
|       |                 |                    |    |                                         |
|  NIC  |                 |       IOCore       |    |             VIO._Cont (SM)              |
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |  |           |  |                    |    |  |           |                          |
|       <=== vc_server <========== write <============== MIOBuffer |                          |
|       |  |           |  |                    |    |  |           |                          |
|       |  +-----------+  |                    |    |  +-----------+                          |
|       |                 | vc->write.vio._cont----->handleEvent(WRITE_READY, &vc->write.vio) |
|       |                 |                    |    |                                         |
+-------+                 +--------------------+    +-----------------------------------------+

对于TCP，这里的IOCore指的就是NetHandler和UnixNetVConnection
```

例如，当状态机（SM）想要实现接收1M字节数据，并且转发给客户端的时候，我们可以通过以下方式实现：

- 创建一个MIOBuffer，用于存放临时接收的数据
- 通过do_io_read在SourceVC上创建一个读数据的VIO，设置读取长度为1M字节，并传入MIOBuffer用于接收临时数据
- IOCore在SourceVC上接收到数据时会将数据生产到VIO中（暂存在MIOBuffer内）
   - 然后呼叫SM->handler(EVENT_READ_READY, SourceVIO)
   - 在handler中可以对SourceVIO进行消费（读取MIOBuffer内暂存的数据），获取本次读取到的一部分数据
- VIO内有一个计数器，当总生产（读取）数据量达到1M字节
   - 那么会由IOCore呼叫SM->handler(EVENT_READ_COMPLETE, SourceVIO)
   - 在handler中需要首先对SourceVIO进行消费（读出），然后关闭SourceVIO。
- 到此就完成了接收1M字节数据的过程。

可以看到，ATS通过VIO向IOCore描述一个包含若干次IO操作的任务，用于VIO操作的MIOBuffer可以很小，仅需要保存一次IO操作所需要的数据，然后由SM对VIO和MIOBuffer进行处理。

或者，可以把VIO看作是一个PIPE，一端是底层IO设备，一端是MIOBuffer，一个VIO创建之前要选择好它的方向，创建后不可修改，如：读VIO，就是从底层IO设备（vc_server）读数据到MIOBuffer；写VIO，就是从MIOBuffer写数据到底层IO设备（vc_server）。

```
+---------------------------------------------------------------+
|                 op=READ, nbytes=X, ndone=Y                    |
|                                                               |
|     VConnection *vc_server ----> MIOBufferAccessor buffer     |
|                                                               |
|  if( ndone < nbytes )                                         |
|      Continuation *_cont->handler(EVENT_READ_READY, thisVIO)  |
|  if( ndone == nbytes )                                        |
|                 _cont->handler(EVENT_READ_COMPLETE, thisVIO)  |
+---------------------------------------------------------------+

+---------------------------------------------------------------+
|               op=WRITE, nbytes=X, ndone=Y                     |
|                                                               |
|     VConnection *vc_server <---- MIOBufferAccessor buffer     |
|                                                               |
|  if( ndone < nbytes )                                         |
|      Continuation *_cont->handler(EVENT_WRITE_READY, thisVIO) |
|  if( ndone == nbytes )                                        |
|                 _cont->handler(EVENT_WRITE_COMPLETE, thisVIO) |
|  else if( wbeEvent )                                          |
|                 _cont->handler(wbeEvent, thisVIO)             |
+---------------------------------------------------------------+
在某些VC的实现中会在写空缓冲区时，再次触发wbeEvent事件，通常为EVENT_WRITE_READY
```

VIO操作包含多种操作类型，可以通过'op'成员变量来确定。可选的值如下：

  - READ 表示读操作
  - WRITE 表示写操作
  - CLOSE 表示请求关闭VConnection
  - ABORT
  - SHUTDOWN_READ
  - SHUTDOWN_WRITE
  - SHUTDOWN_READWRITE
  - SEEK
  - PREAD
  - PWRITE
  - STAT

## 参考资料
- [I_VIO.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VIO.h)
- [I_VConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_VConnection.h)
- [I_IOBuffer.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_IOBuffer.h)
