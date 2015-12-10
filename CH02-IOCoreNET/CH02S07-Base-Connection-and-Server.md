# 基础组件：Connection

用于定义一个网络连接和 Client 端。

# 定义

```
struct Connection {
  SOCKET fd;         ///< 该连接的 fd 描述符，其实是 int 类型
  IpEndpoint addr;   ///< 关联的地址信息
  bool is_bound;     ///< True＝已经与本地IP地址进行了bind操作，通常指调用了Connection::open()方法，而不是listen时的bind()操作
  bool is_connected; ///< True＝已经建立了链接，通常指调用了Connection::connect()方法，对于非阻塞IO，连接可能还在建立中
  int sock_type;     ///< 如果是UDP连接：SOCK_DGRAM，如果是TCP链接：SOCK_STREAM

  // 创建并初始化一个 socket 句柄/描述字
  //     可以传入一个NetVCOptions类型的参数，来设置此 socket 的一些特性，
  //     传递给 connect() 方法时要传递 '同一个' NetVCOptions类型的参数
  //     ALL：TPROXY，sendbuf，revcbuf，call apply_options(opt)
  // 返回 0 表示成功，
  // 小于 0 表示失败，其值为 -ERRNO
  int open(NetVCOptions const &opt = DEFAULT_OPTIONS ///< Socket options.
           );

  // 使用 socket 句柄/描述字，向指定的 IP:PORT 发起连接
  //     通过sockaddr类型参数执行想要连接的目标IP和PORT
  //     通过NetVCOptions类型的参数，来设置此连接是 '阻塞' 还是 '非阻塞'
  //     传递给 open() 方法时要传递 '同一个' NetVCOptions类型的参数
  // 返回 0 表示成功，
  // 小于 0 表示失败，其值为 -ERRNO
  int connect(sockaddr const *to,                       ///< Remote address and port.
              NetVCOptions const &opt = DEFAULT_OPTIONS ///< Socket options
              );

  // 设置内部使用的 socket 地址结构成员
  // 仅用于 ICP
  void
  setRemote(sockaddr const *remote_addr ///< Address and port.
            )
  {
    ats_ip_copy(&addr, remote_addr);
  }

  // 设置 MultiCast 的 send
  int setup_mc_send(sockaddr const *mc_addr, sockaddr const *my_addr, bool non_blocking = NON_BLOCKING, unsigned char mc_ttl = 1,
                    bool mc_loopback = DISABLE_MC_LOOPBACK, Continuation *c = NULL);

  // 设置 MultiCast 的 receive
  int setup_mc_receive(sockaddr const *from, sockaddr const *my_addr, bool non_blocking = NON_BLOCKING, Connection *sendchan = NULL,
                       Continuation *c = NULL);

  // 关闭之前 open() 创建的 socket 句柄/描述字
  // 返回 0 表示成功，
  // 小于 0 表示失败，其值为 -ERRNO
  int close(); // 0 on success, -errno on failure

  // 设置特殊参数
  //     TCP：SOCK_OPT_NO_DELAY，SOCK_OPT_KEEP_ALIVE，SOCK_OPT_LINGER_ON
  //     ALL：SO_MARK，IP_TOS/IPV6_TCLASS
  void apply_options(NetVCOptions const &opt);

  // 析构函数
  // 调用close() 关闭 socket
  // 重置成员：fd=NO_FD，is_bound=false，is_connected=false
  virtual ~Connection();
  
  // 构造函数
  // 初始化成员：fd=NO_FD，is_bound=false，is_connected=false，sock_type=0，addr内存区域填充0
  Connection();

  // NetVCOptions类型的默认值（静态常量，只读）
  // 成员由构造函数调用NetVCOptions::reset()来完成默认值的初始化(P_UnixNetVConnection.h)
  static NetVCOptions const DEFAULT_OPTIONS;

protected:
  // 清理函数：与cleaner类模版配合使用
  // 调用close() 关闭 socket
  void _cleanup();
};
```

## 关于 UnixConnection.cc中的 cleaner模版

