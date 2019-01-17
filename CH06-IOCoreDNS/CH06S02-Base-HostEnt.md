# 基础组件：HostEnt

HostEnt 用于保存 DNS 的解析结果，相当于 NetVConnection 里的 VIO。

由于可能会有多个 DNSEntry 对同一个域名发起解析的请求，因此 HostEnt 可能会被多个 DNSEntry 同时引用。

当状态机接收到 `DNS_EVENT_LOOKUP` 事件时，如果传入的 data 参数不为空，就应该尝试将其转换为 HostEnt 类型，并从中读取及复制有需要的信息，但是不可以使用指针引用其内部的数据并永久保存，因为在状态机返回之后，该 HostEnt 对象可能会被销毁。

## 定义

为了能够自动释放 HostEnt 对象，HostEnt 是继承自引用计数基类 RefCountObj，同时可被自动指针 Ptr 模板进行管理。

```
/**
  All buffering required to handle a DNS receipt. For asynchronous DNS,
  only one HostEntBuf will exist in the system. For synchronous DNS,
  one will exist per call until the user deletes them.

*/
struct HostEnt : RefCountObj {
  // hostent 的定义可通过 man 3 gethostbyname 查看
  // 其结构体内的指针指向成员 hostbuf 所在的地址空间
  struct hostent ent;
  // 保存 DNS 响应中解析出来的 TTL 值
  uint32_t ttl;
  // 用于保存 DNS 响应原始报文的长度
  int packet_size;
  // 用于保存接收到的 DNS 响应原始报文
  char buf[MAX_DNS_PACKET_LEN];
  // 指针数组，在 dns_process() 内对 hostbuf 缓冲区进行解析时被赋值
  u_char *host_aliases[DNS_MAX_ALIASES];
  u_char *h_addr_ptrs[DNS_MAX_ADDRS + 1];
  /* 使用 ink_dn_expand() 方法将 buf 内的报文展开，并将结果保存 hostbuf 缓冲区内
   * 通过按照一定的格式对该缓冲区进行遍历和整理后，可以通过以下数据结构对 hostbuf 内的数据进行访问：
   *   - struct hostent
   *   - host_aliases[]
   *   - h_addr_ptrs[]
   */
  u_char hostbuf[DNS_HOSTBUF_SIZE];

  // 用于保存 SRV 记录的解析结果，在 dns_process() 内对 hostbuf 缓冲区进行解析时被赋值
  SRVHosts srv_hosts;

  // 通过 dnsBufAllocator 释放占用的内存资源
  // 通常被 Ptr 模板调用
  virtual void free();

  // 构造函数
  // 整个区域的数据全部使用 0x00 填充
  HostEnt()
  {
    size_t base = sizeof(force_VFPT_to_top); // preserve VFPT
    memset(((char *)this) + base, 0, sizeof(*this) - base);
  }
};
```

以下是 SRVHosts 与 SRV 的定义

```
struct SRV {
  unsigned int weight;
  unsigned int port;
  unsigned int priority;
  unsigned int ttl;
  unsigned int host_len;
  unsigned int key;
  char host[MAXDNAME];

  SRV() : weight(0), port(0), priority(0), ttl(0), host_len(0), key(0) { host[0] = '\0'; }
};

struct SRVHosts {
  // 包含多少个有效的 SRV 记录
  unsigned srv_host_count;
  // 所有 SRV 记录内 host 成员的累计总字节数
  unsigned srv_hosts_length;
  // HOST_DB_MAX_ROUND_ROBIN_INFO 为 16
  // SRV 记录在 dns_process() 内被填充，多于 16 个时会丢弃剩余的部分
  SRV hosts[HOST_DB_MAX_ROUND_ROBIN_INFO];

  ~SRVHosts() {}

  SRVHosts() : srv_host_count(0), srv_hosts_length(0) {}
};
```

# 参考资料

- [I_DNSProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/I_DNSProcessor.h)
- [SRV.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/SRV.h)

