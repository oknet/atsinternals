# How to design a iocore sub-system

一个子系统至少包含三个部分：

- Handler
- Processor
- Task

## Handler

Handler 是继承自 class Continuation 的状态机，被 EventSystem 周期性回调或者被随时回调。其内部最少包含：

- 保存 Task 对象的队列
    - 一个原子队列及一个本地队列
    - 或者一个保护队列
- int Handler::dispatch(Task \*p)
    - 把 Task 放入 Handler 系统中，由 Handler 来执行对应的任务
    - 通常是直接把 Task 放入原子队列 并 设置 Task::h 指向该 Handler
    - 返回：成功为 0，失败时为 -errno
- int Handler::dispatch_local(Task *p)
    - 与 dispatch 的功能接近，区别是直接把 Task 放入了本地队列
    - 返回：成功为 0，失败时为 -errno
- void Handler::release(Task \*p)
    - 将 Task 从 Handler 中移除
    - 包括将 Task 从 Handler 的队列中删除 等
- void Handler::process_task(Task \*p)
    - 执行 Task 的任务
    - 根据执行的结果，或者 Task 内特定成员的值，决定
        - 是否需要调用 Handler::free_task() 
        - 是否需要将 Task 重新放回队列
        - 或不执行任何操作
- int Handler::mainEvent(int e, void \*p)
    - 被事件系统周期性回调, 遍历 Task 队列
    - 把原子队列导入本地队列
    - 遍历本地队列，取出Task
    - 处理每一个Task内的任务
        - 如果 Task 内的 task\_done 为 true，需要通过 Handler::free\_task() 来回收 Task 对象占用的资源
        - 否则，调用 Handler::precess\_task() 完成 Task 任务
    - 在处理完全部或者一定数量的Task之后，返回到EventSystem
- void Handler::free\_task()
    - 解除 Task 与 Handler 之间的关联(调用 Handler::release(Task \*p))
    - 最终调用 Task::free(t) 回收 Task 对象占用的资源

在 Handler 内，还需要定义一些帮助完成 Task 对象所描述的任务的处理函数，将 Task 携带的成员变量作为输入，通过这些处理函数，实现结果的输出，从而完成任务。


## Processor

Processor 向使用子系统的用户提供了一套API，至少包含以下功能：

- Task \* xxxProcessor::allocate\_task(t)
    - 用来创建一个 Task 对象并初始化
    - 或者通过 new Task 来完成创建 Task 对象
    - 返回：Task 对象
- Task \* xxxProcessor::dispatch(Continuation \* callbackSM, ...)
    - 创建一个 Task 对象，使用传入的 callbackSM 初始化 Task::action 对象，使用传入的其它参数初始化 Task
    - 找到一个适合的 Handler 并尝试对 Handler 上锁
    - 如果上锁失败，则把 Task 放入 Handler 的原子队列(调用 Handler::dispatch(task))
    - 如果上锁成功，则直接放入 Handler 的本地队列(调用 Handler::dispatch\_local(task))
    - 返回 Task 对象的指针
    - 通常适用于没有 pre-Handler 处理过程的情况

## Task

Task 继承自 Continuation，是执行任务的主体，至少包含：

- int task\_done;
    - 表示该 Task 对象的任务是否已经完成
    - 该值为 1 或者为 true 时，Handler 通常会回收该 Task 对象
- int enabled;
    - 表示该 Task 对象的任务是否被允许执行
    - 该值为 1 则 Handler 会执行其所描述的任务，如为 0 则 Handler 暂不执行
- Continuation \*cont;
    - 任务完成时，将回调 cont，并按照 Task 支持的方式传递事件类型和相关数据
- Ptr\<ProxyMutex\> cont\_mutex;
    - 通过 ```cont_mutex = cont->mutex;``` 将 cont->mutex 的引用计数加 1，保证 cont->mutex 不会随着 cont 一起被回收
    - 在回调 cont 之前，要对这个 cont\_mutex 先上锁
    - 必须在上锁之后，才可以从 task\_done 和 cont 读取到真实可靠的数据
