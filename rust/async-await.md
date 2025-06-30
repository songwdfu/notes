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

Future is transpiled into a enum StateMachine that has chunks of codes as variant, where the states are the local variables

TB continued: https://youtu.be/ThjvMReOXYM?t=6473
