# Core: ClassAllocator

ClassAllocator is a C++ class template which inherits from **Allocator**

- Add new 3 methods: alloc(), free(), free_bulk()
   - Input parameter is the type of ```Class *``` to adapt Constructor.
- ```element_size = sizeof(Class)``` is the major difference between ClassAllocator and Allocator

The system allocates memory to all the data structure running in ATS, specially the data transfer between EventSystem and many kinds of SM.

ATS allocates memory dynamically to each incoming session and releases it after the session ends. It is highly frequency operaiton of the memory.

In the session, various memory spaces are allocated/released to different objects which cause seirously fragmentation of the memory.

**Allocator** is designed for fixed size memory blocks which doesn't comply with the object oriented design, therefore ATS introduce a new template **ClassAllocator**.

Use the ClassAllocator template to define a global instance for the specified data structure type (class C), which has a ```proto``` structure:

```
template <class C> class ClassAllocator : public Allocator
{
...
  struct {
    C typeObject;
    int64_t space_holder;
  } proto;
  uint16_r  proxy_index;
};
```

The member ```proto.typeObject``` is declared as a ```class C``` type.

A memory block is pre-allocated by the constructor and split into small pieces. The number of the pieces is ```chunk_size``` and the size of pieces is ```sizeof(C)``` bytes. The pieces are linked to the **free pool** (```InkFreeList *fl```) one by one.

The ```ClassAllocator::alloc()``` pick up a piece from the **free pool** and ```ClassAllocator::free()` return a piece to it.

This reduces the allocation of memory directly from the operating system and fragmentation of memory.

## Methods

Within alloc(), it copy ```proto.typeObject``` directly to the new piece as below:

```
  /** Allocates objects of the templated type. */
  C * 
  alloc()
  {
    void *ptr = ink_freelist_new(this->fl);

    memcpy(ptr, (void *)&this->proto.typeObject, sizeof(C));
    return (C *)ptr;
  }
``` 
The constructor of ```class C``` is not called.

Within free(), the piece returns directly to the freelist:

```
  /**
    Deallocates objects of the templated type.

    @param ptr pointer to be freed.
  */
  void
  free(C *ptr)
  {
    ink_freelist_free(this->fl, ptr);
  }
```

The destructor of ```class C``` is not called.

Both the constructor and the destructor are called on ```proto.typeObject```.

## How to use

To declare a allocator for a specify type with ClassAllocator template:

- Do not initial a ptr object within constructor
   - For example: initial a ptr with new keyword, initial a ptr by malloc, etc...
   - It is safe to initial a ptr with global object.
   - Generally, a C::init() is defined to initial ptr object.
- Do nothing within destructor
   - Generally, a C::clear() is defined to release the resource.
   - and C::destroy() calling C::clear() first and then ClassAllocator::free(this) to do the same thing like destructor.
- If there is a sub object
   - see above

After [TS-4007](https://issues.apache.org/jira/browse/TS-4007) merged,

- The destructor is nolonger called on ```proto.typeObject``` even it is defined.
- The constructor is still called on ```proto.typeObject```.

```
commit e4cb305315e4969748c9dc8414fd96c8d36b4091
Author: Can Selcik <cselcik@linkedin.com>
Date:   Mon Nov 9 22:29:37 2015 -0800

    TS-4007: ClassAllocator: don't attempt to destruct the prototype object when ClassAllocator goes out of scope

diff --git a/lib/ts/Allocator.h b/lib/ts/Allocator.h
index 3b489a7..a21ff90 100644
--- a/lib/ts/Allocator.h
+++ b/lib/ts/Allocator.h
@@ -40,6 +40,7 @@
 #ifndef _Allocator_h_
 #define _Allocator_h_
 
+#include <new>
 #include <stdlib.h>
 #include "ts/ink_queue.h"
 #include "ts/ink_defs.h"
@@ -193,11 +194,12 @@ public:
   */
   ClassAllocator(const char *name, unsigned int chunk_size = 128, unsigned int alignment = 16)
   {
+    ::new ((void*)&proto.typeObject) C();
     ink_freelist_init(&this->fl, name, RND16(sizeof(C)), chunk_size, RND16(alignment));
   }
 
   struct {
-    C typeObject;
+    uint8_t typeObject[sizeof(C)];
     int64_t space_holder;
   } proto;
 };
```

### Define a gloabl allocator

```
ClassAllocatror<NameOfClass> nameofclassAllocator("nameofclassAllocator", 256)
```
The 1st param is the name of memory block,
The 2nd param is the number of pieces in the memory block, the **256** means the memory block could hold 256 ```class NameOfClass``` objects.

### Allocating

```
NameOfClass *obj = nameofclassAllocator.alloc();
obj->init(...);
```
The obj returned from alloc() has its constructor called. The ```obj-> init()`` is used to initialize the ptr object.

### releasing

```
obj->clear();
nameofclassAllocator.free(obj);
obj = NULL;
```
The ```obj->clear()``` is used to release the ptr object.

if there is ```obj->destroy()```:

```
obj->destroy();
obj = NULL;
```

## Reference

- [Allocator.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/Allocator.h)

# Base: Allocator

Allocator is internal memory management technology of ATS, mainly used to reduce memeory allocation times to improve performance.

In additional, store the same size of memory blocks continuously to reduce the memory fragmentation.

## Structure

Allocator is base class of all ClassAllocator:

- **free pool**
   - The pool has a name that sets in constructor
   - Includes at least 1 memory unit
   - Unit size: element\_size * chunk\_size
   - The Unit contains chunk\_size numbers of element\_size bytes of memory
   - The default chunk\_size is 128
- There are 3 methods
   - alloc\_void calls ink\_freelist\_new to acquire a fixed size memory block from **free pool**
   - free\_void calls ink\_freelist\_free to return a memory block to **free pool**
   - free\_void\_bulk calls ink\_freelist\_free\_bulk to return memory blocks to **free pool**
- The constructor calls ink\_freelist\_init to create a **free pool**

## Reference

- [Allocator.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/Allocator.h)
- [ink_queue.h](http://github.com/apache/trafficserver/tree/6.0.x/lib/ts/ink_queue.h)