- EThread \*thread;
    - 一个指向 EThread 对象的指针
    - 在 Task 对象分配时初始化, 表示此 Task 对象与该 EThread 绑定
    - 当需要将此 Task 放入 Handler 的队列时，需要选择该 EThread 中运行的 Handler 对象
- Handler \*h;
    - 一个指向 Handler 对象的指针
    - 当该指针不为空时，只有 Handler 可以释放该 Task 对象
    - 当该指针为空时，可以直接调用 Task::free(t) 释放该 Task 对象
- void Task::init(Continuation \*c, ...) 或者通过构造函数完成
    - 初始化 Task 对象
- void Task::free(EThread \*t)
    - 用来回收 Task 对象占用的资源
- void Task::clear()
    - 可选
    - 用来释放和清理 Task 内的各成员变量
    - 主要用来支持 freelist
    - 通常也会被 init() 调用
- void Task::dispatch(Continuation \*c, …)
    - 可选
    - 只要 Task 处于 Handler 处理过程，可以不对 Task 上锁而直接调用
    - 使用传入的其它参数重新设置必要的 Task 成员
    - 将当前 Task 放入 Task::h 指向的 Handler 对象的原子队列
- void Task::reDispatch()
    - 可选
    - 只要 Task 处于 Handler 处理过程，可以不对 Task 上锁而直接调用
    - 有时候 Task 可以重复执行，通过该方法实现重复多次执行
    - 重新将 Task 对象放入 Handler 的队列并返回
- void Task::pause()
    - 可选
    - 有时候 Task 是周期性执行, 通过该方法暂时停止该 Task 的执行，设置 Task::enabled = 0;
- void Task::resume()
    - 可选
    - 用来恢复被 pause() 的 Task，设置 Task::enabled = 1;
- void Task::finished(int err)
    - 可选
    - 只要 Task 处于 Handler 处理过程，可以不对 Task 上锁而直接调用
    - 当 Task 重复执行时，标记此 Task 已经执行完成，接下来将由 Handler 将 Task 对象回收

需要特别留意以下区别：

- Handler::dispatch(Task *p) 
    - 是将一个 Task 交由该 Handler 管理，或者说是绑定到该 Handler
    - 同时也将 Task 放入该 Handler 的队列里
    - 通常只需要调用一次
- Task::dispatch(Continuation \*c, …) 
    - 此时 Task 已经绑定到 Tash::h 这个 Handler
    - 此操作仅仅是将此 Task 放入 Handler 的队列里
    - 可重复调用
- Task::reDispatch()
    - 通常没有参数传入，不会修改任务内容
    - 用于重复执行 Task 任务


## Task 的生命周期

Task 对象从创建到回收，为了能够完成既定的任务，可以被分为以下三个过程：

- pre-Handler 处理过程
- Handler 处理过程 / in-Handler 处理过程
- post-Handler 处理过程

一个 Task 在交付给 Handler 之前，或从 Handler 归还之后，可能还需要一些其它的处理步骤，才能完成整个任务，对于不同的 sub-system 可能存在或者不存在这些步骤。

对于 pre-Handler 处理过程，需要：

- 在 Processor 定义与之对应的 API 函数，在 Task 定于与之对应的 状态函数 和 处理函数。
    - 处理函数 将会被 API 函数 和 状态函数 调用，
    - 通常这三个函数应该一起被定义。
- 在调用状态函数之前要对 Task 上锁。
- 在调用处理函数之前要对 Handler 上锁。
- 任何时候，对 Handler 上锁之后，就可以通过调用 Handler::dispatch(Task *p) 让 Task 进入到 Handler 处理过程。
- 当从 pre-Handler 处理过程进入到 Handler 处理过程时，需要以回调的方式，让 sub-system 的使用者知道。

