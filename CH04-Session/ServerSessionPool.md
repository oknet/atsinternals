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
// 用来确定一个 ServerSession 是否与指定的 Origin Server IP 及 主机名信息相匹配
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

```
// 传入Origin Server的IP信息和主机名信息，以及想要匹配的模式
// 通过 to_return 返回按照指定的匹配模式，从会话池中找到的 ServerSession
// 返回值：总是为 HSM_NOT_FOUND
HSMresult_t
ServerSessionPool::acquireSession(sockaddr const *addr, INK_MD5 const &hostname_hash, TSServerSessionSharingMatchType match_style,
                                  HttpServerSession *&to_return)
{
  HSMresult_t zret = HSM_NOT_FOUND;
  if (TS_SERVER_SESSION_SHARING_MATCH_HOST == match_style) {
    // MATCH_HOST 模式
    // This is broken out because only in this case do we check the host hash first.
    // 直接从 m_host_pool 中查找
    HostHashTable::Location loc = m_host_pool.find(hostname_hash);
    // 从IP数据结构中获取端口号
    in_port_t port = ats_ip_port_cast(addr);
    // 由于相同主机名可能会有多个端口，必须在 hash 桶里找到匹配端口的 ServerSession 会话
    while (loc && port != ats_ip_port_cast(loc->server_ip))
      ++loc; // scan for matching port.
    // 如果找到了
    if (loc) {
      // 返回找到的 ServerSession
      to_return = loc;
      // 同时从会话池中移除该ServerSession
      // 注意：这里是两个会话池
      m_host_pool.remove(loc);
      m_ip_pool.remove(m_ip_pool.find(loc));
    }
  } else if (TS_SERVER_SESSION_SHARING_MATCH_NONE != match_style) { // matching is not disabled.
    // 否则，判断 不是 MATCH_NONE 的模式，可能是：MATCH_IP，MATCH_BOTH
    // 直接从 m_ip_pool 中查找，这里的匹配是同时根据 IP 和 端口 进行的
    IPHashTable::Location loc = m_ip_pool.find(addr);
    // If we're matching on the IP address we're done, this one is good enough.
    // Otherwise we need to scan further matches to match the host name as well.
    // Note we don't have to check the port because it's checked as part of the IP address key.
    // 如果是 MATCH_BOTH 模式
    if (TS_SERVER_SESSION_SHARING_MATCH_IP != match_style) {
      // 继续判断 hostname_hash 值是否与找到的 ServerSession 匹配
      // 因为一个 IP 上可能会有多个域名
      while (loc && loc->hostname_hash != hostname_hash)
        ++loc;
    }
    // 如果找到了
    if (loc) {
      // 返回找到的 ServerSession
      to_return = loc;
      // 同时从会话池中移除该ServerSession
      // 注意：这里是两个会话池
      m_ip_pool.remove(loc);
      m_host_pool.remove(m_host_pool.find(loc));
    }
  }
  // 总是返回 HSM_NOT_FOUND
  return zret;
}
```

### ServerSessionPool::releaseSession

```
// 将 ServerSession 放入到会话池中
void
ServerSessionPool::releaseSession(HttpServerSession *ss)
{
  // 设置状态，表示该会话由会话池接管
  ss->state = HSS_KA_SHARED;
  // Now we need to issue a read on the connection to detect
  //  if it closes on us.  We will get called back in the
  //  continuation for this bucket, ensuring we have the lock
  //  to remove the connection from our lists
  // 通过 do_io_read 来接收 EOS 事件
  ss->do_io_read(this, INT64_MAX, ss->read_buffer);

  // Transfer control of the write side as well
  // 同样也要接管写操作
  ss->do_io_write(this, 0, NULL);

  // we probably don't need the active timeout set, but will leave it for now
  // 设置超时，这里可能不需要设置active timeout，但是暂时保留了
  ss->get_netvc()->set_inactivity_timeout(ss->get_netvc()->get_inactivity_timeout());
  ss->get_netvc()->set_active_timeout(ss->get_netvc()->get_active_timeout());
  // put it in the pools.
  // 同时放入两个会话池
  m_ip_pool.insert(ss);
  m_host_pool.insert(ss);

  // 输出放入成功的调试信息
  Debug("http_ss", "[%" PRId64 "] [release session] "
                   "session placed into shared pool",
        ss->con_id);
}
```

### ServerSessionPool::eventHandler

