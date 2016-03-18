# 核心组件：sslStartHandShake

在介绍 SSLNetVConnection 之前，要做很多的铺垫，前面介绍的蹦床只是为了实现 NPN 和 ALPN 的协商。

在 SSL/TLS 的协议中，分为 握手过程 和 传输过程，而在传输过程中还会出现 重新协商 的情况。

接下来我们先看看与 握手过程 相关的实现：

  - ret = sslStartHandShake(SSL_EVENT_SERVER, err);
    - sslServerHandShakeEvent
  - ret = sslStartHandShake(SSL_EVENT_CLIENT, err);
    - sslClientHandShakeEvent

在 SSLNetVConnection 中定义了这三个方法，用来实现 SSL 的握手过程：

  - 当 ATS 接受一个Client发起的SSL连接时，ATS作为SSL Server，因此调用 sslServerHandShakeEvent 完成握手
  - 当 ATS 主动向OServer发起SSL连接时，ATS作为SSL Client，因此调用 sslClientHandShakeEvent 完成握手

## 方法

### sslStartHandShake 分析

sslStartHandShake 用来初始化 SSLNetVC 的成员 ```SSL *ssl;```，在SSL握手中，作为 SSL Client 和/或 SSL Server 需要使用不同的方式来初始化该成员。如果初始化失败，则直接报错返回。

在完成 ssl 初始化之后，再根据 event 的值调用 sslServerHandShakeEvent 或 sslClientHandShakeEvent 完成后续握手工作。

```
int
SSLNetVConnection::sslStartHandShake(int event, int &err)
{
  // 用于记录 SSL 握手开始的时间，可用来判断握手过程是否超时
  if (sslHandshakeBeginTime == 0) {
    sslHandshakeBeginTime = Thread::get_hrtime();
    // net_activity will not be triggered until after the handshake
    set_inactivity_timeout(HRTIME_SECONDS(SSLConfigParams::ssl_handshake_timeout_in));
  }
  
  // 根据 event 来判断握手类型
  switch (event) {
  case SSL_EVENT_SERVER:
    // ATS 作为 SSL Server
    if (this->ssl == NULL) {  // 成员 ssl 用来保存 SSL Session 信息，由 OpenSSL 创建
      // 读取 ssl_multicert.config 配置文件
      SSLCertificateConfig::scoped_config lookup;
      IpEndpoint ip;
      int namelen = sizeof(ip);
      safe_getsockname(this->get_socket(), &ip.sa, &namelen);
      // 根据 ssl_multicert.config 配置文件的信息查找是否与当前的ip存在匹配关系
      SSLCertContext *cc = lookup->find(ip);
      // 输出Debug信息
      if (is_debug_tag_set("ssl")) {
        IpEndpoint src, dst;
        ip_port_text_buffer ipb1, ipb2;
        int ip_len;

        safe_getsockname(this->get_socket(), &dst.sa, &(ip_len = sizeof ip));
        safe_getpeername(this->get_socket(), &src.sa, &(ip_len = sizeof ip));
        ats_ip_nptop(&dst, ipb1, sizeof(ipb1));
        ats_ip_nptop(&src, ipb2, sizeof(ipb2));
        Debug("ssl", "IP context is %p for [%s] -> [%s], default context %p", cc, ipb2, ipb1, lookup->defaultContext());
      }

      // Escape if this is marked to be a tunnel.
      // No data has been read at this point, so we can go
      // directly into blind tunnel mode
      // 如果在 ssl_multicert.config 配置文件中定义了与此IP相关的规则，而且action＝tunnel
      if (cc && SSLCertContext::OPT_TUNNEL == cc->opt && this->is_transparent) {
        // 设置为 Blind Tunnel
        this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
        // 强制握手过程为完成
        sslHandShakeComplete = 1;
        // 释放 ssl 成员
        SSL_free(this->ssl);
        this->ssl = NULL;
        // 返回调用者
        // 此时这个SSLVC就变成了一个TCP Blind Tunnel，ATS只是在TCP层双向转发数据
        return EVENT_DONE;
      }

      // 如果没有设置为 Blind Tunnel：
      // Attach the default SSL_CTX to this SSL session. The default context is never going to be able
      // to negotiate a SSL session, but it's enough to trampoline us into the SNI callback where we
      // can select the right server certificate.
      // 使用缺省 context 初始化 ssl 成员。
      // defaultContext() 的设置是不允许进行SSL会话的协商的， －－ 这句不太理解什么意思？？？
      // 但是可以触发 SNI Callback，这样就可以选择一个正确的 Server 端的证书。
      this->ssl = make_ssl_connection(lookup->defaultContext(), this);
#if !(TS_USE_TLS_SNI)
      // 如果当前 OpenSSL 的版本连基本的 TLS SNI Callback 都不支持的话，
      // 就需要在这里初始化用于调试的 SSLTrace 状态
      // set SSL trace
      if (SSLConfigParams::ssl_wire_trace_enabled) {
        bool trace = computeSSLTrace();
        Debug("ssl", "sslnetvc. setting trace to=%s", trace ? "true" : "false");
        setSSLTrace(trace);
      }
#endif
    }
    // 如果创建 ssl 成员失败，报错并返回错误
    if (this->ssl == NULL) {
      SSLErrorVC(this, "failed to create SSL server session");
      return EVENT_ERROR;
    }

    // 最后调用 sslServerHandShakeEvent 进行握手
    return sslServerHandShakeEvent(err);

  case SSL_EVENT_CLIENT:
    // 如果成员 ssl 未创建
    if (this->ssl == NULL) {
      // 使用 client_ctx 初始化并创建成员 ssl
      this->ssl = make_ssl_connection(ssl_NetProcessor.client_ctx, this);
    }

    // 如果创建 ssl 成员失败，报错并返回错误
    if (this->ssl == NULL) {
      SSLErrorVC(this, "failed to create SSL client session");
      return EVENT_ERROR;
    }
    
    // 最后调用 sslClientHandShakeEvent 进行握手
    return sslClientHandShakeEvent(err);

  default:
    // 其它情况，异常错误／调用
    ink_assert(0);
    return EVENT_ERROR;
  }
}
```

