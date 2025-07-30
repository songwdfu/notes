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

tokio mutex guard is `Send`, tokio keep the info that a future holds a mutex. When other future try holding this lock, it is put to sleep.

Green thread is similar to a task described here. `Spawn`ing a task is similar to spawning a green thread.

[Tokio console](https://github.com/tokio-rs/console) visualizes tasks statistics on time spent busy / scheduling / idle, number of time polled, info on task blocking runtime, etc. 

Stealing across multi-core CPU NUMA node's NUMA boundaries could be expensive.

## Resources

Tokio provide resources for I/O for tasks to interact with real world.

`AsyncRead` trait defines things that could be read from in async, e.g. TPC stream, fds, etc. These resources are always found at the bottom of future stack. Tokio provide these interfaces to return a `Poll<Result()>`. 

A `AsyncRead` returns `PENDING` when the underlying resource is not ready, and only wakes up when the underlying resource makes progress. This is achieved with the `cx: &mut Context` (that is also found in `Future` trait) that is passed all-the-way from the top of the future stack to the bottom when called `await`. `Context` has a `Waker` whose `wake` method is called when the future is allowed to make progress again.

Who calls `wake`? The runtime has a I/O event loop service that stores the `Waker` of a `AsyncRead` along with its file descriptor inside it. When the an event of a resource happens it calls `wake` method of the corresponding `Waker` to trigger moving the corresponding top-level task (that shares the same `Context`) to scheduler's the runnable queue. i.e. `Future::poll` is only called after `Future::cx::Waker::wake` is called.

`Waker` is created on-the-fly when the top-level task's `poll` is called. It is cheap to create, just a V-table (virtual dispatch table, list of function ptrs). It stores a ptr to the task, a function pointer to the rust code when `wake` is called. 

`Waker` a struct, not a trait because `Waker` is `Clone`, which is not object-safe, because it names the `Self` type, then `Box<dyn Waker>` or `Box<Arc<dyn Waker>>` cannot work. 

Difference between tokio's `AsyncRead` and `std::io::Read` is that AsyncRead takes `Pin<&mut self>` which is needed for future to have local stack variable. `std::io::Read` does not have `Pin`. Also `AsyncRead` returns `Poll` which is `enum` of `Ready` and `Pending`, while `std::io::Read` does not have a clear way to represent "there are more works to do, but not ready yet".

The scheduler chooses from the runnable task queues, the event loop calls wake and moves futures from non-runnable to runnable.

With <future> IO uring, the `AsyncRead` and `AsyncWrite` traits are not there, details in `tokio_uring` crate. It will replace part of the IO event loop.

`AsyncReadExt` and `AsyncWriteExt` for extention, it provides extra util interfaces such as r/w a fixed bytes, etc. It provides the actual `Future` for `AsyncRead` and `AsyncWrite`, who just provides a method that returns `Poll`.

### Tokio FS

Some OS does not provide async file interfaces. Tokio has a dedicated blocking thread that does the actual FS, the handle waits for it to provide the async interface. This makes tokio FS slower than std FS.

One can use `spawn_blocking` with a closure to interact with the FS and just provide the result as async. 

### Tokio process

When handle to tokio `Child` process, the process is NOT terminated, so is std `Child`. `Child` is not `Future`, but it has a `wait` function to wait on it.

### Tokio IO
Reading from a TCP stream requires a `mut` reference. When shared, it is better not wrap into a `Arc<Mutex<TCPStream>>`, but spawn a thread owning the TCPStream doing r/w, and use a `tokio::sync::mpsc::channel` to commmunicate with other threads.

### Sink and Stream

In the `tokio_stream` crate, `futures::stream::Stream` is similar to an iterator / channel. `futures::sink::Sink` is similar to channel send, which lets you put one element at a time.

Codec / Framing is conversion from element to bytes. A util to be used is `tokio_util::codec`.

## Utils

### tokio::sync

`mpsc` channel, multi-producer-single-consumer. 

`oneshot` channel, can only send and receive once. Can only `await` but not `next` bc it does not impl `Stream`. Provides `blocking_receive` and `send` that are blocking. This bridges async and sync world. Sending request and a `oneshot` receiver together is common.

`broadcast` channel, multiple consumer, every receiver receives every value.

`watch` channel, broadcast only the last sent value. useful for slow reader / only latest update matters

`Notify` a condition variable that is not associated with a mutex. Wrap it into a `Arc` and when notified the future that awaits on `Notify::notified()` is waken up. Is convenient to impl `Waker` in self-defined resources without touching `Poll` interface. 
```rust
let notify = Arc::new(Notify::new());
let notify_clone = notify.copy();
let handle = tokio::spawn(async move {
    notify2.notified().await;
    println!("continue execution");
})
notify.notify_one();
```
`Notify` can also bridge sync and async world, since `notify_one` is sync, some sync resources can call it and unblock a future that called `notified.await`

`Semaphore` similar to c/c++ semaphore. Interfaces are `acquire` that returns a `Result<SemaphorePermit, _>` that is dropped when released.

### JoinSet

TBC: https://youtu.be/o2ob8zkeq2s?t=7881 

## Complications

