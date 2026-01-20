## Send and Sync

Thread safety at type level, instead of doc level.

Both are marker traits, no method. Also auto trait, struct / enum is trait if
members are all trait.

## Send 

it's safe to pass the ownership of the object to the other thread.

`!Send` example: 
- `Rc` where sending violates underlying invariant. Multiple clones of Rc would
hold pointer to the same inner and the ref count is not atomic. Giving any
clone to other thread is race.
- `Mutex` where sending violates OS requirement that thread who hold must be who
release
- Struct whose `drop` refers to thread-locals where sending makes it refer to
the other thread's local

## Sync

`T` is `Sync` iff `&T` is `Send`. Send: it's safe to pass ownership. Sync: it's
safe to pass shared ref.

## Send and Sync

- A `Rc` is `!Send + !Sync` because of the above and also giving a reference
allows other threads to call clone 
- A `MutexGuard` is `Sync + !Send`. It's safe to pass the shared ref of mtx
guard to another thread, because it cannot drop it which needs mut ref. 
- A `Cell` is `Send + !Sync` (for `T: Send`), because it is safe on any threads
as long as no reference is given to other threads. 

`impl Send / Sync` is unsafe, because it's a override from compiler's auto
trait, a guarantee made that it will not cause UB. 
`impl !Sync / !Sync` is unstable but safe. It's useful when the data is send or
sync but the action is not, e.g. accessing thread-locals.

`&T` is `Send` if `T` is `Sync`, but `&mut T` is `Send` only if `T` is `Send`,
because of methods like `mem::replace` that takes ownership from a mut ref. So
`&mut T` is equivalent to `T` in this sense.

`*const T` and `*mut T` are `!Sync + !Send`. Any type that contains raw pointers
has to manually impl `Send` and `Sync`.

Channel `Sender` and `Receiver` are only `Send` if `T` is `Send`, but they are
not `Sync`.

`Arc` requires `T` to be `Send + Sync` bc multiple threads will refer to it
after the `Arc` is cloned, and the last thread that `drop`s the `T` will require
ownership, which might not be the first thread.

To impl `!Send + !Sync` with stable, use `_not_send: PhantomData<Rc<()>>` so the
compiler infers it. To make it only `Send` or only `Sync`, `unsafe impl Send /
Sync` on top of that.
