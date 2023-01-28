# 基础组件：HostDBMD5

每一个 MultiCache 的 Entry 都有唯一的 tag 值，用来声明 Entry 内数据对象的唯一性。在 HostDB 中，将 HostDBInfo 对象作为 Entry 存储到 MultiCache 中，其 tag 值是通过哈希函数计算得来。

由于 HostDBInfo 存在多种查询类型，哈希函数的输入也存在多种不同的类型和来源：

```
/** Host DB record mark.

    The records in the host DB are de facto segregated by roughly the
    DNS query type. We use an intermediate type to provide a little flexibility
    although the type is presumed to be a single byte.
    HostDB 中的数据单元也分为多种类别，并与 DNS 查询类型对应。
    这里使用一个字节来记录数据单元的类型，目前仅定义了 4 种类型，其余预留作为将来的扩展。
 */
enum HostDBMark {
  // 表示 HostDB 用于 域名 正向地址解析，或非以下三种类型
  HOSTDB_MARK_GENERIC, ///< Anything that's not one of the other types.
  // 表示 HostDB 用于 IPv4 反向地址解析
  HOSTDB_MARK_IPV4,    ///< IPv4 / T_A
  // 表示 HostDB 用于 IPv6 反向地址解析
  HOSTDB_MARK_IPV6,    ///< IPv6 / T_AAAA
  // 表示 HostDB 用于 SRV 类型解析
  HOSTDB_MARK_SRV,     ///< Service / T_SRV
};
```

因此这里定义 HostDBMD5 类及方法，以支持不同类型的哈希值计算。

由于 HostDBInfo 中存在一个 HostDBApplicationInfo 类型的成员，其内保存了 `fail_count` 和 `last_failure` 两个值，该值用于表示一个解析结果所提供的服务出现的连续失败的次数以及最后一次失败的时间，详情可以查看 HttpSM 及 HttpTransaction 对于这部分的处理逻辑。

由于同一个域名可以提供 HTTP 和 HTTPS 两种协议，其所对应的 TCP 端口分别为 80 和 443，因此对于不同的服务端口，其源服务器的状态也是需要分别记录的，因此在 HostDBMD5 中计算哈希值时引入了 port 作为数据源的一部分。

但是对于反向地址解析，则未将 port 引入到哈希值的计算过程，主要因为反向地址解析主要用于对源站域名的验证，不存在与服务状态的关系。

## 定义

```
/** Container for an MD5 hash and its dependent data.
    This handles both the host name and raw address cases.
*/
struct HostDBMD5 {
  typedef HostDBMD5 self; ///< Self reference type.

  // 用于存储和计算哈希值，这里使用 MD5 算法
  INK_MD5 hash; ///< The hash value.

  // 用于保存计算哈希值的原始内容，这里分为主机名和 IP 两种类型，分别对应正向解析和反向解析：
  // 1. 用于正向解析：通过 域名 解析 IP 地址
  // 域名类型：host_name 指向域名，host_len 用于描述字符串长度（不包含结尾的 \0）
  char const *host_name; ///< Host name.
  int host_len;          ///< Length of @a _host_name
  // 2. 用于反向解析：通过 IP 地址 解析 域名
  // IP 类型：ip 用于保存 IPv4 或 IPv6 的地址
  IpAddr ip;             ///< IP address.
  // port 用于保存服务端口（对于 SRV 类型，其值总为 0）。
  // 备注：port 在 getbynameport_re，getbyaddr_re 和 getbyname_imm，还有 setby 中接受来自外部传入的值。
  in_port_t port;        ///< IP port (host order).
  // 根据我对代码的理解：
  //  - 对于正向地址解析类型，可以附带 port 值，其它类型的 port 值是无意义的，
  //  - 如果 port 值无意义（例如 SRV 类型）需要将 port 设为 0。
  /// DNS server. Not strictly part of the MD5 data but
  /// it's both used by @c HostDBContinuation and provides access to
  /// MD5 data. It's just handier to store it here for both uses.
  // 指向 DNS Server 对象，非强制性内容
  DNSServer *dns_server;
  // 用来支持 SplitDNS 功能
  SplitDNS *pSD;      ///< Hold the container for @a dns_server.
  // 用于说明计算哈希值时，数据源的类型
  //  - 正向域名解析：一般类型
  //  - 反向地址解析：IPv4 地址类型
  //  - 反向地址解析：IPv6 地址类型
  //  - SRV 类型
  HostDBMark db_mark; ///< Mark / type of record.

  /// Default constructor.
  HostDBMD5();
  /// Destructor.
  ~HostDBMD5();
  /// Recompute and update the MD5 hash.
  void refresh();
  /** Assign a hostname.
      This updates the split DNS data as well.
  */
  self &set_host(char const *name, int len);
};
```

