# Async, Await

[video link](https://www.youtube.com/watch?v=ThjvMReOXYM)

## Async

Async function is a function that returns `impl Future`. The `Future` trait is sth that will be ready in the future.

Await waits for future complete, `future.get()` in cpp. Future does nothing untils it is `await`ed

Async chuck could be thought of executed in a block, between two `await`s. This is coordinated scheduling.

## Await

Await yields until the future is ready. Yield yields until some progress happens using OS primitives

One use of future is to try multiple things. Good for any IO

The executor crate like `tokio` provides IO fds and the top main execution cycle. When everything it manages yields, it yields to the kernel with all things to watch for (epoll)

`tokio::main` just makes everything a async block that the executor blocks on.

`select!` waits for the first that completes, others can be in intermediate state as they are terminated.

`join` and `join_all` executes all input futures concurrently and waits for all of them are completed, returning result in order. `futures::stream::FuturesUnordered` doesn't return in order, push in all the futures and loop through it with await, each time a single result is returned.

`tokio` runtime `block_on` only awaits the single async main function passed into it, even if there are multiple futures in that async main. This only runs it on one thread even if executor starts multiple

### Spawn

`tokio::spawn` moves the future to the executor, now it can run besides the async main future, getting picked up by another thread. `spawn` needs future to be static, i.e. doesn't refer to sth outside of it, of which the lifetime is unknown. `tokio::spawn` returns a future to the return value, which has to be waited on.

using `spawn` also "reserves" core for the async block submitted.

Note: cpp's `std::async(std::launch::async, func)` and rust's `tokio::spawn(async { ... })` both start execution immediately after called, but just creating a `let fut = async { ... }` in rust won't execute it until `fut.await` is called.

### Future

Future is transpiled into a enum StateMachine that has chunks of codes as variant, where the states are the local variables. `impl Future` is actually a `&mut` to a StateMachine, each continuation of the future continues that state machine (that holds local variables in the chunk)

When returning / passing a future, it (the state machine that keeps many local variables) gets memcpy'd around, `tokio::spawn` sends the future to executor and only keeps a handle (a ptr to the future), so there are less memcpy. Another way is to `Box` the future to place it on heap.

### Async func in Trait

Async trait is not supported yet (it gets rewritten into `fn foo() -> impl Future<Output = ...>`), however the size of the future depends on the local variable size of the implementation, which is unknown at compile time.

`async_trait` crate rewrites all Async trait and impl into `fn foo -> Pin<Box<dyn Future<Output = ...>>>`, but now all futures are heap allocated. 

Another approach is to declare associated `type CallFuture: Future<Output = ...>;` Then `fn call() -> Self::CallFuture`, when impl the associated type is declared and known to the compiler

### Async Mutex

Don't use `std::Mutex` with async func, if some job goes `await` holding the mutex guard while another executor job on the same thread waits on the mutex, the thread is blocked and no progress could be made to the job that's in await as well => deadlock! 

`tokio::sync::Mutex` is async-aware, solves this problem but is a lot slower. So use `std::Mutex` as long as the cruitical section does not contain any `await` points and it is short (if it is long, it holds on to the thread + lock without any yield point). 

Don't async spawn something that is not cooperative scheduled.

Future after yield is not guaranteed to run on same thread after yield.

future panic will take effect when executor thread polls it. The stack trace will not include the function that spawned it (it just pushes future to the job queue)
