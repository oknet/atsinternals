# 基础组件：HostEnt

HostEnt 用于保存 DNS 的解析结果，相当于 NetVConnection 里的 VIO。

由于可能会有多个 DNSEntry 对同一个域名发起解析的请求，因此 HostEnt 可能会被多个 DNSEntry 同时引用。

## 定义

为了能够自动释放 HostEnt 对象，HostEnt 是继承自引用计数基类 RefCountObj，同时可被自动指针 Ptr 模板进行管理

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
  uint32_t ttl;
  int packet_size;
  char buf[MAX_DNS_PACKET_LEN];
  u_char *host_aliases[DNS_MAX_ALIASES];
  u_char *h_addr_ptrs[DNS_MAX_ADDRS + 1];
  // 用于存储 hostent 结构
  u_char hostbuf[DNS_HOSTBUF_SIZE];

  SRVHosts srv_hosts;

  virtual void free();

  HostEnt()
  {
    size_t base = sizeof(force_VFPT_to_top); // preserve VFPT
    memset(((char *)this) + base, 0, sizeof(*this) - base);
  }
};
```



# 参考资料

- [I_DNSProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/dns/I_DNSProcessor.h)
