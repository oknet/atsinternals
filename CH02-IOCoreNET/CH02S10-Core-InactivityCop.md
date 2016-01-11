# 核心组件：InactivityCop 状态机

InactivityCop是对早期的超时机制的改进，这个改进降低了EventSystem的压力。

原来的超时控制完全依靠EventSystem的定时事件来实现，导致EventSystem的事件队列内容纳了大量的事件。

对于一个 NetVConnection 的两种超时类型，10K连接就有20K事件长期塞在EventSystem的内部队列里。

因此ATS或者是更早的Inktomi公司，重构了超时管理，提出了InactivityCop状态机，专门用来支持 NetVConnection 的两种超时。

## 定义

```
// 未定义此宏，表示使用新的超市管理机制：InactivityCop状态机
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
  InactivityCop(ProxyMutex *m) : Continuation(m), default_inactivity_timeout(0)
  {
    // 只有一个回调函数：check_inactivity
    SET_HANDLER(&InactivityCop::check_inactivity);
    REC_ReadConfigInteger(default_inactivity_timeout, "proxy.config.net.default_inactivity_timeout");
    Debug("inactivity_cop", "default inactivity timeout is set to: %d", default_inactivity_timeout);

    RecRegisterConfigUpdateCb("proxy.config.net.default_inactivity_timeout", update_cop_config, (void *)this);
  }

  int
  check_inactivity(int event, Event *e)
  {
    (void)event;
    ink_hrtime now = Thread::get_hrtime();
    NetHandler &nh = *get_NetHandler(this_ethread());

    Debug("inactivity_cop_check", "Checking inactivity on Thread-ID #%d", this_ethread()->id);
    // Copy the list and use pop() to catch any closes caused by callbacks.
    forl_LL(UnixNetVConnection, vc, nh.open_list)
    {
      if (vc->thread == this_ethread()) {
        nh.cop_list.push(vc);
      }
    }
    while (UnixNetVConnection *vc = nh.cop_list.pop()) {
      // If we cannot get the lock don't stop just keep cleaning
      MUTEX_TRY_LOCK(lock, vc->mutex, this_ethread());
      if (!lock.is_locked()) {
        NET_INCREMENT_DYN_STAT(inactivity_cop_lock_acquire_failure_stat);
        continue;
      }

      if (vc->closed) {
        close_UnixNetVConnection(vc, e->ethread);
        continue;
      }

      // set a default inactivity timeout if one is not set
      if (vc->next_inactivity_timeout_at == 0 && default_inactivity_timeout > 0) {
        Debug("inactivity_cop", "vc: %p inactivity timeout not set, setting a default of %d", vc, default_inactivity_timeout);
        vc->set_inactivity_timeout(HRTIME_SECONDS(default_inactivity_timeout));
        NET_INCREMENT_DYN_STAT(default_inactivity_timeout_stat);
      } else {
        Debug("inactivity_cop_verbose", "vc: %p now: %" PRId64 " timeout at: %" PRId64 " timeout in: %" PRId64, vc, now,
              ink_hrtime_to_sec(vc->next_inactivity_timeout_at), ink_hrtime_to_sec(vc->inactivity_timeout_in));
      }

      if (vc->next_inactivity_timeout_at && vc->next_inactivity_timeout_at < now) {
        if (nh.keep_alive_queue.in(vc)) {
          // only stat if the connection is in keep-alive, there can be other inactivity timeouts
          ink_hrtime diff = (now - (vc->next_inactivity_timeout_at - vc->inactivity_timeout_in)) / HRTIME_SECOND;
          NET_SUM_DYN_STAT(keep_alive_queue_timeout_total_stat, diff);
          NET_INCREMENT_DYN_STAT(keep_alive_queue_timeout_count_stat);
        }
        Debug("inactivity_cop_verbose", "vc: %p now: %" PRId64 " timeout at: %" PRId64 " timeout in: %" PRId64, vc, now,
              vc->next_inactivity_timeout_at, vc->inactivity_timeout_in);
        vc->handleEvent(EVENT_IMMEDIATE, e);
      }
    }

    // Cleanup the active and keep-alive queues periodically
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

## 参考资料

![How the InactivityCop works](https://cdn.rawgit.com/oknet/atsinternals/master/CH02-IOCoreNET/CH02-IOCoreNet-002.svg)

- [UnixNet.cc](http://github.com/apache/trafficserver/tree/master/iocore/net/UnixNet.cc)
