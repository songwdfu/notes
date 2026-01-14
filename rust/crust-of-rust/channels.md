Channel allows sending T through Sender<T> and Receiver<T>. 

Sender is Send iff T is Send (if T is not send, still ok if Sender is not given
to another thread)

`Mutex::lock()` gives back a `Result` that could be `PoisonError`, signifying
that the last locking panicked.

`#[derive(Clone)]` now desugars to `impl<T: Clone> Clone for Struct<T>`, that
inner has to be Clone.

Channels are in general implemented via Mutex and Condvar. Latch-less impls
could use atomic VecDeque / LinkedList + thread::park and
thread::Thread::notify.

Oneshot channel is much cheaper than normal channels bc it's just an atomic ref.

One optimization one can do for mpsc is batch receiving, which takes multiple
elements at once and reduce number of times that acquires Mutex.


