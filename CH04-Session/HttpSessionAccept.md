# 核心组件： HttpSessionAccept

在阅读此章节之前，请确保已经完全阅读和理解了：

  - [CH02: ProtocolProbeSessionAccept and Trampoline](https://github.com/oknet/atsinternals/blob/master/CH02-IOCoreNET/CH02S12-Core-ProtocolProbeSessionAccept-and-Trampoline.md)
  - [CH03: SSLNextProtocolAccept](https://github.com/oknet/atsinternals/blob/master/CH03-IOCoreSSL/CH03S05-Base-SSLNextProtocolAccept.md)
  - [CH03: SSLNextProtocolTrampoline](https://github.com/oknet/atsinternals/blob/master/CH03-IOCoreSSL/CH03S04-Base-SSLNextProtocolTrampoline.md)
  - [CH03: SessionAccept](https://github.com/oknet/atsinternals/blob/master/CH03-IOCoreSSL/CH03S02-Base-SessionAccept.md)

本章节将以Http协议的Session处理为主，介绍NetVC是如何从IOCore来到各个协议的状态机的。

## 定义

```
/**
   The continuation mutex is NULL to allow parellel accepts in NT. No
   state is recorded by the handler and values are required to be set
   during construction via the @c Options struct and never changed. So
   a NULL mutex is safe.
   注释中对于在构造函数中采用：SessionAccept(NULL) 来初始化Continuation的mutex为NULL做了说明：
      是为了支持在 NT 环境中并行接受新的Session。（在Linux/Unix中也是一样的）
   因为：
      handler不会设置、记录任何状态；
      在构造函数中通过Options结构初始化的值，在之后也永远不会改变（只读）
   所以，这里mutex设置为NULL是安全的。
   实际上在ATS中，所有的XXXSessionAccept的mutex都是可以设置为NULL的。

   Most of the state is simply passed on to the @c ClientSession after
   an accept. It is done here because this is the least bad pathway
   from the top level configuration to the HTTP session.
   
   绝大多数的设置都会在方法XXXSessionAccept::accept()中传递给XXXClientSession。
   为了把配置项传递给Http Session，这是一种不算太差的方式。
*/

class HttpSessionAccept : public SessionAccept, private detail::HttpSessionAcceptOptions
{
private:
  typedef HttpSessionAccept self; ///< Self reference type.
public:
  /** Construction options.
      Provide an easier to remember typedef for clients.
  */
  typedef detail::HttpSessionAcceptOptions Options;

  /** Default constructor.
      @internal We don't use a static default options object because of
      initialization order issues. It is important to pick up data that is read
      from the config file and a static is initialized long before that point.
  */
  // 构造函数
  //     前面已经做了说明，这里初始化基类SessionAccept；
  //     通过传入的opt来初始化各个配置项。
  HttpSessionAccept(Options const &opt = Options()) : SessionAccept(NULL), detail::HttpSessionAcceptOptions(opt) // copy these.
  {
    SET_HANDLER(&HttpSessionAccept::mainEvent);
    return;
  }

  ~HttpSessionAccept() { return; }

  // 每一个 XXXSessionAccept 都需要定义这个 accept 方法，
  // 在基类 SessionAccept 的声明中，accept 被定义为纯虚函数。
  void accept(NetVConnection *, MIOBuffer *, IOBufferReader *);
  // 通常在SSLNextProtocolTrampoline床回调时，可能会通过该Handler间接调用accept方法
  // 因为可以通过 NPN / ALPN 直接获取即将进行的通信协议的类型，从而不需要先读取一段内容进行分析（Probe）
  //   所以，不需要传递 MIOBuffer *, IOBufferReader * 到 XXXSessionAccept；
  // 相反，如果通过 Probe 分析后才能得出即将进行通信的协议类型，
  //   那么就需要调用 accept 方法把 MIOBuffer *, IOBufferReader * 传递进来。
  int mainEvent(int event, void *netvc);

private:
  HttpSessionAccept(const HttpSessionAccept &);
  HttpSessionAccept &operator=(const HttpSessionAccept &);
};
```

## 方法

```
void
HttpSessionAccept::accept(NetVConnection *netvc, MIOBuffer *iobuf, IOBufferReader *reader)
{
  sockaddr const *client_ip = netvc->get_remote_addr();
  const AclRecord *acl_record = NULL;
  ip_port_text_buffer ipb;
  IpAllow::scoped_config ipallow;

  // The backdoor port is now only bound to "localhost", so no
  // reason to check for if it's incoming from "localhost" or not.
  if (backdoor) {
    // backdoor 是ATS内部的一个服务，在最新的6.0.x分支里基本都被砍光了，只留下一个与traffic_manage的心跳探测功能
    // 对此有兴趣的可以翻看比较老的代码，自己研究一下。
    acl_record = IpAllow::AllMethodAcl();
  } else if (ipallow && (((acl_record = ipallow->match(client_ip)) == NULL) || (acl_record->isEmpty()))) {
    // ipallow对应的是 ip_allow.config 这个功能，不在本章节分析范围内，
    // 对此有兴趣的可以自己阅读相关代码，或者等我有空了会做个分析。
    ////////////////////////////////////////////////////
    // if client address forbidden, close immediately //
    ////////////////////////////////////////////////////
    // 如果没有通过 ip_allow 的检查，就执行 do_io_close()
    // ??memleak?? 传入的 MIOBuffer 没有释放？
    // Bug确认：https://issues.apache.org/jira/browse/TS-4697
    Warning("client '%s' prohibited by ip-allow policy", ats_ip_ntop(client_ip, ipb, sizeof(ipb)));
    netvc->do_io_close();

    return;
  }

  // Set the transport type if not already set
  // 设置 netvc 的属性，就是传输类型
  if (HttpProxyPort::TRANSPORT_NONE == netvc->attributes) {
    netvc->attributes = transport_type;
  }

  // 输出 debug 信息
  if (is_debug_tag_set("http_seq")) {
    Debug("http_seq", "[HttpSessionAccept:mainEvent %p] accepted connection from %s transport type = %d", netvc,
          ats_ip_nptop(client_ip, ipb, sizeof(ipb)), netvc->attributes);
  }

  // 创建一个 HttpClientSession
  HttpClientSession *new_session = THREAD_ALLOC_INIT(httpClientSessionAllocator, this_ethread());

  // copy over session related data.
  // 复制大多数的设置到 HttpClientSession 中
  new_session->f_outbound_transparent = f_outbound_transparent;
  new_session->f_transparent_passthrough = f_transparent_passthrough;
  new_session->outbound_ip4 = outbound_ip4;
  new_session->outbound_ip6 = outbound_ip6;
  new_session->outbound_port = outbound_port;
  new_session->host_res_style = ats_host_res_from(client_ip->sa_family, host_res_preference);
  new_session->acl_record = acl_record;

  // 将 netvc 的控制权转交给 HttpClientSession
  new_session->new_connection(netvc, iobuf, reader, backdoor);

  return;
}

int
HttpSessionAccept::mainEvent(int event, void *data)
{
  ink_release_assert(event == NET_EVENT_ACCEPT || event == EVENT_ERROR);
  ink_release_assert((event == NET_EVENT_ACCEPT) ? (data != 0) : (1));

  // 就是一个调用 accept 方法的壳
  if (event == NET_EVENT_ACCEPT) {
    this->accept(static_cast<NetVConnection *>(data), NULL, NULL);
    return EVENT_CONT;
  }

  /////////////////
  // EVENT_ERROR //
  /////////////////
  if (((long)data) == -ECONNABORTED) {
    /////////////////////////////////////////////////
    // Under Solaris, when accept() fails and sets //
    // errno to EPROTO, it means the client has    //
    // sent a TCP reset before the connection has  //
    // been accepted by the server...  Note that   //
    // in 2.5.1 with the Internet Server Supplement//
    // and also in 2.6 the errno for this case has //
    // changed from EPROTO to ECONNABORTED.        //
    /////////////////////////////////////////////////

    // FIX: add time to user_agent_hangup
    HTTP_SUM_DYN_STAT(http_ua_msecs_counts_errors_pre_accept_hangups_stat, 0);
  }

  MachineFatal("HTTP accept received fatal error: errno = %d", -((int)(intptr_t)data));
  return EVENT_CONT;
}
```


## 参考资料

- [HttpSessionAccept.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionAccept.h)
- [HttpSessionAccept.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionAccept.cc)
