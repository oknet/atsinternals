# Base: ProxyMutex

The ProxyMutex is lock object used in continuations and threads.

The ProxyMutex class is the main synchronization object used throughout the Event System. 

It is a reference counted object that provides mutually exclusive access to a resource.

Since the Event System is multithreaded by design, the ProxyMutex is required to protect data structures and state information that could otherwise be affected by the action of concurrent threads.

A ProxyMutex object has an ink_mutex member (defined in ink_mutex.h) which is a wrapper around the platform dependent mutex type. 

This member allows the ProxyMutex to provide the functionallity required by the users of the class without the burden of platform specific function calls.

The ProxyMutex also has a reference to the current EThread holding the lock as a back pointer for verifying that it is released correctly.

## Definition / Method / Member

ProxyMutex class is a wrapper around the platform dependent mutex type.

You have to use new_ProxyMutex() to create a ProxyMutex object instead of "new ProxyMutex".

```
class ProxyMutex : public RefCountObj
{
public:
  /**
    Underlying mutex object.

    The platform independent mutex for the ProxyMutex class. You
    must not modify or set it directly.

  */
  ink_mutex the_mutex;

  /**
    Backpointer to owning thread.

    This is a pointer to the thread currently holding the mutex
    lock.  You must not modify or set this value directly.

  */
  volatile EThreadPtr thread_holding;

  // The counter to implement ReentrantLock.
  //   Increase when locked, Decrease when unlocked.
  //   Mutex is truly released when it becomes 0.
  int nthread_holding;

  // DO NOT directly call this method to release a ProxyMutex object.
  // Please refer to the following details.
  void free();

  /**
    Constructor - use new_ProxyMutex() instead.

    The constructor of a ProxyMutex object. Initializes the state
    of the object but leaves the initialization of the mutex member
    until it is needed (through init()). Do not use this constructor,
    the preferred mechanism for creating a ProxyMutex is via the
    new_ProxyMutex function, which provides a faster allocation.

  */
  ProxyMutex()
  {
    thread_holding = NULL;
    nthread_holding = 0;
  }

  /**
    Initializes the underlying mutex object.

    After constructing your ProxyMutex object, use this function
    to initialize the underlying mutex object with an optional name.

    @param name Name to identify this ProxyMutex. Its use depends
      on the given platform.

  */
  void
  init(const char *name = "UnnamedMutex")
  {
    ink_mutex_init(&the_mutex, name);
  }
};
```

## Function: new_ProxyMutex()

The new_ProxyMutex() function

- Allocate ProxyMutex object by mutexAllocator and "Call" ProxyMutex::ProxyMutex()
- Call ProxyMutex::init()

The mutexAllocator is a global instant of Allocator.

- There is an ProxyMutex in mutexAllocator.
- It allocate memory space for new ProxyMutex 
- It copies data from internal object to overwrite the memory space to replace construct function.

```
// TODO should take optional mutex "name" identifier, to pass along to the init() fun
/**
  Creates a new ProxyMutex object.

  This is the preferred mechanism for constructing objects of the
  ProxyMutex class. It provides you with faster allocation than
  that of the normal constructor.

  @return A pointer to a ProxyMutex object appropriate for the build
    environment.

*/
inline ProxyMutex *
new_ProxyMutex()
{
  ProxyMutex *m = mutexAllocator.alloc();
  m->init();
  return m;
}
```

## How to use ProxyMutex

Declare a ProxyMutex member with Ptr template in a class.

```
Ptr<ProxyMutex> mutex_var_name;
```

After allocating a object of the class, specify a mutex for the object by the following method:

```
// Allocate a new mutex
classObj->mutex_var_name = new_ProxyMutex();
// Or share a existed mutex from other Continuation
classObj->mutex_var_name = Cont->mutex;
```

Just simply set mutex to NULL to destroy a ProxyMutex object.
Ptr will automatically determine whether to release the memory space taken by the mutex.

DO NOT call mutex.free() to release a Ptr<ProxyMutex> mutex object.
In ATS, almost all ProxyMutex are defined with Ptr<ProxyMutex>. It will be destroyed automatically by Ptr template.

```
classObj->mutex_var_name = NULL;
```


## Acquiring/Releasing locks

Included with the ProxyMutex class, there are several macros that allow you to lock/unlock the underlying mutex object.

These macros will be introduced in CH01-Base-Lock Section.


## Ptr template and RefCountObj class

RefCountObj is a base class to enable the Reference-Counted feature for its derived class.
ProxyMutex inherits from it and there is a reference counter (the initial value is 0) inside of ProxyMutex.

Once an object is declared with Ptr<ProxyMutex>, a member m_ptr pointer is declared as ProxyMutex inside of it.

And the Ptr template overload Set, Get, Equal("==") and Not Equal("!=") operator to apply these operators to "ProxyMutex *m_ptr" transparently.

### Set operator overload analysis