### sslServerHandShakeEvent 分析

sslServerHandShakeEvent 主要实现了对 SSL_accept 方法的封装：

  - 在调用 SSL_accept 之前，触发 PreAcceptHook，
  - 处理在 PreAcceptHook 中停留的状态，
  - 在完成 PreAcceptHook 之后，再通过 SSL_accept 完成握手过程

在调用 SSL_accept 完成握手过程时：

  - 会触发 SNI Callback 或者 CERT Callback，
  - 处理在 SNI/CERT Hook 中停留的状态，
  - 在完成 SNI/CERT Hook 之后，再继续调用 SSL_accept 完成握手过程

上面两处 Hook 的停留状态，都需要在 Hook 中调用 reenable 来触发对 sslServerHandShakeEvent 的再次调用，才能继续。

需要注意的是，NetHandler在 读事件 和 写事件 时，都会回调 sslServerHandShakeEvent，因此在阅读代码时要同时考虑 读 和 写 两种情况的回调。

```
int
SSLNetVConnection::sslServerHandShakeEvent(int &err)
{
  // sslPreAcceptHookState 用来支持 PreAcceptHook 的实现
  // 这里首先判断是否已经完成了 PreAcceptHook 阶段
  if (SSL_HOOKS_DONE != sslPreAcceptHookState) {
    // Get the first hook if we haven't started invoking yet.
    // PreAcceptHook 对于每一个 SSLNetVC 只会触发一次，
    //     在回调Hook函数／插件时，还没有向上层状态机传递NET_EVENT_ACCEPT事件
    if (SSL_HOOKS_INIT == sslPreAcceptHookState) {
      // SSL_HOOKS_INIT 状态表示 PreAcceptHook 未被触发
      // 获取 Hook 函数／插件，然后设置状态为 “发起”
      curHook = ssl_hooks->get(TS_VCONN_PRE_ACCEPT_INTERNAL_HOOK);
      sslPreAcceptHookState = SSL_HOOKS_INVOKE;
    } else if (SSL_HOOKS_INVOKE == sslPreAcceptHookState) {
      // SSL_HOOKS_INVOKE 状态表示 PreAcceptHook 在发起中
      // 由于 Hook 函数／插件，可能不只有一个，
      //     因此在完成了第一个函数／插件的调用，还未开始调用第二个时，就可能是这个状态
      //     另外，Hook 函数／插件由于某些原因需要延迟 PreAccept 过程，也可能会停留在这个状态
      // if the state is anything else, we haven't finished
      // the previous hook yet.
      // 获取下一个 Hook 函数／插件
      curHook = curHook->next();
    }
    if (SSL_HOOKS_INVOKE == sslPreAcceptHookState) {
      // 如果在发起中的状态
      if (0 == curHook) { // no hooks left, we're done
        // 下一个 Hook 函数／插件 指向 NULL （0），表示没有啦
        // 那么所有的 Hook 函数／插件 都执行完成了，PreAcceptHook 就全部执行完成
        //     设置为 SSL_HOOKS_DONE 表示已经完成了 PreAcceptHook 阶段
        sslPreAcceptHookState = SSL_HOOKS_DONE;
      } else {
        // SSL_HOOKS_ACTIVE 状态表示，正在执行对 Hook 函数／插件 的回调操作
        // 在 Hook 函数／插件 的回调操作中，通过调用 reenable 重新设置 sslPreAcceptHookState 状态，
        //     对于非 SSL_HOOKS_DONE 状态的情况，统一改为 SSL_HOOKS_INVOKE 状态
        sslPreAcceptHookState = SSL_HOOKS_ACTIVE;
        // 回调 Hook 函数／插件
        // 默认 SSLNetVC 的 mutex 已经被上锁，wrap 尝试对 Hook 函数／插件 的 mutex 上锁，成功则执行同步回调，
        // 失败则创建 ContWrapper 通过 EventSystem 进行异步回调，ContWrapper 的 mutex 共享 SSLNetVC 的 mutex
        //     因此在 EventSystem 回调时会首先锁住 SSLNetVC 的 mutex，
        //     然后在 ContWrapper 的回调函数 event_handler 中再次尝试对 Hook 函数／插件 的 mutex 上锁，
        //     上锁成功则同步回调 Hook 函数／插件，然后释放 ContWrapper，
        //     上锁失败则重新调度 ContWrapper 再次执行。
        ContWrapper::wrap(mutex, curHook->m_cont, TS_EVENT_VCONN_PRE_ACCEPT, this);
    
        // 返回SSL_WAIT_FOR_HOOK，就是表示只有reenable才能继续后面的流程
        return SSL_WAIT_FOR_HOOK;
      }
    } else { // waiting for hook to complete
      // 不是 SSL_HOOKS_INVOKE 状态，例如，可能是 SSL_HOOKS_ACTIVE 状态，
      // 那么就继续 SSL_WAIT_FOR_HOOK
             /* A note on waiting for the hook. I believe that because this logic
                cannot proceed as long as a hook is outstanding, the underlying VC
                can't go stale. If that can happen for some reason, we'll need to be
                more clever and provide some sort of cancel mechanism. I have a trap
                in SSLNetVConnection::free to check for this.
             */
      return SSL_WAIT_FOR_HOOK;
    }
  }

  ////////////
  //
  //  这里存在一个 SSL 的 bug：https://issues.apache.org/jira/browse/TS-4075，但是 Patch 还未经官方确认
  //  情况是这样的：
  //      如果在 SNI/CERT Hook 函数回调中，当前 SSLVC 的 SSL_accept 过程被挂起，
  //          sslHandshakeHookState == HANDSHAKE_HOOKS_CERT 的状态
  //      此时如果客户端关闭了 Socket 连接，那么 epoll_wait 就会发现 socket fd 可读
  //          然后 NetHandler 调用了 net_read_io, 然后发现 SSL 握手未完成，
  //          然后 net_read_io 调用了 ret = sslStartHandShake(SSL_EVENT_SERVER, err); 
  //          然后 sslStartHandShake 调用了 sslServerHandShakeEvent ，就是当前函数
  //          然后在调用 SSLAccept 之前，填充 BIO 缓冲区时，调用 read_raw_data 发现读到了 0 字节
  //          这表示连接中断，返回了 EVENT_ERROR 给 net_read_io
  //      net_read_io 发现遇到了 EVENT_ERROR，就把 SSLVC 給关闭了。
  //      但是，此时 Plugin 还在处理中，等 Plugin 需要针对此 SSLVC 做动作时，就导致 ATS 崩溃了。
  //
  ////////////
  //
  //  修复方案是当发现处于 sslHandshakeHookState == HANDSHAKE_HOOKS_CERT 状态时，返回 SSL_WAIT_FOR_HOOK，
  //  这样就可以推迟 SSLVC 的关闭，这样等 Plugin 需要针对此 SSLVC 做动作时，就不会出现找不到 SSLVC 的情况。
  //
  ////////////

  // If a blind tunnel was requested in the pre-accept calls, convert.
  // Again no data has been exchanged, so we can go directly
  // without data replay.
  // Note we can't arrive here if a hook is active.
  // 在 PreAcceptHook 内可以对SSLVC的属性进行设置，让SSLVC变成Blind Tunnel
  //     在 Hook 内调用 TSVConnTunnel 来设置 SSLVC 成为 Blind Tunnel
  // 此处就是对这个设置进行的响应
  if (TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) {
    // 设置属性为 Blind Tunnel
    this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
    // 释放成员 ssl
    SSL_free(this->ssl);
    this->ssl = NULL;
    // Don't mark the handshake as complete yet,
    // Will be checking for that flag not being set after
    // we get out of this callback, and then will shuffle
    // over the buffered handshake packets to the O.S.
    // 上面的注释暂时没看懂，不明白为何这里不能直接设置为握手已经完成的状态？？
    return EVENT_DONE;
  } else if (TS_SSL_HOOK_OP_TERMINATE == hookOpRequested) {
    // 如果需要在 PreAcceptHook 内终止此 SSLVC，可以设置为 TS_SSL_HOOK_OP_TERMINATE
    // 直接设置为握手完成的状态 －－－－> 这样就能终止此 SSLVC 吗？？
    sslHandShakeComplete = 1;
    return EVENT_DONE;
  }
  // 到这里，所有跟 PreAcceptHook 相关的部分都已经完成了

  int retval = 1; // Initialze with a non-error value

  // All the pre-accept hooks have completed, proceed with the actual accept.
  // 检查一下 ssl 的读通道 rbio，是不是空了
  if (BIO_eof(SSL_get_rbio(this->ssl))) { // No more data in the buffer
    // 没有数据了，就要通过 read_raw_data 读取数据
    //     只有上一个 rbio 被完全消费了，才可以通过 read_raw_data 设置一个新的，
    //     因为 read_raw_data 是会替换掉原来设置的 bio。
    // 这样 SSL_accept 就可以处理这些SSL握手数据了
    // Read from socket to fill in the BIO buffer with the
    // raw handshake data before calling the ssl accept calls.
    retval = this->read_raw_data();
    // 返回 0 表示 EOF
    if (retval == 0) {
      // EOF, go away, we stopped in the handshake
      SSLDebugVC(this, "SSL handshake error: EOF");
      return EVENT_ERROR;
    }
  }

  // 调用 SSLAccept 消费 rbio 内的数据
  // SSLAccept 是对 OpenSSL API SSL_accept 的封装
  ssl_error_t ssl_error = SSLAccept(ssl);
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  // SSL_ERROR_NONE 表示没有错误
  if (ssl_error != SSL_ERROR_NONE) {
    // SSLAccept 调用出错了
    // 这里保存 errno 有用吗？？？
    err = errno;
    SSLDebugVC(this, "SSL handshake error: %s (%d), errno=%d", SSLErrorName(ssl_error), ssl_error, err);

    // start a blind tunnel if tr-pass is set and data does not look like ClientHello
    // 如果 SSLAccept 出错了，那么就看看是否设置了 tr-pass 标志
    char *buf = handShakeBuffer->buf();
    // 如果设置了 tr-pass 标志，而且接收到的数据也不像是 SSL 握手请求
    if (getTransparentPassThrough() && buf && *buf != SSL_OP_HANDSHAKE) {
      SSLDebugVC(this, "Data does not look like SSL handshake, starting blind tunnel");
      // 设置 Blind Tunnel 属性
      this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      // 强制设置为未完成握手的情况，应该跟上面的 if (TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) 处理是一样的
      sslHandShakeComplete = 0;
      // 返回调用者，按照Blind Tunnel来进行处理
      return EVENT_CONT;
    }
  }

  // 根据 SSLAccept 的返回值进行错误处理
  //     参考：https://www.openssl.org/docs/manmaster/ssl/SSL_get_error.html
  switch (ssl_error) {
  case SSL_ERROR_NONE:
    // 没有出错的情况
    // 输出握手成功的debug信息
    if (is_debug_tag_set("ssl")) {
      X509 *cert = SSL_get_peer_certificate(ssl);

      Debug("ssl", "SSL server handshake completed successfully");
      if (cert) {
        debug_certificate_name("client certificate subject CN is", X509_get_subject_name(cert));
        debug_certificate_name("client certificate issuer CN is", X509_get_issuer_name(cert));
        X509_free(cert);
      }
    }

    // 设置握手完成
    sslHandShakeComplete = true;

    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake completed successfully");
    // do we want to include cert info in trace?

    // 记录握手过程使用的时间
    if (sslHandshakeBeginTime) {
      const ink_hrtime ssl_handshake_time = Thread::get_hrtime() - sslHandshakeBeginTime;
      Debug("ssl", "ssl handshake time:%" PRId64, ssl_handshake_time);
      sslHandshakeBeginTime = 0;
      SSL_INCREMENT_DYN_STAT_EX(ssl_total_handshake_time_stat, ssl_handshake_time);
      SSL_INCREMENT_DYN_STAT(ssl_total_success_handshake_count_in_stat);
    }

    {
      const unsigned char *proto = NULL;
      unsigned len = 0;

// If it's possible to negotiate both NPN and ALPN, then ALPN
// is preferred since it is the server's preference.  The server
// preference would not be meaningful if we let the client
// preference have priority.
      // 使用 ALPN 协商模式，获得支持的 proto 字符串
#if TS_USE_TLS_ALPN
      SSL_get0_alpn_selected(ssl, &proto, &len);
#endif /* TS_USE_TLS_ALPN */

      // 在ALPN失败或不存在时，
      // 使用 NPN 协商模式，获得支持的 proto 字符串
#if TS_USE_TLS_NPN
      if (len == 0) {
        SSL_get0_next_proto_negotiated(ssl, &proto, &len);
      }
#endif /* TS_USE_TLS_NPN */

      // len 为支持的协议类型的描述串长度
      if (len) {
        // len 大于 0 表示协商成功了
        // If there's no NPN set, we should not have done this negotiation.
        ink_assert(this->npnSet != NULL);

        // 根据 npnSet 来找到能处理此协议的状态机，保存到 npnEndpoint
        this->npnEndpoint = this->npnSet->findEndpoint(proto, len);
        this->npnSet = NULL;

        // 如果 npnEndpoint 为空，则表示此协议没有对应的状态机可以处理
        if (this->npnEndpoint == NULL) {
          Error("failed to find registered SSL endpoint for '%.*s'", (int)len, (const char *)proto);
          return EVENT_ERROR;
        }

        Debug("ssl", "client selected next protocol '%.*s'", len, proto);
        TraceIn(trace, get_remote_addr(), get_remote_port(), "client selected next protocol'%.*s'", len, proto);
      } else {
        Debug("ssl", "client did not select a next protocol");
        TraceIn(trace, get_remote_addr(), get_remote_port(), "client did not select a next protocol");
      }
    }
    // 返回调用者
    return EVENT_DONE;
    
  case SSL_ERROR_WANT_CONNECT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_CONNECT");
    return SSL_HANDSHAKE_WANT_CONNECT;

  case SSL_ERROR_WANT_WRITE:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_WRITE");
    return SSL_HANDSHAKE_WANT_WRITE;

  case SSL_ERROR_WANT_READ:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_READ");
    if (retval == -EAGAIN) {
      // No data at the moment, hang tight
      SSLDebugVC(this, "SSL handshake: EAGAIN");
      return SSL_HANDSHAKE_WANT_READ;
    } else if (retval < 0) {
      // An error, make us go away
      SSLDebugVC(this, "SSL handshake error: read_retval=%d", retval);
      return EVENT_ERROR;
    }
    return SSL_HANDSHAKE_WANT_READ;

// This value is only defined in openssl has been patched to
// enable the sni callback to break out of the SSL_accept processing
#ifdef SSL_ERROR_WANT_SNI_RESOLVE
  case SSL_ERROR_WANT_X509_LOOKUP:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_X509_LOOKUP");
    return EVENT_CONT;
  case SSL_ERROR_WANT_SNI_RESOLVE:
    // SSL_ERROR_WANT_SNI_RESOLVE 的出现表示 SSL 握手过程在 SNI Callback 中挂起了，
    //     需要再次调用 SSL_accept 才能完成握手过程。
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_SNI_RESOLVE");
#elif SSL_ERROR_WANT_X509_LOOKUP
  case SSL_ERROR_WANT_X509_LOOKUP:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_X509_LOOKUP");
#endif
#if defined(SSL_ERROR_WANT_SNI_RESOLVE) || defined(SSL_ERROR_WANT_X509_LOOKUP)
    if (this->attributes == HttpProxyPort::TRANSPORT_BLIND_TUNNEL || TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) {
      // 如果在 Hook 里面设置为 Blind Tunnel 了，就需要做一个处理
      this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      sslHandShakeComplete = 0;
      return EVENT_CONT;
    } else {
      // 如果没有设置为 Blind Tunnel，那么就是停在 Hook 函数／插件里面了，因此需要等待
      //  Stopping for some other reason, perhaps loading certificate
      return SSL_WAIT_FOR_HOOK;
    }
#endif

  case SSL_ERROR_WANT_ACCEPT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_ACCEPT");
    return EVENT_CONT;

  // 下面这些都是SSL异常的错误处理
  case SSL_ERROR_SSL: {
    SSL_CLR_ERR_INCR_DYN_STAT(this, ssl_error_ssl, "SSLNetVConnection::sslServerHandShakeEvent, SSL_ERROR_SSL errno=%d", errno);
    char buf[512];
    unsigned long e = ERR_peek_last_error();
    ERR_error_string_n(e, buf, sizeof(buf));
    TraceIn(trace, get_remote_addr(), get_remote_port(),
            "SSL server handshake ERROR_SSL: sslErr=%d, ERR_get_error=%ld (%s) errno=%d", ssl_error, e, buf, errno);
    return EVENT_ERROR;
  }

  case SSL_ERROR_ZERO_RETURN:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_ZERO_RETURN");
    return EVENT_ERROR;
  case SSL_ERROR_SYSCALL:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_SYSCALL");
    return EVENT_ERROR;
  default:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_OTHER");
    return EVENT_ERROR;
  }
}
```

