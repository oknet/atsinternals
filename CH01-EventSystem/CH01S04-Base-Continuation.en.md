# Base：Continuation

Technically, Continuation model is designed with Continuation as the bottom layer. This is a primitive technology. Microsoft improved this model to Co-Routine programming model(method).

What is Continuation ? In order to understand ATS source code, you have to know the Continuation first. I would like to share my opinion:

- It is a notepad to record the next todo task with a single line.
- The notepad is assigned to a worker and start working on the task in the notepad.
- A worker is allow to return the notepad in any time as long as they erase the complete task and write done the next todo task.
- The notepad will be assigned again once it returns by one worker.
- Tear up the notepad is a termination task cause there is no return of this notepad.
- A notepad is only assigned to one worker to avoid other worker works on the same task at the same time.

Derived class of Continuation:

- Generally, the derived class of Continuation is a state machine.
- An operation guide attached with the notepad, each step has its own code name which match an operational program.
- Code name indicate the todo task
- A worker finds the operational program in the operation guide with this code name.

Continuation is the foundation of asynchronous callback mechanism in multi-thread event programming.

- In ATS, Continuation is one of the most primary abstract structures.
- All advanced structures such as Action, Event, VC package the Continuation data sturcture.

Some information from the comments of ATS source code:

- The Continuation class represents the main abstraction mechanism used throughout the IO Core Event System to communicate its users the occurrence of an event.
- Continuations are typically subclassed in order to implement event-driven state machines.
- By including additional state and methods, continuations can combine state with control flow, and they are generally used to support split-phase, event-driven control flow.


## Define and Structure

Base class for all state machines to receive notification of events.

```
#define CONTINUATION_EVENT_NONE 0

#define CONTINUATION_DONE 0
#define CONTINUATION_CONT 1

typedef int (Continuation::*ContinuationHandler)(int event, void *data);

class Continuation: private force_VFPT_to_top
{
public:
  /**
    The current continuation handler function.

    The current handler should not be set directly. In order to
    change it, first aquire the Continuation's lock and then use
    the SET_HANDLER macro which takes care of the type casting
    issues.

  */
  ContinuationHandler handler;

  /**
    The Continuation's lock.

    A reference counted pointer to the Continuation's lock. This
    lock is initialized in the constructor and should not be set
    directly.

  */
  Ptr<ProxyMutex> mutex;

  LINK(Continuation, link);

  /**
    Receives the event code and data for an Event.

    This function receives the event code and data for an event and
    forwards them to the current continuation handler. The processor
    calling back the continuation is responsible for acquiring its
    lock.

    @param event Event code to be passed at callback (Processor specific).
    @param data General purpose data related to the event code (Processor specific).
    @return State machine and processor specific return code.

  */
  int handleEvent(int event = CONTINUATION_EVENT_NONE, void *data = 0) {
      return (this->*handler) (event, data);
  }

    Continuation(ProxyMutex * amutex = NULL);
};

inline Continuation::Continuation(ProxyMutex *amutex)
  : handler(NULL), mutex(amutex)
{
}

#define SET_HANDLER(_h) (handler = ((ContinuationHandler)_h))
#define SET_CONTINUATION_HANDLER(_c, _h) (_c->handler = ((ContinuationHandler)_h))
```

A Continuation is a lightweight data structure,

- It implements a single method ```int handleEvent(int , void *)```
   - handleEvent() is a interface for Processor and EThread.
   - It can be inerited to add additional states and methods.
- With which the user is called back ```ContinuationHandler handler```
   - which is invoked when events arrive.
   - To set/change with the following methods (two macros defined in the header file)：
      - SET\_HANDLER(\_h)
      - SET\_CONTINUATION\_HANDLER(\_c, \_h)

In ATS, we do not create the object of type Continuation directly but the instances of Continuation inheritance class, therefore there is no direct match of ClassAllocator objects.

## ProxyMutex object

Given the multithreaded nature of the Event System, every continuation carries a reference to a ProxyMutex object to protect its state and ensure atomic operations.

This ProxyMutex object must be allocated by continuation-derived classes or by clients of the IO Core Event System and it is required as a parameter to  the Continuation's class constructor.

## TSAPI

**Continuations** are instances of the opaque data type ``TSCont``.

Note: an opaque data type is a data type whose concrete data structure is not defined in an interface.


## 参考资料

- [I_Continuation.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/eventsystem/I_Continuation.h)