## 方法

### 构造函数 HostDBMD5::HostDBMD5()

用于初始化 HostDBMD5 对象。

```
HostDBMD5::HostDBMD5() : host_name(0), host_len(0), port(0), dns_server(0), pSD(0), db_mark(HOSTDB_MARK_GENERIC)
{
}
```

### 析构函数 HostDBMD5::~HostDBMD5()

用于释放对 SplitDNSConfig 对象的引用，引用为 0 时自动释放，详情可参考 SplitDNS 部分。

```
HostDBMD5::~HostDBMD5()
{
  if (pSD)
    SplitDNSConfig::release(pSD);
}
```

### HostDBMD5::refresh()

计算 HostDB 对象的哈希值，这里没有对 db_mark 进行判断，而是做出以下设定：

- 如果 host_name 非空，就是 域名 正向地址解析或 SRV 类型的解析，那么 `db_mark` 的值就是 `HOSTDB_MARK_GENERIC` 或 `HOSTDB_MARK_SRV`
- 否则就一定是 IP 地址反向解析，那么 `db_mark` 的值就是 `HOSTDB_MARK_IPV4` 或 `HOSTDB_MARK_IPV6`
 
```
void
HostDBMD5::refresh()
{
  MD5Context ctx;

  if (host_name) {
    // 对于 域名 正向地址解析或 SRV 类型的解析，其哈希值计算的输入元素及顺序是：
    //   域名，端口，HostDB 类型，DNS 服务器（如有）
    char const *server_line = dns_server ? dns_server->x_dns_ip_line : 0;
    uint8_t m               = static_cast<uint8_t>(db_mark); // be sure of the type.

    ctx.update(host_name, host_len);
    ctx.update(reinterpret_cast<uint8_t *>(&port), sizeof(port));
    ctx.update(&m, sizeof(m));
    if (server_line)
      ctx.update(server_line, strlen(server_line));
  } else {
    // 对于 IP 地址 反向解析，其哈希值计算的输入元素及顺序是：
    //   0x00, 0x00, IPv4 或 IPv6（网络字节序）, 0x00, 0x00
    // 其在 IPv4 或 IPv6（网络字节序）的前后添加了两个 0x00，共 4 个字节
    // INK_MD5 the ip, pad on both sizes with 0's
    // so that it does not intersect the string space
    //
    char buff[TS_IP6_SIZE + 4];
    int n = ip.isIp6() ? sizeof(in6_addr) : sizeof(in_addr_t);
    memset(buff, 0, 2);
    memcpy(buff + 2, ip._addr._byte, n);
    memset(buff + 2 + n, 0, 2);
    ctx.update(buff, n + 4);
  }
  ctx.finalize(hash);
}
```

这里 MD5Context 的 `ctx.update()` 方法实际上调用了 OpenSSL 库的 `MD5_Update()`，`ctx.finalize()` 方法则是调用了 OpenSSL 库的 `MD5_Final()`。

- 其中 `ctx.update()` 用于将内容追加到数据区，
- 而 `ctx.finalize()` 则对数据区内存储的内容进行 MD5 计算，生成摘要。

### HostDBMD5::set_host(char const *name, int len)

设置 HostDBMD5 的域名，该域名会参与哈希值的计算。

```
HostDBMD5 &
HostDBMD5::set_host(char const *name, int len)
{
  host_name = name;
  host_len  = len;
#ifdef SPLIT_DNS
  if (host_name && SplitDNSConfig::isSplitDNSEnabled()) {
    const char *scan;
    // I think this is checking for a hostname that is just an address.
    for (scan = host_name; *scan != '\0' && (ParseRules::is_digit(*scan) || '.' == *scan || ':' == *scan); ++scan)
      ;
    if ('\0' != *scan) {
      // config is released in the destructor, because we must make sure values we
      // get out of it don't evaporate while @a this is still around.
      if (!pSD)
        pSD = SplitDNSConfig::acquire();
      if (pSD) {
        dns_server = static_cast<DNSServer *>(pSD->getDNSRecord(host_name));
      }
    } else {
      dns_server = 0;
    }
  }
#endif // SPLIT_DNS
  return *this;
}
```

注意：调用该方法后，哈希值不会自动更新，需要显示调用 `refresh()` 方法才会更新哈希值。

## 参考资料

- [HostDB.cc](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/HostDB.cc)
- [P_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/P_HostDBProcessor.h)
- OpenSSL MD5
   - [MD5_Update](https://www.openssl.org/docs/man1.0.2/man3/MD5_Update.html)
   - [MD5_Final](https://www.openssl.org/docs/man1.0.2/man3/MD5_Final.html)