I'm try to explain the Set operator overload with a simple example:

  PtrMutex ＝ p;

```
temp_ptr = PtrMutex->m_ptr;
if (m_ptr != p) {
    PtrMutex->m_ptr = p;
    if (PtrMutex->m_ptr) 
        PtrMutex->m_ptr->refcount_inc();
    if (temp_ptr && temp_ptr->refcount_dec() == 0) 
        temp_ptr->free();
}
return(PtrMutex);
```

A slightly more complex example:

First，we create two ProxyMutex instants and one Ptr<ProxyMutex> instant

- Ptr\<ProxyMutex\> PtrMutex;
- ProxyMutex *p=new_ProxyMutex();
- ProxyMutex *m=new_ProxyMutex();

Set "p" to PtrMutex (PtrMutex = p):

- PtrMutex.m_ptr=p
- Increase refcount for PtrMutex.m_ptr which means the refcount of p is 1.
- return with PtrMutex

Then, Set "m" to PtrMutex (PtrMutex = m):

- Save m_ptr to temp_ptr which means save p to temp_ptr
- PtrMutex.m_ptr=m
- Increase refcount for PtrMutex.m_ptr which means the refcount of m is 1.
- Decrease refcount for temp_ptr which means the refcount of p is reduced by 1 and is 0 now.
- If the refcount of temp_ptr is 0, call temp_ptr->free(). It is destroy & free p.
- return with PtrMutex

At last, Set NULL to PtrMutex (PtrMutex = NULL):

- Save m_ptr to temp_ptr which means save m to temp_ptr
- PtrMutex.m_ptr=NULL, we do not change the refcount of m_ptr due to the m_ptr is NULL
- Decrease refcount for temp_ptr which means the refcount of m is reduced by 1 and is 0 now.
- If the refcount of temp_ptr is 0, call temp_ptr->free(). It is destroy & free m.
- return with PtrMutex

If it is PtrMutex1=PtrMutex2 ?

- It will be transformed to "PtrMutex1 ＝ PtrMutex2.m_ptr"
- Then referring to the above flow

### Definition of RefCountObj

RefCountObj is atomic version for reference counting, the non-atomic version is named NonAtomicRefCountObj.

```
class RefCountObj : public ForceVFPTToTop
{
public:
  RefCountObj() : m_refcount(0) {}
  RefCountObj(const RefCountObj &s) : m_refcount(0)
  {
    (void)s;
    return;
  }
  virtual ~RefCountObj() {}
  RefCountObj &operator=(const RefCountObj &s)
  {
    (void)s;
    return (*this);
  }

  int refcount_inc();
  int refcount_dec();
  int refcount() const;

  virtual void
  free()
  {
    delete this;
  }

  volatile int m_refcount;
};
```

### Definition of Ptr template

The Ptr template also has a non-atomic version that is named NonAtomicPtr.

```
template <class T> class Ptr
{
public:
  explicit Ptr(T *p = 0);
  Ptr(const Ptr<T> &);
  ~Ptr();

  void clear();
  Ptr<T> &operator=(const Ptr<T> &);
  Ptr<T> &operator=(T *);

  T *
  to_ptr()
  {
    if (m_ptr && m_ptr->m_refcount == 1) {
      T *ptr = m_ptr;
      m_ptr = 0;
      ptr->m_refcount = 0;
      return ptr;
    }
    return 0;
  }
  operator T *() const { return (m_ptr); }
  T *operator->() const { return (m_ptr); }
  T &operator*() const { return (*m_ptr); }

  int operator==(const T *p) { return (m_ptr == p); }
  int operator==(const Ptr<T> &p) { return (m_ptr == p.m_ptr); }
  int operator!=(const T *p) { return (m_ptr != p); }
  int operator!=(const Ptr<T> &p) { return (m_ptr != p.m_ptr); }

  RefCountObj *
  _ptr()
  {
    return (RefCountObj *)m_ptr;
  }

  T *m_ptr;
};

// Set overload for PtrObj = obj
template <class T> inline Ptr<T> &Ptr<T>::operator=(T *p)
{
  T *temp_ptr = m_ptr;

  if (m_ptr == p)
    return (*this);

  m_ptr = p;

  if (m_ptr != 0) {
    _ptr()->refcount_inc();
  }

  if ((temp_ptr) && ((RefCountObj *)temp_ptr)->refcount_dec() == 0) {
    ((RefCountObj *)temp_ptr)->free();
  }

  return (*this);
}

// Set overload for PtrObj1 = PtrObj2
// It is transformed to PtrObj1 = PtrObj2.m_ptr;
template <class T> inline Ptr<T> &Ptr<T>::operator=(const Ptr<T> &src)
{
  return (operator=(src.m_ptr));
}
```


## Reference

- [I_Lock.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Lock.h)
- [Ptr.h]
(http://github.com/apache/trafficserver/tree/master/lib/ts/Ptr.h)
