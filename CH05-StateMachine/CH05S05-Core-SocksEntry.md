# 核心组件：SocksEntry

SocksEntry 是ATS作为Socks Client向Socks代理服务器发起Socks请求的状态机。

当 上层SM 调用 netProcessor.connect_re() 的时候，根据 socks 的配置判断出是否需要通过 Socks代理服务器 发起到目标IP和端口的连接，但是这一切对 上层SM 都是透明的。

## 定义

```
// 与SSL协议一样，Socks协议可以理解为一种传输层的协议
//   区别是Socks只有握手过程，但是没有通信加解密功能
// 因此，这里也像SSLNetVConnection一样定义了SocksNetVC
// 不过由于Socks协议不需要提供通信加解密功能，
// 所以，它可以与UnixNetVConnection共享net_read_io和write_to_net_io
typedef UnixNetVConnection SocksNetVC;

struct SocksEntry : public Continuation {
  // 用来保存发送给 Socks代理服务器 的Socks请求
  // 同时用来接收自 Socks代理服务器 返回的Socks响应
  MIOBuffer *buf;
  IOBufferReader *reader;

  // 由 connect_re() 创建的用来与 Socks代理服务器连接的 NetVC
  SocksNetVC *netVConnection;

  // Changed from @a ip and @a port.
  // 保存目标的IP地址和端口，将通过Socks代理服务器来访问该目标
  IpEndpoint target_addr; ///< Original target address.
  // Changed from @a server_ip, @a server_port.
  // 保存Socks代理服务器的IP地址和端口（下面的英文注释是错误的）
  IpEndpoint server_addr; ///< Origin server address.

  // 用来保存当前尝试与 Socks代理服务器 连接的次数
  int nattempts;

  // 在Socks握手完成后，会回调该Action连接的状态机
  Action action_;
  // 在与 Socks代理服务器 建立连接的所有尝试失败后，将向上述状态回调 NET_EVENT_OPEN_FAILED，并传递 -lerrno 值
  int lerrno;
  // 通过 schedule_*() 方法设置超时时间，保存其返回值
  // 如遇到 Socks 超时，SocksEntry 会收到 EVENT_INTERVAL 事件
  //   注意：如遇到 NetVConnection 超时，仍然会收到 EVENT_*_TIMEOUT 事件
  Event *timeout;
  // 与 Socks代理服务器 通信时采用的协议版本
  unsigned char version;

  // 表示ATS与Socks代理服务器正处于握手过程中，
  // true: ATS已经向Socks代理服务器发出了请求
  bool write_done;

  // 用于实现Socks5认证的函数指针，Socks4协议时为NULL
  SocksAuthHandler auth_handler;
  // 用于表示发起的Socks请求中的Socks命令字
  // 如果其值为 NORMAL_SOCKS，则表示 CONNECT 命令，也是SocksEntry能够支持的
  // 如果其值不为 NORMAL_SOCKS，ATS 则会在发送了Socks请求之后与目标服务器建立 Blind Tunnel
  unsigned char socks_cmd;

  // socks server selection:
  // 如果有多个 Socks代理服务器可以使用时，将逐个尝试
  ParentConfigParams *server_params;
  HttpRequestData req_data; // We dont use any http specific fields.
  ParentResult server_result;

  // 初始化回调函数
  int startEvent(int event, void *data);
  // 主处理回调函数
  int mainEvent(int event, void *data);
  // 当存在多个可用的 Socks代理服务器 时，通过findServer选择下一个
  void findServer();
  // 初始化 SocksEntry 实例
  void init(ProxyMutex *m, SocksNetVC *netvc, unsigned char socks_support, unsigned char ver);
  // 回调上层状态机 连接成功 或 失败，然后释放SocksEntry
  void free();

  // 构造函数
  SocksEntry()
    : Continuation(NULL),
      netVConnection(0),
      nattempts(0),
      lerrno(0),
      timeout(0),
      version(5),
      write_done(false),
      auth_handler(NULL),
      socks_cmd(NORMAL_SOCKS)
  {
    memset(&target_addr, 0, sizeof(target_addr));
    memset(&server_addr, 0, sizeof(server_addr));
  }
};

typedef int (SocksEntry::*SocksEntryHandler)(int, void *);

extern ClassAllocator<SocksEntry> socksAllocator;
```

