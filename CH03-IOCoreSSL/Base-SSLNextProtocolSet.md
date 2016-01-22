# 基础部件：SSLNextProtocolSet

为了实现 NPN 与 ALPN，ATS定义了SSLNextProtocolSet来支持对多种协议的处理。

SSLNextProtocolSet 主要用来生成支持的协议列表

  - 通过register和unregister方法添加和删除支持的协议
  - 每次添加、删除（bug？）之后都会重新按照 NPN 和／或 ALPN 的要求生成支持的的协议列表 npn，长度为 npnsz
  - 通过 advertise 来获取生成好的协议列表，在 NPN 和／或 ALPN 交互时使用
  - 通过 find 来寻找与特定协议对应的状态机

## 定义

```
class SSLNextProtocolSet
{
public:
  // 构造函数
  // 初始化 npn=NULL，npnsz=0 
  SSLNextProtocolSet();
  // 析构函数
  // 释放npn的内存，然后遍历 endpoints 链表，逐个delete链表上的元素
  ~SSLNextProtocolSet();

  // 注册一个状态机和支持的协议串
  // 不能对同一个协议重复注册
  // 注册一个新协议的时候，会new NextProtocolEndpoint并放入endpoints链表中
  // true=成功注册，遍历 endpoints 生成最新的 npn 和 npnsz
  // false=注册失败，协议已经注册 或 协议串超过255字节
  bool registerEndpoint(const char *, Continuation *);
  // 注销一个状态机和支持的协议串
  // 遍历 endpoints，比对协议字符串，找到要注销的就直接调用 endpoints 链表的remove方法删除
  // true=成功注销
  // false=注销失败，没有找到要注销的协议字符串
  // 这里存在一个内存泄漏：在从链表remove掉元素之后，没有通过delete方法释放该元素占用的内存
  // 这里还有一个bug：没有刷新 npn 和 npnsz
  // 不过，纵观整个ATS的代码，好像没有任何组件调用过这个注销方法～
  bool unregisterEndpoint(const char *, Continuation *);
  // 获取当前的 npn 和 npnsz，分别存入 *out 和 *len
  // true=成功，false=当前没有npn和npnsz
  // 其后有const关键词，表示此函数不会修改类成员的值
  bool advertiseProtocols(const unsigned char **out, unsigned *len) const;
  // 遍历 endpoints 链表，逐个比对协议名称
  // 返回该链表中完全匹配的那个NextProtocolEndpoint类型的对象内的 endpoint 成员
  // 返回NULL表示链表中没有匹配的NextProtocolEndpoint类型的对象
  // 其后有const关键词，表示此函数不会修改类成员的值
  Continuation *findEndpoint(const unsigned char *, unsigned) const;

  struct NextProtocolEndpoint {
    // NOTE: the protocol and endpoint are NOT copied. The caller is
    // responsible for ensuring their lifetime.
    // 构造函数
    // 成员protocol指向传入的参数protocol
    // 成员endpoint指向传入的参数endpoint
    // 由于没有进行copy操作，所以调用者要确保传入的参数指向的对象不被释放
    NextProtocolEndpoint(const char *protocol, Continuation *endpoint);
    // 析构函数
    // 没有任何操作的空函数，因为不需要回收资源等
    ~NextProtocolEndpoint();

    const char *protocol;
    Continuation *endpoint;
    LINK(NextProtocolEndpoint, link);

    typedef DLL<NextProtocolEndpoint> list_type;
  };

private:
  SSLNextProtocolSet(const SSLNextProtocolSet &);            // disabled
  SSLNextProtocolSet &operator=(const SSLNextProtocolSet &); // disabled

  // mutable 关键词好像是为了对应advertiseProtocols和findEndpoint两个方法
  // 但是感觉没有什么必要？
  mutable unsigned char *npn;
  mutable size_t npnsz;

  // 定义一个双向链表
  // 这里为啥不用 DLL<NextProtocolEndpoint> endpoints; ？非要用个typedef ？
  NextProtocolEndpoint::list_type endpoints;
};
```

## 参考资料

- [P_SSLNextProtocolSet.h](http://github.com/apache/trafficserver/tree/master/iocore/net/P_SSLNextProtocolSet.h)
- [SSLNextProtocolSet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/SSLNextProtocolSet.cc)
