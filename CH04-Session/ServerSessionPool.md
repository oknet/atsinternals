# 核心组件：ServerSessionPool

## 定义

```
source: proxy/http/HttpProxyAPIEnums.h
/// Server session sharing values - match
typedef enum {
  TS_SERVER_SESSION_SHARING_MATCH_NONE,  // 不支持从会话池中获取会话
  TS_SERVER_SESSION_SHARING_MATCH_BOTH,  // 必须同时匹配 IP 和 主机名
  TS_SERVER_SESSION_SHARING_MATCH_IP,    // 只匹配 IP
  TS_SERVER_SESSION_SHARING_MATCH_HOST   // 只匹配 主机名
} TSServerSessionSharingMatchType;

/// Server session sharing values - pool
typedef enum {
  TS_SERVER_SESSION_SHARING_POOL_GLOBAL,  // 跨线程共享会话池，
  TS_SERVER_SESSION_SHARING_POOL_THREAD,  // 线程内共享会话池
} TSServerSessionSharingPoolType;
```

```
/** A pool of server sessions.

    This is a continuation so that it can get callbacks from the server sessions.
    This is used to track remote closes on the sessions so they can be cleaned up.
    继承自 Continuation，这样就可以被 ServerSession 回调。
    ServerSessionPool 用来跟踪源服务器Session的关闭，这样可以及时清理占用的内存。
    
    @internal Cleanup is the real reason we will always need an IP address mapping for the
    sessions. The I/O callback will have only the NetVC and thence the remote IP address for the
    closed session and we need to be able find it based on that.
    由于 do_io 操作的回调，只会传递 NetVConnection，我们只能够从其中获取到源服务器的IP。
    因此，我们需要建立源服务器IP地址与会话的映射关系，这样才能完成清理的工作。
*/
class ServerSessionPool : public Continuation
{
public:
  /// Default constructor.
  /// Constructs an empty pool.
  // 构造函数，用来创建空的会话池
  ServerSessionPool();
  /// Handle events from server sessions.
  // 处理来自会话的回调，通常为异常数据、超时、连接关闭
  int eventHandler(int event, void *data);

protected:
  /// Interface class for IP map.
  // IP地址与会话的映射表的方法
  struct IPHashing {
    typedef uint32_t ID;
    typedef sockaddr const *Key;
    typedef HttpServerSession Value;
    typedef DList(HttpServerSession, ip_hash_link) ListHead;

    static ID
    hash(Key key)
    {
      return ats_ip_hash(key);
    }
    static Key
    key(Value const *value)
    {
      return &value->server_ip.sa;
    }
    static bool
    equal(Key lhs, Key rhs)
    {
      return ats_ip_addr_port_eq(lhs, rhs);
    }
  };

  /// Interface class for FQDN map.
  // 主机名与会话的映射表的方法
  struct HostHashing {
    typedef uint64_t ID;
    typedef INK_MD5 const &Key;
    typedef HttpServerSession Value;
    typedef DList(HttpServerSession, host_hash_link) ListHead;

    static ID
    hash(Key key)
    {
      return key.fold();
    }
    static Key
    key(Value const *value)
    {
      return value->hostname_hash;
    }
    static bool
    equal(Key lhs, Key rhs)
    {
      return lhs == rhs;
    }
  };

  // 通过模版组合 TSHashTable 与对应的方法
  typedef TSHashTable<IPHashing> IPHashTable;     ///< Sessions by IP address.
  typedef TSHashTable<HostHashing> HostHashTable; ///< Sessions by host name.

public:
  /** Check if a session matches address and host name.
      判定一个会话是否匹配 IP地址 和 主机名
   */
  static bool match(HttpServerSession *ss, sockaddr const *addr, INK_MD5 const &host_hash,
                    TSServerSessionSharingMatchType match_style);

  /** Get a session from the pool.
      从会话池中获得一个会话

      The session is selected based on @a match_style equivalently to @a match. If found the session
      is removed from the pool.
      根据 match_style 指定的匹配方式，调用match方法，从会话池选中适合的会话。
      会话一旦被选中，则从会话池中移除。

      @return A pointer to the session or @c NULL if not matching session was found.
      server_session 指向选中会话的指针，或者指向 NULL 表示没有找到适合的会话。
      这是的函数返回值为 HSMresult_t 类型，但是并未有代码对其进行判断，
      !!! 在当前的代码中，总是返回 HSM_NOT_FOUND !!!
              
  */
  HSMresult_t acquireSession(sockaddr const *addr, INK_MD5 const &host_hash, TSServerSessionSharingMatchType match_style,
                             HttpServerSession *&server_session);
  /** Release a session to to pool.
      将一个会话放回到会话池
   */
  void releaseSession(HttpServerSession *ss);

  /// Close all sessions and then clear the table.
  // 关闭所有的会话，并且清空 IP映射表 和 主机名映射表
  void purge();

  // Pools of server sessions.
  // Note that each server session is stored in both pools.
  // IP映射表（池）
  IPHashTable m_ip_pool;
  // 主机名映射表（池）
  HostHashTable m_host_pool;
  // 注意，每一个会话都会存在于两个表中
};
```

## 方法

### ServerSessionPool::ServerSessionPool