## 方法

SocksEntry::init(mutex, vc, socks_support, ver)

- 用于初始化 SocksEntry 对象
  - 初始化用于建立Socks会话的交互缓冲区
  - 设置Socks协议版本
  - 初始回调函数为 startEvent
  - 获得目标IP和端口
  - 设置一个定时回调，实现对 Socks代理服务器 连接建立超时的管理
- 参数 mutex 为发起该连接的状态机的 mutex
  - 也就是 netProcessor.connect_re() 调用者的 mutex
- 参数 vc 为已经设置了目标IP和端口，即将发起 connect() 调用的 SocksNetVC
  - 该方法从vc中获得目标IP和端口并保存在 target_addr 成员
  - 然后将选中的 Socks 代理服务器的IP和端口保存在 server_addr 成员
- 参数 socks_support 默认为 NORMAL_SOCKS
  - 该参数主要用于重新建立到 Socks 代理服务器的连接，例如：在连接超时，多个Socks代理服务器可用时
  - 设置 options.socks_support 为 NO_SOCKS 后，调用 connect_re() 方法就不会再进入到 SocksEntry
- 参数 ver 表示 Socks 协议的版本
  - 默认为 (unisgned char) 5，表示采用 Socks 5 协议与 Socks代理服务器 建立通信
- 作为 init() 方法的调用者，将会使用 server_addr 成员替代 vc 的目标IP和端口
  - 这样在稍后发起的 connect() 调用，就会与 Socks 代理服务器建立 TCP 连接

```
void
SocksEntry::init(ProxyMutex *m, SocksNetVC *vc, unsigned char socks_support, unsigned char ver)
{
  // 初始化SocksEntry的mutex，与调用者共享同一个 mutex
  mutex = m;
  // 创建会话交互缓冲区
  buf = new_MIOBuffer();
  reader = buf->alloc_reader();

  // 设置 socks_cmd 默认值，通常为 NORMAL_SOCKS(0)
  socks_cmd = socks_support;

  // 设置所采用的 Socks 协议的版本
  if (ver == SOCKS_DEFAULT_VERSION)
    version = netProcessor.socks_conf_stuff->default_version;
  else
    version = ver;

  // 设置事件回调函数为 startEvent
  SET_HANDLER(&SocksEntry::startEvent);

  // 这里在 6.0.x 分支上存在bug，应该为 vc->get_remote_addr()
  // 从 vc 上获得目标IP和端口，保存在 target_addr 成员
  // 将来构造 Socks 请求时，通过 target_addr 成员构造目标IP和端口
  ats_ip_copy(&target_addr, vc->get_local_addr());

// 该宏在 I_Socks.h 中定义
#ifdef SOCKS_WITH_TS
  // 初始化 req_data，在调用 findParent() 方法查找 socks.config 规则时使用
  req_data.hdr = 0;
  req_data.hostname_str = 0;
  req_data.api_info = 0;
  req_data.xact_start = time(0);

  assert(ats_is_ip4(&target_addr));
  // 保存目标IP和端口到 req_data.dest_ip
  ats_ip_copy(&req_data.dest_ip, &target_addr);

  // we dont have information about the source. set to destination's
  // 使用目标IP和端口初始化 req_data.src_ip 只是为了保证此处有一个合法的信息
  ats_ip_copy(&req_data.src_ip, &target_addr);

  // 通过Socks配置文件，加载可用的 Socks 代理服务器
  server_params = SocksServerConfig::acquire();
#endif

  // 重置尝试次数，然后开始查找可用的 Socks 代理服务器
  nattempts = 0;
  findServer();

  // 创建一个定时回调，在 n 秒之后回调 SocksEntry::startEvent 判定超时情况
  timeout = this_ethread()->schedule_in(this, HRTIME_SECONDS(netProcessor.socks_conf_stuff->server_connect_timeout));
  write_done = false;
}
```

SocksEntry::findServer()

