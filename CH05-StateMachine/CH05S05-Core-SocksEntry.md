# 核心组件：SocksEntry

SocksEntry 是ATS作为Socks Client向Socks代理服务器发起Socks请求的状态机。

## 定义

```
// 与SSL协议一样，Socks协议可以理解为一种传输层的协议
//   区别是Socks只有握手过程，但是没有通信加解密功能
// 因此，这里也像SSLNetVConnection一样定义了SocksNetVC
// 不过由于Socks协议不需要提供通信加解密功能，
// 所以，它可以与UnixNetVConnection共享net_read_io和write_to_net_io
typedef UnixNetVConnection SocksNetVC;

struct SocksEntry : public Continuation {
  // 用来保存发送给 Socks代理服务器 的Socks请求
  // 同时用来接收自 Socks代理服务器 返回的Socks响应
  MIOBuffer *buf;
  IOBufferReader *reader;

  // 已经通过connect_req发起了到Socks代理服务器连接的 NetVC
  SocksNetVC *netVConnection;

  // Changed from @a ip and @a port.
  // 保存目标的IP地址和端口，将通过Socks代理服务器来访问该目标
  IpEndpoint target_addr; ///< Original target address.
  // Changed from @a server_ip, @a server_port.
  // 保存Socks代理服务器的IP地址和端口（下面的英文注释是错误的）
  IpEndpoint server_addr; ///< Origin server address.

  // 用来保存当前尝试与 Socks代理服务器 连接的次数
  int nattempts;

  // 在Socks握手完成后，会回调该Action连接的状态机
  Action action_;
  // 在与 Socks代理服务器 建立连接的所有尝试失败后，将向上述状态回调 NET_EVENT_OPEN_FAILED，并传递 -lerrno 值
  int lerrno;
  // 通过 schedule_*() 方法设置超时时间，保存其返回值
  // 如遇到 Socks 超时，SocksEntry 会收到 EVENT_INTERVAL 事件
  //   注意：如遇到 NetVConnection 超时，仍然会收到 EVENT_*_TIMEOUT 事件
  Event *timeout;
  // 与 Socks代理服务器 通信时采用的协议版本
  unsigned char version;

  // 表示ATS与Socks代理服务器正处于握手过程中，
  // true: ATS已经向Socks代理服务器发出了请求
  bool write_done;

  // 用于实现Socks5认证的函数指针，Socks4协议时为NULL
  SocksAuthHandler auth_handler;
  // 用于表示发起的Socks请求中的Socks命令字
  // 如果其值为 NORMAL_SOCKS，则表示 CONNECT 命令，也是SocksEntry能够支持的
  // 如果其值不为 NORMAL_SOCKS，ATS 则会在发送了Socks请求之后与目标服务器建立 Blind Tunnel
  unsigned char socks_cmd;

  // socks server selection:
  // 如果有多个 Socks代理服务器可以使用时，将逐个尝试
  ParentConfigParams *server_params;
  HttpRequestData req_data; // We dont use any http specific fields.
  ParentResult server_result;

  // 初始化回调函数
  int startEvent(int event, void *data);
  // 主处理回调函数
  int mainEvent(int event, void *data);
  // 当存在多个可用的 Socks代理服务器 时，通过findServer选择下一个
  void findServer();
  // 初始化 SocksEntry 实例
  void init(ProxyMutex *m, SocksNetVC *netvc, unsigned char socks_support, unsigned char ver);
  // 回调上层状态机 连接成功 或 失败，然后释放SocksEntry
  void free();

  // 构造函数
  SocksEntry()
    : Continuation(NULL),
      netVConnection(0),
      nattempts(0),
      lerrno(0),
      timeout(0),
      version(5),
      write_done(false),
      auth_handler(NULL),
      socks_cmd(NORMAL_SOCKS)
  {
    memset(&target_addr, 0, sizeof(target_addr));
    memset(&server_addr, 0, sizeof(server_addr));
  }
};

typedef int (SocksEntry::*SocksEntryHandler)(int, void *);

extern ClassAllocator<SocksEntry> socksAllocator;
```

## 参考资料

- [P_Socks.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_Socks.h)
- [Socks.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/Socks.cc)