### sslClientHandShakeEvent 分析

sslClientHandShakeEvent 主要实现了：

  - ATS 作为 SSL Client 向 OServer 发起一个 SSL Handshake 的功能
  
```
int
SSLNetVConnection::sslClientHandShakeEvent(int &err)
{
#if TS_USE_TLS_SNI
  // 对 SNI 功能的支持
  if (options.sni_servername) {
    if (SSL_set_tlsext_host_name(ssl, options.sni_servername)) {
      Debug("ssl", "using SNI name '%s' for client handshake", options.sni_servername.get());
    } else {
      Debug("ssl.error", "failed to set SNI name '%s' for client handshake", options.sni_servername.get());
      SSL_INCREMENT_DYN_STAT(ssl_sni_name_set_failure);
    }
  }
#endif

  // 在 SSLInitClientContext 方法中，初始化了 ssl_client_data_index，
  //     用来在 SSL 会话描述符中申请一个内存块，此index表示内存块的编号，
  //     对于可以通过此index值来存／取该内存块的数据，
  //     在 SSL 会话描述符中有多个这样的内存块，可以存储应用层的数据
  // 在 ATS 中，利用这个特性，把 SSLVC 的实例内存地址（this）存储到了 SSL 会话描述符中
  SSL_set_ex_data(ssl, get_ssl_client_data_index(), this);
  // 向 OServer 发起 SSL Handshake，SSLConnect 是对 SSL_connect 的封装
  ssl_error_t ssl_error = SSLConnect(ssl);
  // SSL 跟踪调试状态
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  // 根据 SSLConnect 的返回值 ssl_error 进行错误处理
  switch (ssl_error) {
  // SSL_ERROR_NONE 表示没有错误
  case SSL_ERROR_NONE:
    // 输出调试信息
    if (is_debug_tag_set("ssl")) {
      X509 *cert = SSL_get_peer_certificate(ssl);

      Debug("ssl", "SSL client handshake completed successfully");
      // if the handshake is complete and write is enabled reschedule the write
      // 为何在开启了调试信息时要做 writeReschedule ？？？
      if (closed == 0 && write.enabled)
        writeReschedule(nh);
      if (cert) {
        debug_certificate_name("server certificate subject CN is", X509_get_subject_name(cert));
        debug_certificate_name("server certificate issuer CN is", X509_get_issuer_name(cert));
        X509_free(cert);
      }
    }
    SSL_INCREMENT_DYN_STAT(ssl_total_success_handshake_count_out_stat);

    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake completed successfully");
    // do we want to include cert info in trace?

    // 设置握手完成
    sslHandShakeComplete = true;
    return EVENT_DONE;

  // 下面是其它错误情况的处理
  case SSL_ERROR_WANT_WRITE:
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_WANT_WRITE");
    SSL_INCREMENT_DYN_STAT(ssl_error_want_write);
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_WRITE");
    return SSL_HANDSHAKE_WANT_WRITE;

  case SSL_ERROR_WANT_READ:
    SSL_INCREMENT_DYN_STAT(ssl_error_want_read);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_WANT_READ");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_READ");
    return SSL_HANDSHAKE_WANT_READ;

  case SSL_ERROR_WANT_X509_LOOKUP:
    SSL_INCREMENT_DYN_STAT(ssl_error_want_x509_lookup);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_WANT_X509_LOOKUP");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_X509_LOOKUP");
    break;

  case SSL_ERROR_WANT_ACCEPT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_ACCEPT");
    return SSL_HANDSHAKE_WANT_ACCEPT;

  case SSL_ERROR_WANT_CONNECT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake ERROR_WANT_CONNECT");
    break;

  case SSL_ERROR_ZERO_RETURN:
    SSL_INCREMENT_DYN_STAT(ssl_error_zero_return);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, EOS");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake EOS");
    return EVENT_ERROR;

  case SSL_ERROR_SYSCALL:
    err = errno;
    SSL_INCREMENT_DYN_STAT(ssl_error_syscall);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, syscall");
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL client handshake Syscall Error: %s", strerror(errno));
    return EVENT_ERROR;
    break;

  case SSL_ERROR_SSL:
  default: {
    err = errno;
    // FIXME -- This triggers a retry on cases of cert validation errors....
    Debug("ssl", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_SSL");
    SSL_CLR_ERR_INCR_DYN_STAT(this, ssl_error_ssl, "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_SSL errno=%d", errno);
    Debug("ssl.error", "SSLNetVConnection::sslClientHandShakeEvent, SSL_ERROR_SSL");
    char buf[512];
    unsigned long e = ERR_peek_last_error();
    ERR_error_string_n(e, buf, sizeof(buf));
    TraceIn(trace, get_remote_addr(), get_remote_port(),
            "SSL client handshake ERROR_SSL: sslErr=%d, ERR_get_error=%ld (%s) errno=%d", ssl_error, e, buf, errno);
    return EVENT_ERROR;
    }
    break;
  }
  return EVENT_CONT;
}
```

