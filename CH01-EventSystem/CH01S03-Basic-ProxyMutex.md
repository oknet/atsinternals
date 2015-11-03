# 基础部件：ProxyMutex

ProxyMutex 是在Continuation和Thread中使用的互斥锁。

在整个事件系统中（Event System），ProxyMutex是最基本的同步对象。

它是一个引用计数的对象，提供对资源的互斥访问保护。

由于事件系统是多线程的模式，通过ProxyMutex可以保护数据结构和状态信息，否则将会受到并发线程的影响。

ProxyMutex对象有一个ink_mutex类型（在ink_mutex.h中定义）的成员，它是对不同平台的互斥锁类型的封装。

ProxyMutex通过ink_mutex来实现互斥锁的功能，这样就没有特定平台的函数调用的负担。

ProxyMutex也有一个指针，指回到当前持有锁的EThread，用于验证它是否正确的被释放。

## 定义／方法／成员

ProxyMutex 类主要是对底层mutex的封装，用来支持跨越多个平台的mutex系统。

在创建一个ProxyMutex对象时，首选的方式是使用new_ProxyMutex()函数。

```
class ProxyMutex : public RefCountObj
{
public:
  // 内部互斥对象
  ink_mutex the_mutex;

  // 指向持有该锁的EThread
  volatile EThreadPtr thread_holding;

  // 加锁、解锁计数器
  // 每加锁一次加1，解锁一次减1，但是减为0时才真正解锁
  int nthread_holding;

  // 在ATS中不要直接调用此方法来释放资源，请参阅后面的详细内容
  void free();

  /**
    构造函数 - 请调用 new_ProxyMutex() 来代替。

    ProxyMutex 对象的构造函数。初始化对象的状态，但是不对mutex成员进行初始化，
    在使用之前可以通过init()对mutex进行初始化。请不要使用这个构造函数，创建一个
    ProxyMutex 实例的最佳方式是通过 new_ProxyMutex() 函数，同时提供了更快的
    内存分配策略。
  */
  ProxyMutex()
  {
    thread_holding = NULL;
    nthread_holding = 0;
  }

  /**
    初始化内部 mutex 对象。

    在 ProxyMutex 对象构造完成后，使用这个方法初始化内部 mutex 对象
    同时也可以对该对象赋予一个名字（某些平台可能不支持）。
  */
  void
  init(const char *name = "UnnamedMutex")
  {
    ink_mutex_init(&the_mutex, name);
  }
};
```

## new_ProxyMutex 函数

该函数通过内存池，快速分配一个空间给ProxyMutex对象，执行构造函数，然后再执行init()函数。

```
// TODO：支持可选的mutex名称，在调用init()时传入
/**
  创建一个新的 ProxyMutex 对象。

  这时一个创建ProxyMutex对象的首选方法。它可以快速的分配对象占用的内存。
*/
inline ProxyMutex *
new_ProxyMutex()
{
  ProxyMutex *m = mutexAllocator.alloc();
  m->init();
  return m;
}
```

## ProxyMutex的使用方法

在一个class里定义ProxyMutex类型时，通常定义为一个Ptr指针：

```
Ptr<ProxyMutex> mutex_var_name;
```

在分配这个class的实例之后，如果必须要为它指定一个mutex，可以通过以下方法：

```
// 分配一个新的mutex
classObj->mutex_var_name = new_ProxyMutex();
// 与某个Continuation共享同一个mutex
classObj->mutex_var_name = Cont->mutex;
```

当我们不再需要mutex时，例如classObj需要被释放时，只需要简单的将mutex指向NULL，Ptr将自动判断是否可以释放这个mutex占用的内存空间。

千万不要调用mutex.free()方法来释放Ptr\<ProxyMutex\> mutex。

在ATS的实现中，几乎所有的ProxyMutex都是采用Ptr\<ProxyMutex\>方式定义的。

```
classObj->mutex_var_name = NULL;
```

## 获得锁/释放锁：

与ProxyMutex类一同提供了几个宏，可以让您锁定/解锁内部的互斥对象。

这部分将会在Lock部分介绍。

## Ptr模版 与 RefCountObj类

RefCountObj是一个引用计数类，ProxyMutex继承自这个类，那么ProxyMutex的实例mutex就内置了一个计数器，初始值为0。

通过Ptr\<ProxyMutex\>声明的实例，由Ptr模版对ProxyMutex类再次封装，其实例内部有一个m_ptr指针，类型为ProxyMutex。

同时Ptr模版的定义中，对赋值操作符“＝”进行了重载，来实现接受ProxyMutex类型的赋值，同时对==, != 比较操作也做了重载。

### 操作符＝（赋值）重载函数分析

PtrMutex ＝ p的过程如下：

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

首先我们分配两个ProxyMutex实例以及一个智能指针Ptr\<ProxyMutex\> PtrMutex

- Ptr\<ProxyMutex\> PtrMutex;
- ProxyMutex *p=new_ProxyMutex();
- ProxyMutex *m=new_ProxyMutex();

首先我们将p赋值给它：

- PtrMutex.m_ptr=p
- PtrMutex.m_ptr 的引用计数加1，其实就是p的引用计数为1了
- 返回PtrMutex

然后我们再将m赋值给它：

- temp_ptr暂存之前的p
- PtrMutex.m_ptr=m
- PtrMutex.m_ptr 的引用计数加1，其实就是m的引用计数为1了
- temp_ptr 的引用计数减1，其实就是p的引用计数减1，变为0了
- 判断temp_ptr的引用计数是否为0，为0，调用temp_ptr->free()，p被释放回收了。
- 返回PtrMutex

然后我们再将NULL赋值给它

- temp_ptr暂存上面的m
- PtrMutex.m_ptr=NULL，为NULL，那么引用计数不能增加，所以m的引用计数不变，仍然为1
- temp_ptr 的引用计数减1，其实就是m的引用计数减1，变为0了
- 判断temp_ptr的引用计数是否为0，为0，调用temp_ptr->free()，m被释放回收了。
- 返回PtrMutex

如果是 PtrMutex1=PtrMutex2 这样的操作呢？

- 转化为PtrMutex1 ＝ PtrMutex2.m_ptr
- 参考上面的流程就可以了

### RefCountObj的定义

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

### Ptr的定义

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

// 操作符 ＝ 的重载
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
template <class T> inline Ptr<T> &Ptr<T>::operator=(const Ptr<T> &src)
{
  return (operator=(src.m_ptr));
}
```


## 参考资料

- [I_Lock.h]
(http://github.com/apache/trafficserver/tree/master/iocore/eventsystem/I_Lock.h)
- [Ptr.h]
(http://github.com/apache/trafficserver/tree/master/lib/ts/Ptr.h)