- 用来找到可以个可用的 Socks代理服务器
- 在 socks.config 中可以为特定的目标指定 Socks代理服务器，这样可以根据不同的用途使用不同的 Socks代理服务器
- 如果在 socks.config 中没有找到，还可以从 records.config 中查找 proxy.config.socks.default_servers
- 在与 Socks代理服务器 尝试建立TCP连接时，
  - 可以设置最大的重试次数（per_server_connection_attempts）
  - 还可以设置总的重试次数（connection_attempts）
    - 避免可用 Socks代理服务器 较多时，反复重试导致连接超时
    - 或者将此值设置为: 可用Socks代理服务器数量 * per_server_connection_attempts，以保证尝试所有的 Socks代理服务器

```
void
SocksEntry::findServer()
{
  nattempts++;

#ifdef SOCKS_WITH_TS
  if (nattempts == 1) {
    ink_assert(server_result.r == PARENT_UNDEFINED);
    // 寻找一个可用的 Socks代理服务器，
    server_params->findParent(&req_data, &server_result);
  } else {
    socks_conf_struct *conf = netProcessor.socks_conf_stuff;
    // 如果不是第一次尝试，那么要判断是否达到了重试次数上限
    //     如果未达上限，直接返回，这样会再次尝试与该 Socks代理服务器 建立连接
    // 参考: proxy.config.socks.per_server_connection_attempts
    if ((nattempts - 1) % conf->per_server_connection_attempts)
      return; // attempt again

    // 达到了重试次数上限
    // 设置该 Socks代理服务器 为down的状态
    server_params->markParentDown(&server_result);

    // 参考: proxy.config.socks.connection_attempts
    if (nattempts > conf->connection_attempts)
      server_result.r = PARENT_FAIL;
    else
      server_params->nextParent(&req_data, &server_result);
  }

  // 对于 ParentSelection 相关的分析这里暂时不展开介绍。
  // server_result.r == PARENT_DIRECT: 表示未匹配中规则，同时 default_servers 没有设置
  // server_result.r == PARENT_SPECIFIED: 表示找到了一个可用的 Socks代理服务器
  // server_result.r == PARENT_FAIL: 表示虽然匹配了一条 socks.config 中的规则，但是该规则没有配置有效的 Socks代理服务器
  switch (server_result.r) {
  case PARENT_SPECIFIED:
    // Original was inet_addr, but should hostnames work?
    // ats_ip_pton only supports numeric (because other clients
    // explicitly want to avoid hostname lookups).
    // 在 socks.config 中配置 Socks代理服务器 的规则时，如果填写了域名，而不是IP的话，
    //     在 setup_socks_servers() 加载规则的过程中，会把域名解析为IP地址
    //     解析失败时，会把 hostname 设置为 255.255.255.255 这个地址
    // 因此这里使用 pton 方法把 hostname 直接写入到 server_addr
    //     由于存在上述加载配置时的错误处理策略，因此 pton 方法应该不会失败
    if (0 == ats_ip_pton(server_result.hostname, &server_addr)) {
      ats_ip_port_cast(&server_addr) = htons(server_result.port);
    } else {
      Debug("SocksParent", "Invalid parent server specified %s", server_result.hostname);
    }
    break;

  default:
    ink_assert(!"Unexpected event");
  case PARENT_DIRECT:
  case PARENT_FAIL:
    // 在没有找到可用的 Socks代理服务器 时，将 server_addr 清零
    // 我认为这里应该使用 target_addr 恢复 server_addr 让 ATS 可以尝试直接访问目标IP和端口
    memset(&server_addr, 0, sizeof(server_addr));
  }
#else
  if (nattempts > netProcessor.socks_conf_stuff->connection_attempts)
    memset(&server_addr, 0, sizeof(server_addr));
  else
    ats_ip_copy(&server_addr, &g_socks_conf_stuff->server_addr);
#endif // SOCKS_WITH_TS

  char buff[INET6_ADDRSTRLEN];
  Debug("SocksParents", "findServer result: %s:%d", ats_ip_ntop(&server_addr.sa, buff, sizeof(buff)),
        ats_ip_port_host_order(&server_addr));
}
```