```
// 会话池的事件处理函数
int
ServerSessionPool::eventHandler(int event, void *data)
{
  NetVConnection *net_vc = NULL;
  HttpServerSession *s = NULL;

  switch (event) {
  case VC_EVENT_READ_READY:
  // The server sent us data.  This is unexpected so
  //   close the connection
  // 收到了来自 Origin Server 的数据，这是异常数据，应该关闭连接。
  //     此处没有立即关闭，是正常的，继续往下看。
  /* Fall through */
  case VC_EVENT_EOS:
  case VC_EVENT_ERROR:
  case VC_EVENT_INACTIVITY_TIMEOUT:
  case VC_EVENT_ACTIVE_TIMEOUT:
    // 获取 NetVConnection
    net_vc = static_cast<NetVConnection *>((static_cast<VIO *>(data))->vc_server);
    break;

  default:
    ink_release_assert(0);
    return 0;
  }

  sockaddr const *addr = net_vc->get_remote_addr();
  // 获取 http_config_params 对象
  HttpConfigParams *http_config_params = HttpConfig::acquire();
  bool found = false;

  // 由于 IP地址 会话池的哈希表是根据 IP＋端口 或者 ServerSession 来查找的，
  // 但是此处只有 NetVC 对象，所以只能根据 IP＋端口 来找，
  // 但是去往同一个 Origin Server 的连接可能会有多个，因此需要进行遍历，找到与此 NetVC 关联的 ServerSession
  for (ServerSessionPool::IPHashTable::Location lh = m_ip_pool.find(addr); lh; ++lh) {
    // 如果找到了与 NetVC 对应的 ServerSession
    if ((s = lh)->get_netvc() == net_vc) {
      // if there was a timeout of some kind on a keep alive connection, and
      // keeping the connection alive will not keep us above the # of max connections
      // to the origin and we are below the min number of keep alive connections to this
      // origin, then reset the timeouts on our end and do not close the connection
      // 如果是一个超时事件（出去超时事件，其它都是遇到了严重错误，需要关闭 NetVC）
      //   并且 ServerSession 当前是被会话池管理的状态
      //   并且该会话开启了源服务器连接限制
      if ((event == VC_EVENT_INACTIVITY_TIMEOUT || event == VC_EVENT_ACTIVE_TIMEOUT) && s->state == HSS_KA_SHARED &&
          s->enable_origin_connection_limiting) {
        // 判定当前服务器是否低于最小 keep alive 连接维持限定
        bool connection_count_below_min =
          s->connection_count->getCount(s->server_ip) <= http_config_params->origin_min_keep_alive_connections;

        // 如果低于最小 keep alive 连接维持限定
        //   就需要保持这个连接，即使超时也不会关闭它
        //   简单重设超时后，继续保持它在会话池中
        if (connection_count_below_min) {
          Debug("http_ss", "[%" PRId64 "] [session_bucket] session received io notice [%s], "
                           "reseting timeout to maintain minimum number of connections",
                s->con_id, HttpDebugNames::get_event_name(event));
          s->get_netvc()->set_inactivity_timeout(s->get_netvc()->get_inactivity_timeout());
          s->get_netvc()->set_active_timeout(s->get_netvc()->get_active_timeout());
          // 设置为已经找到，然后跳出 for 循环
          found = true;
          break;
        }
      }
      // 如果不是超时事件，就是遇到了严重错误或者对方关闭了连接
      // 或者，虽然是超时事件，但是 keep alive 连接已经足够了
      // 那么，就把这个 NetVConnection 从会话池中移除，然后关闭它
      // We've found our server session. Remove it from
      //   our lists and close it down
      Debug("http_ss", "[%" PRId64 "] [session_pool] session %p received io notice [%s]", s->con_id, s,
            HttpDebugNames::get_event_name(event));
      ink_assert(s->state == HSS_KA_SHARED);
      // Out of the pool! Now!
      m_ip_pool.remove(lh);
      m_host_pool.remove(m_host_pool.find(s));
      // Drop connection on this end.
      s->do_io_close();
      // 设置为已经找到，然后跳出 for 循环
      found = true;
      break;
    }
    // 如果 NetVC 与 ServerSession 不对应，则继续 for 循环遍历
  }

  // 释放 http_config_params 对象
  HttpConfig::release(http_config_params);
  if (!found) {
    // 如果没有找到 ServerSession，说明出现了连接泄漏的情况
    // We failed to find our session.  This can only be the result
    //  of a programming flaw
    Warning("Connection leak from http keep-alive system");
    ink_assert(0);
  }
  return 0;
}
```

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

## 参考资料

- [HttpProxyAPIEnums.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyAPIEnums.h)
- [HttpSessionManager.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.h)
- [HttpSessionManager.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.cc)
