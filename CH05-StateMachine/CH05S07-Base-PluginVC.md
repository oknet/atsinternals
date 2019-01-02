# 基础组件：PluginVC

PluginVC 是伪装成 NetVC，用来实现在两个“延续”(Continuation)之间进行双向数据传输的状态机。

PluginVC 是以 PluginVCCore 作为核心，其内包含两个 PluginVC 成员，一个是 active\_vc，一个是 passive\_vc。

- PluginVCCore 就像是一个缩小版的 IOCore Net Subsystem，它只负责管理 active\_vc 和 passive\_vc。
- 每一个 PluginVC 通过 4 个 Event 的不间断回调，实现了数据的双向传输
   - sm_lock_retry_event （对 vio.mutex 上锁后可以读写）
   - core_lock_retry_event （对 PluginVC::mutex 上锁后可以读写）
   - active_event (被 PluginVC 回调时可以读写）
   - inactive_event (被 PluginVC 回调时可以读写）
   - 除了 inactive_event 是周期性事件，其它三个都是立即事件
- 每一个 PluginVC 可以分别在不同的线程被回调，可用于实现跨线程的数据传输

在 TSHttpConnectWithPluginId() 的实现中，

- 通过 PluginVCCore::connect() 与 HttpSM 建立连接，passive\_vc 被 HttpSM 作为 ClientVC
- 在调用返回后，Plugin 获得 active\_vc，并且 passive\_vc 被设置 internal request 标志
- 其调用者将作为 Http Client，需要向 active\_vc 中写入一个合法的 HTTP 请求
- 然后 PluginVCCore 负责把数据传输到 passive\_vc 一侧，并且向 HttpSM 发送 READ\_READY 事件

在 TSHttpTxnServerIntercept() 的实现中，

- HttpSM 会通过 PluginVCCore::connect\_re() 获得成员 active\_vc，并将其作为 ServerVC
- 在 PluginVCCore::connect\_re() 被调用后，Plugin 会收到 TS\_EVENT\_NET\_ACCEPT 时间，并获得 passive\_vc
- 其调用者将作为 Http Origin Server，将会从 passive\_vc 一侧读取到 HttpSM 发送给 OS 的请求
- 然后将一个合法的 HTTP 响应写入 passive\_vc，PluginVCCore 将负责把数据传输到 active\_vc 一侧

```
+-------------+                      +--------------+                      +-------------------+
| Plugin Cont |                      |              |                      |                   |
|      as     +----- active_vc ------+ PluginVCCore +----- passive_vc -----+ HttpClientSession |
|    Client   |                      |              |                      |                   |
+-------------+                      +--------------+                      +---------+---------+
                                                                                     |
                                                                                     |
                                                                           +---------+---------+
                                                                           |       HttpSM      |
                                                                           |         or        |
                                                                           |     HttpTunnel    |
                                                                           +---------+---------+
                                                                                     |
                                                                                     |
+-------------+                      +--------------+                      +---------+---------+
| Plugin Cont |                      |              |                      |                   |
|      as     +----- passive_vc -----+ PluginVCCore +------ active_vc -----+ HttpServerSession |
|    Server   |                      |              |                      |                   |
+-------------+                      +--------------+                      +-------------------+
```

## 定义

PluginIdentity 是用来识别 PluginVC 身份的基类

- 它的存在主要是用来在必要的时候判断 NetVC 是不是一个真正的 NetVC 还是由 PluginVC 伪装的 NetVC
- 可以通过 PluginId 获得创建此 PluginVC 的插件的信息

由于 PluginVC 须要伪装成 NetVConnection，因此它继承自两个类。

```
class PluginVC : public NetVConnection, public PluginIdentity
{
  friend class PluginVCCore;

public:
  // 构造函数，需要传入 PluginVCCore 对象
  PluginVC(PluginVCCore *core_obj);
  // 析构函数
  ~PluginVC();

  // 开始：实现基本的 NetVC 的方法
  virtual VIO *do_io_read(Continuation *c = NULL, int64_t nbytes = INT64_MAX, MIOBuffer *buf = 0);

  virtual VIO *do_io_write(Continuation *c = NULL, int64_t nbytes = INT64_MAX, IOBufferReader *buf = 0, bool owner = false);

  virtual void do_io_close(int lerrno = -1);
  virtual void do_io_shutdown(ShutdownHowTo_t howto);

  // Reenable a given vio.  The public interface is through VIO::reenable
  virtual void reenable(VIO *vio);
  virtual void reenable_re(VIO *vio);

  // Timeouts
  virtual void set_active_timeout(ink_hrtime timeout_in);
  virtual void set_inactivity_timeout(ink_hrtime timeout_in);
  virtual void cancel_active_timeout();
  virtual void cancel_inactivity_timeout();
  virtual void add_to_keep_alive_queue();
  virtual void remove_from_keep_alive_queue();
  virtual bool add_to_active_queue();
  virtual ink_hrtime get_active_timeout();
  virtual ink_hrtime get_inactivity_timeout();

  // Pure virutal functions we need to compile
  virtual SOCKET get_socket();
  virtual void set_local_addr();
  virtual void set_remote_addr();
  virtual int set_tcp_init_cwnd(int init_cwnd);
  virtual int set_tcp_congestion_control(int);

  virtual void apply_options();

  virtual bool get_data(int id, void *data);
  virtual bool set_data(int id, void *data);
  // 结束
  
  // 获取另外一端的 PluginVC 对象
  virtual PluginVC *
  get_other_side()
  {
    return other_side;
  }

  // 用来实现 PluginIdentity 的方法
  //@{ @name Plugin identity.
  /// Override for @c PluginIdentity.
  virtual const char *
  getPluginTag() const
  {
    return plugin_tag;
  }
  /// Override for @c PluginIdentity.
  virtual int64_t
  getPluginId() const
  {
    return plugin_id;
  }

  /// Setter for plugin tag.
  virtual void
  setPluginTag(const char *tag)
  {
    plugin_tag = tag;
  }
  /// Setter for plugin id.
  virtual void
  setPluginId(int64_t id)
  {
    plugin_id = id;
  }
  //@}

  // 主回调处理函数
  int main_handler(int event, void *data);

private:
  // 从 PluginVCCore 的缓冲区向 Read VIO 的缓冲区传输数据，并回调
  void process_read_side(bool);
  // 从 Write VIO 的缓冲区向 PluginVCCore 的缓冲区传输数据，并回调
  void process_write_side(bool);
  void process_close();
  void process_timeout(Event **e, int event_to_send);

  void setup_event_cb(ink_hrtime in, Event **e_ptr);

  void update_inactive_time();
  int64_t transfer_bytes(MIOBuffer *transfer_to, IOBufferReader *transfer_from, int64_t act_on);

  // magic 值，用于调试
  uint32_t magic;
  // PluginVC 的类型，枚举值
  //   PLUGIN_VC_UNKNOWN: 未知（默认值，防止未初始化的情况）
  //   PLUGIN_VC_ACTIVE：主动端
  //   PLUGIN_VC_PASSIVE：被动端
  PluginVC_t vc_type;
  // 指回到 PluginVCCore 的指针
  PluginVCCore *core_obj;

  // 指向另外一端的 PluginVC 的指针
  PluginVC *other_side;

  // 与NetVC里的 read 和 write 成员功能一致
  // PluginVCState 类型与 NetState 类型的定义非常接近
  PluginVCState read_state;
  PluginVCState write_state;

  // 标记是否需要调用 process_read_side() 和 process_write_side()
  bool need_read_process;
  bool need_write_process;

  // 表示 PluginVC 是否已经被关闭，与 NetVC::closed 一致
  volatile bool closed;
  // 由使用 PluginVC 的状态机触发的，对 main_handler 进行回调的 Event
  Event *sm_lock_retry_event;
  // 由 PluginVC / PluginVCCore 触发的，对 main_handler 进行回调的 Event
  Event *core_lock_retry_event;

  // 是否可以被释放
  bool deletable;
  // 重入计数，与 NetVC::recursion 一致
  int reentrancy_count;

  // active 超时
  ink_hrtime active_timeout;
  // 在 active 超时后回调 PluginVC 的 Event
  Event *active_event;

  // inactive 超时
  ink_hrtime inactive_timeout;
  ink_hrtime inactive_timeout_at;
  // 在 inactive 超时后回调 PluginVC 的 Event
  // 此 Event 是一个周期性事件，每次回调 main_handler 后，通过判断上述两个变量确定是否超时
  Event *inactive_event;

  // 用于 PluginIdentify
  const char *plugin_tag;
  int64_t plugin_id;
};
```

### 初始化及构造

PluginVC 不会单独出现，它总是作为 PluginVCCore 的成员对象存在。
因此 PluginVC 都是通过 PluginVCCore::alloc() 创建 PluginVCCore 实例来获得。

```
PluginVCCore *
PluginVCCore::alloc()
{
  PluginVCCore *pvc = new PluginVCCore;
  pvc->init();
  return pvc; 
}
```

通过调用 init() 完成初始化

- 创建 mutex
- 初始化 active_vc 和 passive_vc 两个 PluginVC 成员
- 创建双向数据传输的MIOBuffer和IOBufferReader

```
void
PluginVCCore::init()
{
  // 使用独立的 mutex
  mutex = new_ProxyMutex();

  // 初始化 active 端的 PluginVC
  active_vc.vc_type = PLUGIN_VC_ACTIVE;
  active_vc.other_side = &passive_vc;
  active_vc.core_obj = this;
  active_vc.mutex = mutex;
  active_vc.thread = this_ethread();

  // 初始化 passive 端的 PluginVC
  passive_vc.vc_type = PLUGIN_VC_PASSIVE;
  passive_vc.other_side = &active_vc;
  passive_vc.core_obj = this;
  passive_vc.mutex = mutex;
  passive_vc.thread = active_vc.thread;

  // 初始化两个PluginVC之间的数据传递缓冲区
  // 只有对 PluginVC 和 与之对应的 SM 上锁之后，才可以访问这两个缓冲区
  // p_to_a:
  //   从 passive 端的 write VIO 获取数据写入 p_to_a_buffer
  //   从 p_to_a_reader 获得数据写入 active 端的 read VIO
  p_to_a_buffer = new_MIOBuffer(BUFFER_SIZE_INDEX_32K);
  p_to_a_reader = p_to_a_buffer->alloc_reader();
  // a_to_p:
  //   从 active 端的 write VIO 获取数据写入 a_to_p_buffer
  //   从 a_to_p_reader 获得数据写入 passive 端的 read VIO
  a_to_p_buffer = new_MIOBuffer(BUFFER_SIZE_INDEX_32K);
  a_to_p_reader = a_to_p_buffer->alloc_reader();

  Debug("pvc", "[%u] Created PluginVCCore at %p, active %p, passive %p", id, this, &active_vc, &passive_vc);
}
```

由于在 PluginVC 中需要频繁的设置回调，因此创建了 setup_event_cb() 以简化代码逻辑。

在函数的头部有一段说明，为了避免锁的问题，传递给 setup_event_cb 的Event指针，是两个不同的Event

- sm_lock_retry_event
  - 获得当前 PluginVC 的 VIO 的锁之后，才可以对该 event 设置回调
  - 通常在 PluginVC 回调 SM 之后，SM 对 PluginVC 执行 do_io, reenable 等操作时，都会通过该 Event 重新回调 PluginVC
  - 如果未获得 VIO 的锁，直接通过该 Event 设置对 PluginVC 的回调，可能会导致 Event 泄露
- core_lock_retry_event
  - 获得当前 PluginVC 自身的锁之后，才可以对该 event 设置回调
  - 由于 PluginVCCore 和 两个PluginVC成员共享同一个 mutex，因此在两个PluginVC之间需要相互通知的时候，通过此 Event 可以相互安排对方的回调

```
// void PluginVC::setup_event_cb(ink_hrtime in)
//
//    Setup up the event processor to call us back.
//      We've got two different event pointers to handle
//      locking issues
//
void
PluginVC::setup_event_cb(ink_hrtime in, Event **e_ptr)
{
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  // 传入的 Event 指针必须为 NULL，否则，
  //   需要先cancel()，然后再创建Event，不然会导致内存泄露
  // 这里的设计是：
  //   不重设Event，也就是不重复创建回调
  //   只有未创建回调的时候，才会创建新的Event
  if (*e_ptr == nullptr) {
    // We locked the pointer so we can now allocate an event
    //   to call us back
    // 如果 in == 0 表示须要立即回调
    if (in == 0) {
      // 需要一个 REGULAR 类型的线程来执行回调
      if (this_ethread()->tt == REGULAR) {
        *e_ptr = this_ethread()->schedule_imm_local(this);
      } else {
        *e_ptr = eventProcessor.schedule_imm(this);
      }
    // 否则表示定时回调
    } else {
      // 需要一个 REGULAR 类型的线程来执行回调
      if (this_ethread()->tt == REGULAR) {
        *e_ptr = this_ethread()->schedule_in_local(this, in);
      } else {
        *e_ptr = eventProcessor.schedule_in(this, in);
      }
    }
  }
}
```

### PluginVC Read I/O 流程分析

PluginVC::do_io_read() 前面的代码与 NetVC::do_io_read() 相似

- 设置 MIOBuffer
- 设置 VIO

然后是设置

- need_read_process = true 表示 main_handler 须要进行读操作处理
- 通过 setup_event_cb() 安排对 PluginVC::main_handler 的回调

最后是返回 VIO

```
void
PluginVC::do_io_read(Continuation *c, int64_t nbytes, MIOBuffer *buf)                                                                        
{
  ink_assert(!closed);
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  // 设置 VIO
  if (buf) {
    read_state.vio.buffer.writer_for(buf);
  } else {
    read_state.vio.buffer.clear();
  }

  // BUG：buffer.clear() 之后，可能会继续设置nbytes > 0
  // Note: we set vio.op last because process_read_side looks at it to
  //  tell if the VConnection is active.
  read_state.vio.mutex     = c->mutex;
  read_state.vio._cont     = c;
  read_state.vio.nbytes    = nbytes;
  read_state.vio.ndone     = 0;
  read_state.vio.vc_server = (VConnection *)this;
  read_state.vio.op        = VIO::READ;

  Debug("pvc", "[%u] %s: do_io_read for %" PRId64 " bytes", core_obj->id, PVC_TYPE, nbytes);

  // Since reentrant callbacks are not allowed on from do_io
  //   functions schedule ourselves get on a different stack
  // BUG：buffer.clear() 之后，need_read_process 应该被设置为 false
  need_read_process = true;
  setup_event_cb(0, &sm_lock_retry_event);

  return &read_state.vio;
}
```
在设置 need_read_process 之后，EventSystem将会回调 PluginVC::main_handler()，然后调用 process_read_side(false) 实现了：

- 将 PluginVCCore缓冲区 内的数据读取到 状态机的读缓冲区，
- 然后再通知另外一端继续向 PluginVCCore缓冲区 写入数据的流程。

由于 process_read_side() 包含两种情况的处理，非常的复杂，因此，我会在代码里进行标记，未标记的部分则是两种情况都需要的公共代码。

```
// void PluginVC::process_read_side()
//
//   This function may only be called while holding
//      this->mutex & while it is ok to callback the
//      read side continuation
//
// 在调用该函数之前需要满足以下条件：
//
//   - 已经对 PluginVC::mutex 上锁
//   - 已经对 PluginVC::read_state.vio.mutex 上锁
//     - 满足回调 PluginVC::read_state.vio._cont 的条件。
//
//   Does read side processing
//
void
PluginVC::process_read_side(bool other_side_call)
{
  ink_assert(!deletable);
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  // TODO: Never used??
  // MIOBuffer *core_buffer;

  IOBufferReader *core_reader;

  /* 根据当前 PluginVC 的类型（Active / Passive）选择对应的 MIOBuffer 和 IOBufferReader
   * 用来从 Passive端 向 Active端 传送数据的 MIOBuffer 和 IOBufferReader
   *   core_obj->p_to_a_buffer
   *   core_obj->p_to_a_reader
   * 用来从 Active端 向 Passive端 传送数据的 MIOBuffer 和 IOBufferReader
   *   core_obj->a_to_p_buffer
   *   core_obj->a_to_p_reader
   * 处理读操作，需要读取数据，因此这里只需要获得 IOBufferReader，因此 MIOBuffer 部分的代码被注释掉了。
   * 如果当前 PluginVC 是 Active端，而当前又是 读操作，因此是从 Passive 向 Active 传送数据
   */
  if (vc_type == PLUGIN_VC_ACTIVE) {
    // core_buffer = core_obj->p_to_a_buffer;
    core_reader = core_obj->p_to_a_reader;
  } else {
    // 相反的，则是从 Active 向 Passive 传送数据
    ink_assert(vc_type == PLUGIN_VC_PASSIVE);
    // core_buffer = core_obj->a_to_p_buffer;
    core_reader = core_obj->a_to_p_reader;
  }

  /* 以下部分仅在 other_side_call == true 时需要执行
   * 但是在 other_side_call == false 时，也可以执行，但是没有任何效果
   * 参考：
   *   - https://github.com/apache/trafficserver/pull/4746
   *   - TS-3339 （https://issues.apache.org/jira/browse/TS-3339）
   */
  //************ START: other_side_call == true ************
  // 将 need_write_process 重置，这条应该放在下面的 if 语句里面
  need_read_process = false;

  // 在未上锁之前进行一次 read_state 的检查，是为了避免在后面上锁失败的时候，仍然强制执行读流程
  // BUG：这里缺少了对 read_state.shutdown 的判断
  if (read_state.vio.op != VIO::READ || closed) {
    return;
  }
  /* 如果 other_side_call == true，那么 read_state.vio.mutex 是一定没有上锁的，
   * 因此，这里要尝试对其上锁，然后才能进一步完成 vio 的读写操作。
   * 如果 other_side_call == false，那么对 process_read_side() 的调用一定来自：
   *
   *   - PluginVC::main_handler()
   *   - PluginVC::reenable_re()
   *
   * 此时 read_state.vio.mutex 是已经上锁的状态。
   */
  // Acquire the lock of the read side continuation
  EThread *my_ethread = mutex->thread_holding;
  ink_assert(my_ethread != NULL);
  MUTEX_TRY_LOCK(lock, read_state.vio.mutex, my_ethread);
  if (!lock.is_locked()) {
    Debug("pvc_event", "[%u] %s: process_read_side lock miss, retrying", core_obj->id, PVC_TYPE);

    // 如果上锁失败，则强制执行一次读流程，相当于执行了 PluginVC::reenable(read_state.vio)
    // 唯一不同的是使用了 core_lock_retry_event 事件来执行回调
    need_read_process = true;
    setup_event_cb(PVC_LOCK_RETRY_TIME, &core_lock_retry_event);
    // 从 other_side 处理中返回到 local_side 写流程（process_write_side）
    return;
  }
  //************ END: other_side_call == true ************

  Debug("pvc", "[%u] %s: process_read_side", core_obj->id, PVC_TYPE);
  need_read_process = false;

  // 判断 read shutdown
  // BUG：在上锁后，应该对 read_state 重新进行判断 (TS-3339)，这里只判断了 shutdown 是不对的。
  // Check read_state.shutdown after the lock has been obtained.
  if (read_state.shutdown) {
    return;
  }

  // 判断 ntodo
  // Check the state of our read buffer as well as ntodo
  int64_t ntodo = read_state.vio.ntodo();
  if (ntodo == 0) {
    return;
  }

  // 准备从 PluginVCCore 的内部 MIOBuffer 读取数据到 read VIO
  int64_t bytes_avail = core_reader->read_avail();
  int64_t act_on = MIN(bytes_avail, ntodo);

  Debug("pvc", "[%u] %s: process_read_side; act_on %" PRId64 "", core_obj->id, PVC_TYPE, act_on);

  // ntodo 不可能为 0, 因此这里实际上是判断 bytes_avail
  // 如果 PluginVCCore 内部的缓冲区没有可读取的数据，
  if (act_on <= 0) {
    // 并且对端已经关闭, 或者 对端 写关闭，
    // 则回调 read VIO 状态机，遇到了 EOS（这里是模拟了 read(socket) 返回 0 的场景）
    if (other_side->closed || other_side->write_state.shutdown) {
      read_state.vio._cont->handleEvent(VC_EVENT_EOS, &read_state.vio);
    }
    return;
  }
  // Bytes available, try to transfer from the PluginVCCore
  //   intermediate buffer
  //
  /* 从 core_reader 读取数据, 写入 vio buffer
   * 临时保存在 PluginVCCore 缓冲区内的最大数据量为 32 KB (PVC_DEFAULT_MAX_BYTES)。
   * 数据读取操作同时也会受到接收缓冲区 water_mark 值的限制，
   *
   *   - 如果接收缓冲区内仍有数据可读，
   *   - 同时可读取的数据容量大于 water_mark 值，则不再向接收缓冲区填充更多数据
   *
   * 为了保证 PluginVC 数据流的传输效率，这里将 water_mark 值的最小值设置为了 32 KB，
   * 这样每次最多可以向接收缓冲区传输 32 KB 的数据，以尽最大可能将 core_buffer 内的数据传输到接收缓冲区。
   * 根据 water_mark 功能的设定：
   *
   *   - 目标缓冲区内已有数据容量大于等于 water_mark 值时，
   *      - 为 high_water 状态，此时不继续填充数据
   *      - 否则为 low_water 状态，应该继续填充数据
   *   - 当处与 low_water 状态时，不回调 VIO 状态机，只有处于 high_water 状态时才回调状态机。
   *   - water_mark 的默认值为 0，表示任何时候均处于 high_water 状态，
   *      - 因此，读取任意数据都会回调 VIO 状态机
   *
   * 可以看到，超量填充数据，不会让 high_water 状态发生改变，
   * 因此，将低于 32 KB 的 water_mark 值拉升至 32 KB 不会对原有逻辑产生任何影响。
   */
  MIOBuffer *output_buffer = read_state.vio.get_writer();

  int64_t water_mark = output_buffer->water_mark;
  water_mark = MAX(water_mark, PVC_DEFAULT_MAX_BYTES);
  int64_t buf_space = water_mark - output_buffer->max_read_avail();
  // 如果接收缓冲区处于 high_water 状态，则直接返回。
  if (buf_space <= 0) {
    Debug("pvc", "[%u] %s: process_read_side no buffer space", core_obj->id, PVC_TYPE);
    return;
  }
  /* 计算实际要传输的数据量，
   *
   *   - act_on 表示 core_buffer 内可以被传输的数据容量
   *   - buf_space 表示接收缓冲区可以接收的数据量
   *
   * 这里取两者的最小值，以确保传输后，接收缓冲区的可读数据量仍然不超过 water_mark 与 PluginVCCore 缓冲区的传输限制。
   */
  act_on = MIN(act_on, buf_space);

  // 数据传输
  int64_t added = transfer_bytes(output_buffer, core_reader, act_on);
  if (added <= 0) {
    // Couldn't actually get the buffer space.  This only
    //   happens on small transfers with the above
    //   PVC_DEFAULT_MAX_BYTES factor doesn't apply
    Debug("pvc", "[%u] %s: process_read_side out of buffer space", core_obj->id, PVC_TYPE);
    return;
  }

  read_state.vio.ndone += added;

  Debug("pvc", "[%u] %s: process_read_side; added %" PRId64 "", core_obj->id, PVC_TYPE, added);

  // 根据 ntodo 确定回调 READY 或 COMPLETE 事件
  if (read_state.vio.ntodo() == 0) {
    read_state.vio._cont->handleEvent(VC_EVENT_READ_COMPLETE, &read_state.vio);
  } else {
    read_state.vio._cont->handleEvent(VC_EVENT_READ_READY, &read_state.vio);
  }

  // 刷新 inactive timeout 类似 net_activity()
  update_inactive_time();

  // Wake up the other side so it knows there is space available in
  //  intermediate buffer
  // 由于消费了 core buffer 的数据, 尽可能通知 对端 来填充数据
  if (!other_side->closed) {
    if (!other_side_call) {
      // 来自本地 PluginVC::main_handler
      // 通过对端的 process_write_side(true) 完成数据填充
      /* 此处并不满足调用 process_write_side 的要求，可参考 PR #4746 进行改进：
       *
       *   - https://github.com/apache/trafficserver/pull/4746
       */
      other_side->process_write_side(true);
    } else {
      // 来自对端 PluginVC::main_handler
      // 此时对端的 write_state.vio.mutex 是上锁的
      // 因此, 可以安全的执行 reenable() 通过 sm_lock_retry_event 进行回调
      other_side->write_state.vio.reenable();
    }
  }
}
```

### PluginVC Write I/O 流程分析

PluginVC::do_io_write() 前面的代码与 NetVC::do_io_write() 相似

- 设置 MIOBuffer
- 设置 VIO

然后是设置

- need_write_process = true 表示 main_handler 须要进行写操作处理
- 通过 setup_event_cb() 安排对 PluginVC::main_handler 的回调

最后是返回 VIO

```
VIO *
PluginVC::do_io_write(Continuation *c, int64_t nbytes, IOBufferReader *abuffer, bool owner)
{
  ink_assert(!closed);
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  // 设置 VIO
  if (abuffer) {
    ink_assert(!owner);
    write_state.vio.buffer.reader_for(abuffer);
  } else {
    write_state.vio.buffer.clear();
  }

  // BUG：buffer.clear() 之后，可能会继续设置nbytes > 0
  // Note: we set vio.op last because process_write_side looks at it to
  //  tell if the VConnection is active.
  write_state.vio.mutex = c ? c->mutex : this->mutex;
  write_state.vio._cont = c;
  write_state.vio.nbytes = nbytes;
  write_state.vio.ndone = 0;
  write_state.vio.vc_server = (VConnection *)this;
  write_state.vio.op = VIO::WRITE;

  Debug("pvc", "[%u] %s: do_io_write for %" PRId64 " bytes", core_obj->id, PVC_TYPE, nbytes);

  // Since reentrant callbacks are not allowed on from do_io
  //   functions schedule ourselves get on a different stack
  // BUG：buffer.clear() 之后，need_read_process 应该被设置为 false
  need_write_process = true;
  setup_event_cb(0, &sm_lock_retry_event);

  return &write_state.vio;
}

```

在设置 need_write_process 之后，EventSystem将会回调 PluginVC::main_handler()，然后调用 process_write_side(false) 实现了：

- 将 状态机的写缓冲区 内的数据写入到 PluginVCCore缓冲区，
- 然后再通知另外一端继续从 PluginVCCore缓冲区 读取数据的流程。

由于 process_write_side() 包含两种情况的处理，非常的复杂，因此，我会在代码里进行标记，未标记的部分则是两种情况都需要的公共代码。

```
// void PluginVC::process_write_side(bool cb_ok)
//
//   This function may only be called while holding
//      this->mutex & while it is ok to callback the
//      write side continuation
//
// 在调用该函数之前需要满足以下条件：
//
//   - 已经对 PluginVC::mutex 上锁
//   - 已经对 PluginVC::write_state.vio.mutex 上锁
//     - 满足回调 PluginVC::write_state.vio._cont 的条件。
//
//   Does write side processing
//
void
PluginVC::process_write_side(bool other_side_call)
{
  ink_assert(!deletable);
  ink_assert(magic == PLUGIN_VC_MAGIC_ALIVE);

  /* 根据当前 PluginVC 的类型（Active / Passive）选择对应的 MIOBuffer 和 IOBufferReader
   * 用来从 Passive端 向 Active端 传送数据的 MIOBuffer 和 IOBufferReader
   *   core_obj->p_to_a_buffer
   *   core_obj->p_to_a_reader
   * 用来从 Active端 向 Passive端 传送数据的 MIOBuffer 和 IOBufferReader
   *   core_obj->a_to_p_buffer
   *   core_obj->a_to_p_reader
   * 处理写操作，需要写入数据，因此这里只需要获得 MIOBuffer。
   * 如果当前 PluginVC 是 Active端，而当前又是 写操作，因此是从 Active 向 Passive 传送数据，
   * 相反的，则是从 Passive 向 Active 传送数据。
   */
  MIOBuffer *core_buffer = (vc_type == PLUGIN_VC_ACTIVE) ? core_obj->a_to_p_buffer : core_obj->p_to_a_buffer;

  /* 以下部分仅在 other_side_call == true 时需要执行
   * 但是在 other_side_call == false 时，也可以执行，但是没有任何效果
   * 参考：
   *   - https://github.com/apache/trafficserver/pull/4746
   *   - TS-3339 （https://issues.apache.org/jira/browse/TS-3339）
   */
  //************ START: other_side_call == true ************
  // 将 need_write_process 重置，这条应该放在下面的 if 语句里面
  need_write_process = false;

  // 在未上锁之前进行一次 write_state 的检查，是为了避免在后面上锁失败的时候，仍然强制执行写流程
  if (write_state.vio.op != VIO::WRITE || closed || write_state.shutdown) {
    return;
  }
  /* 如果 other_side_call == true，那么 write_state.vio.mutex 是一定没有上锁的，
   * 因此，这里要尝试对其上锁，然后才能进一步完成 vio 的读写操作。
   * 如果 other_side_call == false，那么对 process_write_side() 的调用一定来自：
   *
   *   - PluginVC::main_handler()
   *   - PluginVC::reenable_re()
   *
   * 此时 write_state.vio.mutex 是已经上锁的状态。
   */
  // Acquire the lock of the write side continuation
  EThread *my_ethread = mutex->thread_holding;
  ink_assert(my_ethread != NULL);
  MUTEX_TRY_LOCK(lock, write_state.vio.mutex, my_ethread);
  if (!lock.is_locked()) {
    Debug("pvc_event", "[%u] %s: process_write_side lock miss, retrying", core_obj->id, PVC_TYPE);

    // 如果上锁失败，则强制执行一次写流程，相当于执行了 PluginVC::reenable(write_state.vio)
    // 唯一不同的是使用了 core_lock_retry_event 事件来执行回调
    need_write_process = true;
    setup_event_cb(PVC_LOCK_RETRY_TIME, &core_lock_retry_event);
    // 从 other_side 处理中返回到 local_side 读流程（process_read_side）
    return;
  }
  //************ END: other_side_call == true ************

  Debug("pvc", "[%u] %s: process_write_side", core_obj->id, PVC_TYPE);
  need_write_process = false;

  // BUG：在上锁后，应该对 write_state 重新进行判断 (TS-3339)

  // 判断 ntodo
  // Check the state of our write buffer as well as ntodo
  int64_t ntodo = write_state.vio.ntodo();
  if (ntodo == 0) {
    return;
  }

  // 准备从 write VIO 向 PluginVCCore 的内部 MIOBuffer 写入数据
  IOBufferReader *reader = write_state.vio.get_reader();
  int64_t bytes_avail = reader->read_avail();
  int64_t act_on = MIN(bytes_avail, ntodo);

  Debug("pvc", "[%u] %s: process_write_side; act_on %" PRId64 "", core_obj->id, PVC_TYPE, act_on);

  // 如果在写入前发现对端已经关闭，或者对端执行了 shutdown(READ)，
  // 则回调 write VIO 的状态机，遇到了写失败（这里是模拟了 socket 上的 SIGPIPE 信号出发后的写失败）
  if (other_side->closed || other_side->read_state.shutdown) {
    write_state.vio._cont->handleEvent(VC_EVENT_ERROR, &write_state.vio);
    return;
  }

  // 如果 write VIO 里没有任何可以发送的内容，就向状态机发送 VC_EVENT_WRITE_READY 信号，
  // 让状态机向 write VIO 里的 IOBuffer 填充数据。
  if (act_on <= 0) {
    if (ntodo > 0) {
      // Notify the continuation that we are "disabling"
      //  ourselves due to to nothing to write
      write_state.vio._cont->handleEvent(VC_EVENT_WRITE_READY, &write_state.vio);
    }
    // BUG？：在回调 VC_EVENT_WRITE_READY 事件之后，应该重新计算 bytes_avail，然后继续数据传输过程。
    // 这里直接返回，那么就要等到下一次 PluginVC::main_handler() 被回调的时候，才能完成数据传输。
    return;
  }
  // Bytes available, try to transfer to the PluginVCCore
  //   intermediate buffer
  //
  /* 临时保存在 PluginVCCore 缓冲区内的最大数据量为 32 KB (PVC_DEFAULT_MAX_BYTES)。
   *
   * 当一侧的 PluginVC 向另外一侧的 PluginVC 传输数据时，总是先将数据放入这个缓冲区，然后再等待另一次将数据取走。
   * 为了避免该缓冲区保存的数据量过大，因此 PluginVCCore 将此缓冲区的最大值进行了上述限定。
   * 因此一个 PluginVCCore 的缓冲区最大占用 32 KB * 2 = 64 KB （双向数据传输）。
   *
   * 以下代码判断该缓冲区是否达到限定的 32 KB，如果已经达到该限定，则不进行任何的数据传输，直接返回。
   */
  int64_t buf_space = PVC_DEFAULT_MAX_BYTES - core_buffer->max_read_avail();
  if (buf_space <= 0) {
    Debug("pvc", "[%u] %s: process_write_side no buffer space", core_obj->id, PVC_TYPE);
    return;
  }
  // 计算实际要传输的数据量，以确保在数据传输后，仍然不超过上述缓冲区的最大限定。
  act_on = MIN(act_on, buf_space);

  // 数据传输
  int64_t added = transfer_bytes(core_buffer, reader, act_on);
  if (added < 0) {
    // Couldn't actually get the buffer space.  This only
    //   happens on small transfers with the above
    //   PVC_DEFAULT_MAX_BYTES factor doesn't apply
    Debug("pvc", "[%u] %s: process_write_side out of buffer space", core_obj->id, PVC_TYPE);
    return;
  }

  write_state.vio.ndone += added;

  Debug("pvc", "[%u] %s: process_write_side; added %" PRId64 "", core_obj->id, PVC_TYPE, added);

  // 根据 ntodo 确定回调 READY 或 COMPLETE 事件
  if (write_state.vio.ntodo() == 0) {
    write_state.vio._cont->handleEvent(VC_EVENT_WRITE_COMPLETE, &write_state.vio);
  } else {
    write_state.vio._cont->handleEvent(VC_EVENT_WRITE_READY, &write_state.vio);
  }

  // 刷新 inactive timeout 类似 net_activity()
  update_inactive_time();

  // Wake up the read side on the other side to process these bytes
  // 由于向 core buffer 内写入了新的数据，尽可能通知 对端 来读取数据
  if (!other_side->closed) {
    if (!other_side_call) {
      // 来自本地 PluginVC::main_handler
      // 通过对端的 process_read_side(true) 完成数据读取
      /* 此处并不满足调用 process_read_side 的要求，可参考 PR #4746 进行改进：
       *
       *   - https://github.com/apache/trafficserver/pull/4746
       */
      other_side->process_read_side(true);
    } else {
      // 来自对端 PluginVC::main_handler
      // 此时对端的 read_state.vio.mutex 是上锁的
      // 因此, 可以安全的执行 reenable() 通过 sm_lock_retry_event 进行回调
      other_side->read_state.vio.reenable();
    }
  }
}
```

## 参考资料

- TSHttpConnectWithPluginId
- [Plugin.h](http://github.com/apache/trafficserver/tree/master/proxy/Plugin.h)
- [PluginVC.h](http://github.com/apache/trafficserver/tree/master/proxy/PluginVC.h)
- [PluginVC.cc](http://github.com/apache/trafficserver/tree/master/proxy/PluginVC.cc)
- [Pull Request #4746](https://github.com/apache/trafficserver/pull/4746)
