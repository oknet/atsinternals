# DNS 及 HostDB 系统概述

DNS 子系统是一个底层系统，它不被上层协议状态机直接调用，而是通过 HostDB 子系统间接使用。

在 ATS 的设计里，使用 HostDB 子系统作为 DNS 解析结果的缓存，当上层协议状态机需要对 DNS 解析时，先向 HostDB 发起查询请求，而 HostDB 作为 DNS 解析结果的缓存，可以极大的提升 DNS 解析的效率。

我们知道 DNS 系统的设计是允许并希望用户自己建立 DNS Cache Server，以加速 DNS 解析的并降低根服务器的压力，通常 ISP 提供商也会向自己的用户提供公共的 DNS 缓存服务。

HostDB 子系统就相当于是 ATS 系统内部的 DNS Cache Server，它完全模拟并实现了 DNS 缓存服务的功能：

- 接收 DNS 解析请求（通过 HostDBProcessor 提供的接口）
- 查询内部的 DNS 结果缓存
- 对 DNS 结果缓存进行持久化保存，即使 ATS Crash 也可以确保缓存结果不丢失
- 当在缓存中找不到结果时，或者缓存过期时，会通过 DNS 子系统进行查询（通过 DNSProcessor 提供的接口）

另外，为了实现反向代理等功能，社区对 HostDB 的功能进行了响应的扩展。

接下来我将按照以下顺序来介绍域名解析系统：

- DNSEntry
- HostEnt
- DNSHandler
- DNSConnection
- DNSProcessor
- HostDBContinuation
- HostDBProcessor

最后，再来介绍 ATS 对 DNS 解析功能的高级扩展功能：SplitDNS。