```
ServerSessionPool::ServerSessionPool() : Continuation(new_ProxyMutex()), m_ip_pool(1023), m_host_pool(1023)
{
  SET_HANDLER(&ServerSessionPool::eventHandler);
  m_ip_pool.setExpansionPolicy(IPHashTable::MANUAL);
  m_host_pool.setExpansionPolicy(HostHashTable::MANUAL);
}
```

### initialize_thread_for_http_sessions

```
source: proxy/http/HttpSessionManager.cc
// Initialize a thread to handle HTTP session management
void
initialize_thread_for_http_sessions(EThread *thread, int /* thread_index ATS_UNUSED */)
{
  thread->server_session_pool = new ServerSessionPool;
}
```

在 UnixNetProcessor::start() 中会调用 initialize_thread_for_http_sessions 为每一个线程创建 ServerSessionPool 对象。

```
int
UnixNetProcessor::start(int, size_t)
{
  EventType etype = ET_NET;

  netHandler_offset = eventProcessor.allocate(sizeof(NetHandler));
  pollCont_offset = eventProcessor.allocate(sizeof(PollCont));

  // etype is ET_NET for netProcessor
  // and      ET_SSL for sslNetProcessor
  upgradeEtype(etype);

  n_netthreads = eventProcessor.n_threads_for_type[etype];
  netthreads = eventProcessor.eventthread[etype];
  for (int i = 0; i < n_netthreads; ++i) {
    initialize_thread_for_net(netthreads[i]);                                                                                        
    extern void initialize_thread_for_http_sessions(EThread * thread, int thread_index);
    initialize_thread_for_http_sessions(netthreads[i], i); 
  }
...
}
```

此处实际上可以理解为使用 netProcessor 干了 httpProcessor 的事情，但是 ATS 里并未定义 httpProcessor。


### ServerSessionPool::match

```
bool
ServerSessionPool::match(HttpServerSession *ss, sockaddr const *addr, INK_MD5 const &hostname_hash,
                         TSServerSessionSharingMatchType match_style)
{
  return TS_SERVER_SESSION_SHARING_MATCH_NONE != match_style && // if no matching allowed, fail immediately.
         // The hostname matches if we're not checking it or it (and the port!) is a match.
         (TS_SERVER_SESSION_SHARING_MATCH_IP == match_style ||
          (ats_ip_port_cast(addr) == ats_ip_port_cast(ss->server_ip) && ss->hostname_hash == hostname_hash)) &&
         // The IP address matches if we're not checking it or it is a match.
         (TS_SERVER_SESSION_SHARING_MATCH_HOST == match_style || ats_ip_addr_port_eq(ss->server_ip, addr));
}
```

使用 match_style 表示匹配规则，将上面的内容转换为 if-else 的句法：

```
  if (TS_SERVER_SESSION_SHARING_MATCH_NONE == match_style) {
    // 如果什么都不匹配，直接返回 false
    return false;
  }
  
  // 如果是IP的匹配模式，
  if (TS_SERVER_SESSION_SHARING_MATCH_IP == match_style ) {
    // 判断IP和PORT都相等，返回 true，否则返回flase
    if (ats_ip_addr_port_eq(ss->server_ip, addr)) {
      return true;
    } else {
      return false;
    }
  // 否则（不是IP的匹配模式，可能是：MATCH_HOST，MATCH_BOTH）
  } else if (ats_ip_port_cast(addr) == ats_ip_port_cast(ss->server_ip) && ss->hostname_hash == hostname_hash) {
    // 如果匹配的是端口号＋主机名的哈希值（已经满足MATCH_HOST的条件）
    // ats_ip_port_cast(addr) 返回的是端口号
    // The IP address matches if we're not checking it or it is a match.
    // 判断匹配条件为 MATCH_HOST，则直接返回 true
    if (TS_SERVER_SESSION_SHARING_MATCH_HOST == match_style) {
      return true;
    // 否则就是 MATCH_BOTH 的情况，需要继续判断 IP和PORT是否都相等
    } else if (ats_ip_addr_port_eq(ss->server_ip, addr)) {
      // 相等，满足 MATCH_BOTH 的条件，返回 true，否则返回 false
      return true;
    } else {
      return false;
    }
  // 既不是 IP 匹配模式，又不满足 MATCH_HOST 的条件
  // 所以，肯定无法匹配，直接返回 false
  } else {
    return false;
  }
```

### ServerSessionPool::acquireSession

### ServerSessionPool::releaseSession

### ServerSessionPool::eventHandler

### ServerSessionPool::purge

```
void
ServerSessionPool::purge()
{
  for (IPHashTable::iterator last = m_ip_pool.end(), spot = m_ip_pool.begin(); spot != last; ++spot) {
    spot->do_io_close();
  }
  m_ip_pool.clear();
  m_host_pool.clear();
}
```

# 核心组件：HttpSessionManager

## 定义

单实例，只有一个 HttpSessionManager 类声明的对象 httpSessionManager：

```
HttpSessionManager httpSessionManager;
```

## 方法

### HttpSessionManager::init

```
void
HttpSessionManager::init()                                                                                                                                                                                                            
{
  m_g_pool = new ServerSessionPool;
}
```


## 参考资料

- [HttpProxyAPIEnums.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyAPIEnums.h)
- [HttpSessionManager.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.h)
- [HttpSessionManager.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.cc)
