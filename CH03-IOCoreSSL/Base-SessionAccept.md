# 基础组件：SessionAccept

SessionAccept 是一个基类，提供 accept方法 和 mainEvent回调函数。

当在一个TCP连接中需要进行多个事务时，ATS抽象了这个 SessionAccept 基类。

这里多个事务可以是：

  - 不同类型，例如SSL支持
    - 首先是SSL握手过程
    - 然后是NPN/ALPN选择的协议对应的状态机
  - 相同类型，例如HTTP的Keep alive支持
    - 首先是一个HTTP请求
    - 结束上一个HTTP请求，再开始一个HTTP请求

## 定义

```
class SessionAccept : public Continuation
{
public:
  SessionAccept(ProxyMutex *amutex) : Continuation(amutex) { SET_HANDLER(&SessionAccept::mainEvent); }

  ~SessionAccept() {}

  virtual void accept(NetVConnection *, MIOBuffer *, IOBufferReader *) = 0;

private:
  virtual int mainEvent(int event, void *netvc) = 0;
};
```

## 参考资料

- [I_SessionAccept.h](http://github.com/apache/trafficserver/tree/master/iocore/net/I_SessionAccept.h)
