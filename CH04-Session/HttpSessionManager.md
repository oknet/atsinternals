# 核心组件：HttpSessionManager

单实例，只有一个 HttpSessionManager 类声明的对象 httpSessionManager：

httpSessionManager 是 ServerSessionPool 的管理者，实际上它跟 Processor 有相似之处。

由于 ServerSessionPool 在 ATS 中被设计为两层：

  - 一层是在线程中的，每个线程里有一个本地的 ServerSessionPool
  - 而在 httpSessionManager 中还包含了一个全局的 ServerSessionPool

```
HttpSessionManager httpSessionManager;
```

但是需要注意的是，其实 ServerSessionPool 一共有三层设计：

  - 全局 ServerSessionPool
  - 线程 ServerSessionPool
  - ClientSession 的 Slave ServerSession

下面在 acquire_session 方法中会看到对上述三层设计的实现。

## 定义

```
class HttpSessionManager
{
public:
  // 构造函数，初始化全局会话池为 NULL
  HttpSessionManager() : m_g_pool(NULL) {}

  ~HttpSessionManager() {}

  // 从会话池中选取一个ServerSession，然后将此ServerSession关联到指定的 HttpSM
  HSMresult_t acquire_session(Continuation *cont, sockaddr const *addr, const char *hostname, HttpClientSession *ua_session,
                              HttpSM *sm);
  // 将指定的 ServerSession 归还到会话池
  HSMresult_t release_session(HttpServerSession *to_release);
  // 关闭并清除全局会话池中所有的会话
  void purge_keepalives();
  // 为全局会话池创建对象
  void init();
  // 未定义，可能是没有实现完整
  int main_handler(int event, void *data);

private:
  /// Global pool, used if not per thread pools.
  /// @internal We delay creating this because the session manager is created during global statics init.
  // 全局会话池
  //     由于SessionManager创建时，系统正在初始化全局统计系统
  //     因此，我们不在构造函数中初始化全局会话池，而是在全局统计系统初始化完成后再调用 init 方法初始化全局会话池
  ServerSessionPool *m_g_pool;
};
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

### HttpSessionManager::purge_keepalives

```
// TODO: Should this really purge all keep-alive sessions?
// Does this make any sense, since we always do the global pool and not the per thread?
// 从注释上来看，这里仍然有一些疑问，因为这个方法只关闭和释放全局会话池中的会话
// 这个方法只在 HttpSM::do_http_server_open 中调用，
//     当判断超出了最大的服务器连接（server_max_connections）时调用此方法。
void
HttpSessionManager::purge_keepalives()
{
  EThread *ethread = this_ethread();

  MUTEX_TRY_LOCK(lock, m_g_pool->mutex, ethread);
  if (lock.is_locked()) {
    m_g_pool->purge();
  } // should we do something clever if we don't get the lock?
}
```

### HttpSessionManager::acquire_session

```
HSMresult_t
HttpSessionManager::acquire_session(Continuation * /* cont ATS_UNUSED */, sockaddr const *ip, const char *hostname,
                                    HttpClientSession *ua_session, HttpSM *sm)
{
  // 找到匹配的 ServerSession 后会保存到 to_return 变量
  HttpServerSession *to_return = NULL;
  // 从 HttpSM 获得匹配方式（MATCH_NONE, MATCH_IP, MATCH_HOST, MATCH_BOTH）
  TSServerSessionSharingMatchType match_style =
    static_cast<TSServerSessionSharingMatchType>(sm->t_state.txn_conf->server_session_sharing_match);
  // 保存传入的 hostname 的哈希值
  INK_MD5 hostname_hash;
  // 指示是否找到匹配的 ServerSession，作为函数返回值
  HSMresult_t retval = HSM_NOT_FOUND;

  // 计算传入的 hostname 的哈希值
  ink_code_md5((unsigned char *)hostname, strlen(hostname), (unsigned char *)&hostname_hash);

  // First check to see if there is a server session bound
  //   to the user agent session
  // 首先，查看 ClientSession 上是否已经有一个关联的ServerSession
  // 注意：这里并没有获取 ClientSession 锁的操作！！！
  //     因为，acquire_session 只在 HttpSM::do_http_server_open 中调用，
  //     而 HttpSM，ClientSession，ServerSession 被设计为共享 Client NetVConnection 的 mutex，
  //     因此，这里访问传入的 HttpSM 和 HttpClientSession 对象时并不需要上锁。
  to_return = ua_session->get_server_session();
  if (to_return != NULL) {
    // 存在一个关联的ServerSession，那么就把这个ServerSession从ClientSession上拿走
    // 此时，是吧ClientSession当作只能保存一个ServerSession的会话池来使用
    ua_session->attach_server_session(NULL);

    // 然后通过 match 方法来进行判定：
    //     之前关联在 ClientSession 上的 ServerSession 是否能够匹配输入的条件
    if (ServerSessionPool::match(to_return, ip, hostname_hash, match_style)) {
      // 匹配成功
      Debug("http_ss", "[%" PRId64 "] [acquire session] returning attached session ", to_return->con_id);
      // 设置 ServerSession 状态为被状态机接管
      to_return->state = HSS_ACTIVE;
      // 调用 HttpSM::attach_server_session 完成 ServerSession 到 HttpSM 的绑定
      sm->attach_server_session(to_return);
      // 返回匹配完成（成功）
      return HSM_DONE;
    }
    // Release this session back to the main session pool and
    //   then continue looking for one from the shared pool
    // 如果没有匹配成功
    Debug("http_ss", "[%" PRId64 "] [acquire session] "
                     "session not a match, returning to shared pool",
          to_return->con_id);
    // 那么就把这个之前关联在 ClientSession 上的 ServerSession 放入到会话池中，以备将来使用
    to_return->release();
    // 清除 to_return，接下来要存储会话池中查找的结果
    // 注：感觉应该把这行放在大括号外面
    to_return = NULL;
  }

  // Now check to see if we have a connection in our shared connection pool
  // 如果在 ClientSession 上没有找到合适的 ServerSession，那么就要从会话池中查找了
  EThread *ethread = this_ethread();

  // 根据当前 HttpSM 对于会话池的使用方式，选择使用全局会话池还是线程本地会话池，
  //     获得所选会话池对应的锁。
  ProxyMutex *pool_mutex = (TS_SERVER_SESSION_SHARING_POOL_THREAD == sm->t_state.http_config_param->server_session_sharing_pool) ?
                             ethread->server_session_pool->mutex :
                             m_g_pool->mutex;
  // 尝试对会话池上锁
  MUTEX_TRY_LOCK(lock, pool_mutex, ethread);
  if (lock.is_locked()) {
    // 成功上锁
    if (TS_SERVER_SESSION_SHARING_POOL_THREAD == sm->t_state.http_config_param->server_session_sharing_pool) {
      // 如果使用的是线程本地会话池
      // 调用对应会话池的 acquireSession 方法获得匹配的 ServerSession
      retval = ethread->server_session_pool->acquireSession(ip, hostname_hash, match_style, to_return);
      Debug("http_ss", "[acquire session] thread pool search %s", to_return ? "successful" : "failed");
    } else {
      // 如果使用的是全局会话池
      // 调用对应会话池的 acquireSession 方法获得匹配的 ServerSession
      retval = m_g_pool->acquireSession(ip, hostname_hash, match_style, to_return);
      Debug("http_ss", "[acquire session] global pool search %s", to_return ? "successful" : "failed");
    }
    if (to_return) {
      // 找到了匹配的 ServerSession
      Debug("http_ss", "[%" PRId64 "] [acquire session] return session from shared pool", to_return->con_id);
      // 设置 ServerSession 状态为被状态机接管
      to_return->state = HSS_ACTIVE;
      // Holding the pool lock and the sm lock
      // the attach_server_session will issue the do_io_read under the sm lock
      // Must be careful to transfer the lock for the read vio because
      // the server VC may be moving between threads TS-3266
      // 调用 HttpSM::attach_server_session 完成 ServerSession 到 HttpSM 的绑定
      sm->attach_server_session(to_return);
      // 返回匹配完成（成功）
      retval = HSM_DONE;
    }
  } else {
    // 上锁失败
    // 返回需要重试
    retval = HSM_RETRY;
  }
  return retval;
}
```

ServerSessionPool::acquireSession 方法：

  - 只会返回找到的 ServerSession
  - 该 ServerServer 的 NetVConnection 上发生的读写事件，仍然会回调 ServerSessionPool 的状态机

因此，在获得 ServerSession 之后，必须：

  - 立即通过 do_io_read 和 do_io_write 方法重设该 ServerServer 的 I/O 回调
  - 然后才能释放 ServerSessionPool 的锁

而 HttpSM::attach_server_session 就是这样做的，如果暂时不想进行任何读写，至少也要执行：

  - do_io_read(sm, 0, NULL);
  - do_io_write(sm, 0, NULL);

### HttpSessionManager::release_session

```
HSMresult_t
HttpSessionManager::release_session(HttpServerSession *to_release)
{
  EThread *ethread = this_ethread();
  // 根据会话当初被分配的模式（默认为全局会话池）选择匹配的会话池
  ServerSessionPool *pool =
    TS_SERVER_SESSION_SHARING_POOL_THREAD == to_release->sharing_pool ? ethread->server_session_pool : m_g_pool;
  bool released_p = true;

  // The per thread lock looks like it should not be needed but if it's not locked the close checking I/O op will crash.
  // 尝试对会话池上锁
  MUTEX_TRY_LOCK(lock, pool->mutex, ethread);
  if (lock.is_locked()) {
    // 上锁成功
    // 通过 releaseSession 放回到会话池中
    //     其中会通过 do_io_read 和 do_io_write 重设 I/O 回调
    pool->releaseSession(to_release);
  } else {
    // 上锁失败
    Debug("http_ss", "[%" PRId64 "] [release session] could not release session due to lock contention", to_release->con_id);
    released_p = false;
  }

  // 根据是否把 ServerSession 成功放入会话池，返回不同的结果
  return released_p ? HSM_DONE : HSM_RETRY;
}
```

## TODO

通过分析 HttpSessionManager 的代码，感觉这是一个半成品：

  - 声明了回调函数 int main_handler(int event, void *data);
    - 但是却没有继承自 Continuation 类
  - 遇到上锁失败时都没有进行自我重新调度，
    - 而是返回 HSM_RETRY 提示调用者重试
  - 在 HttpSM 和 HttpServerSession 中调用 HttpSessionManager 的方法之后，
    - 总会有“TODO”，“FIXME”这样的内容，提示需要进一步完善

## ServerSession 与 NetVConnection 对于 mutex 的使用

首先看一段代码，来自 HttpSM::attach_server_session 方法

```
void
HttpSM::attach_server_session(HttpServerSession *s)
{
  hsm_release_assert(server_session == NULL);
  hsm_release_assert(server_entry == NULL);
  hsm_release_assert(s->state == HSS_ACTIVE);
  server_session = s; 
  server_session->transact_count++;

  // Set the mutex so that we have something to update
  //   stats with
  server_session->mutex = this->mutex;
...
}
```

这段代码执行时：

  - ServerSession 刚刚从会话池中取出
  - ServerSession 的 I/O 回调仍然指向会话池
  - 会话池已经被上锁，因此 ServerSession 的 I/O 回调不会进行

HttpSM 把 ServerSession 的锁修改为了 HttpSM 的锁，此时以下实例共享同一把锁：

  - Client UnixNetVConnection
  - ClientSession
  - HttpSM
  - ServerSession

唯独 Server UnixNetVConnection 使用的锁是不同的。

谁会对 ServerSession 上锁？

  - 我们知道 HttpServerSession 上是没有任何回调函数的，因此 NetHandler 不会对其上锁
  - 但是如果 ServerSession 总是共享 HttpSM 和 ClientSession 的锁
    - NetHandle 在回调 HttpSM 和 ClientSession 时会上锁，此时也就相当于同时锁定了 ServerSession 的锁
  - 在 HttpServerSession 被放入到会话池中时，并未把锁切换为会话池的锁
    - 那么会话池在被 NetHandler 回调时 HttpServerSession 其实是没有上锁的
    - 但是此时 HttpServerSession 已经从 HttpSM 和 HttpClientSession 脱离，不会被其它对象访问

再回头看 HttpServerSession 是作为 HttpClientSession 的 slave 而存在的，因此可以确定：

  - ServerSession 是 ClientSession 的附属 或者是 SM 的附属
  - 为了安全的访问 ServerSession 的成员，需要 ServerSession 共享其属主的锁
  - 这样在其属主被回调时，其属主自然被上锁，而同时 ServerSession 也因为共享同一把锁也被锁定

这样属主（HttpClientSession 或 HttpSM）被回调时，就可以直接访问 HttpServerSession 的成员了。

## 参考资料

- [HttpProxyAPIEnums.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyAPIEnums.h)
- [HttpSessionManager.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.h)
- [HttpSessionManager.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.cc)