在 pre-Handler 执行的过程里，可能会遇到各种错误，从而没有进行到 Handler 处理过程，因此为了让 sub-system 的使用者明确的知道一个 Task 是否进入到了 Handler 处理过程，需要：

- 在 pre-Handler 过程如果遇到错误无法进入到 Handler 处理过程时，
    - 向使用者回调 ERROR事件 和 错误信息
    - 然后将 Task 回收
- 任何时候，调用 Handler::dispatch(Task *p) 成功之后，
    - 都要向使用者回调 SUCCESS事件 和 Task 自身
    - 进入到 Handler 处理过程之后，Handler 负责在任务完成后将 Task 回收，除非进行到 post-Handler 处理流程


在 Processor 里可以提供与 pre-Handler() 函数对应的 API，由 sub-system 的使用者调用：

- Action \* Processor::doTask(Continuation \*callbackSM, …)
    - 可选
    - 创建一个 Task 对象，使用传入的 callbackSM 初始化 Task::action 对象，使用传入的其它参数初始化 Task
    - 选取一个 Handler 并对其上锁
    - 如果上锁成功，则直接调用 Task::preHandler()，然后回调 callbackSM，返回：ACTION\_RESULT\_DONE
    - 如果上锁失败，则设置与 Task::preHandler() 对应的 Task::preEvent(int e, void \*data) 作为状态处理函数，通过事件系统回调 Task 完成 pre-Handler 流程
    - 返回：Task 的 action 成员

在 ATS 中，回调分为同步和异步，参照 ATS 内部的通用方式，向调用者返回 Action 对象 或 ACTION\_RESULE\_DONE，因此需要在 Task 中增加 Action 对象：

- Action Task::action;
    - 一个 Action 对象，注意不是指针
    - 在 pre-Handle 任务完成时，将回调 Action 对象指向的状态机
    - 如该 Action 对象被取消，则不回调状态机，并调用 Task::free(t) 回收

对于 Handler / in-Handler 处理过程，

- 除了 Task::finished(), Task::dispatch(), Task::reDispatch() 之外，对 Task 的操作，都需要对 Task 所在的 Handler 上锁
- 当 sub-system 的使用者收到 Handler 的通知后，可以选择：
    - 通过 Task::reDispatch() 让 Task 重复执行，
    - 通过 Task::pause() 暂停 Task 执行，
    - 通过 Task::finished(int err) 将 Task 设置为处理完成，让 Handler 完成对 Task 的回收，
    - 通过 Handler::release(Task *p) 让 Task 从 Handler 处理过程进入到 post-Handler 处理过程。
- 在 Handler 处理过程中，可以随时调用 Task::finished(int err) 终止 Task 的执行，并让 Handler 回收 Task 对象。

对于 post-Handler 处理过程，

- 由 sub-system 的使用者发起，在 Handler 回调使用者时，使用者通过 Handler::release(Task *p) 将 Task 与 Handler 分离，让 Task 进入到 post-Handler 过程。
- 然后使用者将尝试对 Task 上锁，然后调用 Task::postHandler 或 将 Task::postEvent(int e, void \*data) 设置为状态函数，通过事件系统重新调度 Task。
- 最后总是由 sub-system 的使用者调用 Task::free(t) 来回收 Task。

## 共享 Handler 处理过程

当两种 Task 的 Handler 处理过程相同，但是 pre-Handler 和 post-Handler 处理过程不同时，可以合并两种 Task，此时：

- 一个 Task 会有
   - pre-red-Handler()，pre-blue-Handler() 等多个处理函数
   - redEvent(int e, void \*data)，blueEvent(int e, void \*data) 等多个状态函数
   - Processor::doRead(), Processor::doBlue() 等多个 API 函数
- 根据 Task 初始化时传入的信息不同，会从多个状态函数选择一个作为起始状态函数，

这样就可以让一个 Task 拥有不同的 pre-Handler 处理过程，但是共享同一个 Hander 处理过程。







