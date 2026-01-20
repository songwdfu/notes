# Smart Pointers

## `Cell`

Contains an immutable inner struct. Can modify stuff through shared reference
(interior mutability).

A `Cell` can be created / swapped, when created it takes immutable ref to self.
For inner that is `Copy`, it has `get` that gets a copy of inner instead of a
ref to it. This guarantees safety since no one has pointer to inner

`Cell` is `!Sync`. thread unsafe.

Usually used in threadlocals, flag / counter that only one thread modifies in
multiple places.

## `UnsafeCell`

In `Cell` we need to modify the inner value while only having a shared
reference to self. Mutating it with shared ref requires `UnsafeCell`. This is
Core to iterior mutability. Holds a type and can get raw exclusive ref anyhow.
Deref the *T gives a T, should be wrapped in `unsafe{}` 

## `RefCell`

A mutable memory location with dyn checking borrow rules.

Provide `borrow` and `borrow_mut`. Good for graph and trees.

`borrow` and `borrow_mut` return `Ref` and `RefMut` type, which are keep ref to
RefCell to decrement refcount upon `Drop`. This has lifetime of the refcell.

`Sync` version is `RwLock`, counter is atomic, block until borrow check is met.
also `Mutex`, which only support `borrow_mut`

## `Rc`

Refcounted shared pointer to alloc on the heap. Disallow mut for inner. use
`Rc<Cell<>>` for interior mutability. `Clone` gives a ptr to same obj on heap.
It is `!Sync`.

`Box<T>` puts things on heap but `Clone` alloc new T on heap.
`Box<T>::into_raw` consumes the `Box` and return raw `*T`. Be careful if `Box`
goes OOS, it is freed!
`Box<T>::from_row` turns ptr back to `Box`

`* mut` and `* const` are raw pointers, does not have compile time borrow
checks. Can only deref it into unsafe block and get a shared / exclusive ref.
`* const` cannot be turned into a mut ref

support `T: ?Size` this allows the generic `T` to be dynamic sized. ? opts out
the default `Sized` requirement for generics.

`Arc` is `Sync` and `Send` version is `Rc`, same except refcount is not `Cell`
but atomics.

### `std::ptr::NonNull`

Tells compiler the value is not Null. So compiler can use Null to represent sth
else. `Option<NonNull<T>>` has 0 overhead since None is represented as null. Is
`!Send` and `!Sync`

Could be used to store a `* mut` and use `NonNull::as_ptr()` to get it back.
using `NonNull::as_ref()` gives a shared ref, no need for `&*ptr`.

### Drop check

Rust treats `Drop`ing of a variable (when it goes OOS) as usage of every field
within the type that is being dropped. However if the definition of the struct
contains only the pointer to that type, compiler does not know it owns the
object and wouldn't check on it (dropping a ptr to a already dropped obj is
fine). Adding a `std::marker::PhantomData<T>` in the struct makes the compiler
know it owns a `T` and would perform drop check over T. So if the thing that's
pointed to by the ptr is dropped, the code will not compile.

e.g.
```rust
struct Rc<T> {
    inner: NonNull<RcInner<T>>,
    _marker: PhantomData<RcInner<T>>,
}

impl<T> Rc<T> {
    pub fn new(v: T) -> Self {
        let inner = Box::new(RcInner {
            value: v,
            refcount: Cell::new(1),
        })
    }
}

int main() {
    let (y, x);
    x = String::from("hi");
    y = Rc::new(x);

    // Rust drops in reverse order of declaration
    // drop x
    // drop y <- not ok!! when dropping the last Rc, Rc::Drop will drop inner,
    // it calls to drop(x), which is on already dropped value. This is only
    // checked if _marker is here.
}

```

## `std::borrow::Cow`

Is a `enum` that could be `Borrowed` or `Owned`. `Cow` impls `Deref`, returns
shared ref to either the borrowed or owned. Useful when things could be modified
or not modified depending on ifelse.