## SNI/CERT Hook 的实现

在上面的 sslServerHandShakeEvent 分析中，并未看到 SNI/CERT Hook 的实现部分，只看到了 PreAccept Hook 的实现，那么 SNI/CERT Hook 是如何实现的呢？

由于在 OpenSSL 1.0.2d 之后提供了 Certificate Callback，但是之前的版本是 SNI Callback，有一定区别，因此在ATS中通过宏定义来实现了对两个不同版本的 OpenSSL 库的兼容。

但是更早期的版本里，好像连 SNI Callback 也没有实现，在 configure 的时候，会检测 ```SSL_CTX_set_tlsext_servername_callback``` 是否可用：

  - 可用，那么定义 TS_USE_TLS_SNI 为 true (1)
  - 不可用，就定义 TS_USE_TLS_SNI 为 false (0)

在 Client 连接到 ATS 的 SSL Server 进行握手时，SSL Server 需要下发一个证书给 Client，因此 ATS 规定需要在 ssl_multicert.config 中配置一个规则，指明使用的证书：

  - 必须设置 ssl_cert_name=<file.pem>
    - 当 Client 访问的时候，就会将这个证书提供给客户端
    - 在加载证书的时候，会根据其内签发的域名来匹配SNI
  - 其它选项都可以可选的，不是必须配置的
  - 如果设置 dest_ip=*
    - 那么对应的证书则作为缺省证书使用，
    - 当SNI内包含的域名，无法在所有的证书中找到时，则使用这个缺省证书。
  - 当没有设置 dest_ip=* 的缺省证书时，
    - 在 SSLParseCertificateConfiguration 方法加载完配置后，会创建一个 dest_ip=* 的记录，但是该记录没有证书。
    - 那么在找不到证书时，握手就无法建立。

