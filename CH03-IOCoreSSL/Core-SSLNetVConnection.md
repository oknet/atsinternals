# 核心组件 SSLNetVConnection

SSLNetVConnection 继承自 UnixNetVConnection，对部分方法进行了重载：

  - net_read_io
  - load_buffer_and_write
  - SSLNetVConnection
  - do_io_close
  - free
  - reenable
  - getSSLHandShakeComplete
  - setSSLHandShakeComplete
  - getSSLClientConnection
  - setSSLClientConnection
  - getSSLSessionCacheHit
  - setSSLSessionCacheHit

同时增加了一些方法：

  - enableRead
  - read_raw_data
  - initialize_handshake_buffers
  - free_handshake_buffers
  - sslStartHandShake
  - sslServerHandShakeEvent
  - sslClientHandShakeEvent
  - registerNextProtocolSet
  - endpoint
  - advertise_next_protocol
  - select_next_protocol
  - sslContextSet
  - callHooks
  - calledHooks
  - get_ssl_iobuf
  - set_ssl_iobuf
  - get_ssl_reader
  - isEosRcvd

在UnixNetVConnection的基础上构建了对SSL会话的支持。

# 定义

# 参考资料

- [P_SSLNetVConnection.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNetVConnection.h)
- [SSLNetVConnection.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNetVConnection.cc)


  
