# Basic: Lock

The following analysis base on ATS 6.0 which may be different from the previous versions cause there is reconstruction of I_Lock.h in 2014.

To ensure the access of sharing resources with multi-thread, ATS defines several methods to get a lock: 

- Blocked lock, blocks until the lock to the ProxyMutex is acquired.
   - SCOPED\_MUTEX\_LOCK
- Non-blocking lock, it needs to confirm whether it locks successfully or not after calling it.
   - MUTEX\_TRY\_LOCK
   - MUTEX\_TRY\_LOCK\_SPIN
   - MUTEX\_TRY\_LOCK\_FOR (The feature is the same as MUTEX_TRY_LOCK)
- Release lock/Unlock
   - MUTEX\_RELEASE

## Definition

Normally we always call locks with macro. (I will discuss the reason in the following chapters)

There is a DEBUG version of each macro. In order to descript it in simplification way, we removed the DEBUG version in this chapter.

Blocked lock:

```
/**
  Blocks until the lock to the ProxyMutex is acquired.

  This macro performs a blocking call until the lock to the ProxyMutex
  is acquired. This call allocates a special object that holds the
  lock to the ProxyMutex only for the scope of the function or
  region. It is a good practice to delimit such scope explicitly
  with '&#123;' and '&#125;'.

  @param _l Arbitrary name for the lock to use in this call
  @param _m A pointer to (or address of) a ProxyMutex object
  @param _t The current EThread executing your code.

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

Non-blocking lock:

```
/**
  Attempts to acquire the lock to the ProxyMutex.

  This macro attempts to acquire the lock to the specified ProxyMutex
  object in a non-blocking manner. After using the macro you can
  see if it was successful by comparing the lock variable with true
  or false (the variable name passed in the _l parameter).

  @param _l Arbitrary name for the lock to use in this call (lock variable)
  @param _m A pointer to (or address of) a ProxyMutex object
  @param _t The current EThread executing your code.

*/
#define MUTEX_TRY_LOCK(_l, _m, _t) MutexTryLock _l(_m, _t)

/**
  Attempts to acquire the lock to the ProxyMutex.

  This macro performs up to the specified number of attempts to
  acquire the lock on the ProxyMutex object. It does so by running
  a busy loop (busy wait) '_sc' times. You should use it with care
  since it blocks the thread during that time and wastes CPU time.

  @param _l Arbitrary name for the lock to use in this call (lock variable)
  @param _m A pointer to (or address of) a ProxyMutex object
  @param _t The current EThread executing your code.
  @param _sc The number of attempts or spin count. It must be a positive value.

*/
#define MUTEX_TRY_LOCK_SPIN(_l, _m, _t, _sc) MutexTryLock _l(_m, _t, _sc)

/**
  Attempts to acquire the lock to the ProxyMutex.

  This macro attempts to acquire the lock to the specified ProxyMutex
  object in a non-blocking manner. After using the macro you can
  see if it was successful by comparing the lock variable with true
  or false (the variable name passed in the _l parameter).

  @param _l Arbitrary name for the lock to use in this call (lock variable)
  @param _m A pointer to (or address of) a ProxyMutex object
  @param _t The current EThread executing your code.
  @param _c Continuation whose mutex will be attempted to lock.

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
  // I will introduce it in following chapters
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

Release lock/unlock:

```
/**
  Releases the lock on a ProxyMutex.

  This macro releases the lock on the ProxyMutex, provided it is
  currently held. The lock must have been successfully acquired
  with one of the MUTEX macros.

  @param _l Arbitrary name for the lock to use in this call (lock
    variable) It must be the same name as the one used to acquire the
    lock.

*/
#define MUTEX_RELEASE(_l) (_l).release()

// In the MutexLock::release() and the MutexTryLock::release(),
//   calls Mutex_unlock(m, t) to unlock the mutex.

