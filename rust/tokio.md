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

Trouble: when the future calls something that blocks the current worker OS thread, so it couldn't execute other futures. Don't pass future to tokio that runs long time without `await` points. 

Impirically, chuck >100ms should be run with `spawn_blocking` / use `block_inplace` 

`spawn_blocking` takes a closure, spawns it into the blocking thread pool of tokio runtime (starts a new blocking thread / reuse existing), returns a handle. Tokio can reuse the blocking threads for other closures. The handle a future that can be waited on. 

`block_in_place` can accept closure that is not `Send`. It is called on a tokio worker thread, that thread saves its state to be inherited by a new tokio worker thread, and the current thread becomes non-worker thread that can sync execute the closure without moving it. The benefit of doing this is the now non-worker thread would not block other futures.

Async has more book-keeping, but gets the benefit of using green threads. Works well if large number of tasks / either of the concurrent events.

### Shutdown

When all futures end, runtime yields back to the caller of `block_on`. Most times the caller awaits on the returned handle. 

`shutdown_background` will drop tasks when they yield after get called `poll` upon, whether `READY` or `PENDIND`.

If the program returns, the threads are killed and the futures never called `drop` for cleanup.

### Send Bound & LocalSet

To run futures that are not `Send`. `LocalSet` type is a set of tasks that are guaranteed to be run on the same thread, not sent between threads. `LocalSet::spawn_local` creates future for them. It could only be used as top-level task (on runtime's `block_on`, `run_until`, etc.) or start a new thread and send it via channel.

### Tokio Mutex

`lock` method of tokio mutex returns future. `std::Mutex` is more efficient. Reason why tokio mutex is necessary is: Waiting on the mutex blocks the worker thread, also await while holding the lock could take long time. `std::Mutex`''s mutex guard is also not `Send`, which makes the future that contains that not `Send`. 

tokio mutex is aware when a task calls `mutex.lock().await` if the lock is held by other task, in this case the requiring task is put to sleep and the worker thread picks up other tasks. (it yields when cannot acquire lock).

TBC: https://youtu.be/o2ob8zkeq2s?t=3419

## Resources

## Utils

## Complications
