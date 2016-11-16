# Block and Notification

There is exception for everything. Some syscalls will be called by SM. The syscall would be block and wait for condition.
It will cause block as well as not return to engine in time. For example: NetHandler calls epoll_wait().

We do not perform a syscall which would cause block if there is follow-up task of the engine otherwise we could.
During the block, We should break the block and return to engine once there is a new task.

We designed a notification mechanism in the engine to break the block and inform the SM that there are new tasks.

The Rules to implement a SM:

- Save current status and return to engine whenever there is block and waiting.
- Reschedule the SM to perform continuously in next loop
- If we required better performance from block, we should support notification mechanism in the SM and return to engine if needed.

I will introduce the components and implementation of event system in the following chapter.
