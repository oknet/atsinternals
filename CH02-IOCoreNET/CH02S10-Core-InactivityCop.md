# 核心组件：InactivityCop 状态机

InactivityCop是对早期的超时机制的改进，这个改进降低了EventSystem的压力。

原来的超时控制完全依靠EventSystem的定时事件来实现，导致EventSystem的事件队列内容纳了大量的事件。

对于一个 NetVConnection 的两种超时类型，10K连接就有20K事件长期塞在EventSystem的内部队列里。

因此ATS或者是更早的Inktomi公司，重构了超时管理，提出了InactivityCop状态机，专门用来支持 NetVConnection 的两种超时。

对于此部分改进的一些讨论，可以参阅官方JIRA的两个Issue：

  - [TS-3313 New World order for connection management and timeouts](https://issues.apache.org/jira/browse/TS-3313)
  - [TS-1405 apply time-wheel scheduler about event system](https://issues.apache.org/jira/browse/TS-1405)

对于InactivityCop管理的几个队列的介绍，可以参见 NetHandler 的章节。

## 定义

```
// 未定义此宏，表示使用新的超时管理机制：InactivityCop状态机
#ifndef INACTIVITY_TIMEOUT
int update_cop_config(const char *name, RecDataT data_type, RecData data, void *cookie);

// INKqa10496
// One Inactivity cop runs on each thread once every second and
// loops through the list of NetVCs and calls the timeouts
// 每个线程运行了一个InactivityCop状态机，每秒执行一次
// 便利队列中的NetVC，并回调超时的NetVC状态机
class InactivityCop : public Continuation
{
public:
  // 构造函数
  // 注意这里初始化 ProxyMutex，传入的是NetHandler的ProxyMutex
  InactivityCop(ProxyMutex *m) : Continuation(m), default_inactivity_timeout(0)
  {
    // 只有一个回调函数：check_inactivity
    SET_HANDLER(&InactivityCop::check_inactivity);
    REC_ReadConfigInteger(default_inactivity_timeout, "proxy.config.net.default_inactivity_timeout");
    Debug("inactivity_cop", "default inactivity timeout is set to: %d", default_inactivity_timeout);

    // 配置文件重新加载时，对于指定配置项的回调函数
    RecRegisterConfigUpdateCb("proxy.config.net.default_inactivity_timeout", update_cop_config, (void *)this);
  }

  // Inactivity Timeout 处理函数
  // 前面讲了，InactivityCop会引用NetHandler的ProxyMutex，因此在此函数的回调期间，NetHandler也是被锁定的状态
  // 因为InactivityCop会访问到NetHandler的一些私有成员和队列
  int
  check_inactivity(int event, Event *e)
  {
    (void)event;
    ink_hrtime now = Thread::get_hrtime();
    NetHandler &nh = *get_NetHandler(this_ethread());

    Debug("inactivity_cop_check", "Checking inactivity on Thread-ID #%d", this_ethread()->id);
    // Copy the list and use pop() to catch any closes caused by callbacks.
    // forl_LL 是一个宏，类似于foreach，用来遍历NetHandler的open_list队列
    forl_LL(UnixNetVConnection, vc, nh.open_list)
    {
      // 如果vc的管理线程是当前线程，就把vc放到cop_list队列（堆栈）里
      // 不过，真的会有不属于当前线程管理的vc被放入到open_list里吗？？？
      if (vc->thread == this_ethread()) {
        nh.cop_list.push(vc);
      }
    }
    // 接下来遍历cop_list队列（堆栈）
    while (UnixNetVConnection *vc = nh.cop_list.pop()) {
      // If we cannot get the lock don't stop just keep cleaning
      // 尝试对vc上锁
      MUTEX_TRY_LOCK(lock, vc->mutex, this_ethread());
      // 上锁失败则跳过此vc，继续处理后面的vc
      if (!lock.is_locked()) {
        NET_INCREMENT_DYN_STAT(inactivity_cop_lock_acquire_failure_stat);
        continue;
      }

      // 如果vc已经关闭，则调用close方法关闭vc，然后继续处理后面的vc
      if (vc->closed) {
        close_UnixNetVConnection(vc, e->ethread);
        continue;
      }

      // set a default inactivity timeout if one is not set
      // 如果开启了Inactivity Timeout功能：default_inactivity_timeout > 0
      //     当前vc的超时计时器为0，那么就初始化为默认值
      if (vc->next_inactivity_timeout_at == 0 && default_inactivity_timeout > 0) {
        Debug("inactivity_cop", "vc: %p inactivity timeout not set, setting a default of %d", vc, default_inactivity_timeout);
        vc->set_inactivity_timeout(HRTIME_SECONDS(default_inactivity_timeout));
        NET_INCREMENT_DYN_STAT(default_inactivity_timeout_stat);
      } else {
        Debug("inactivity_cop_verbose", "vc: %p now: %" PRId64 " timeout at: %" PRId64 " timeout in: %" PRId64, vc, now,
              ink_hrtime_to_sec(vc->next_inactivity_timeout_at), ink_hrtime_to_sec(vc->inactivity_timeout_in));
      }

      // 对于已经设置了计时器的vc：vc->next_inactivity_timeout_at > 0
      // 而且满足Inactivity Timeout的情况：vc->next_inactivity_timeout_at < now
      if (vc->next_inactivity_timeout_at && vc->next_inactivity_timeout_at < now) {
        // 如果此vc存在于keep_alive_queue中，则进行一个全局计数器的统计
        if (nh.keep_alive_queue.in(vc)) {
          // only stat if the connection is in keep-alive, there can be other inactivity timeouts
          // 由于在net_activity()中：next_inactivity_timeout_at = get_hrtime() + inactivity_timeout_in
          // 因此：diff 就是上一次 net_activity 执行时到当前时刻的时间差
          ink_hrtime diff = (now - (vc->next_inactivity_timeout_at - vc->inactivity_timeout_in)) / HRTIME_SECOND;
          NET_SUM_DYN_STAT(keep_alive_queue_timeout_total_stat, diff);
          NET_INCREMENT_DYN_STAT(keep_alive_queue_timeout_count_stat);
        }
        Debug("inactivity_cop_verbose", "vc: %p now: %" PRId64 " timeout at: %" PRId64 " timeout in: %" PRId64, vc, now,
              vc->next_inactivity_timeout_at, vc->inactivity_timeout_in);
        // 对于超时的vc，则回调mainEvent来处理，上层状态机会收到超时事件的回调
        // 此处对mainEvent的回调，总是能满足mainEvent中对NetHandler，ReadVIO和WriteVIO的上锁要求
        // 因此这个回调总是以同步方式完成
        vc->handleEvent(EVENT_IMMEDIATE, e);
      }
    }

    // Cleanup the active and keep-alive queues periodically
    // 遍历完所有当前线程管理的vc后，先后调用下面两个方法，在后面分析
    nh.manage_active_queue();
    nh.manage_keep_alive_queue();

    return 0;
  }

  // 提供给update_cop_config的方法，用来设置缺省inactivity_timeout的值
  void
  set_default_timeout(const int x)
  {
    default_inactivity_timeout = x;
  }

private:
  int default_inactivity_timeout; // only used when one is not set for some bad reason
};

// 这个是从配置文件读取配置，更新InactivityCop的缺省inactivity_timeout的值
int
update_cop_config(const char *name, RecDataT data_type ATS_UNUSED, RecData data, void *cookie)
{
  InactivityCop *cop = static_cast<InactivityCop *>(cookie);
  ink_assert(cop != NULL);

  if (cop != NULL) {
    if (strcmp(name, "proxy.config.net.default_inactivity_timeout") == 0) {
      Debug("inactivity_cop_dynamic", "proxy.config.net.default_inactivity_timeout updated to %" PRId64, data.rec_int);
      cop->set_default_timeout(data.rec_int);
    }
  }

  return REC_ERR_OKAY;
}

#endif
```

## Active Queue & Keep-Alive Queue

在 NetHandler 中声明了两个队列：active_queue 和 keep_alive_queue，

- 只用于存放 Client 与 ATS 之间的 NetVC
- 但是 ATS 与 Origin Server 之间的 NetVC 不会放入这两个队列，而是由 ServerSessionPooll 来管理
- 这两个队列是针对 HTTP 协议设计的

显然这两个队列出现在 NetHandler 的成员中，是不太合理的，这样的设计需要进一步的抽象 / 优化。

在 HTTP 协议中，一个会话可以包含多个事务，事务之间可以存在不定长时间的间隙，

- 如果这个会话正在处理一个事务，那么这个会话对应的 NetVC 就被放入 active_queue
- 如果这个会话正处于两个事务之间的间隙，那么这个会话对应的 NetVC 就被放入 keep_alive_queue

当 ATS 的可用连接数不足（达到允许的连接数上限）时，会强制关闭 keep_alive_queue 中的 NetVC，甚至是 active_queue 中的 NetVC，然后才能接受这个新的会话。

## NetHandler的延伸：(add_to|remote_from)_active_queue

添加NetVC到active_queue，或从active_queue中移除NetVC

```
bool
NetHandler::add_to_active_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);
  Debug("net_queue", "max_connections_per_thread_in: %d active_queue_size: %d keep_alive_queue_size: %d",
        max_connections_per_thread_in, active_queue_size, keep_alive_queue_size);

  // 首先整理active_queue，看看是否超出最大连接的限制
  // if active queue is over size then close inactive connections
  if (manage_active_queue() == false) {
    // 超出限制，直接返回失败
    // there is no room left in the queue
    return false;
  }

  if (active_queue.in(vc)) {
    // 如果已经在队列中，则从队列移除，后面在添加进去之后就放在头部，可以被尽快处理
    // already in the active queue, move the head
    active_queue.remove(vc);
  } else {
    // 如果不在队列中，则要尝试从active_queue里移除，因为一个NetVC不能同时出现在active_queue和keep_alive_queue
    // in the keep-alive queue or no queue, new to this queue
    remove_from_keep_alive_queue(vc);
    // 因为之前此NetVC不在keep_alive_queue里，所以次数要对计数器做增量
    ++active_queue_size;
  }
  // 将NetVC放入active_queue
  active_queue.enqueue(vc);

  return true;
}

void
NetHandler::remove_from_active_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);
  if (active_queue.in(vc)) {
    // 如果已经在队列中，则从队列移除
    active_queue.remove(vc);
    // 并对计数器做递减
    --active_queue_size;
  }
}
```

## NetHandler的延伸：manage_active_queue

对active_queue的管理，在InactivityCop的主流程里只看到了对Inactivity Timeout的检查和处理，那么Active Timeout的处理是在哪里完成的呢？

下面的 manage_active_queue 并未实现Active Timeout的处理，只是在active_queue达到上限值时，主动遍历active_queue关掉Inactivity Timeout和Active Timeout的NetVC。

通过阅读代码，我觉得在InactivityCop的实现中，Active Timeout好像被弄坏了，不能用了。

```
bool
NetHandler::manage_active_queue()
{
  // 此处用来做最大连接数的限制
  const int total_connections_in = active_queue_size + keep_alive_queue_size;
  Debug("net_queue", "max_connections_per_thread_in: %d max_connections_active_per_thread_in: %d total_connections_in: %d "
                     "active_queue_size: %d keep_alive_queue_size: %d",
        max_connections_per_thread_in, max_connections_active_per_thread_in, total_connections_in, active_queue_size,
        keep_alive_queue_size);

  // 没有达到最大连接数则不进行处理
  if (max_connections_active_per_thread_in > active_queue_size) {
    return true;
  }

  // 达到最大连接数时则进行连接主动回收
  // 获取当前时间
  ink_hrtime now = Thread::get_hrtime();

  // loop over the non-active connections and try to close them
  UnixNetVConnection *vc = active_queue.head;
  UnixNetVConnection *vc_next = NULL;
  int closed = 0;
  int handle_event = 0;
  int total_idle_time = 0;
  int total_idle_count = 0;
  // 遍历active_queue，主动关闭超时的连接
  for (; vc != NULL; vc = vc_next) {
    vc_next = vc->active_queue_link.next;
    if ((vc->next_inactivity_timeout_at <= now) || (vc->next_activity_timeout_at <= now)) {
      _close_vc(vc, now, handle_event, closed, total_idle_time, total_idle_count);
    }
    // 恢复到最大连接之下，则不对后面的vc进行处理了
    if (max_connections_active_per_thread_in > active_queue_size) {
      return true;
    }
  }

  // 返回 false 表示不能再接受新vc了，达到了连接的最大值
  // 在ATS的代码里，只有add_to_active_queue()会对此值进行判断
  return false; // failed to make room in the queue, all connections are active
}
```

## NetHandler的延伸：（add_to|remove_from)_keep_alive_queue

添加NetVC到keep_alive_queue，或从keep_alive_queue中移除NetVC

```
void
NetHandler::add_to_keep_alive_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);

  if (keep_alive_queue.in(vc)) {
    // 如果已经在队列中，则从队列移除，后面在添加进去之后就放在头部，可以被尽快处理
    // already in the keep-alive queue, move the head
    keep_alive_queue.remove(vc);
  } else {
    // 如果不在队列中，则要尝试从active_queue里移除，因为一个NetVC不能同时出现在active_queue和keep_alive_queue
    // in the active queue or no queue, new to this queue
    remove_from_active_queue(vc);
    // 因为之前此NetVC不在keep_alive_queue里，所以次数要对计数器做增量
    ++keep_alive_queue_size;
  }
  // 将NetVC放入keep_alive_queue
  keep_alive_queue.enqueue(vc);

  // 整理keep_alive_queue
  // if keep-alive queue is over size then close connections
  manage_keep_alive_queue();
}

void
NetHandler::remove_from_keep_alive_queue(UnixNetVConnection *vc)
{
  Debug("net_queue", "NetVC: %p", vc);
  if (keep_alive_queue.in(vc)) {
    // 如果已经在队列中，则从队列移除
    keep_alive_queue.remove(vc);
    // 并对计数器做递减
    --keep_alive_queue_size;
  }
}
```

## NetHandler的延伸：manage_keep_alive_queue

在Http/1.1的实现里，有一个叫做Keep Alive的功能，具有以下行为：

  - 处理完一个请求后，不立即关闭连接，变成IDLE状态
  - 处于IDLE状态的连接有独立的超时时间设置

这就对超时控制有了新的要求。

ATS通过把此类型的vc放入到keep_alive_queue中进行处理，下面就是管理keep_alive_queue的方法。

```
void
NetHandler::manage_keep_alive_queue()
{
  // 此处用来做最大连接数的限制
  uint32_t total_connections_in = active_queue_size + keep_alive_queue_size;
  // 获取当前时间
  ink_hrtime now = Thread::get_hrtime();

  Debug("net_queue", "max_connections_per_thread_in: %d total_connections_in: %d active_queue_size: %d keep_alive_queue_size: %d",
        max_connections_per_thread_in, total_connections_in, active_queue_size, keep_alive_queue_size);

  // 没有达到最大连接数则不进行处理，这里与active_queue部分的判定不同
  // 用了 active_queue_size + keep_alive_queue_size 的和做判断，因为：
  //     keep_alive_queue 是一个可选的部分，只是为了能够减少TCP重建连接的成本
  // 所以：它其实是借用了active_queue的连接数配额，在active_queue需要的时候，则不能继续增大keep_alive_queue
  if (!max_connections_per_thread_in || total_connections_in <= max_connections_per_thread_in) {
    return;
  }

  // 达到最大连接数时则进行连接主动回收
  // loop over the non-active connections and try to close them
  UnixNetVConnection *vc_next = NULL;
  int closed = 0;
  int handle_event = 0;
  int total_idle_time = 0;
  int total_idle_count = 0;
  // 遍历keep_alive_queue，主动关闭一部分连接
  // 因为是keep_alive连接，关闭之后，只是影响一点性能
  // 如果active_queue把所有的连接都用光了，那么keep_alive_queue就会是空的
  for (UnixNetVConnection *vc = keep_alive_queue.head; vc != NULL; vc = vc_next) {
    vc_next = vc->keep_alive_queue_link.next;
    _close_vc(vc, now, handle_event, closed, total_idle_time, total_idle_count);

    // 恢复到最大连接之下，则不对后面的vc进行处理了
    total_connections_in = active_queue_size + keep_alive_queue_size;
    if (total_connections_in <= max_connections_per_thread_in) {
      break;
    }
  }

  if (total_idle_count > 0) {
    Debug("net_queue", "max cons: %d active: %d idle: %d already closed: %d, close event: %d mean idle: %d\n",
          max_connections_per_thread_in, total_connections_in, keep_alive_queue_size, closed, handle_event,
          total_idle_time / total_idle_count);
  }
  // 由于keep_alive_queue是一个可选的连接池，因此不存在失败的情况，所以不用返回什么值
}
```

## NetHandler的延伸：_close_vc

在manage_active_queue()和manage_keep_alive_queue()都调用了 _close_vc 这个方法，来主动关闭一些vc。

在InactivityCop的主回调函数中，对cop_list队列遍历时，发现了超时情况时，执行的策略与此方法相似，但又有一点不同。

传入参数：

  - vc 待关闭的VC
  - now 用来计算超时时间的，表示当前时间的值

返回多个计数器的值：

  - handle_event 当本次处理强制vc超时，进行关闭vc的操作时，对此值增量
  - closed 当本次处理遇到了原本就已经被设置为关闭状态的vc时，对此值增量
  - total_idle_time 累加此vc空闲状态的时间到此计数器，单位：秒
  - total_idle_count 当本次处理遇到vc空闲状态的时间超过1秒时，对此值增量

```
void
NetHandler::_close_vc(UnixNetVConnection *vc, ink_hrtime now, int &handle_event, int &closed, int &total_idle_time,
                      int &total_idle_count)
{
  if (vc->thread != this_ethread()) {
    return;
  }
  // 尝试对vc上锁
  MUTEX_TRY_LOCK(lock, vc->mutex, this_ethread());
  // 上锁失败则返回
  if (!lock.is_locked()) {
    return;
  }
  // 计算上一次 net_activity 执行时到当前时刻的时间差：diff值，单位：秒
  ink_hrtime diff = (now - (vc->next_inactivity_timeout_at - vc->inactivity_timeout_in)) / HRTIME_SECOND;
  if (diff > 0) {
    total_idle_time += diff;
    ++total_idle_count;
    NET_SUM_DYN_STAT(keep_alive_queue_timeout_total_stat, diff);
    NET_INCREMENT_DYN_STAT(keep_alive_queue_timeout_count_stat);
  }
  Debug("net_queue", "closing connection NetVC=%p idle: %u now: %" PRId64 " at: %" PRId64 " in: %" PRId64 " diff: %" PRId64, vc,
        keep_alive_queue_size, ink_hrtime_to_sec(now), ink_hrtime_to_sec(vc->next_inactivity_timeout_at),
        ink_hrtime_to_sec(vc->inactivity_timeout_in), diff);

  // 如果vc已经关闭，则调用close方法关闭vc
  if (vc->closed) {
    close_UnixNetVConnection(vc, this_ethread());
    ++closed;
  } else {
    // 如果vc没有关闭，则设置为超时状态，然后回调vc状态机按照超时vc处理
    vc->next_inactivity_timeout_at = now;
    // create a dummy event
    Event event;
    event.ethread = this_ethread();
    // 此处对mainEvent的回调，总是能满足mainEvent中对NetHandler，ReadVIO和WriteVIO的上锁要求
    // 因此这个回调总是以同步方式完成，因此使用临时变量event是安全的
    vc->handleEvent(EVENT_IMMEDIATE, &event);
    ++handle_event;
  }
}
```

## 参考资料

![How the InactivityCop works](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-002.svg)

- [UnixNet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)
