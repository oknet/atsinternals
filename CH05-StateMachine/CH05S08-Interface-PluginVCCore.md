# 接口：PluginVCCore 和 API

PluginVC 可用于两个状态机之间进行数据传输，但是大多数情况都是用于 HttpSM 与 Plugin 之间的数据传输。Plugin 可作为虚拟的客户端与 HttpSM 建立通信，也可以作为虚拟的服务端与 HttpSM 建立通信，这取决于调用了哪个 API。

在 ATS 的插件开发库中，通过调用 PluginVCCore 提供的接口实现了，实现了以下五个 API：

- Plugin 作为客户端
  - TSHttpConnectWithPluginId
  - TSHttpConnect
  - TSHttpConnectTransparent
- Plugin 作为服务端
  - TSHttpTxnServerIntercept
  - TSHttpTxnIntercept

在 PluginVCCore 的设计里，如果使用回调方式向目标状态机传递 PluginVC 对象时，其总是通过

- `NET_EVENT_ACCEPT` 事件向目标状态机传递 `passive_vc`
- `NET_EVENT_OPEN` 事件向目标状态机传递 `active_vc`

当 Plugin 作为客户端时，通常通过 API 的返回值获得 `active_vc`，当 Plugin 作为服务端时，通常需要等待 `NET_EVENT_ACCEPT` 事件的回调，才可以获得 `passive_vc`。

## 定义

PluginVCCore 的对外提供的接口与 NetProcessor 基本一致，但是其定位与 NetHandler 又比较接近。下面我们来看一下它的定义：

