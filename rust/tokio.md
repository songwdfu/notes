# Decrusting Tokio Crate

## Tokio Runtime

Takes the future, returns the result

### Scheduler

Pick a future and calls its `poll` method that returns either `READY` or `PENDING`. When `PENDING`, the future is put back to the runtime to be polled again.

Multi-thread scheduler returns 1 OS thread per CPU core.  Current-thread scheduler uses cur thread.

`Runtime::block_on` interface takes a future and return the execution result. 

`Runtime::spawn` only puts the future onto queue of tasks of runtime and returns a handle. The handle can be used to await or abort the passed future

Every future that is `spawn`ed (put onto the queue) becomes a task. They could contain more futures internally that are not tasks. tokio executor only knows tasks but not nested futures.

Only when `block_on` is called, tokio execute tasks on the queue, polling ALL the tasks it has on the queue until THE ONE passed to `block_on` is `READY`

Each thread has local queue, loops and execute futures on their queue. When empty pull from global queue or steal from other threads'queues. It is a stealing scheduler! This requires future to be `Send`

std::thread executes sync closure, tokio::runtime::spawn executes async future.

Every runtime has two queues (which are thread-safe linkedlist of tasks): runnable and non-runnable tasks, only choose and poll runnable queue. tasks in runnable queue is believed to be able to make progress. Tasks are moved between two queues. when thread does not have anything runnable, it is parked and only unparked when sth becomes runnable.

OS CPU scheduler also impl work stealing: CPU core pulls pending thread, so OS level threads can move between CPU threads.

Having one thread per task is expensive bc context switch is kernel space. In executor one thread pickup multiple tasks, similar to grenn threading.

### Blocking

TO BE CONTINUED: https://youtu.be/o2ob8zkeq2s?t=1753

## Resources

## Utils

## Complications