在 [UnixConnection.cc](https://github.com/apache/trafficserver/tree/master/iocore/net/UnixConnection.cc) 中定义了一个cleaner的模版类：

```
// 由于使用了 未命名命名空间 的定义，因此这个 cleaner模版 只能在 UnixConnection.cc 中使用
namespace
{
 /* 这是一个为了方便资源清理的结构
    通常 method 是 object 析构的时候调用的清理并释放资源的方法
    但是可以通过 cleaner::reset 方法来避免在析构时调用 method 方法。
    这种特性在 分配，检查，返回 的流程设计中可能没什么用处，
    但是对于以下情况却很有用处：
      - 有多个资源 （每一种都可以有自己的资源清理方法）
      - 有多个检查点
    此时，可以在 分配 的时候创建一个cleaner，就不用在所有需要的地方都检查资源的情况了
        如果创建成功就调用 reset 方法，这个分支通常在代码里只有一处
        否则就执行正常的析构函数，释放资源
    例子：
    self::some_method (...) {
      // 分配 资源
      cleaner<self> clean_up(this, &self::cleanup);
      // 修改 或 检查 资源
      if (fail)
        return FAILURE; // 失败，自动调用 cleanup()，资源被释放
      // 成功
      clean_up.reset(); // 调用reset()方法阻止析构，cleanup() 就不会被调用了
      return SUCCESS;
  */
template <typename T> struct cleaner {
  T *obj;                      ///< Object instance.
  typedef void (T::*method)(); ///< Method signature.
  method m;

  cleaner(T *_obj, method _method) : obj(_obj), m(_method) {}
  ~cleaner()
  {
    if (obj)
      (obj->*m)();
  }
  void
  reset()
  {
    obj = 0;
  }
};
}
```


# 基础组件：Server

继承自 Connection 基类，用于定义一个 Server 端。

# 定义

```
struct Server : public Connection {
  // 准备接受客户端连接的本地IP地址
  IpEndpoint accept_addr;

  // True = 进来的连接是透明的，对应ATS的tr-in
  bool f_inbound_transparent;

  // True = 使用kernel的HTTP accept filter
  // 这是BSD系统上的一个高级过滤器，Linux只有DEFER_ACCEPT与其类似，所以在Linux系统上，此值通常为false
  // 其功能大概是（我没用过，不是100%确定）
  //     1. 在kernel内完成TCP握手，
  //     2. 然后读取第一个请求包，
  //     3. 是HTTP请求，从accept()调用返回
  //     4. 不是HTTP请求，关闭TCP连接，等待下一个连接，或者accept()返回错误
  bool http_accept_filter;

  //
  // Use this call for the main proxy accept
  //
  // 这个没有定义，在ATS的代码中也没有使用
  int proxy_listen(bool non_blocking = false);

  // 接受一个新连接
  //     使用 this->fd 这个 之前创建好的 listen fd 执行accept()
  //     新连接的描述符存入 c->fd
  //     为新链接设置FD_CLOEXEC，nonblocking，SEND_BUF_SIZE
  // 返回 0 表示成功
  // 小于 0 表示失败，其值为 -ERRNO
  int accept(Connection *c);

  // 创建一个可用于 accept() 的listen fd
  //     使用 accept_addr 填充 addr 成员
  //     创建 TCP socket，SOCK_STREAM，设置为non blocking
  //     通过 bind() 绑定到IP和PORT
  // 接下来就可以用 setup_fd_for_listen() 设置一些必要的参数，然后调用 accept() 来接受新连接了
  int listen(bool non_blocking = false, int recv_bufsize = 0, int send_bufsize = 0, bool transparent = false);
  
  // 设置成员 fd 的参数
  //     send buf size, recv buf size, FD_CLOEXEC, SO_LINGER, IPV6_V6ONLY for ipv6
  //     SO_REUSEADDR, TCP_NODELAY, SO_KEEPALIVE, SO_TPROXY, TCP_MAXSEG, non blocking
  // 为调用 accept() 方法做准备
  // 之前要首先调用 listen() 创建一个 socket
  int setup_fd_for_listen(bool non_blocking = false, int recv_bufsize = 0, int send_bufsize = 0,
                          bool transparent = false ///< Inbound transparent.
                          );

  // 构造函数
  // 通过 Connection() 初始化成员：fd=NO_FD，is_bound=false，is_connected=false，sock_type=0，addr内存区域填充0
  // 初始化成员 f_inbound_transparent=false，accept_addr内存区域填充0
  Server() : Connection(), f_inbound_transparent(false) { ink_zero(accept_addr); }
};
```

# 参考资料

- [P_Connection.h]
(https://github.com/apache/trafficserver/tree/master/iocore/net/P_Connection.h)
- [Connection.cc]
(https://github.com/apache/trafficserver/tree/master/iocore/net/Connection.cc)
- [UnixConnection.cc]
(https://github.com/apache/trafficserver/tree/master/iocore/net/UnixConnection.cc)
