# 基础知识：NPN 和 ALPN

在了解ATS的SSL实现之前，首先需要了解一下 NPN 和 ALPN，他们都是为了解决同一个问题：

  - 在建立SSL会话之后，解析出来的明文，是什么协议？

于是 TLS-NPN 就诞生了，TLS-NPN全称是Transport Layer Security - Next Protocol Negotiation。

TLS-NPN 是 Google 为了支持 SPDY 协议作为一个应用层协议使用，而为TLS定义的一个扩展。

所以正式的名字叫做 TLS-NPN，简写为 NPN （不是电子元件三极管，哈哈～）


## TLS-NPN 扩展

以下有关支持 NPN 扩展的 SSL 握手的详细流程请参考：[RFC 5246 Section 7.3](https://tools.ietf.org/html/rfc5246#section-7.3) 

首先是 SSL Full HandShake 过程，如何附带 NPN 扩展

```
Client                                               Server

ClientHello (带有NP扩展标志)  -------->
                                                 ServerHello (带有NP扩展标志 & 支持的协议列表)
                                                Certificate*
                                          ServerKeyExchange*
                                         CertificateRequest*
                             <--------       ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
EncryptedExtensions（包含NP信息）
Finished                     -------->
                                          [ChangeCipherSpec]
                             <--------              Finished
Application Data             <------->      Application Data
```

然后是 SSL Abbreviated HandShake 过程，如何附带 NPN 扩展

```
Client                                                Server

ClientHello (带有NP扩展标志)    -------->
                                                 ServerHello (带有NP扩展标志 & 支持的协议列表)
                                          [ChangeCipherSpec]
                              <--------             Finished
[ChangeCipherSpec]
EncryptedExtensions（包含NP信息）
Finished                      -------->
Application Data              <------->     Application Data
```

上面提到的 NP 全称是 Next Protocol，NP信息是一个结构体：

```
struct {
  opaque selected_protocol<0..255>;
  opaque padding<0..255>;
} NextProtocol;
```

这个结构体的长度是32字节的整数倍，selected_protocol是一个字符串，表示选择的协议类型，目前支持：

  - http/1.1
  - spdy/1
  - spdy/2

但是需要注意的是，NP扩展，只针对连接，而不是会话：

  - 因此在会话重用的时候（Abbreviated HandShake）也需要重新进行NP扩展的识别。
  - 同样的，在出现会话重协商（Renegotiation）时，也需要重新进行NP扩展的识别。

其它信息：

  - Next protocol negotiation 扩展编号为：13172
  - NextProtocol handshake 消息编号为：67

## TLS-ALPN 扩展

## 参考资料

- [Draft NPN](http://tools.ietf.org/html/draft-agl-tls-nextprotoneg-04)
- [Draft ALPN](http://tools.ietf.org/html/draft-friedl-tls-applayerprotoneg-00)
- [RFC7301 ALPN](https://tools.ietf.org/html/rfc7301)
- [GoogleCode technotes](https://github.com/agl/technotes.git)
- [NPN and ALPN](https://www.imperialviolet.org/2013/03/20/alpn.html)
- [NPN 与 ALPN](https://zlb.me/2013/07/19/npn-and-alpn/)
- [SPDY简介](https://zlb.me/2013/01/07/spdy-intro/)