对 ssl_multicert.config 的加载和解析过程：

  - SSLNetProcessor::start(int number_of_ssl_threads, size_t stacksize)
    - SSLConfig::startup()
      - 加载 SSL配置到 ConfigProcessor
      - SSLConfig::reconfigure()
      - SSLConfigParams *params = new SSLConfigParams;
      - params->initialize();  // 这一行负责加载records.config中与SSL相关的配置
      - 保存 params 到 ConfigProcessor
    - SSLCertificateConfig::startup()
      - 加载 ssl_multicert.config 里定义的证书信息到 ConfigProcessor
      - SSLCertificateConfig::reconfigure()
        - 声明 SSLConfig::scoped_config params;  // 构造函数从 ConfigProcessor 加载相关配置完成初始化
        - 声明 SSLCertLookup *lookup = new SSLCertLookup();
        - 调用 SSLParseCertificateConfiguration(params, lookup)
        - 保存 lookup 到 ConfigProcessor

SSLParseCertificateConfiguration 方法负责：

  - 解析 ssl_multicert.config 配置文件
  - 调用 ssl_store_ssl_context 方法来加载证书
  - 全部处理完成后，会检查是否已经设置了缺省证书
  - 如果没有设置，则调用 ssl_store_ssl_context 方法添加一个 空（NULL）证书，作为缺省证书

