# 基础部件：Lock

由于I_Lock.h在2014年进行了重构，所以下文分析的ATS 6.0版本中包含的代码与早期版本可能不一样。

为了保证多线程操作对于共享资源的访问，ATS定义了以下几种锁的操作：

- 阻塞型上锁，阻塞直到完成上锁
   - SCOPED_MUTEX_LOCK
- 非阻塞型上锁，需要在调用后进行判断，有可能没有成功上锁
   - MUTEX_TRY_LOCK
   - MUTEX_TRY_LOCK_SPIN
   - MUTEX_TRY_LOCK_FOR 这个跟MUTEX_TRY_LOCK的功能一样，可能是开源时被阉割了
- 释放锁／解锁
   - MUTEX_RELEASE

## 定义

实际使用中，通过宏定义的方式来调用Lock。为什么通过此方式？（后面讨论）

每个宏都一个DEBUG版本，为了描述简便，下面的介绍中删除了DEBUG版本。

阻塞型上锁部分：

```
/**
  阻塞，直到获得对ProxyMutex的锁。

  这个宏将阻塞，直到获得对ProxyMutex的锁。
  调用此宏，将分配一个特殊的对象持有对ProxyMutex的锁，但是该对象只存在于函数或者区域的作用域中。
  例如，可以通过大括号来限定作用域。

  PS：换句话说，就是这个特殊的对象是一个局部变量。

  @param _l 给锁起个临时的名字
  @param _m 指向ProxyMutex的指针
  @param _t 当前EThread线程

*/
#define SCOPED_MUTEX_LOCK(_l, _m, _t) MutexLock _l(_m, _t)
class MutexLock
{
private:
  Ptr<ProxyMutex> m;

public:
  MutexLock(ProxyMutex *am, EThread *t) : m(am)
  {
    Mutex_lock(m, t);
  }

  ~MutexLock() { Mutex_unlock(m, m->thread_holding); }
};
```

非阻塞型上锁部分：

```
/**
  尝试获得对ProxyMutex的锁。

  这个宏将以非阻塞方式，尝试获得对ProxyMutex的锁。
  在使用此宏之后，你可以通过对_l这个锁的变量进行判断（true或者false），来确认是否获得锁。

  @param _l 给锁起个临时的名字
  @param _m 指向ProxyMutex的指针
  @param _t 当前EThread线程

*/
#define MUTEX_TRY_LOCK(_l, _m, _t) MutexTryLock _l(_m, _t)

/**
  尝试获得对ProxyMutex的锁。

  这个宏将按照指定的次数，多次尝试获得对ProxyMutex的锁。
  它通过一个快速的循环实现多次（'_sc'）尝试，因此有可能造成CPU资源的浪费，所以你需要小心的使用它。

  @param _l 给锁起个临时的名字
  @param _m 指向ProxyMutex的指针
  @param _t 当前EThread线程
  @param _sc 尝试的次数，必须大于0

*/
#define MUTEX_TRY_LOCK_SPIN(_l, _m, _t, _sc) MutexTryLock _l(_m, _t, _sc)

/**
  尝试获得对ProxyMutex的锁。

  这个宏将以非阻塞方式，尝试获得对ProxyMutex的锁。
  在使用此宏之后，你可以通过对_l这个锁的变量进行判断（true或者false），来确认是否获得锁。

  @param _l 给锁起个临时的名字
  @param _m 指向ProxyMutex的指针
  @param _t 当前EThread线程
  @param _c 将要被锁定的Mutex来自这个Continuation

*/
#define MUTEX_TRY_LOCK_FOR(_l, _m, _t, _c) MutexTryLock _l(_m, _t)
class MutexTryLock
{
private:
  Ptr<ProxyMutex> m;
  bool lock_acquired;

public:
  MutexTryLock(ProxyMutex *am, EThread *t) : m(am)
  {
    lock_acquired = Mutex_trylock(m, t);
  }

  MutexTryLock(ProxyMutex *am, EThread *t, int sp) : m(am)
  {
    lock_acquired = Mutex_trylock_spin(m, t, sp);
  }

  ~MutexTryLock()
  {
    if (lock_acquired)
      Mutex_unlock(m.m_ptr, m.m_ptr->thread_holding);
  }

  // Spin till lock is acquired
  // MutexTryLock内的阻塞Lock，具体用法后面有分析
  void
  acquire(EThread *t)
  {
    MUTEX_TAKE_LOCK(m.m_ptr, t);
    lock_acquired = true;
  }

  void
  release()
  {
    // generate a warning because it shouldn't be done.
    ink_assert(lock_acquired);
    if (lock_acquired) {
      Mutex_unlock(m.m_ptr, m.m_ptr->thread_holding);
    }
    lock_acquired = false;
  }

  bool
  is_locked() const
  {
    return lock_acquired;
  }
  const ProxyMutex *
  get_mutex()
  {
    return m.m_ptr;
  }
};
```