```
class PluginVCCore : public Continuation
{
  friend class PluginVC;

public:
  // 构造函数
  //   - 将 active_addr_struct 和 passive_addr_struct 清空
  //   - 对 nextid 执行原子增量操作，用于统计创建了多少个 PluginVCCore
  //   - 同时将 nextid 的原值赋值给 id 作为当前 PluginVCCore的编号
  PluginVCCore();
  // 析构函数（空）
  ~PluginVCCore();

  // 静态方法，通过 new 创建 PluginVCCore 对象，并调用 init() 方法完成初始化
  // 与 NetProcessor 的 allocate_vc() 功能一致
  static PluginVCCore *alloc();
  // 初始化 PluginVCCore 以及其内包含的两个 PluginVC 对象
  void init();
  // 设置用于接收 passive_vc 的状态机，该状态机的地址保存在成员 connect_to
  // PluginVCCore 会向该状态机回调 NET_EVENT_ACCEPT 事件，同时传递 passive_vc
  void set_accept_cont(Continuation *c);

  // 向 connect_to 指向的状态机回调事件：
  //   - NET_EVENT_ACCEPT
  //   - NET_EVENT_ACCEPT_FAILED
  // 这里声明为状态机函数，是因为在回调前需要对 connect_to->mutex 上锁，
  // 如果上锁失败会重新调度，最终完成向 connect_to 回调事件并传递 passive_vc 的功能
  int state_send_accept(int event, void *data);
  int state_send_accept_failed(int event, void *data);

  // 尝试回收 PluginVCCore 和 PluginVC 占用的资源并释放内存
  void attempt_delete();

  // 调用 state_send_accept，然后返回 active_vc
  // 会将成员 connected 设置为 true，表示两端的 PluginVC 完成了连接的建立
  PluginVC *connect();
  // 模拟了 netProcessor.connect_re() 的设计
  // 调用 state_send_accept，然后以回调 c 的方式向 c 传递 active_vc
  // 永远返回 ACTION_RESULT_DONE
  // 会将成员 connected 设置为 true，表示两端的 PluginVC 完成了连接的建立
  Action *connect_re(Continuation *c);
  // 专用于销毁两端的 PluginVC 未完成连接（connected == false）的 PluginVCCore 实例
  void kill_no_connect();

  // 设置 active_vc 和 passive_vc 的虚拟 IP 地址
  // PluginVC 是伪装成 NetVC，因此需要为 active_vc 和 passive_vc 填写一个 IP 地址
  /// Set the active address.
  void set_active_addr(in_addr_t ip, ///< IPv4 address in host order.
                       int port      ///< IP Port in host order.
                       );
  /// Set the active address and port.
  void set_active_addr(sockaddr const *ip ///< Address and port used.
                       );
  /// Set the passive address.
  void set_passive_addr(in_addr_t ip, ///< IPv4 address in host order.
                        int port      ///< IP port in host order.
                        );
  /// Set the passive address.
  void set_passive_addr(sockaddr const *ip ///< Address and port.
                        );

  // 保存自定义数据到 vc 上，目前未使用，
  // 而是由 PluginVC::set_data 和 PluginVC::get_data 直接存取成员 active_data 和 passive_data
  void set_active_data(void *data);
  void set_passive_data(void *data);

  // 分别调用 active_vc 和 passive_vc 的 set_is_transparent() 设置 TCP 透明传输
  void set_transparent(bool passive_side, bool active_side);

  // 用于同时设置 active_vc 和 passive_vc 的 plugin_id 和 plugin_tag
  /// Set the plugin ID for the internal VCs.
  void set_plugin_id(int64_t id);
  /// Set the plugin tag for the internal VCs.
  void set_plugin_tag(char const *tag);

  // PluginVCCore 内包含的两个 PluginVC 成员，注意这里不是指针对象。
  // The active vc is handed to the initiator of
  //   connection.  The passive vc is handled to
  //   receiver of the connection
  PluginVC active_vc;
  PluginVC passive_vc;

private:
  // 私有方法及成员
  // 回收 PluginVCCore 和 两个 PluginVC 占用的资源，并释放内存
  void destroy();

  // 通过 set_accept_cont() 方法设置
  Continuation *connect_to;
  // 表示是否将 passive_vc 传递给了目标状态机
  // 也可以理解为 两个 PluginVC 成功的将状态机与 Plugin 连接在了一起
  bool connected;

  // 由 Passive 端向 Active 端发送数据时，数据会先缓存在 p_to_a_buffer，
  // 然后 Active 端可以通过 p_to_a_reader 读取这些数据。
  MIOBuffer *p_to_a_buffer;
  IOBufferReader *p_to_a_reader;

  // 由 Active 端向 Passive 端发送数据时，数据会先缓存在 a_to_p_buffer，
  // 然后 Passive 端可以通过 a_to_p_reader 读取这些数据。
  MIOBuffer *a_to_p_buffer;
  IOBufferReader *a_to_p_reader;

  // 用来保存虚拟的 passive 和 active 端的 IP 地址及端口
  IpEndpoint passive_addr_struct;
  IpEndpoint active_addr_struct;

  // 用来保存 passive 和 active 端的自定义数据
  // 用途不明
  void *passive_data;
  void *active_data;

  // 用于统计创建了多少个 PluginVCCore 对象，可以拿出来定义成全局变量
  static vint32 nextid;
  // 用于保存当前 PluginVCCore 的 ID 编号
  unsigned id;
};
```

## 方法

### PluginVCCore::connect

向目标状态机发起连接（可以看做是模拟了 TCP 握手过程），目标状态机会收到 `NET_EVENT_ACCEPT` 和 `passive_vc`，最后向调用者返回 `active_vc`。

```
PluginVC *
PluginVCCore::connect()
{
  // Make sure there is another end to connect to
  if (connect_to == NULL) {
    return NULL;
  }

  connected = true;
  // 调用 state_send_accept 向目标状态机回调 `NET_EVENT_ACCEPT` 事件和 `passive_vc`
  state_send_accept(EVENT_IMMEDIATE, NULL);

  return &active_vc;
}
```

### PluginVCCore::connect_re

该方法的实现参考了 `netProcessor.connect_re()`，其行为表现也几乎一致。

