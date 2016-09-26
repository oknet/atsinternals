# Introduction

One of the core componets of Apache Traffic Server is Event Driven Engine(Event System), which accomplish external interactive with event. There are several events processed by multiple thread (EThread) in the engine simultaneously. EThread is created and managed by a singleton processor named eventProcessor.

Let's image this is a straight-six(or more) engine but the power is Event instead of petrol(gasoline).

```
+-------------------+
|   eventProcessor  |
+---+---+---+---+---+
| E | E | . | E | E |
| T | T | . | T | T |
| h | h | . | h | h |
| r | r | . | r | r |
| e | e | . | e | e |
| a | a | . | a | a |
| d | d | . | d | d |
+---+---+---+---+---+
```

## Event System

It consists of three parts:

- Base
   - ProxyMutex
   - Continuation
   - Lock
- Core
   - ClassAllocator (Global object allocator)
      - Inherited from the Allocator base class
      - Used to allocate memory for various types of object
   - ProxyAllocator (Thread local object allocator)
      - implement by freelist
   - EThread (run / process Event)
      - Inherited from Thread base class
   - Event (Signal / command to the engine)
      - Inherited from Action base class
- Interface
   - eventProcessor
      - Inherited from Processor base class
      - Singleton, create EThread group / pool for the engine

In the next chapter, I will descript the mechanism of the Event System generally then I will analysis the source code comprehensively from the directory iocore/eventsystem.

