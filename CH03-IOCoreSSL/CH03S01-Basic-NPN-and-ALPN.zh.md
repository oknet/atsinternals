# 基础知识：NPN 和 ALPN

在了解ATS的SSL实现之前，首先需要了解一下 NPN 和 ALPN，他们都是为了解决同一个问题：

  - 在建立SSL会话之后，解析出来的明文，是什么协议？

在OSI的7层模型里，SSL 被普遍理解为处于第6层的位置，并且被广泛用于很多场景，但是：

  - 对于解密后的管道内的协议是什么协议？怎么使用？
  - 只能由Client与Server约定好。
  - 例如在443端口上，约定了使用SSL协议来加密HTTP/1.x的流量。

随着 SPDY 协议的诞生，在同一个端口上使用多种可能的协议进行通信的需求被提出，

  - 由于数据内容已经被加密了，无法直接判断，只能解密数据之后才能知道协议类型
  - 需要在数据解密之前就知道是哪种类型的协议数据被加密了
  - 于是 TLS-NPN 就诞生了

## TLS-NPN 扩展

TLS-NPN 全称是Transport Layer Security - Next Protocol Negotiation，它是 Google 为了支持 SPDY 协议作为一个应用层协议使用，而为TLS定义的一个扩展。

TLS-NPN 可以简写为 NPN （不是电子元件三极管，哈哈～）

TLS-NPN 扩展实现了在SSL握手过程中：

  - 由Server告知Client它可以在SSL会话解密后识别哪些协议，
  - 然后Client再告知Server它的这次通信会把哪种协议加密后传送给Server。