- 调用者通常将自身作为 Continuation 传入
- 通过 `state_send_accept` 回调 Plugin 状态机 `NET_EVENT_ACCEPT` 事件和 `passive_vc`
- 向传入的 `c` 回调 `NET_EVENT_OPEN` 事件和 `active_vc`

```
Action *
PluginVCCore::connect_re(Continuation *c)
{
  // Make sure there is another end to connect to
  if (connect_to == NULL) {
    return NULL;
  }

  EThread *my_thread = this_ethread();
  // 这里使用了 MUTEX_TAKE_LOCK，根据下面的注释，这里应该总是会上锁成功，而且不会产生阻塞
  MUTEX_TAKE_LOCK(this->mutex, my_thread);

  connected = true;
  state_send_accept(EVENT_IMMEDIATE, NULL);

  // We have to take out our mutex because rest of the
  //   system expects the VC mutex to held when calling back.
  // We can use take lock here instead of try lock because the
  //   lock should never already be held.

  c->handleEvent(NET_EVENT_OPEN, &active_vc);
  MUTEX_UNTAKE_LOCK(this->mutex, my_thread);

  return ACTION_RESULT_DONE;
}
```

### PluginVCCore::state\_send\_accept

向 Passive 端的状态机回调 `NET_EVENT_ACCEPT` 事件和 `passive_vc`。

由于在回调之前需要对状态机先上锁，然后才可以进行回调，因此这里设计成回调函数，一旦上锁失败可以通过事件系统重新调度。

```
int
PluginVCCore::state_send_accept(int /* event ATS_UNUSED */, void * /* data ATS_UNUSED */)
{
  MUTEX_TRY_LOCK(lock, connect_to->mutex, this_ethread());

  if (lock.is_locked()) {
    connect_to->handleEvent(NET_EVENT_ACCEPT, &passive_vc);
  } else {
    SET_HANDLER(&PluginVCCore::state_send_accept);
    eventProcessor.schedule_in(this, PVC_LOCK_RETRY_TIME);
  }

  return 0;
}
```

### PluginVCCore::state\_send\_accept\_failed

与 `PluginVCCore::state_send_accept` 一样，只是回调的事件类型不同。

```
int
PluginVCCore::state_send_accept_failed(int /* event ATS_UNUSED */, void * /* data ATS_UNUSED */)
{
  MUTEX_TRY_LOCK(lock, connect_to->mutex, this_ethread());

  if (lock.is_locked()) {
    connect_to->handleEvent(NET_EVENT_ACCEPT_FAILED, NULL);
    destroy();
  } else {
    SET_HANDLER(&PluginVCCore::state_send_accept_failed);
    eventProcessor.schedule_in(this, PVC_LOCK_RETRY_TIME);
  }

  return 0;
}
```

### HttpSessionAccept 无锁化

在第四章节里，介绍 HttpSessionAccept 对象时，提到了所有的 XXXSessionAccept 的 mutex 都应该指向 NULL，这样才能够并发的接收新连接，并创建新的 HttpSM 状态机。

延伸开来，所有会接受 `NET_EVENT_ACCEPT` 事件的状态机，其 mutex 都应该指向 NULL，并且向状态机回调 `NET_EVENT_ACCEPT` 事件之前，并不需要对目标状态机进行上锁操作。

但是可以看到，在 `PluginVCCore::state_send_accept` 和 `PluginVCCore::state_send_accept` 方法的实现里，都先对目标状态机进行上锁操作之后，才回调了目标状态机。

这样就导致了创建新会话的操作变成了串行化，由此导致涉及到 PluginVC 应用时的性能非常差。

那么为什么 PluginVC 会这么设计呢？下面我以 `stats_over_http` 插件为例，解析这样设计的原因。

首先找到插件里调用 API 的位置，可以看到负责接收 `NET_EVENT_ACCEPT` 事件和 `passive_vc` 的是 `stats_dostuff()`。