ssl_store_ssl_context 方法负责

  - 向 SSLCertLookup 类型的容器添加域名和证书
  - 如果添加的证书是一个缺省证书，则会：
    - 将此证书设置为 SSLCertLookup 类型容器内的缺省证书
    - 同时调用 ssl_set_handshake_callbacks 方法，设置 SNI/CERT Hook 到 SSL 会话上

注意，scoped_config 的定义有两处：

  - SSLConfig::scoped_config
    - 构造函数从 ConfigProcessor 获取 SSLConfigParams 类型的结构数据，完成初始化
  - SSLCertificateConfig::scoped_config
    - 构造函数从 ConfigProcessor 获取 SSLCertLookup 类型的结构数据，完成初始化

```
source: P_SSLConfig.h
struct SSLConfig {
  static void startup();
  static void reconfigure();
  static SSLConfigParams *acquire();
  static void release(SSLConfigParams *params);

  typedef ConfigProcessor::scoped_config<SSLConfig, SSLConfigParams> scoped_config;

private:
  static int configid;
};

struct SSLCertificateConfig {
  static bool startup();
  static bool reconfigure();
  static SSLCertLookup *acquire();
  static void release(SSLCertLookup *params);

  typedef ConfigProcessor::scoped_config<SSLCertificateConfig, SSLCertLookup> scoped_config;

private:
  static int configid;
};
```

ssl_set_handshake_callbacks 方法通过宏定义来判定使用哪个 OpenSSL 的API方法来设置 Hook 函数。

需要注意的是，如果 TS_USE_TLS_SNI 为 0 的话，那么 ssl_set_handshake_callbacks 就是一个空函数。

```
source: SSLUtils.cc
static void
ssl_set_handshake_callbacks(SSL_CTX *ctx)
{
#if TS_USE_TLS_SNI
// Make sure the callbacks are set
#if TS_USE_CERT_CB
  SSL_CTX_set_cert_cb(ctx, ssl_cert_callback, NULL);
#else
  SSL_CTX_set_tlsext_servername_callback(ctx, ssl_servername_callback);
#endif
#endif
}
```

下面分别是 ssl_cert_callback 和 ssl_servername_callback 两个回调函数：

  - ssl_cert_callback
    - 当 OpenSSL 版本大于等于 1.0.2d 的时候，采用Cert Callback
  - ssl_servername_callback
    - 否则，采用 SNI Callback
  - 这两个回调函数都需要确保必须调用，且只调用一次 set_context_cert 方法
    - 因为 set_context_cert 方法用来设置缺省的证书还有通信加密算法
    - 而且由于 SNI/CERT Hook 功能的存在，这两个回调函数，可能会被多次回调
  - 然后再回调 SNI/CERT Hook

而 set_context_cert 方法是两个回调函数的公共部分：

  - set_context_cert
    - 由上面两个方法调用，用来设置默认的SSL证书信息
    - 首先根据 Server Name 在 SSLCertLookup 中查找，
    - 如果找不到，再根据 IP 查找。

同样，如果 TS_USE_TLS_SNI 为 0 的话，上面这三个方法就不会被定义出来。


