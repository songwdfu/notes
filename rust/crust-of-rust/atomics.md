## Atomics

Normally AtomicX will be put on the heap. Putting it on the stack requires you
passing ref around, which is tricky bc lifetimes.

`AtomicPtr` is an atomic `UnsafeCell` which might be useful.

use `AtomicX::compare_exchange` instead of `compare_swap` bc the former allows
specifying different mem ordering for success and failure. Often the mem
ordering for failure is just `Relaxed` since it's not necessary to sync with
other threads after failing to acquire.

Bc MESI protocal, compare_exchange spin is expensive, do
```
while compare_exchange(...) {
    while locked.load() {}
}
```
so the inner loop is not exclusive access.

`compare_exchange_weak` is allowed to fail spuriously but is cheaper on arm. x86
has cmpxchg, however arm only have load-linked store-conditional. store-cond
does not require exclusive access and is cheaper, but could fail if another
thread got it. arm impls strict `compare_exchange` using retries. Therefore, if
it's called in a loop, should always use `compare_exchange_weak`

`fetch_add` is exclusive access. `fetch_update` is a `compare_exchange_weak`
loop internally, similar for operations that are not supported by the
archetecture.

## Mem ordering

`Ordering::{Relaxed, Acquire, Release, AcqRel, SeqCst}`. 

When under `Relaxed`, a thread is allowed to see any value ever written to the
mem location. The only restrictions are dependencies and branches. 

Using `Release`, **if the store is loaded** by some later `Acquire`, all
previous writes become visible to that thread that perfermed `Acquire` or
stronger loading of the value, no reads or writes on the current thread can be
reordered to after this store.

Using `Acquire`, **if the load loads** some previous `Release`, all writes in
that other threads that `Release` the save variable are visible in the current
thread, no reads or writes on the current thread can be reordered to before this
load.  

`AcqRel` is both `Release` and `Acquire` semantics, often used in ops like
`fetch_add` that is both load and store. 

However, `Acquire` and `Release` does not guarantee immediate visibility. The
synchronization only happens if the load loaded the stored value.  

`SeqCst` is strictly sequential and is necessary under multi-producer
multi-consumer scenario where each consumer must observe the same order of
producer execution. This requires full mfence so is expensive.

e.g. case when acq/rel does not work:
```
thread 1: fn write_x() { x.store(rel) }
thread 2: fn write_y() { y.store(rel) }

thread 3: fn read_x_then_y() {
    while !x.load(acq) {}
    if y.load(acq) {
        z.fetch_add(1);
    }
}

thread 4: fn read_y_then_x() {
    while !y.load(acq) {}
    if x.load(acq) {
        z.fetch_add(1);
    }
}
```
Then it's possible to observe `z == 0`. The only dependency here is `write_x`
before `read_x_then_y`, and `write_y` before `write_y_then_x`. However it is
possible for thread 3 and thread 4 to observe different ordering of execution.
Under acq/rel, the visible write guarantees are only true **if** the stored
value is loaded, however it is possible that the `if y.load()` and `if x.load()`
may **both** observe the old value. Only `SeqCst` ensures global seq ordering
such that at least one reads the new value (serializeable).

## ThreadSanitizer / Loom 

ThreadSan detects unsynchronized r/w to memory location for the specific
execution, however does not catch all. 

Loom implements CDSChecker for atomics. Intrumenting the concurrent prog using
the loom atomics instead of std. Loom atomics `load` returns one of the valid
values acc to the specified mem ordering. Loom runs it multiple times so we
observe what happens when different ordering happens. Replace primitives with
loom primitives and `loom::model` takes a closure and run that testing over it.

Loom does not model SeqCst now, SeqCst is still AcqRel.

Some mocking of, e.g., syscalls are necessary to use Loom.

## MFENCE

`std::sync::atomic::compiler_fence` does not prevent hardware reordering. Only
prevents compiler reordering and is only useful to prevent a thread racing with
itself when using signal handlers.

`std::sync::atomic::fence` establish happens-before relationship without
mentioning mem location. It syncs with other thread having atomic ops or
`fence`. The prereq of fence syncing with fence / atomic on another threadis
that there exists an syncing atomic load-store in between the (two) fences. 

e.g.
```
fence(release) // A
store
        load
        fence(acquire) // B
```
prevents all r/w before A to be reordered after any r/w after B.

Syncing btw mfences might be useful when storing and loading lots of atomics,
where all load and stores can just be `Relaxed` and only add fences to be
`Acquire` and `Release`.

The difference btw fence and load-store is: While an atomic store-release
operation prevents all preceding reads and writes from moving past the
store-release, an atomic_thread_fence with std::memory_order_release ordering
prevents all preceding reads and writes from moving past all subsequent stores.
This is useful when on a single thread:

e.g. In this snippet, write B can be reordered before write A
```
// write mem A
x.store(release)
// write mem B
```
But in this snippet, write B cannot be reordered before A. 
```
// write mem A
mfence(release)
x.store(relaxed)
// write mem B
```

## Volatile

`std::ptr::{read_volatile, write_volatile}`: when working with mmap'd devices,
read from mem might have side effect (e.g. moving ringbuf pointers), in that
case, `read_volatile` ensures reads reaches the main mem. It is not for syncing
purposes.


