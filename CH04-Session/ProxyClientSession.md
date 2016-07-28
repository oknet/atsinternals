# 基础组件：ProxyClientSession

ProxyClientSession 是所有 XXXClientSession的基类。

任何 XXXClientSession 都必需定义 start()，new_connection()，get_netvc()，release_netvc() 这四个方法:

  - new_connection()
    - 用来接受 XXXSessionAccept 的调用，传入 netvc，表示开始一个新的会话（Session）
  - start()
    - 表示开始一个新的事务（Transaction）
  - get_netvc()
    - 用来获得传入的 netvc
  - release_netvc()
    - 用来解除与 netvc 的关联

## 定义

```
class ProxyClientSession : public VConnection
{
public:
  ProxyClientSession();

  virtual void destroy();
  virtual void start() = 0;

  virtual void new_connection(NetVConnection *new_vc, MIOBuffer *iobuf, IOBufferReader *reader, bool backdoor) = 0;

  virtual void
  ssn_hook_append(TSHttpHookID id, INKContInternal *cont)
  {
    this->api_hooks.prepend(id, cont);
  }

  virtual void
  ssn_hook_prepend(TSHttpHookID id, INKContInternal *cont)
  {
    this->api_hooks.prepend(id, cont);
  }

  virtual NetVConnection *get_netvc() const = 0;
  virtual void release_netvc() = 0;

  APIHook *
  ssn_hook_get(TSHttpHookID id) const
  {
    return this->api_hooks.get(id);
  }

  void *
  get_user_arg(unsigned ix) const
  {
    ink_assert(ix < countof(user_args));
    return this->user_args[ix];
  }

  void
  set_user_arg(unsigned ix, void *arg)
  {
    ink_assert(ix < countof(user_args));
    user_args[ix] = arg;
  }

  // Return whether debugging is enabled for this session.
  bool
  debug() const
  {
    return this->debug_on;
  }

  bool
  hooks_enabled() const
  {
    return this->hooks_on;
  }

  bool
  has_hooks() const
  {
    return this->api_hooks.has_hooks() || http_global_hooks->has_hooks();
  }

  // Initiate an API hook invocation.
  void do_api_callout(TSHttpHookID id);

  static int64_t next_connection_id();

protected:
  // XXX Consider using a bitwise flags variable for the following flags, so that we can make the best
  // use of internal alignment padding.

  // Session specific debug flag.
  bool debug_on;
  bool hooks_on;

private:
  APIHookScope api_scope;
  TSHttpHookID api_hookid;
  APIHook *api_current;
  HttpAPIHooks api_hooks;
  void *user_args[HTTP_SSN_TXN_MAX_USER_ARG];

  ProxyClientSession(ProxyClientSession &);                  // noncopyable
  ProxyClientSession &operator=(const ProxyClientSession &); // noncopyable

  int state_api_callout(int event, void *edata);
  void handle_api_return(int event);

  friend void TSHttpSsnDebugSet(TSHttpSsn, int);
};
```

## 方法

### 调试相关

debug()

  - 返回成员 debug_on
  - 用来指示当前Session是否开启了 Debug 功能


### 自定义数据

void *user_args[HTTP_SSN_TXN_MAX_USER_ARG];

  - Session提供了这样的数组来保存自定义数据

set_user_arg(unsigned ix, void *arg)

  - 可以在Session上关联一个指针，指向自定义数据
  - ix 为索引号码，最大值为 HTTP_SSN_TXN_MAX_USER_ARG-1

get_user_arg(unsigned ix)

  - 获取指向自定义数据的指针
  - ix 为索引号码，最大值为 HTTP_SSN_TXN_MAX_USER_ARG-1

### Hook 相关

hooks_enabled()

  - 返回成员 hooks_on
  - 用来指示当前Session是否开启了 Hook 功能
  - 在构造函数中，默认初始化为 true
  - 只有在 HttpClientSession 中，backdoor 为 true 时，才会设置为 false

has_hooks()

  - 通过查看 hook 链表，来确认当前的 Session 是否有任何的 hook 需要触发

ssn_hook_prepend(TSHttpHookID id, INKContInternal *cont)

  - 在 id 这个 HookID 上添加一个回调 cont
  - 如果该 id 上已经有了其它的回调 cont，则把这个 cont 放在最前面

ssn_hook_append(TSHttpHookID id, INKContInternal *cont)

  - 在 id 这个 HookID 上添加一个回调 cont
  - 如果该 id 上已经有了其它的回调 cont，则把这个 cont 放在最后面

do_api_callout(TSHttpHookID id)

  - 获取指定 Hook id 的回调 cont，并直接调用 state_api_callout 对 cont 进行回调
  - 同时设置 Session 的 handleEvent 为 state_api_callout

state_api_callout(int event, void *edata)

  - 作为 Session 的 handleEvent 方法，用来处理一个 Hook id 绑定了多个 cont 的情况
  - 当第一个 cont 传回 CONTINUE 事件时，会继续回调下一个 cont
  - 当所有的 cont 都回调完成时，调用 handle_api_return 继续

handle_api_return(int event)

  - 用来实现从特定 Hook id 的 cont 回调之后，返回到 ATS 的流程中

## 从 Hook ID 到 Plugin

在ATS中，Hook ID 作为ATS主线流程中的一个路口，可以用来回调 Plugin，然后再返回到这个点，继续运行ATS的主线流程。

有这么一个路口，但是不一定每次都要拐出去，因为可能没有 Plugin 在这个岔路口的末端。

