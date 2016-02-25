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

## 定义

```
int
SSLNetVConnection::sslStartHandShake(int event, int &err)
{
  if (sslHandshakeBeginTime == 0) {
    sslHandshakeBeginTime = Thread::get_hrtime();
    // net_activity will not be triggered until after the handshake
    set_inactivity_timeout(HRTIME_SECONDS(SSLConfigParams::ssl_handshake_timeout_in));
  }
  switch (event) {
  case SSL_EVENT_SERVER:
    if (this->ssl == NULL) {
      SSLCertificateConfig::scoped_config lookup;
      IpEndpoint ip;
      int namelen = sizeof(ip);
      safe_getsockname(this->get_socket(), &ip.sa, &namelen);
      SSLCertContext *cc = lookup->find(ip);
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
      if (cc && SSLCertContext::OPT_TUNNEL == cc->opt && this->is_transparent) {
        this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
        sslHandShakeComplete = 1;
        SSL_free(this->ssl);
        this->ssl = NULL;
        return EVENT_DONE;
      }


      // Attach the default SSL_CTX to this SSL session. The default context is never going to be able
      // to negotiate a SSL session, but it's enough to trampoline us into the SNI callback where we
      // can select the right server certificate.
      this->ssl = make_ssl_connection(lookup->defaultContext(), this);
#if !(TS_USE_TLS_SNI)
      // set SSL trace
      if (SSLConfigParams::ssl_wire_trace_enabled) {
        bool trace = computeSSLTrace();
        Debug("ssl", "sslnetvc. setting trace to=%s", trace ? "true" : "false");
        setSSLTrace(trace);
      }
#endif
    }

    if (this->ssl == NULL) {
      SSLErrorVC(this, "failed to create SSL server session");
      return EVENT_ERROR;
    }

    return sslServerHandShakeEvent(err);

  case SSL_EVENT_CLIENT:
    if (this->ssl == NULL) {
      this->ssl = make_ssl_connection(ssl_NetProcessor.client_ctx, this);
    }

    if (this->ssl == NULL) {
      SSLErrorVC(this, "failed to create SSL client session");
      return EVENT_ERROR;
    }

    return sslClientHandShakeEvent(err);

  default:
    ink_assert(0);
    return EVENT_ERROR;
  }
}

int
SSLNetVConnection::sslServerHandShakeEvent(int &err)
{
  if (SSL_HOOKS_DONE != sslPreAcceptHookState) {
    // Get the first hook if we haven't started invoking yet.
    if (SSL_HOOKS_INIT == sslPreAcceptHookState) {
      curHook = ssl_hooks->get(TS_VCONN_PRE_ACCEPT_INTERNAL_HOOK);
      sslPreAcceptHookState = SSL_HOOKS_INVOKE;
    } else if (SSL_HOOKS_INVOKE == sslPreAcceptHookState) {
      // if the state is anything else, we haven't finished
      // the previous hook yet.
      curHook = curHook->next();
    }
    if (SSL_HOOKS_INVOKE == sslPreAcceptHookState) {
      if (0 == curHook) { // no hooks left, we're done
        sslPreAcceptHookState = SSL_HOOKS_DONE;
      } else {
        sslPreAcceptHookState = SSL_HOOKS_ACTIVE;
        ContWrapper::wrap(mutex, curHook->m_cont, TS_EVENT_VCONN_PRE_ACCEPT, this);
        return SSL_WAIT_FOR_HOOK;
      }
    } else { // waiting for hook to complete
             /* A note on waiting for the hook. I believe that because this logic
                cannot proceed as long as a hook is outstanding, the underlying VC
                can't go stale. If that can happen for some reason, we'll need to be
                more clever and provide some sort of cancel mechanism. I have a trap
                in SSLNetVConnection::free to check for this.
             */
      return SSL_WAIT_FOR_HOOK;
    }
  }

  // If a blind tunnel was requested in the pre-accept calls, convert.
  // Again no data has been exchanged, so we can go directly
  // without data replay.
  // Note we can't arrive here if a hook is active.
  if (TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) {
    this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
    SSL_free(this->ssl);
    this->ssl = NULL;
    // Don't mark the handshake as complete yet,
    // Will be checking for that flag not being set after
    // we get out of this callback, and then will shuffle
    // over the buffered handshake packets to the O.S.
    return EVENT_DONE;
  } else if (TS_SSL_HOOK_OP_TERMINATE == hookOpRequested) {
    sslHandShakeComplete = 1;
    return EVENT_DONE;
  }

  int retval = 1; // Initialze with a non-error value

  // All the pre-accept hooks have completed, proceed with the actual accept.
  if (BIO_eof(SSL_get_rbio(this->ssl))) { // No more data in the buffer
    // Read from socket to fill in the BIO buffer with the
    // raw handshake data before calling the ssl accept calls.
    retval = this->read_raw_data();
    if (retval == 0) {
      // EOF, go away, we stopped in the handshake
      SSLDebugVC(this, "SSL handshake error: EOF");
      return EVENT_ERROR;
    }
  }

  ssl_error_t ssl_error = SSLAccept(ssl);
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  if (ssl_error != SSL_ERROR_NONE) {
    err = errno;
    SSLDebugVC(this, "SSL handshake error: %s (%d), errno=%d", SSLErrorName(ssl_error), ssl_error, err);

    // start a blind tunnel if tr-pass is set and data does not look like ClientHello
    char *buf = handShakeBuffer->buf();
    if (getTransparentPassThrough() && buf && *buf != SSL_OP_HANDSHAKE) {
      SSLDebugVC(this, "Data does not look like SSL handshake, starting blind tunnel");
      this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      sslHandShakeComplete = 0;
      return EVENT_CONT;
    }
  }

  switch (ssl_error) {
  case SSL_ERROR_NONE:
    if (is_debug_tag_set("ssl")) {
      X509 *cert = SSL_get_peer_certificate(ssl);

      Debug("ssl", "SSL server handshake completed successfully");
      if (cert) {
        debug_certificate_name("client certificate subject CN is", X509_get_subject_name(cert));
        debug_certificate_name("client certificate issuer CN is", X509_get_issuer_name(cert));
        X509_free(cert);
      }
    }

    sslHandShakeComplete = true;

    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake completed successfully");
    // do we want to include cert info in trace?

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

#if TS_USE_TLS_ALPN
      SSL_get0_alpn_selected(ssl, &proto, &len);
#endif /* TS_USE_TLS_ALPN */

#if TS_USE_TLS_NPN
      if (len == 0) {
        SSL_get0_next_proto_negotiated(ssl, &proto, &len);
      }
#endif /* TS_USE_TLS_NPN */

      if (len) {
        // If there's no NPN set, we should not have done this negotiation.
        ink_assert(this->npnSet != NULL);

        this->npnEndpoint = this->npnSet->findEndpoint(proto, len);
        this->npnSet = NULL;

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
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_SNI_RESOLVE");
#elif SSL_ERROR_WANT_X509_LOOKUP
  case SSL_ERROR_WANT_X509_LOOKUP:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_X509_LOOKUP");
#endif
#if defined(SSL_ERROR_WANT_SNI_RESOLVE) || defined(SSL_ERROR_WANT_X509_LOOKUP)
    if (this->attributes == HttpProxyPort::TRANSPORT_BLIND_TUNNEL || TS_SSL_HOOK_OP_TUNNEL == hookOpRequested) {
      this->attributes = HttpProxyPort::TRANSPORT_BLIND_TUNNEL;
      sslHandShakeComplete = 0;
      return EVENT_CONT;
    } else {
      //  Stopping for some other reason, perhaps loading certificate
      return SSL_WAIT_FOR_HOOK;
    }
#endif

  case SSL_ERROR_WANT_ACCEPT:
    TraceIn(trace, get_remote_addr(), get_remote_port(), "SSL server handshake ERROR_WANT_ACCEPT");
    return EVENT_CONT;

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


int
SSLNetVConnection::sslClientHandShakeEvent(int &err)
{
#if TS_USE_TLS_SNI
  if (options.sni_servername) {
    if (SSL_set_tlsext_host_name(ssl, options.sni_servername)) {
      Debug("ssl", "using SNI name '%s' for client handshake", options.sni_servername.get());
    } else {
      Debug("ssl.error", "failed to set SNI name '%s' for client handshake", options.sni_servername.get());
      SSL_INCREMENT_DYN_STAT(ssl_sni_name_set_failure);
    }
  }

#endif

  SSL_set_ex_data(ssl, get_ssl_client_data_index(), this);
  ssl_error_t ssl_error = SSLConnect(ssl);
  bool trace = getSSLTrace();
  Debug("ssl", "trace=%s", trace ? "TRUE" : "FALSE");

  switch (ssl_error) {
  case SSL_ERROR_NONE:
    if (is_debug_tag_set("ssl")) {
      X509 *cert = SSL_get_peer_certificate(ssl);

      Debug("ssl", "SSL client handshake completed successfully");
      // if the handshake is complete and write is enabled reschedule the write
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

    sslHandShakeComplete = true;
    return EVENT_DONE;

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
  } break;
  }
  return EVENT_CONT;
}
```

## 参考
