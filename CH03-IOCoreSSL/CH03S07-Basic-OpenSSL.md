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

OpenSSL 的其它库函数对于资源的读写都是通过 BIO 抽象层来完成的，例如：

  - SSL_accept 和 SSL_connect
  - SSL_read 和 SSL_write

我们会发现 BIO 与 VIO 非常相似但是又不一样：

  - BIO抽象了对内存块和socket fd的访问，是一个资源描述符
  - VIO则是连接了socket fd与内存块，同时设置了数据流的方向，是一个数据管道

# 参考资料

- [OpenSSL Online Docs](https://www.openssl.org/docs/manmaster/ssl/SSL.html)