```
source: plugins/stats_over_http/stats_over_http.c
static int                                                                                                                           
stats_origin(TSCont contp ATS_UNUSED, TSEvent event ATS_UNUSED, void *edata)                                                            
{
...
  TSDebug(PLUGIN_NAME, "Intercepting request");                                                                                         
                                                                                                                                        
  icontp = TSContCreate(stats_dostuff, TSMutexCreate());                                                                                
  my_state = (stats_state *)TSmalloc(sizeof(*my_state));                                                                                
  memset(my_state, 0, sizeof(*my_state));                                                                                               
  TSContDataSet(icontp, my_state);                                                                                                      
  TSHttpTxnIntercept(icontp, txnp);                                                                                                     
  goto cleanup;
...
}
```

然后再来看 `stats_dostuff()` 的实现，可以看到这个函数里不光要处理 `NET_EVENT_ACCEPT` 事件，还要处理 VIO 的读写。

```
source: plugins/stats_over_http/stats_over_http.c
static int                                                                                                                              
stats_dostuff(TSCont contp, TSEvent event, void *edata)                                                                                 
{                                                                                                                                       
  stats_state *my_state = TSContDataGet(contp);                                                                                         
  if (event == TS_EVENT_NET_ACCEPT) {                                                                                                   
    my_state->net_vc = (TSVConn)edata;                                                                                                  
    stats_process_accept(contp, my_state);                                                                                              
  } else if (edata == my_state->read_vio) {                                                                                             
    stats_process_read(contp, event, my_state);                                                                                         
  } else if (edata == my_state->write_vio) {                                                                                            
    stats_process_write(contp, event, my_state);                                                                                        
  } else {                                                                                                                              
    TSReleaseAssert(!"Unexpected Event");                                                                                               
  }                                                                                                                                     
  return 0;                                                                                                                             
}  
```

可以看到在 `stats_over_http` 插件里，没有独立的 XXXSessionAccept 函数，而是使用 `stats_dostuff()` 这一个函数实现了 HttpSessionAccept 和 HttpSM 的所有功能。

对于 PluginVC 的性能问题，可以找到 TS-4924 和 TS-4955 两个官方的 Issue，最终在 Pull Request #1096 通过修改 `PluginVCCore::state_send_accept` 和 `PluginVCCore::state_send_accept` 两个方法解决了 PluginVC 的瓶颈问题，但是 `stats_over_http` 插件的性能问题仍然是存在的。

参考：

