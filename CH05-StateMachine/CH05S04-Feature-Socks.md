# 新特性 / 功能：Socks

下面将要介绍的是 Socks 功能，Socks 功能包含了：

- Socks Proxy
  - 它是ATS里设计最简单的一个代理功能，它利用到了 OneWayTunnel 和 TwoWayTunnel
  - 它使得ATS可以对Socks Client的请求进行解析
  - 我们可以通过对 Socks Proxy 代码的分析，学习如何使用 Tunnel 状态机。
- Socks Parent
  - 它允许ATS通过Socks代理服务器对源服务器发起访问
  - 它使得ATS可以作为Socks Client

## SOCKS_WITH_TS

首先，介绍一个宏定义 SOCKS_WITH_TS，它在 I_Socks.h 中进行了定义，在 Socks 代码中会有大量的地方判断这个宏是否定义，以应用不同的代码设计。
定义了这个宏之后，会有一些增强的新特性，因此，在后面进行代码分析的时候，我们就按照具有新特性的代码进行全面的分析。
```
source: I_Socks.h
/*When this is being compiled with TS, we enable more features the use
  non modularized stuff. namely:
  ip_ranges and multiple socks server support.
*/
#define SOCKS_WITH_TS
```
## socks_conf_struct

接着，介绍一个结构，该结构存储了 Socks 功能的全局配置项。

```
source: P_Socks.h
struct socks_conf_struct {
  // 下面这些配置项是用来服务Socks Parent功能的
  int socks_needed;                   // true: 所有ATS对源服务器发起的连接都需要通过Socks代理服务器
                                      //   该配置也是整个 Socks 功能的总开关
  int server_connect_timeout;         // 连接Socks代理服务器的超时时间
  int socks_timeout;                  // 与Socks代理服务器建立连接后的通信超时时间
  unsigned char default_version;      // 与Socks代理服务器进行通信时，默认采用的Socks协议版本
  char *user_name_n_passwd;           // 指向 用户名 和 密码 的指针：[len(short int)] + "用户名" + [len(short int)] + "密码"
  int user_name_n_passwd_len;         // 用户名和密码的字节数＋2 bytes

  int per_server_connection_attempts; // 当指定了多个Socks代理服务器时，对于每一个Socks代理服务器尝试重连的次数
  int connection_attempts;            // 总尝试次数

  // the following ports are used by SocksProxy
  // 下面这些配置项是用来服务Socks Proxy功能的
  int accept_enabled;       // true：开启对Socks Client请求的解析功能
                            //   要求 socks_needed 必须为 true，否则解析出来的 Socks 请求，没办法发送给Socks代理服务器
  int accept_port;          // 对来自指定端口的Socks Client的请求进行解析
                            //   在开启 accept_enabled 之后，必需设置一个服务端口用来接收 Socks 请求
  unsigned short http_port; // 如果Socks Client的请求为CONNECT，并且目标端口为指定端口，则将该请求转入 HttpSM 进行处理

  IpMap ip_map;             // 当 socks_needed == true 时，这个表里包含的 目标IP地址 将不通过 Socks代理服务器

  socks_conf_struct()                    // 配置文件缺省值为
    : socks_needed(0),                   // false(0)（必需项，true＝最小配置项）
      server_connect_timeout(0),         // 10 秒
      socks_timeout(100),                // 100 秒
      default_version(5),                // Version 4
      user_name_n_passwd(NULL),          // NULL (非必需项)
      user_name_n_passwd_len(0),         // 0 (非必需项)
      per_server_connection_attempts(1), // 1
      connection_attempts(0),            // 4
      accept_enabled(0),                 // false(0)（必需项，true＝最小配置项）
      accept_port(0),                    // 1080
      http_port(1080)                    // 80，这里是一个小bug，不过没关系，在配置文件加载时缺省值是正确的。
  {
  }
};
```

## 参考资料

- [I_Socks.h](http://github.com/apache/trafficserver/tree/master/iocore/net/I_Socks.h)
- [P_Socks.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_Socks.h)
- [mgmt/RecordsConfig.cc](http://github.com/apache/trafficserver/tree/master/mgmt/RecordsConfig.cc)