以下有关支持 NPN 扩展的 SSL 握手的详细流程请参考：[RFC 5246 Section 7.3](https://tools.ietf.org/html/rfc5246#section-7.3) 

首先是 SSL Full HandShake 过程，如何附带 NPN 扩展

```
Client                                                 Server

ClientHello (带有NP扩展标志)  -------->
                                    ServerHello (带有NP扩展标志
                                               & 支持的协议列表)
                                                 Certificate*
                                           ServerKeyExchange*
                                          CertificateRequest*
                              <--------       ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
EncryptedExtensions（包含NP信息）
Finished                      -------->
                                           [ChangeCipherSpec]
                              <--------              Finished
Application Data              <------->      Application Data
```

然后是 SSL Abbreviated HandShake 过程，如何附带 NPN 扩展

```
Client                                                 Server

ClientHello (带有NP扩展标志)  -------->
                                   ServerHello (带有NP扩展标志
                                              & 支持的协议列表)
                                           [ChangeCipherSpec]
                              <--------              Finished
[ChangeCipherSpec]
EncryptedExtensions（包含NP信息）
Finished                      -------->
Application Data              <------->      Application Data
```

上面提到的 NP 全称是 Next Protocol，NP信息是一个结构体：

```
struct {
  opaque selected_protocol<0..255>;
  opaque padding<0..255>;
} NextProtocol;
```

这个结构体的长度是32字节的整数倍，selected_protocol是一个字符串，表示选择的协议类型，目前支持：

  - http/1.0
  - http/1.1
  - spdy/1
  - spdy/2
  - spdy/3
  - spdy/3.1

实现NPN扩展需要在SSL会话握手过程里插入`EncryptedExtensions`：

  - 如果客户端收到的 `ServerHello` 中包含了一个NP信息，那么客户端也要回应一个NP信息
  - 客户端发送 `ChangeCipherSpec` 和 `Finished` 之间，增加了一个发送 `EncryptedExtensions` 信息的部分
  - 在 `EncryptedExtensions` 中包含了一系列各式各样的扩展信息，客户端可以把NP信息插入其中

但是需要注意的是，NP扩展，只针对连接，而不是会话：

  - 因此在会话重用的时候（Abbreviated HandShake）也需要重新进行NP扩展的识别。
  - 同样的，在出现会话重协商（Renegotiation）时，也需要重新进行NP扩展的识别。

其它信息：

  - Next protocol negotiation 扩展编号为：13172 (0x3374)
  - NextProtocol handshake 消息编号为：67 (0x43)
  - OpenSSL 1.0.0d 版本开始支持 NPN 功能



## TLS-ALPN 扩展

ALPN 的全称为 Application Layer Protocol Negotiation，被设计为 NPN 的替代者。

由于 NPN 需要在 Finished 之前插入一个流程，实现的不够完美，因此 ALPN 的实现，完全在 ClientHello 和 ServerHello 中进行。

TLS-ALPN 扩展实现了在SSL握手过程中：

  - 由Client告知Server它可以通过哪些协议把数据传递给Server
  - 然后Server再从这些协议中选择一个它可以支持的协议告诉Client

可以看到 NPN 是服务器端提供列表，客户端来选择，而 ALPN 是客户端提供列表，由服务器端来选择。

以下有关支持 ALPN 扩展的 SSL 握手的详细流程请参考：[RFC 7301 Section 3.1](https://tools.ietf.org/html/rfc7301#section-3.1) 

首先是 SSL Full HandShake 过程，如何附带 ALPN 扩展：

```
Client                                                                     Server

ClientHello (带有ALPN扩展标志 & 支持的协议列表)   -------->
                                                      ServerHello (带有ALPN扩展标志
                                                                       & 选中的协议)
                                                                     Certificate*
                                                               ServerKeyExchange*
                                                              CertificateRequest*
                                                  <--------       ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
Finished                                          -------->
                                                               [ChangeCipherSpec]
                                                  <--------              Finished
Application Data                                  <------->      Application Data
```

然后是 SSL Abbreviated HandShake 过程，如何附带 ALPN 扩展

```
Client                                                                     Server

ClientHello (带有ALPN扩展标志 & 支持的协议列表)   -------->
                                                      ServerHello (带有ALPN扩展标志
                                                                       & 选中的协议)
                                                               [ChangeCipherSpec]
                                                  <--------              Finished
[ChangeCipherSpec]
Finished                                          -------->
Application Data                                  <------->      Application Data
```

这个ALPN扩展就是在SSL会话握手过程中，

  - 客户端发送 ClientHello 时，包含了一个客户端支持的协议列表（协议类型及名称与NPN一样）
  - 服务端发送 ServerHello 时，从客户端提供的协议中选择一个将结果告诉客户端

客户端的ClientHello中包含的列表格式如下：

```
   opaque ProtocolName<1..2^8-1>;

   struct {
       ProtocolName protocol_name_list<2..2^16-1>
   } ProtocolNameList;
```

服务端ServerHello中同样包含一个协议列表，跟上面的格式一样，但是只能含有一项。

如果客户端发送的协议列表，服务端都不支持，ServerHello 中则包含：

```
enum {
       no_application_protocol(120),
       (255)
   } AlertDescription;
```

但是需要注意的是，ALPN扩展，同样只针对连接，而不是会话：

  - 因此在会话重用的时候（Abbreviated HandShake）也需要重新进行ALPN协商过程。
  - 同样的，在出现会话重协商（Renegotiation）时，也需要重新进行ALPN协商过程。

其它信息：

  - Application Layer Protocol Negotiation 扩展编号为：16 (0x10)
  - OpenSSL 1.0.2 版本开始支持 ALPN 功能

## NPN / ALPN 与 SSL 和上层协议之间的关系

```
 SPDY <------- HTTP/1.x
   |
   |
   V
  SSL <------- NPN/ALPN
   |
   |
   V
  TCP
```

## 参考资料

- [NPN and ALPN](https://www.imperialviolet.org/2013/03/20/alpn.html)
- [NPN 与 ALPN](https://zlb.me/2013/07/19/npn-and-alpn/)
- [SPDY简介](https://zlb.me/2013/01/07/spdy-intro/)
- NPN
  - [Draft NPN](http://tools.ietf.org/html/draft-agl-tls-nextprotoneg-04)
  - [GoogleCode technotes](https://github.com/agl/technotes.git)
  - [NPN protocol and explanation about its need to tunnel SPDY over HTTPS](https://tools.ietf.org/agenda/82/slides/tls-3.pdf)
- ALPN
  - [Draft ALPN](http://tools.ietf.org/html/draft-friedl-tls-applayerprotoneg-00)
  - [RFC7301 ALPN](https://tools.ietf.org/html/rfc7301)
  - [Wikipedia APLN](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation)