那么什么时候能够 / 需要回调 Plugin ？在ATS中，设计了三个层次：

  - 全局（Global Hooks）
    - 就是一种既定道路，到了这里就一定要回调 Plugin
    - 通常不区分协议
  - 会话（Session Hooks）
    - 也是一种既定道路，到了这里就一定要回调 Plugin
    - 但是这些Hook点通常是与协议紧密关联的
  - 事务（Transaction Hooks）
    - 是一种临时设定的道路，可以认为是基于条件回调 Plugin
    - 在 Session Hooks 回调 Plugin 之后，在 Plugin 中进行一些条件判断后才会设置是否触发 Hook 对 Plugin 的回调

有的 Hook ID 可以作为“会话”级使用，也可以作为“事务”级使用，因此ATS约定：

  - 如果一个 Hook ID 可以有多个级别的 Plugin 回调
  - 那么先调用全局（Global Hooks）
  - 然后调用会话（Session Hooks）
  - 最后调用事务（Transaction Hooks）

如果一个 Hook ID 上有多个 Plugin 等待回调：

  - 按照各个 Plugin 注册到这个 Hook ID 的顺序进行
  - 同一个 Hook ID 上等待回调的 Plugin 被放在一个链表里
  - 通过 ssn_hook_prepend 和 ssn_hook_append 可以在这个链表的 头部 和 尾部 注册新的 Plugin

ATS 定义了三个方法来支持：

  - Hook ID 的触发 do_api_callout(id)
  - 对 Plugin 的回调 state_api_callout(event, data)
  - 从 Plugin 返回到 ATS 的主流程 handle_api_return(id)

这三个方法在 Global，Session 和 Transaction / SM 内都有对应的实现。

在 ProxyClientSession 中定义的这一套，三个方法是处于 Global 和 Session 层次的：

```
void
ProxyClientSession::do_api_callout(TSHttpHookID id)
{
  // 通过assert来限定只支持HTTP协议的 SSN_START 和 SSN_CLOSE 两个 Hook
  // 这里实现的并不是特别好，对于希望扩展ATS来支持更多协议的时候，就需要把这里改写到HttpClientSession里面去
  ink_assert(id == TS_HTTP_SSN_START_HOOK || id == TS_HTTP_SSN_CLOSE_HOOK);

  // 三个与 Plugin 回调相关的变量
  // api_hookid 用来记录这一次是触发了哪一个hook id的回调
  this->api_hookid = id;
  // api_scope 是用来说明这个hook id适用的最高层级，可以看到这里设置为Global
  this->api_scope = API_HOOK_SCOPE_GLOBAL;
  // api_current 是用来表示当前正在回调的 plugin
  //   plugin 也是一个状态机，需要通过多次回调才能完成
  //   然后会继续回调当前 hook id 上关联的下一个 plugin
  this->api_current = NULL;

  // has_hooks() 的判断是动态的，
  //   每次从 hook 链表中取走一个 plugin，最后 hook 链表就变为 NULL
  if (this->hooks_on && this->has_hooks()) {
    // 切换ProxyClientSession的handler
    SET_HANDLER(&ProxyClientSession::state_api_callout);
    // 通过state_api_callout来进行首次回调
    this->state_api_callout(EVENT_NONE, NULL);
  } else {
    // 如果这个 Hook ID 上没有 plugin，那么就直接通过 handle_api_return 回到 ATS 的主流程
    // 这里实现的并不是特别好，对于希望扩展ATS来支持更多协议的时候，就需要把这里改写到HttpClientSession里面去
    this->handle_api_return(TS_EVENT_HTTP_CONTINUE);
  }
}
```

```
int
ProxyClientSession::state_api_callout(int event, void * /* data ATS_UNUSED */)
{
  switch (event) {
  case EVENT_NONE:
  case EVENT_INTERVAL:
  case TS_EVENT_HTTP_CONTINUE:
    if (likely(is_valid_hook(this->api_hookid))) {
      if (this->api_current == NULL && this->api_scope == API_HOOK_SCOPE_GLOBAL) {
        this->api_current = http_global_hooks->get(this->api_hookid);
        this->api_scope = API_HOOK_SCOPE_LOCAL;
      }

      if (this->api_current == NULL && this->api_scope == API_HOOK_SCOPE_LOCAL) {
        this->api_current = ssn_hook_get(this->api_hookid);
        this->api_scope = API_HOOK_SCOPE_NONE;
      }

      if (this->api_current) {
        bool plugin_lock = false;
        APIHook *hook = this->api_current;
        Ptr<ProxyMutex> plugin_mutex;

        if (hook->m_cont->mutex) {
          plugin_mutex = hook->m_cont->mutex;
          plugin_lock = MUTEX_TAKE_TRY_LOCK(hook->m_cont->mutex, mutex->thread_holding);
          if (!plugin_lock) {
            SET_HANDLER(&ProxyClientSession::state_api_callout);
            mutex->thread_holding->schedule_in(this, HRTIME_MSECONDS(10));
            return 0;
          }
        }

        this->api_current = this->api_current->next();
        hook->invoke(eventmap[this->api_hookid], this);

        if (plugin_lock) {
          Mutex_unlock(plugin_mutex, this_ethread());
        }

        return 0;
      }
    }

    handle_api_return(event);
    break;

  case TS_EVENT_HTTP_ERROR:
    this->handle_api_return(event);
    break;

  // coverity[unterminated_default]
  default:
    ink_assert(false);
  }

  return 0;
}
```


## 参考资料

- [ProxyClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/ProxyClientSession.h)
- [ProxyClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/ProxyClientSession.cc)