```
source: SSLUtils.cc
#if TS_USE_TLS_SNI
int
set_context_cert(SSL *ssl)
{
  SSL_CTX *ctx = NULL;
  SSLCertContext *cc = NULL;
  SSLCertificateConfig::scoped_config lookup;
  // 获取 SNI
  const char *servername = SSL_get_servername(ssl, TLSEXT_NAMETYPE_host_name);
  SSLNetVConnection *netvc = (SSLNetVConnection *)SSL_get_app_data(ssl);
  bool found = true;
  // 返回值默认为 1 表示成功
  int retval = 1;

  Debug("ssl", "set_context_cert ssl=%p server=%s handshake_complete=%d", ssl, servername, netvc->getSSLHandShakeComplete());
  // set SSL trace (we do this a little later in the USE_TLS_SNI case so we can get the servername
  // 设置 SSL Trace
  if (SSLConfigParams::ssl_wire_trace_enabled) {
    bool trace = netvc->computeSSLTrace();
    Debug("ssl", "sslnetvc. setting trace to=%s", trace ? "true" : "false");
    netvc->setSSLTrace(trace);
  }

  // catch the client renegotiation early on
  // 处理 CVE-2011-1473 客户端发起的 SSL 重协商漏洞
  //     该问题是由于SSL协议的客观因素引起的。
  //     因为服务器在进行密钥的计算时，其消耗的计算资源是客户端的数十倍
  //     所以如果可以允许客户端主动发起Renegotiation，那么将可以造成DoS攻击。
  // 参考：
  //     TS-1467: Disable client initiated renegotiation (SSL) DDoS by default
  //     Github: https://github.com/apache/trafficserver/commit/d43b5d685e55795a755413b92d3a8827b86c4a03
  // 该问题的修补不只有这一处，因此需要整体参考github上的补丁
  if (SSLConfigParams::ssl_allow_client_renegotiation == false && netvc->getSSLHandShakeComplete()) {
    Debug("ssl", "set_context_cert trying to renegotiate from the client");
    retval = 0; // Error
    goto done;
  }

  // The incoming SSL_CTX is either the one mapped from the inbound IP address or the default one. If we
  // don't find a name-based match at this point, we *do not* want to mess with the context because we've
  // already made a best effort to find the best match.
  // 查找与该 SNI 匹配的 SSL_CTX 设置
  if (likely(servername)) {
    cc = lookup->find((char *)servername);
    if (cc && cc->ctx)
      ctx = cc->ctx;
    // 开启了Tunnel功能，那么就直接透传
    if (cc && SSLCertContext::OPT_TUNNEL == cc->opt && netvc->get_is_transparent()) {
      netvc->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      netvc->setSSLHandShakeComplete(true);
      // 返回 -1 表示挂起 SSL 握手过程
      retval = -1;
      goto done;
    }
  }

  // If there's no match on the server name, try to match on the peer address.
  // 当通过 SNI 无法找到匹配的 SSL_CTX 时，使用 IP 再次查找
  if (ctx == NULL) {
    IpEndpoint ip;
    int namelen = sizeof(ip);

    safe_getsockname(netvc->get_socket(), &ip.sa, &namelen);
    cc = lookup->find(ip);
    if (cc && cc->ctx)
      ctx = cc->ctx;
    // 此处没有对 Tunnel 进行判断，因为这里是到了 SNI/CERT Callback，
    // 而根据 IP 进行判断是否要做 Tunnel 没必要在这么靠后的位置。
  }

  // 如果匹配成功，就要把 SSL_CTX 设置到当前的 SSL 会话上
  if (ctx != NULL) {
    SSL_set_SSL_CTX(ssl, ctx);
#if HAVE_OPENSSL_SESSION_TICKETS
    // Reset the ticket callback if needed
    // 如果支持 Session Ticket，还要设置 Session Ticket Callback
    SSL_CTX_set_tlsext_ticket_key_cb(ctx, ssl_callback_session_ticket);
#endif
  } else {
    found = false;
  }

  ctx = SSL_get_SSL_CTX(ssl);
  Debug("ssl", "ssl_cert_callback %s SSL context %p for requested name '%s'", found ? "found" : "using", ctx, servername);

  // 如果当前的 SSL 会话没有 SSL_CTX，那么就要返回错误
  if (ctx == NULL) {
    // 返回值为 0 表示错误
    retval = 0;
    goto done;
  }
done:
  return retval;
}

// Use the certificate callback for openssl 1.0.2 and greater
// otherwise use the SNI callback
#if TS_USE_CERT_CB
/**
 * Called before either the server or the client certificate is used
 * Return 1 on success, 0 on error, or -1 to pause
 */
static int
ssl_cert_callback(SSL *ssl, void * /*arg*/)
{
  SSLNetVConnection *netvc = (SSLNetVConnection *)SSL_get_app_data(ssl);
  bool reenabled;
  int retval = 1;

  // Do the common certificate lookup only once.  If we pause
  // and restart processing, do not execute the common logic again
  // 确保必须执行，而且只执行一次 set_context_cert 方法
  if (!netvc->calledHooks(TS_SSL_CERT_HOOK)) {
    retval = set_context_cert(ssl);
    if (retval != 1) {
      return retval;
    }
  }

  // Call the plugin cert code
  // 回调 SNI/CERT Hook
  reenabled = netvc->callHooks(TS_SSL_CERT_HOOK);
  // If it did not re-enable, return the code to
  // stop the accept processing
  // 根据返回值来确认是否要挂起握手过程
  if (!reenabled) {
    retval = -1; // Pause
  }

  // Return 1 for success, 0 for error, or -1 to pause
  return retval;
}
#else
static int
ssl_servername_callback(SSL *ssl, int * /* ad */, void * /*arg*/)
{
  SSLNetVConnection *netvc = (SSLNetVConnection *)SSL_get_app_data(ssl);
  bool reenabled;
  int retval = 1;

  // Do the common certificate lookup only once.  If we pause
  // and restart processing, do not execute the common logic again
  // 确保必须执行，而且只执行一次 set_context_cert 方法
  if (!netvc->calledHooks(TS_SSL_CERT_HOOK)) {
    retval = set_context_cert(ssl);
    if (retval != 1) {
      goto done;
    }
  }

  // Call the plugin SNI code
  // 回调 SNI/CERT Hook
  reenabled = netvc->callHooks(TS_SSL_SNI_HOOK);
  // If it did not re-enable, return the code to
  // stop the accept processing
  // 根据返回值来确认是否要挂起握手过程
  if (!reenabled) {
    retval = -1;
  }

done:
  // Map 1 to SSL_TLSEXT_ERR_OK
  // Map 0 to SSL_TLSEXT_ERR_ALERT_FATAL
  // Map -1 to SSL_TLSEXT_ERR_READ_AGAIN, if present
  // 由于早期版本的 OpenSSL API 的返回值是一组宏定义
  // 因此下面的代码做一下简单的翻译工作
  switch (retval) {
  case 1:
    retval = SSL_TLSEXT_ERR_OK;
    break;
  case -1:
#ifdef SSL_TLSEXT_ERR_READ_AGAIN
    retval = SSL_TLSEXT_ERR_READ_AGAIN;
#else
    Error("Cannot pause SNI processsing with this version of openssl");
    retval = SSL_TLSEXT_ERR_ALERT_FATAL;
#endif
    break;
  case 0:
  default:
    retval = SSL_TLSEXT_ERR_ALERT_FATAL;
    break;
  }
  return retval;
}
#endif
#endif /* TS_USE_TLS_SNI */
```