- [TS-4924](https://issues.apache.org/jira/browse/TS-4924)
- [TS-4955](https://issues.apache.org/jira/browse/TS-4955)
- [Pull Request #1096](https://github.com/apache/trafficserver/pull/1096)
  - [Commit 3526b772](https://github.com/apache/trafficserver/commit/3526b772efaa644e1bff1853d818b5792613e145)

## API - Plugin 作为客户端

### TSHttpConnectWithPluginId

由 Plugin 发起，Plugin 控制 Active 端，HttpSM 控制 Passive 端

- Plugin 需要向 active_vc 写入合法的 HTTP 请求
- HttpSM 将会对该请求进行解析，就像从通常的 TCP 客户端那里收到了 HTTP 请求一样
- 但是 HttpSM 会认为该请求是一个内部请求(Internal Request)，在某些逻辑处理上稍微有差别

参数：

- addr
  - 用于模拟客户端的 IP 地址和端口
- tag, id
  - 可以在日志中输出 tag 和 id 的信息，分别使用：`%<pitag>` 和 `%<piid>`


```
TSVConn
TSHttpConnectWithPluginId(sockaddr const *addr, char const *tag, int64_t id)
{
  sdk_assert(addr);

  sdk_assert(ats_is_ip(addr));
  sdk_assert(ats_ip_port_cast(addr));

  // plugin_http_accept 是一个全局的 HttpSessionAccept 对象
  if (plugin_http_accept) {
    // 创建 PluginVCCore
    PluginVCCore *new_pvc = PluginVCCore::alloc();

    // 将客户端地址填入 Active 端
    new_pvc->set_active_addr(addr);
    // 设置 plugin id 和 tag
    new_pvc->set_plugin_id(id);
    new_pvc->set_plugin_tag(tag);
    // 设置目标状态机，这里要跟 HttpSM 建立连接，因此填入 HttpSessionAccept 类型的对象
    new_pvc->set_accept_cont(plugin_http_accept);

    // 向目标状态机发起连接（模拟 TCP 握手）
    // 目标状态机会收到 NET_EVENT_ACCEPT 和 passive_vc
    // 并返回 active_vc
    PluginVC *return_vc = new_pvc->connect();

    // 判断 active_vc 不为 NULL，需要把 passive_vc 设置为内部请求
    if (return_vc != NULL) {
      PluginVC *other_side = return_vc->get_other_side();

      if (other_side != NULL) {
        other_side->set_is_internal_request(true);
      }
    }

    // 将 active_vc 返回给 Plugin
    return reinterpret_cast<TSVConn>(return_vc);
  }

  return NULL;
}
```

### TSHttpConnect

与 TSHttpConnectWithPluginId 相同，只是传入了默认的 tag 和 id 的值。

```
TSVConn
TSHttpConnect(sockaddr const *addr)
{
  return TSHttpConnectWithPluginId(addr, "plugin", 0);
}
```

TSHttpConnectTransparent

使用 PluginVC 模拟了透明代理的访问，同时设置了虚拟的客户端地址，同时也设置了目标服务器的地址。

```
TSVConn
TSHttpConnectTransparent(sockaddr const *client_addr, sockaddr const *server_addr)
{
  sdk_assert(ats_is_ip(client_addr));
  sdk_assert(ats_is_ip(server_addr));
  sdk_assert(!ats_is_ip_any(client_addr));
  sdk_assert(ats_ip_port_cast(client_addr));
  sdk_assert(!ats_is_ip_any(server_addr));
  sdk_assert(ats_ip_port_cast(server_addr));

  // plugin_http_transparent_accept 是一个全局的 HttpSessionAccept 对象
  // 与 plugin_http_accept 不同的是，它的 HttpSessionAcceptOptions 设置了 f_outbound_transparent 为 true
  if (plugin_http_transparent_accept) {
    // 创建 PluginVCCore
    PluginVCCore *new_pvc = PluginVCCore::alloc();

    // set active address expects host ordering and the above casts do not
    // swap when it is required
    // 将客户端地址填入 Active 端，将服务端地址填入 Passive 端
    new_pvc->set_active_addr(client_addr);
    new_pvc->set_passive_addr(server_addr);
    // 设置 Active 和 Passive 两端都是是透明模式
    new_pvc->set_transparent(true, true);
    // 设置目标状态机，这里要跟 HttpSM 建立连接，因此填入 HttpSessionAccept 类型的对象
    new_pvc->set_accept_cont(plugin_http_transparent_accept);

    // 向目标状态机发起连接（模拟 TCP 握手）
    // 目标状态机会收到 NET_EVENT_ACCEPT 和 passive_vc
    // 并返回 active_vc
    PluginVC *return_vc = new_pvc->connect();

    // 判断 active_vc 不为 NULL，需要把 passive_vc 设置为内部请求
    if (return_vc != NULL) {
      PluginVC *other_side = return_vc->get_other_side();

      if (other_side != NULL) {
        other_side->set_is_internal_request(true);
      }
    }

    // 将 active_vc 返回给 Plugin
    return reinterpret_cast<TSVConn>(return_vc);
  }

  return NULL;
}
```


## API - Plugin 作为服务端

### TSHttpTxnServerIntercept

由 Plugin 发起，HttpSM 控制 Active 端，Plugin 控制 Passive 端

- HttpSM 将 `active_vc` 看做 `server_vc`，并向 `active_vc` 写入合法的 HTTP 请求
- Plugin 将会作为源服务器，从 ATS 接收到该 HTTP 请求
- Plugin 需要对该请求进行解析，并作出响应
- HttpSM 会像处理源服务的响应一样处理 Plugin 做出的响应

参数：

- contp
  - plugin 端用于接收 `NET_EVENT_ACCEPT` 和 `passive_vc` 的状态机
- txnp
  - 指向 HttpSM 

该 API 可以在以下 Hook 点触发时调用：

- `TS_HTTP_TXN_START_HOOK`
- `TS_HTTP_READ_REQUEST_HDR_HOOK`
- `TS_HTTP_PRE_REMAP_HOOK`
- `TS_HTTP_POST_REMAP_HOOK`
- `TS_HTTP_OS_DNS_HOOK`
- `TS_HTTP_SELECT_ALT_HOOK`
- `TS_HTTP_READ_CACHE_HDR_HOOK`

如果 HttpSM 由于种种原因未发起到目标服务器的连接，在 Http 事务完成后，Plugin 的状态机会收到 `TS_EVENT_NET_ACCEPT_FAILED` 事件。

```
void
TSHttpTxnServerIntercept(TSCont contp, TSHttpTxn txnp)
{
  sdk_assert(sdk_sanity_check_txn(txnp) == TS_SUCCESS);
  sdk_assert(sdk_sanity_check_continuation(contp) == TS_SUCCESS);

  HttpSM *http_sm = (HttpSM *)txnp;
  INKContInternal *i = (INKContInternal *)contp;

  // Must have a mutex
  // 用于接收 NET_EVENT_ACCEPT 和 passive_vc 的状态机需要有互斥锁
  sdk_assert(sdk_sanity_check_null_ptr((void *)i->mutex) == TS_SUCCESS);

  // 告知 HttpSM，PluginVC 的类型
  http_sm->plugin_tunnel_type = HTTP_PLUGIN_AS_SERVER;
  // 将 PluginVCCore 对象保存于 HttpSM
  // HttpSM 通过调用 plugin_tunnel->connect_re() 获得 active_vc，并通过回调将 passive_vc 传递给 Plugin
  http_sm->plugin_tunnel = PluginVCCore::alloc();
  // 设置 Passive 端用于接收回调的状态机
  http_sm->plugin_tunnel->set_accept_cont(i);
}
```

### TSHttpTxnIntercept

与 TSHttpTxnServerIntercept 的实现几乎一致，唯一不同的是 `plugin_tunnel_type` 为 `HTTP_PLUGIN_AS_INTERCEPT`。

但是在 HttpSM 的处理逻辑上，却存在很大的不同：

- 该操作会跳过 Cache Lookup 的检测，
- 该操作会跳过 ACL 的检测，
- 该操作会跳过 DNS 的解析，
- 该操作会跳过 Remap 的处理。
- 对于 Plugin 发送给 HttpSM 的响应，也不会被 Cache 缓存。

该 API 可以在以下 Hook 点触发时调用：

- `TS_HTTP_TXN_START_HOOK`
- `TS_HTTP_READ_REQUEST_HDR_HOOK`

```
void
TSHttpTxnIntercept(TSCont contp, TSHttpTxn txnp)
{
  sdk_assert(sdk_sanity_check_txn(txnp) == TS_SUCCESS);
  sdk_assert(sdk_sanity_check_continuation(contp) == TS_SUCCESS);

  HttpSM *http_sm = (HttpSM *)txnp;
  INKContInternal *i = (INKContInternal *)contp;

  // Must have a mutex
  sdk_assert(sdk_sanity_check_null_ptr((void *)i->mutex) == TS_SUCCESS);

  http_sm->plugin_tunnel_type = HTTP_PLUGIN_AS_INTERCEPT;
  http_sm->plugin_tunnel = PluginVCCore::alloc();
  http_sm->plugin_tunnel->set_accept_cont(i);
}
```

## 参考资料

- [PluginVC.h](http://github.com/apache/trafficserver/tree/6.0.x/proxy/PluginVC.h)
- [PluginVC.cc](http://github.com/apache/trafficserver/tree/6.0.x/proxy/PluginVC.cc)
- [Pull Request #1096](https://github.com/apache/trafficserver/pull/1096)
