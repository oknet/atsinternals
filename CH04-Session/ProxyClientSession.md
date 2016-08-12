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

ProxyClientSession 不能直接使用，必须定义其继承类来使用。

```
class ProxyClientSession : public VConnection
{
public:
  ProxyClientSession();

  // 回收 ClientSession 的资源
  //   释放成员 api_hooks 占用的内存
  //   释放成员 mutex 占用的内存
  // 注意：
  //   在继承类中需要定义此方法，用来释放对象占用的内存空间
  //   调用该方法之前必须首先执行 do_io_close 和 release_netvc 两个方法。
  // 这里是否应该声明为纯虚函数？
  virtual void destroy();
  // 开始一个事务
  virtual void start() = 0;

  // 开始一个会话
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

  // 获取与ClientSession对应的netvc
  virtual NetVConnection *get_netvc() const = 0;
  // 与netvc解除关联，但是不会关闭netvc
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
  // 当前正在处理的Hook信息
  //     Hook 等级（全局、会话、事务）
  APIHookScope api_scope;
  //     Hook ID
  TSHttpHookID api_hookid;
  //     当前正在回调的与此Hook ID关联的Plugin回调函数
  APIHook *api_current;
  // 所有注册到当前 ClientSession 的Plugin回调函数与Hook ID的对应关系
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

  - do_api_callout 发起对指定Hook ID上的plugin函数的调用
  - 如果没有任何plugin hook在指定的Hook ID上，就直接调用handle_api_return回到ATS的主流程
  - 否则，通过state_api_callout来实现对plugin函数的回调

```
// 发起对指定Hook ID上的plugin函数的调用
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
// Hook ID的合法性检查
static bool
is_valid_hook(TSHttpHookID hookid)
{
  return (hookid >= 0) && (hookid < TS_HTTP_LAST_HOOK);
}
```

```
// 实现对plugin函数的回调
int
ProxyClientSession::state_api_callout(int event, void * /* data ATS_UNUSED */)
{
  switch (event) {
  case EVENT_NONE:  // 表示首次触发，此时：
                    //   this->api_hookid  为触发的Hook ID
                    //   this->api_scope   为触发的层次：Global, Local, None 分别表示全局、会话、事务但哥级别
                    //   this->api_current 为NULL，之后将会指向正在执行的plugin回调函数
  case EVENT_INTERVAL:  // 表示这是直接来自EventSystem的定时回调
                        // 通常在回调 Plugin 时需要上锁，如果上锁失败，则会安排EventSystem延时重新回调
  case TS_EVENT_HTTP_CONTINUE:  // 表示这是来自 Plugin 的通知消息，让ATS继续处理这个会话
                                // 通过 TSHttpSsnReenable 可以向 ClientSession 发送消息
    // 判断 api_hookid 是否是一个合法有效的Hook ID
    if (likely(is_valid_hook(this->api_hookid))) {
      // api_current == NULL 表示当前没有正在执行的plugin回调函数
      // api_scope == GLOBAL 表示需要从全局层级开始查找plugin回调函数的设定
      if (this->api_current == NULL && this->api_scope == API_HOOK_SCOPE_GLOBAL) {
        // 获取 api_hookid 对应的全局层级的 plugin 回调函数
        // 如果没有找到，那么返回NULL，则api_current == NULL
        // 这里 http_global_hooks 为全局变量
        this->api_current = http_global_hooks->get(this->api_hookid);
        // 设置下一个探查层级为会话层级
        this->api_scope = API_HOOK_SCOPE_LOCAL;
      }

      // 如果获取全局层级失败，则 api_current == NULL
      //     则继续获 api_hookid 对应的取会话层级的 plugin 回调函数
      // 如果获取全局层级成功，则跳过
      if (this->api_current == NULL && this->api_scope == API_HOOK_SCOPE_LOCAL) {
        // 获取 api_hookid 对应的会话层级的 plugin 回调函数
        // 如果没有找到，那么返回NULL，则api_current == NULL
        // 这里 ssn_hook_get 为成员函数
        this->api_current = ssn_hook_get(this->api_hookid);
        // 设置下一个探查层级为事务层级
        this->api_scope = API_HOOK_SCOPE_NONE;
      }

      // 由于 TS_HTTP_SSN_START_HOOK 和 TS_HTTP_SSN_CLOSE_HOOK 不支持会话层级
      // 因此这里没有继续判断，获取会话层级的 plugin 回调函数

      // 如果获取 plugin 回调函数成功，这里可能是 全局 或 会话 层级
      if (this->api_current) {
        bool plugin_lock = false;
        APIHook *hook = this->api_current;
        // 创建自动指针
        Ptr<ProxyMutex> plugin_mutex;

        // 每一个 plugin 的回调函数都由 Continuation 来封装，因此都会有一个 mutex
        // 如果这个mutex被正确设置了，则需要对mutex上锁，
        // 但是，也会存在mutex未设置的情况，此时则跳过上锁过程。
        if (hook->m_cont->mutex) {
          plugin_mutex = hook->m_cont->mutex;
          // 对 plugin 的 Cont 尝试上锁
          plugin_lock = MUTEX_TAKE_TRY_LOCK(hook->m_cont->mutex, mutex->thread_holding);
          // 上锁失败，则在 10ms 之后重新调用 state_api_callout 重试
          if (!plugin_lock) {
            SET_HANDLER(&ProxyClientSession::state_api_callout);
            mutex->thread_holding->schedule_in(this, HRTIME_MSECONDS(10));
            return 0;
          }
          // 注意此处没有自动解锁！！
        }

        // 如果有多个 plugin 都对同一个Hook ID有兴趣，那么则继续设置该Hook ID对应的下一个 plugin 的回调函数
        // 如果这是该 Hook ID 在当前层级的最后一个 plugin，那么会返回 NULL
        this->api_current = this->api_current->next();
        // 回调 plugin 的 Hook 函数
        // 对于没有正确设置mutex的情况，不对plugin的mutex上锁，那么直接回调 plugin 难道不会有问题吗？？？
        // invoke 方法，相当于直接调用了 plugin 里设置的回调函数，通过eventmap数组把Hook ID转换为对应的Event ID
        hook->invoke(eventmap[this->api_hookid], this);

        // 如果之前成功上锁，这里要显示解锁
        if (plugin_lock) {
          Mutex_unlock(plugin_mutex, this_ethread());
        }

        // 成功回调之后，直接返回
        // 等待 Plugin 调用 TSHttpSsnReenable 方法，此时会重新调用 ClientSession 的 handleEvent()
        // 而 handleEvent 则指向 state_api_callout
        // Plugin 会传入两种类型的事件：TS_EVENT_HTTP_CONTINUE 或 TS_EVENT_HTTP_ERROR
        // 感觉这里设计的也不是特别好，与HTTP协议相关的EVENT，竟然放在了ProxyClientSession的基类里
        // 这里实际上是返回的 EVENT_DONE = 0
        return 0;
      }
    }

    // 如果在 全局 和 会话 级别都没有找到与 Hook ID 关联的 Plugin，则直接通过 handle_api_return 返回到ATS的主流程
    handle_api_return(event);
    break;

  case TS_EVENT_HTTP_ERROR:  // 表示这是来自 Plugin 的通知消息，通知ATS这个会话出现了错误，需要由ATS终止这个会话
                             // 通过 TSHttpSsnReenable 可以向 ClientSession 发送消息
    // 通过将 TS_EVENT_HTTP_ERROR 传递给 handle_api_return 来完成错误处理
    this->handle_api_return(event);
    break;

  // coverity[unterminated_default]
  default:
    ink_assert(false);
  }

  // 这里实际上是返回的 EVENT_DONE = 0
  return 0;
}
```

```
// 回到ATS的主流程
void
ProxyClientSession::handle_api_return(int event)
{
  // 保存当前的 Hook ID
  TSHttpHookID hookid = this->api_hookid;
  // 设置回调函数为 state_api_callout
  // 这是一个保护性设置，但是在应用中不会被回调
  //     如果不幸被回调了，会触发assert
  SET_HANDLER(&ProxyClientSession::state_api_callout);

  // 重置 ClientSession 的 Hook 状态
  this->api_hookid = TS_HTTP_LAST_HOOK;
  this->api_scope = API_HOOK_SCOPE_NONE;
  this->api_current = NULL;

  // 根据之前保存的 Hook ID 来执行对应的操作
  switch (hookid) {
  case TS_HTTP_SSN_START_HOOK:
    // 如果 Hook ID 是 SSN START
    if (event == TS_EVENT_HTTP_ERROR) {
      // 如果之前通过 TSHttpSsnReenable 传递的是 ERROR，则执行关闭ClientSession的操作
      this->do_io_close();
    } else {
      // 如果是 CONTINUE，则开始一个事务
      this->start();
    }
    break;
  case TS_HTTP_SSN_CLOSE_HOOK: {
    // 如果 Hook ID 是 SSN CLOSE
    NetVConnection *vc = this->get_netvc();
    if (vc) {
      // 由于是 SSN CLOSE HOOK，无论出现任何错误，都是要关闭ClientSession的，所以就不需要对 event 进行判断了
      // 关闭 netvc
      vc->do_io_close();
      // 把保存了 netvc 的成员变量设置为 NULL
      this->release_netvc();
    }
    // 首先，释放继承类的成员对象
    // 然后，释放 ProxyClientSession 的 api_hooks 和 mutex
    // 最后，回收 ClientSession 对象的内存
    this->destroy();
    break;
  }
  default:
    Fatal("received invalid session hook %s (%d)", HttpDebugNames::get_api_hook_name(hookid), hookid);
    break;
  }
}
```
可以看到在ProxyClientSession这个基类里只实现了对SSN START和SSN CLOSE两个Hook点返回到ATS主流程之后的处理。

但是在 SSN START 遇到错误关闭 ClientSession 时，与 SSN CLOSE 之后关闭 ClientSession 的代码又完全不一样，这是为什么？

  - 在调用 this->do_io_close() 之后，仍然需要触发 SSN CLOSE Hook
  - 在 SSN CLOSE Hook 之后，直接关闭NetVC，并释放 ClientSession 的资源就可以了

所以：

  - this->do_io_close() 被设计用来
    - 执行对 ClientSession 的关闭，只是设置状态为关闭，但是不回收任何资源
    - 然后触发 SSN CLOSE Hook
  - 由 Plugin 返回后，来到 handle_api_return 的 SSN CLOSE Hook 的处理点
    - 在这里才是最后关闭 NetVC
    - 然后回收 ClientSession 的资源

## 与 SessionAccept 的关系

XxxSessionAccept::accept()

  - cs = THREAD_ALLOC(XxxClientSession)
  - cs->new_connection()

XxxClientSession::new_connection()

  - do_api_callout(TS_HTTP_SSN_START_HOOK)
  - handle_api_return(TS_HTTP_SSN_START_HOOK)
  - start()

XxxClientSession::start()

  - 开始事务处理
  - Http 协议相对简单，在 start() 中调用 new_transaction()
  - H2 协议相对复杂，需要做更多的操作，则未定义 new_transaction()

XxxxSessionAccept 通常用来创建 XxxxClientSession，例如：

  - HttpSessionAccept 创建 HttpClientSession
  - Http2SessionAccept 创建 Http2ClientSession

## 参考资料

- [ProxyClientSession.h](http://github.com/apache/trafficserver/tree/master/proxy/ProxyClientSession.h)
- [ProxyClientSession.cc](http://github.com/apache/trafficserver/tree/master/proxy/ProxyClientSession.cc)
