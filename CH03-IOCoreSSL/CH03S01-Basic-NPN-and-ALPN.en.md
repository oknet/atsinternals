# Basic Knowledge: NPN and APLN
Before you reading the implementation of SSL in ATS, Let's learn about NPN and APLN first. They are supposed to solve the same issue:

  - Which protocol will be performed over plaintext after establishing a SSL connection?

In the OSI model, SSL, widely used in many scenarios, is generally considered in the 6th layer. However,

  - which protocol will be applied to the decrypted pipe, and how to apply it?
  - It can only be decided by an agreement between the client and the server.
  - For instance, on port 443, it's decided that we use SSL to encrypt traffic in HTTP/1.x.

As SPDY is created, a new requirement is raised which is using several possible application layer protocols on same port,

  - We don't know which protocol is applied unless data is decrypted.
  - We should know the protocol before data gets decrypted.
  - Therefore TLS-NPN extension is introduced.

## TLS-NPN extension

TLS-NPN is short for Transport Layer Security - Next Protocol Negotiation (instead of NPN bipolar transistor, haha), it is implemented as an extension of TLS by Google to support SPDY as an application layer protocol.

TLS-NPN extension implements that in SSL handshake:

  - The server informs the client its supporting protocols.
  - Then the client responses the server which protocol will be applied on this connection.

Please refer to [RFC 5246 Section 7.3](https://tools.ietf.org/html/rfc5246#section-7.3) the detail procedures for supporting the NPN extension of SSL handshake.

A full handshake with `EncryptedExtensions` has the following flow (contrast with [section 7.3 of RFC 5246](https://tools.ietf.org/html/rfc5246#section-7.3)) that explains how to attach NPN extension:

```
Client                                                 Server

ClientHello (NPN extension)   -------->
                                   ServerHello (NPN extension &
                                              list of protocols)
                                                 Certificate*
                                           ServerKeyExchange*
                                          CertificateRequest*
                              <--------       ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
EncryptedExtensions(NPN extension is included)
Finished                      -------->
                                           [ChangeCipherSpec]
                              <--------              Finished
Application Data              <------->      Application Data
```

An abbreviated handshake with `EncryptedExtensions` has the following flow:

```
Client                                                 Server

ClientHello (NPN extension)   -------->
                                   ServerHello (NPN extension &
                                              list of protocols)
                                           [ChangeCipherSpec]
                              <--------              Finished
[ChangeCipherSpec]
EncryptedExtensions(NPN extension is included)
Finished                      -------->
Application Data              <------->      Application Data
```

One of extensions in the `EncryptedExtensions` message is `next_protocol_negotiation`(TBD) of which the `extension_data` has the following format:

```
struct {
  opaque selected_protocol<0..255>;
  opaque padding<0..255>;
} NextProtocolNegotiationEncryptedExtension;
```

The length of this structure is an integer multiple of 32 bytes. The `selected_protocol` is a string indicating the selected protocol which could be one in the following list:

  - http/1.0
  - http/1.1
  - spdy/1
  - spdy/2
  - spdy/3
  - spdy/3.1

In SSL handshake with NPN,

  - The `EncryptedExtensions` message contains a series of Extension structures.
  - If the server included a `NextProtocolNegotiationEncryptedExtension` extension in its ServerHello message,
  - the client MUST include an Extension with extension_type equal to `NextProtocolNegotiationEncryptedExtension`.

Unlike many other TLS extensions, this extension does not establish properties of the session, only of the connection. When session resumption or session tickets [RFC5077](https://tools.ietf.org/html/rfc5077) are used, the previous contents of this extension are irrelevant and only the values in the new handshake messages are considered.

Other information:

  - the extension number of next_protocol_negotiation is 13172 (0x3374)
  - the message number of NextProtocol is 67 (0x43)
  - OpenSSL supports NPN starting with 1.0.0d

## TLS-ALPN extension

ALPN is short for Application Layer Protocol Negotiation, designed to replace NPN.

Due to NPN needs to insert a message before Finished, which is not perfect.

The ALPN extension is intended to follow the typical design of TLS protocol extensions. Specifically, the negotiation is performed entirely within the client/server hello exchange in accordance with the established TLS architecture.

TLS-ALPN extension implements that in SSL handshake:

  - The client informs the server its supporting protocols.
  - Then the server responses the client which protocol will be applied on this connection.

Please refer to [RFC 7301 Section 3.1](https://tools.ietf.org/html/rfc7301#section-3.1) the detail procedures for supporting the ALPN extension of SSL handshake.

A full handshake with the `application_layer_protocol_negotiation` extension in the ClientHello and ServerHello messages has the following flow (contrast with [section 7.3 of RFC 5246](https://tools.ietf.org/html/rfc5246#section-7.3)) that explains how to attach ALPN extension:

```
Client                                                                     Server

ClientHello (ALPN extension & list of protocols)  -------->
                                                      ServerHello (ALPN extension &
                                                                 selected protocol)
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

An abbreviated handshake with the `application_layer_protocol_negotiation` extension has the following flow:

```
Client                                                                     Server

ClientHello (ALPN extension & list of protocols)  -------->
                                                      ServerHello (ALPN extension &
                                                                 selected protocol)
                                                               [ChangeCipherSpec]
                                                  <--------              Finished
[ChangeCipherSpec]
Finished                                          -------->
Application Data                                  <------->      Application Data
```

In SSL handshake with ALPN,

  - ClientHello contains the list of protocols advertised by the client.
  - the server sends the selected protocol as part of the TLS ServerHello message.

ClientHello includes a "ProtocolNameList" structure

```
   opaque ProtocolName<1..2^8-1>;

   struct {
       ProtocolName protocol_name_list<2..2^16-1>
   } ProtocolNameList;
```

ServerHello also includes a "ProtocolNameList" structure the same as the above but only contains one "ProtocolName" that is also advertised in ClientHello.

In the event that the server supports no protocols that the client advertises, then the server SHALL respond with a fatal `no_application_protocol` alert:

```
enum {
       no_application_protocol(120),
       (255)
   } AlertDescription;
```

Unlike many other TLS extensions, this extension does not establish properties of the session, only of the connection. When session resumption or session tickets [RFC5077](https://tools.ietf.org/html/rfc5077) are used, the previous contents of this extension are irrelevant and only the values in the new handshake messages are considered.

Other information:

  - Application Layer Protocol Negotiation extension number is 16 (0x10)
  - OpenSSL supports ALPN starting with 1.0.2

## Relationship between SSL with NPN/ALPN and protocols from other layers

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

## Reference

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
