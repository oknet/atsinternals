# Basic Knowledge: NPN and APLN
Before you read the implementation of SSL in ATS, Let's learn about NPN and APLN first. They are supposed to solve the same issue:
  - Which protocol will be performed over plaintext after SSL connection is established?
In the OSI model, SSL, widely used in many scenarios, is generally considered in the 6th layer. However,
  - which protocol will be applied to the decrypted pipe, and how to apply it?
  - It can only be decided by an agreement between the client and the server.
  - For instance, on port 443, it's decided that we use SSL to encrypt traffic in HTTP/1.x.

As SPDY is created, a requirement that using several possible application layer protocols on the same port has been put forward,

  - We can't tell which protocol is applied unless data gets decrypted.
  - We need to know which protocol before data gets decrypted.
  - Therefore TLS-NPN extension is implemented.

TLS-NPN extension implements that in SSL handshake:

  - The server tells the client what protocols it supports.
  - Then the client tells the server which protocol will be applied on this connection.

TLS-NPN is short for Transport Layer Security - Next Protocol Negotiation (instead of NPN bipolar transistor, haha), it is implemented as an extension of TLS by Google to support SPDY as an application layer protocol.


## TLS-NPN extension

A full handshake with `EncryptedExtensions` has the following flow (contrast with [RFC 5246 Section 7.3](https://tools.ietf.org/html/rfc5246#section-7.3))

```
Client                                               Server

ClientHello (NPN extension)   -------->
                                               ServerHello
                                                 (NPN extension &
                                                  list of protocols)
                                              Certificate*
                                        ServerKeyExchange*
                                       CertificateRequest*
                            <--------      ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
EncryptedExtensions(NPN extension is included)
Finished                     -------->
                                        [ChangeCipherSpec]
                            <--------             Finished
Application Data             <------->     Application Data
```

An abbreviated handshake with `EncryptedExtensions` has the following flow:

```
Client                                                Server

ClientHello (NPN extension)    -------->
                                               ServerHello
                                                 (NPN extension &
                                                  list of protocols)
                                        [ChangeCipherSpec]
                             <--------            Finished
[ChangeCipherSpec]
EncryptedExtensions(NPN extension is included)
Finished                      -------->
Application Data              <------->    Application Data
```

One of extensions in the `EncryptedExtensions` message is `next_protocol_negotiation`(TBD) of which the `extension_data` has the following format:

```
struct {
  opaque selected_protocol<0..255>;
  opaque padding<0..255>;
} NextProtocolNegotiationEncryptedExtension;
```

This structure has a length of an integer multiple of 32. selected_protocol is a string indicating the protocol selected, which could be one in the following list:

  - http/1.0
  - http/1.1
  - spdy/1
  - spdy/2
  - spdy/3
  - spdy/3.1

In SSL handshake, TLS NPN sends a NextProtocol message, after the client sends ChangeCipherSpec and before it sends Finished.

Unlike many other TLS extensions, this extension does not establish properties of the session, only of the connection. When session resumption or session tickets [RFC5077](https://tools.ietf.org/html/rfc5077) are used, the previous contents of this extension are irrelevant and only the values in the new handshake messages are considered.

Other information:

  - the extension number of next_protocol_negotiation is 13172 (0x3374)
  - the message number of NextProtocol is 67 (0x43)
  - OpenSSL supports NPN starting with 1.0.0d

## TLS-ALPN extension

ALPN is short for Application Layer Protocol Negotiation, designed to be a replacement of NPN.

Since NPN needs to insert a message before Finished, which is not perfect. Therefore the negotiation is done by ClientHello and ServerHello with ALPN.


The detailed flow for a full SSL handshake with ALPN is as follows. Please refer to [RFC 7301 Section 3.1](https://tools.ietf.org/html/rfc7301#section-3.1)


```
Client                                              Server

ClientHello                     -------->       ServerHello
 (ALPN extension &                               (ALPN extension &
  list of protocols)                              selected protocol)
                                               Certificate*
                                               ServerKeyExchange*
                                               CertificateRequest*
                               <--------       ServerHelloDone
Certificate*
ClientKeyExchange
CertificateVerify*
[ChangeCipherSpec]
Finished                        -------->
                                               [ChangeCipherSpec]
                               <--------       Finished
Application Data                <------->       Application Data
```

Then here is SSL Abbreviated HandShake with ALPN:

```
Client                                              Server

ClientHello                     -------->       ServerHello
 (ALPN extension &                               (ALPN extension &
  list of protocols)                              selected protocol)
                                               [ChangeCipherSpec]
                               <--------       Finished
[ChangeCipherSpec]
Finished                        -------->
Application Data                <------->       Application Data

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

ServerHello extension is structured the same as described above for the client "extension_data", except that the "ProtocolNameList" MUST contain  exactly one "ProtocolName".

In the event that the server supports no protocols that the client advertises, then the server SHALL respond

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
