# DNS 及 HostDB 系统概述

DNS 子系统是一个底层系统，它不被上层协议状态机直接调用，而是通过 HostDB 子系统间接使用。在 ATS 的设计里，使用 HostDB 子系统作为 DNS 解析结果的缓存，当上层协议状态机需要对 DNS 解析时，先向 HostDB 发起查询请求，而 HostDB 作为 DNS 解析结果的缓存，可以极大的提升 DNS 解析的效率。

接下来，我将按照以下顺序来介绍域名解析系统：

- DNSEntry
- HostEnt
- DNSHandler
- DNSConnection
- DNSProcessor
- HostDBContinuation
- HostDBProcessor

最后，再来介绍 ATS 对 DNS 解析功能的高级扩展功能：SplitDNS。

## DNS 子系统

DNS 子系统主要包含以下几个部分：

- DNS 请求发送部分
  - `DNSProcessor::getby()` 方法
  - `write_dns()` 方法
  - `write_dns_event()` 方法
- DNS 响应接收部分
  - `DNSHandler::recv_dns()` 方法
  - `dns_process()` 方法
- 解析结果回调部分
  - `dns_result()` 方法
  - `DNSEntry::post()` 方法
  - `DNSEntry::postEvent()` 方法
- Name Server 健康检查
  - 主要在 `DNSHandler::mainEvent()` 方法内实现
  - `DNSHandler::recover()` 方法
  - `DNSHandler::retry_named()` 方法
  - `DNSHandler::try_primary_named()` 方法
  - `switch_named()` 方法
  - `DNSHandler::failover()` 方法
  - `DNSHandler::rr_failure()` 方法
  - `DNSHandler::received_one()` 方法
  - `DNSHandler::sent_one()` 方法
  - `DNSHandler::failover_now()` 方法
  - `DNSHandler::failover_soon()` 方法

由于 DNS 请求与响应不是按照“请求1--响应1--请求2--响应2--...”顺序进行，因此每一个请求和响应都包含了一个 Query ID，在接收到一个响应时，可以通过这个 Query ID 找到与之关联的请求。

在 DNS 子系统里，通过 `DNSHandler::qid_in_flight[]` 数组实现了一个 bitmap，用来生成 Query ID，并确保 Query ID 的唯一性。

所有的 DNS 请求都由 `DNSEntry` 对象来描述，在发起请求之前，会将 `DNSEntry` 放入到 `DNSHandler::entries` 队列里，这样在收到 DNS 响应之后，就可以根据 Query ID 去 `DNSHandler::entries` 队列里查找对应的请求。

当发起一个新的 DNS 请求时，会首先在 `DNSHandler::entries` 队列里查询，是否存对同一个域名的查询请求，如果已经存在，那么这个 DNSEntry 对象是一个头结点，其内部有一个 `DNSEntry::dups` 队列来保存请求相同域名的 DNSEntry 对象，当收到 DNS 响应时，头结点将把 DNS 解析结果与 `DNSEntry::dups` 队列内的 DNSEntry 对象共享，从而减少 DNS 请求发送的次数，同时提高了 DNS 子系统的工作效率。

因此 `DNSHandler::entries` 队列是可以按照 Query Domain Name 和 Query ID 两种方式进行检索的。

可以把 `DNSHandler::entries` 队列和 `DNSEntry::dups` 队列组合起来，看成是一个保存了许多 DNSEntry 对象的表格，每一行的所有 DNSEntry 对象所请求的域名都是相同的，表格第一列的 DNSEntry 是 `DNSEntry::dups` 队列的头结点，负责发起 DNS 请求，并将 DNS 结果共享给 `DNSEntry::dups` 队列内的 DNSEntry。


## HostDB 子系统

我们知道 DNS 系统的设计是允许并希望用户自己建立 DNS Cache Server，以加速 DNS 解析的并降低根服务器的压力，通常 ISP 提供商也会向自己的用户提供公共的 DNS 缓存服务。

HostDB 子系统就相当于是 ATS 系统内部的 DNS Cache Server，它完全模拟并实现了 DNS 缓存服务的功能：

- 接收 DNS 解析请求（通过 HostDBProcessor 提供的接口）
- 查询内部的 DNS 结果缓存
- 对 DNS 结果缓存进行持久化保存，即使 ATS Crash 也可以确保缓存结果不丢失
- 当在缓存中找不到结果时，或者缓存过期时，会通过 DNS 子系统进行查询（通过 DNSProcessor 提供的接口）

另外，为了实现反向代理等功能，社区对 HostDB 的功能进行了相应的扩展。




