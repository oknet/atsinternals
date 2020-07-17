# DNS 子系统概述

DNS 子系统是一个底层系统，它不被上层协议状态机直接调用，而是通过 HostDB 子系统间接使用。
在最早期的开源代码里还有 ATS 作为 DNS Cache Server 的相关代码，但是在正式发布的第一个版本里这部分功能被删除了。

接下来，我将按照以下顺序来介绍域名解析系统：

- DNSEntry
- HostEnt
- DNSHandler
- DNSConnection
- DNSProcessor
- SplitDNS

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

可以把 `DNSHandler::entries` 队列和 `DNSEntry::dups` 队列组合起来，看成是一个保存了许多 DNSEntry 对象的表格，每一行的所有 DNSEntry 对象所请求的域名都是相同的，表格第一列的 DNSEntry 是 `DNSEntry::dups` 队列的头结点，负责发起 DNS 请求，并将 DNS 结果共享给 `DNSEntry::dups` 队列内的 DNSEntry。

在 `DNSHandler::entries` 表中每一行的第一个 DNSEntry 对象有以下状态：

- 从未发送过任何 DNS 解析请求（新请求）
- 曾经发送过 DNS 解析请求，又分为：
  - 正在等待 DNS 响应（在途请求）
  - 准备再次发送 DNS 解析请求（重试请求）

而 `DNSHandler::entries` 表可以按照 Query Domain Name 和 Query ID 两种方式进行检索，由于在途 DNS 解析请求的最大数量被控制在 2048 (dns_max_dns_in_flight) 个以内，所以对 `DNSHandler::entries` 表采用了遍历的方式进行检索，但是新的 DNS 请求是直接追加到 `DNSHandler::entries` 表，而该表不设置最大长度，因此进行遍历操作的效率是不可控的，这里是否可以使用 Hash 方式进行优化？是否可以拆分为两个表，一个专门用来存储新的 DNS 解析请求？

ATS 最初是支持 DNS Proxy/Cache 功能的，可以接收来自客户端的 DNS 请求，可以将 ATS 作为 DNS 服务器使用。

但是这部分功能由于几乎没有用户使用，而且功能也不够完善，因此被社区砍掉了：

- TS-422 Remove DNS proxy support.
- commit: 82bab7efec97c234090211875ee097758315c9c4

如果需要将 ATS 作为 DNS 服务器来使用的话，可以参考一下这部分被砍掉的代码。