SocksEntry::free()

- 与 Socks代理服务器 建立 Socks 通信之后 或 遇到任何错误 通过调用 free() 方法通知状态机，然后释放 SocksEntry 占用的内存。
- 需要取消 超时事件
- 需要在出错时关闭 netVC
- 需要考虑状态机已经关闭的情况（如状态机已经关闭会通过Action对象调用cancel方法）
- 需要重置 do_io 操作，然后才能释放 SocksEntry 对象

```
void
SocksEntry::free()
{
  MUTEX_TRY_LOCK(lock, action_.mutex, this_ethread());
  // 在 6.0.x 的版本里 lock 前面少了一个感叹号。
  // 这个bug是在 53e56ffc7499648bc096d63f9c0f9da0ea9212ba 修复 TS-3156 时产生的
  if (!lock.is_locked()) {
    // 这里认为调用 free 方法之前必然已经持有 action_.mutex 的锁了，
    //   如果不能立即上锁成功，则是出现了异常
    // 我认为这里应该是早期的代码存在其它的bug导致问题，在这里加锁以避免问题产生
    //   所以我觉得这里应该完全没有必要上锁，只需要判断 mutex->thread_holding == this_ethread()
    // Socks continuation share the user's lock
    // so acquiring a lock shouldn't fail
    ink_assert(0);
    return;
  }

  // 取消之前设置的超时
  // 这里应该在 cancel 之后，将 timeout 赋值为 NULL
  if (timeout)
    timeout->cancel(this);

#ifdef SOCKS_WITH_TS
  if (!lerrno && netVConnection && server_result.retry)
    server_params->recordRetrySuccess(&server_result);
#endif

  // 下面的逻辑有些混乱
  // 如果被cancel或者出错时，关闭 netVC
  //   这里应该在关闭netVC后设置 netVC 为 NULL
  if ((action_.cancelled || lerrno) && netVConnection)
    netVConnection->do_io_close();

  // 如果没有被cancel，那么需要回调状态机
  if (!action_.cancelled) {
    // 如果上面把 netVC 设置为 NULL，这里就完全可以根据 netVC 是否为 NULL 对状态机回调不同的事件。
    if (lerrno || !netVConnection) {
      // 如果出错 或 netVC 为 NULL，则回调 OPEN_FAILED 并传递错误信息
      Debug("Socks", "retryevent: Sent errno %d to HTTP", lerrno);
      NET_INCREMENT_DYN_STAT(socks_connections_unsuccessful_stat);
      action_.continuation->handleEvent(NET_EVENT_OPEN_FAILED, (void *)(intptr_t)(-lerrno));
    } else {
      // 否则要回调 OPEN 并且传递 netVC 给状态机
      // 说明成功建立了 Socks 通信
      // 这里的 do_io 重设存在问题，不应该使用 this，应该使用 NULL
      netVConnection->do_io_read(this, 0, 0);
      netVConnection->do_io_write(this, 0, 0);
      netVConnection->action_ = action_; // assign the original continuation
      // 这里猜测是想把 target_addr 复制回 netVC->server_addr 以实现完全透明的通过 SocksEntry 建立连接
      // 但是这儿写成了 server_addr
      ats_ip_copy(&netVConnection->server_addr, &server_addr);
      Debug("Socks", "Sent success to HTTP");
      NET_INCREMENT_DYN_STAT(socks_connections_successful_stat);
      action_.continuation->handleEvent(NET_EVENT_OPEN, netVConnection);
    } 
  }
  // 回收对资源的引用
#ifdef SOCKS_WITH_TS
  SocksServerConfig::release(server_params);
#endif

  // 释放资源
  free_MIOBuffer(buf);
  // 回收 action_->mutex 的引用计数
  action_ = NULL;
  // 回收 this->mutex 的引用计数
  mutex = NULL;
  // 回收 SocksEntry 对象
  socksAllocator.free(this);
}
```

## 参考资料

- [P_Socks.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_Socks.h)
- [Socks.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/Socks.cc)
- [ParentSelection.cc](http://github.com/apache/trafficserver/tree/master/proxy/ParentSelection.cc)
