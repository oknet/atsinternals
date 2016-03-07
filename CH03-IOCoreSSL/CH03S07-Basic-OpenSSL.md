# 基础知识：OpenSSL

ATS 对 TLS/SSL 的底层支持是通过 OpenSSL 开发库实现的，因此我们要先对 OpenSSL 的开发库做一个简单的了解。

使用 OpenSSL 的步骤如下：

  - 通过 SSL_CTX_new 方法创建一个 SSL_CTX 对象
    - 该对象可以用来承载多种属性，如：证书，加解密的算法等
  - 等网络连接创建之后，可以通过 SSL_set_fd 或 SSL_set_bio 方法把 socket fd 与 SSL 对象关联起来
  - 使用 SSL_accept 或 SSL_connect 方法完成 SSL 握手过程
  - 然后使用 SSL_read 和 SSL_write 方法 接收 和 发送 数据
  - 最后，使用 SSL_shutdown 方法来关闭 TLS/SSL 连接

## BIO

OpenSSL 提出了一个 BIO 的概念，BIO与读/写对应，因此分为 Read BIO 和 Write BIO。

  - 可以通过 SSL_set_bio() 一次性同时设置 Read BIO 和 Write BIO
  - 或者通过 SSL_set_rbio() 单独设置 Read BIO
  - 或者通过 SSL_set_wbio() 单独设置 Write BIO

BIO 可以跟一个 内存块 关联：

  - BIO_new(BIO_s_mem())
  - BIO_new_mem_buf(ptr, len)

BIO 也可以跟一个 socket fd 关联：

  - BIO_new_fd(int fd, int close_flag)
  - BIO_set_fd(BIO *b, int fd, int close_flag)

或者直接在 SSL 对象上设置 socket fd：

  - SSL_set_fd
    - 同时设置读/写双方向
  - SSL_set_rfd
    - 只设置用于读取数据的 socket fd 到 SSL 对象
  - SSL_set_wfd
    - 只设置用于写入数据的 socket fd 到 SSL 对象

在向 SSL 对象上设置 socket fd 时，实际上是：

  - 创建了一个与 socket fd 关联的 BIO，
  - 然后再把 BIO 关联到 SSL 对象。
  - 如果 SSL 对向已经与 BIO 关联，那么就先通过 BIO_free() 将其释放掉。

在实际应用中，可以随时让 BIO 在 内存块 与 socket fd 之间进行切换。

OpenSSL 的其它库函数对于资源的读写都是通过 BIO 抽象层来完成的，例如：

  - SSL_accept 和 SSL_connect
  - SSL_read 和 SSL_write

我们会发现 BIO 与 VIO 非常相似但是又不一样：

  - BIO抽象了对内存块和socket fd的访问，是一个资源描述符
  - VIO则是连接了socket fd与内存块，同时设置了数据流的方向，是一个数据管道

## SSL 握手过程

作为Server端时，需要通过调用 SSL_accept 实现 SSL 握手过程，在调用 SSL_accept 之前，可能需要进行以下的设置：

  - 关联证书信息到 SSL 对象
  - 设置 SNI/Cert Callback 函数 
  - 设置 Session Ticket Callback 函数

作为Client端时，需要通过调用 SSL_connect 实现 SSL 握手过程，在调用 SSL_connect 之前，可能需要进行以下的设置：

  - 关联证书信息到 SSL 对象
  - 设置 SNI
  - 设置 NPN/ALPN 参数

## SSL 数据传输过程

在完成了 SSL 握手过程之后，就可以进行数据传输了：

  - SSL_read
    - 从 Read BIO 读取信息，解密后写入到指定的内存地址
  - SSL_write
    - 从指定内存地址获取信息，写入到 Write BIO，加密后发送

## SSL 错误信息

OpenSSL API 调用后会返回一些错误信息，对常见的错误信息解释如下：

  - SSL_ERROR_WANT_READ
    - 表示在调用 read() 时遇到了 EAGAIN
    - 需要在socket fd可读的时候，重新调用
  - SSL_ERROR_WANT_WRITE
    - 表示在调用 write() 时遇到了 EAGAIN
    - 需要在socket fd可写的时候，重新调用
  - SSL_ERROR_WANT_ACCEPT
    - 表示在调用 SSL_accept() 时遇到了阻塞的情况
    - 需要在socket fd可读的时候，重新调用
  - SSL_ERROR_WANT_CONNECT
    - 表示在调用 SSL_connect() 时遇到了阻塞的情况
    - 需要在socket fd可写的时候，重新调用
  - SSL_ERROR_WANT_X509_LOOKUP
    - 表示 SNI/CERT Callback 要求挂起 SSL_accept 过程
    - 并且需要再次调用 SSL_accept() 来触发对 SNI/CERT Callback 的调用
    - 直到 SNI/CERT Callback 允许 SSL_accept 过程继续
    - 根据业务需要，在适当的时候重新调用 SSL_accept

上面的错误状态都是遇到了阻塞的情况，只需要在适当的时候重新调用之前调用过的API就可以了。

  - SSL_ERROR_SYSCALL
    - 表示在执行系统调用时遇到了特殊的错误
    - 可以通过 errno 值来判断系统调用的错误类型
  - SSL_ERROR_SSL
    - OpenSSL 库函数出错，通常表示协议层出现了错误
  - SSL_ERROR_ZERO_RETURN
    - 表示对端关闭了SSL连接
    - 但是需要注意，这并不意味着TCP连接也一定关闭了

上面这些是真正的错误，需要相应的错误处理，例如：关闭连接，或者重试？

  - SSL_ERROR_NONE
    - 表示操作正常完成，没有错误 

只有这个，才是没有任何错误的完成了一个方法的调用。

# 参考资料

- [OpenSSL Online Docs](https://www.openssl.org/docs/manmaster/ssl/SSL.html)