inline void
Mutex_unlock(ProxyMutex *m, EThread *t)
{
  if (m->nthread_holding) {
    ink_assert(t == m->thread_holding);
    m->nthread_holding--;
    if (!m->nthread_holding) {
#ifdef DEBUG
      if (Thread::get_hrtime() - m->hold_time > MAX_LOCK_TIME)
        lock_holding(m->srcloc, m->handler);
#ifdef MAX_LOCK_TAKEN
      if (m->taken > MAX_LOCK_TAKEN)
        lock_taken(m->srcloc, m->handler);
#endif // MAX_LOCK_TAKEN
      m->srcloc  = SourceLocation(nullptr, nullptr, 0);
      m->handler = nullptr;
#endif // DEBUG
      ink_assert(m->thread_holding);
      m->thread_holding = 0;
      ink_mutex_release(&m->the_mutex);
    }
  }
}

inline void
Mutex_unlock(Ptr<ProxyMutex> &m, EThread *t)
{
  Mutex_unlock(m.get(), t);
}
```

## Usage

ATS locks are designed as automatic locks. Although it quotes this macro at the beginning of the function, you barely see MUTEX_RELEASE in the code. How it works?


In these destructors of ```class MutexLock``` and ```class MutexTryLock```, the Mutex\_unlock() is called:

- Mutex\_unlock(m, m->thread\_holding);
- Mutex\_unlock(m.m\_ptr, m.m\_ptr->thread\_holding);

Expands the locking macros, I found they declare a temporary class instance of ```MutexLock``` or ```MutexTryLock```. There are valid scopes of all the temporary variables. Due to the lock release automatically without the valid scope, we don't release it manually.

It is very easy to set the valid scope with ```{``` and ```}```, the lock releases outside the valid scope.

### About the MutexTryLock::acquire

How to acquire a lock in blocking mode when you fail to acquire lock with MUTEX\_TRY\_LOCK ?

Generally we will try the following ways:

- Call MUTEX\_RELEASE(\_l) to release the temporary lock object
- And call SCOPED\_MUTEX\_LOCK to acquire a lock in blocking mode

The ```MutexTryLock``` introduced a new method ```acquire()``` to this particular case.

- Call ```_l.acquire(ethread)``` directly to block until the lock to the ProxyMutex is acquired.
- Please careful use it

## DEPRECATED

The above methods are actually:

  - Create a temporary lock object through a macro
  - Object's destructor calls the unlock method
  - The destructor function is called when the lock object leave the valid scope
  - This is automatic lock that is always automatically unlocked.

But these methods don't apply for the followings:

```
        bool plugin_lock = false;
        APIHook *hook    = this->api_current;
        Ptr<ProxyMutex> plugin_mutex;

        if (hook->m_cont->mutex) {
          plugin_mutex = hook->m_cont->mutex;
          // The mutex is locked here
          plugin_lock = MUTEX_TAKE_TRY_LOCK(hook->m_cont->mutex, mutex->thread_holding);
          if (!plugin_lock) {
            SET_HANDLER(&ProxyClientSession::state_api_callout);
            mutex->thread_holding->schedule_in(this, HRTIME_MSECONDS(10));
            return 0;
          }
          // The mutex is automatically unlocked if we acquire the lock by MUTEX_TRY_LOCK
          // Now the mutex is not automatically unlocked since we acquire the lock by MUTEX_TAKE_TRY_LOCK.
        }

        this->api_current = this->api_current->next();
        hook->invoke(eventmap[this->api_hookid], this);

        if (plugin_lock) {
          // Has to unlick manually since it is not a automatic lock.
          Mutex_unlock(plugin_mutex, this_ethread());
        }
```

There are several un-recommended locking methods still in use.

- Block lock, Block it until lock it
   - MUTEX\_TAKE\_LOCK
   - MUTEX\_TAKE\_LOCK\_FOR
- Non-blocking lock, required to check the return value (true/false) to confirm whether or not lock it successfully
   - MUTEX\_TAKE\_TRY\_LOCK
   - MUTEX\_TAKE\_TRY\_LOCK\_FOR
   - MUTEX\_TAKE\_TRY\_LOCK\_FOR\_SPIN
- Release lock/unlock
   - MUTEX\_UNTAKE\_LOCK
   - Mutex\_unlock


## 参考资料

- [I_Lock.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/eventsystem/I_Lock.h)
- [ProxyMutex](CH01S03-Basic-ProxyMutex.en.md)