释放锁／解锁部分

```
/**
  释放ProxyMutex上的锁

  这个宏将释放ProxyMutex上的锁。
  _l 必须是已经成功上锁之后才可以调用这个宏。

  @param _l 之前已经完成上锁操作时，给锁设置的临时名字。

*/
#define MUTEX_RELEASE(_l) (_l).release()
```

## 使用

ATS的锁被设计为自动锁，你会看到在代码里很少看到使用MUTEX_RELEASE这个宏，但是明明在函数的开头调用了加锁的宏，这是什么原理呢？

可以在```class MutexLock```和```class MutexTryLock```的定义中看到，析构函数中都调用了：

- Mutex_unlock(m, m->thread_holding);
- Mutex_unlock(m.m_ptr, m.m_ptr->thread_holding);
- 这两种使用方法有区别吗？**

而且再看看上锁使用的宏，这些宏都是使用```class MutexLock```或```class MutexTryLock```来声明了一个临时的类实例，凡是临时变量总有作用域的限制，所以离开作用域的时候，就会调用析构函数，因此锁就自动释放了，所以就不需要显示释放。

而且我们可以使用成对的大括号```{ ... }```，非常方便的设置作用域，只要在作用域的开头上锁，离开作用域自然就解锁了。

### TryLock的特殊用法

当使用MUTEX_TRY_LOCK之后，发现没有获得锁，但是后面又需要上锁的情况如何处理？

通常情况可能会这样做：

- 显示解锁，然后重新调用SCOPED_MUTEX_LOCK来上锁

但是TryLock的设计，为这种特殊情况增加了一个方法acquire()

- 可以直接调用 _l.acquire(ethread) 实现阻塞，直到加锁成功
- 但是要小心使用

## DEPRECATED

在最后，有几个被标记为不建议使用的上锁方法，但是在代码中仍然有很多地方在使用他们

- 阻塞型上锁，阻塞直到完成上锁
   - MUTEX_TAKE_LOCK
   - MUTEX_TAKE_LOCK_FOR
- 非阻塞型上锁，需要在调用后根据返回值（true/false）进行判断，有可能没有成功上锁
   - MUTEX_TAKE_TRY_LOCK
   - MUTEX_TAKE_TRY_LOCK_FOR
   - MUTEX_TAKE_TRY_LOCK_FOR_SPIN
- 释放锁／解锁
   - MUTEX_UNTAKE_LOCK 或者 Mutex_unlock

前面介绍的加锁方法，实际上是：

  - 通过宏定义，创建了一个临时的对象
  - 对象的析构函数来调用解锁方法
  - 这样临时对象离开作用域的时候会被自动释放
  - 从而触发析构函数的执行，完成自动解锁

但是有些时候，没法使用上面的这种方式，例如下面这段代码：

```
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
```

可以看到上锁的时候，是在一个 if 语句的作用域里，如果用上面介绍的第一种方式来上锁，离开 if 语句的作用域就会自动解锁，这显然是不正确的，所以这时就需要使用第二种方法进行上锁，这样在离开 if 语句的作用域的时候就不会自动解锁，而是需要在最后进行显示解锁。

## 参考资料

- [I_Lock.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Lock.h)
- [ProxyMutex](CH01S03-Basic-ProxyMutex.zh.md)