## SNI/CERT Hook 的回调

在 SSLVC 中，PRE ACCEPT Hook 与 SNI/CERT Hook 的回调处理都是特殊的方式。

在 sslServerHandShakeEvent 中已经介绍了 PRE ACCEPT Hook 的回调，下面是对 SNI/CERT Hook 的回调进行分析。

抵达 callHooks 方法的调用栈如下：

  - SSLAccept()
    - SSL_accept()
      - ssl_servername_callback() / ssl_cert_callback()
        - callHooks()

```
bool
SSLNetVConnection::callHooks(TSHttpHookID eventId)
{
  // Only dealing with the SNI/CERT hook so far.
  // TS_SSL_SNI_HOOK and TS_SSL_CERT_HOOK are the same value
  ink_assert(eventId == TS_SSL_CERT_HOOK);

  // First time through, set the type of the hook that is currently
  // being invoked
  // 将 SNI/CERT Hook 的状态由“初始状态”修改“中间状态”
  if (this->sslHandshakeHookState == HANDSHAKE_HOOKS_PRE) {
    this->sslHandshakeHookState = HANDSHAKE_HOOKS_CERT;
  }

  // 只有“中间状态”时才设置 Hook 函数
  // 如果有多个 Plugin 都 Hook 在 SNI/CERT 处理上时，
  //     Hook 函数就是一个链表，需要一个一个的按照顺序回调
  if (this->sslHandshakeHookState == HANDSHAKE_HOOKS_CERT && eventId == TS_SSL_CERT_HOOK) {
    if (curHook != NULL) {
      // 如果之前已经设置过，那么就取下一个 Hook 函数
      curHook = curHook->next();
    } else {
      // 如果之前没有设置过，那么就获取第一个 Hook 函数
      curHook = ssl_hooks->get(TS_SSL_CERT_INTERNAL_HOOK);
    }
  } else {
    // Not in the right state, or no plugins registered for this hook
    // reenable and continue
    // 状态不正确，或者没有plugin注册这个 Hook 点
    return true;
  }

  // 如果 curHook 不为空，则发起对 Hook 函数的调用
  bool reenabled = true;
  SSLHandshakeHookState holdState = this->sslHandshakeHookState;
  if (curHook != NULL) {
    // Otherwise, we have plugin hooks to run
    this->sslHandshakeHookState = HANDSHAKE_HOOKS_INVOKE;
    // TS_SSL_CERT_HOOK 是最奇葩的 Hook 了，它没有自己的 TS_EVENT_xxxx_HOOK 值
    curHook->invoke(eventId, this);
    // 如果在 Hook 函数中调用了 TSVConnReenable(ssl_vc)，那么就会间接调用了 ssl_vc->reenable(ssl_vc->nh)
    // 在 reenbale 中如果判断所有的 Hook 函数都执行完了，那么就会设置为 HANDSHAKE_HOOKS_DONE 的状态
    // 只有此时 reenabled 才会为 true
    reenabled = (this->sslHandshakeHookState != HANDSHAKE_HOOKS_INVOKE);
  }
  this->sslHandshakeHookState = holdState;
  // 如果 reenabled 为 true 表示不需要挂起 SSL_accept 过程
  // 否则就会在 SSL_accept 过程中挂起，直到 SSL_accept 被再次调用，然后回调本函数
  // 本函数返回 true 才会让 SSL_accept 过程完成。
  return reenabled;
}
```

需要注意 SSLNetVConnection::reenable() 方法是多态的，下面这个才是 TSAPI TSVConnReenable(ssl_vc) 调用的那个：

```
void
SSLNetVConnection::reenable(NetHandler *nh)
{
  if (this->sslPreAcceptHookState != SSL_HOOKS_DONE) {
    this->sslPreAcceptHookState = SSL_HOOKS_INVOKE;
    this->readReschedule(nh);
  } else {
    // Reenabling from the handshake callback
    //
    // Originally, we would wait for the callback to go again to execute additinonal
    // hooks, but since the callbacks are associated with the context and the context
    // can be replaced by the plugin, it didn't seem reasonable to assume that the
    // callback would be executed again.  So we walk through the rest of the hooks
    // here in the reenable.
    if (curHook != NULL) {
      curHook = curHook->next();
      if (curHook != NULL) {
        // Invoke the hook
        curHook->invoke(TS_SSL_CERT_HOOK, this);
      }
    }
    if (curHook == NULL) {
      this->sslHandshakeHookState = HANDSHAKE_HOOKS_DONE;
      this->readReschedule(nh);
    }
  }
}
```

## 参考

  - [OpenSSL::SSL_accept](https://www.openssl.org/docs/manmaster/ssl/SSL_accept.html)
  - [OpenSSL::SSL_get_rbio](https://www.openssl.org/docs/manmaster/ssl/SSL_get_rbio.html)
  - [OpenSSL::SSL_set_bio](https://www.openssl.org/docs/manmaster/ssl/SSL_set_bio.html)
  - [OpenSSL::SSL_connect](https://www.openssl.org/docs/manmaster/ssl/SSL_connect.html)
  - [OpenSSL::SSL_get_error](https://www.openssl.org/docs/manmaster/ssl/SSL_get_error.html)
  - [OpenSSL::SSL_CTX_set_verify](https://www.openssl.org/docs/manmaster/ssl/SSL_CTX_set_verify.html)
  - [SSLUtils.cc](https://github.com/apache/trafficserver/blob/master/iocore/net/SSLUtils.cc)
