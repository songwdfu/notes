Drop check cares for two things: does the drop impl uses the T? does the drop
impl drops the T?

1. When the type is generic over T and drop is implemented, the compiler assumes
   drop uses T (for drop impl to be sound for generic type, the generic args
must outlive it), unless `#[may_dangle]` is used.

2. When the fields contain a T by value, or we `impl<T> Drop for Type<T>` and
   `#[may_dangle]` is not used, compiler assumes we'll drop a T during drop. 

When `#[may_dangle]` is used, the compiler assumes nothing about the drop impl
except the following edge case: When `#[may_dangle]` is used, the inner type T
will still be drop checked iff the outer type owns T, this is by the outer type
containing T by value or `PhantomData<T>`.

---

When a type is generic over a type and impls drop, dropping the type would
always be treated as a use of the generic type. Even if the type only contains
`*mut T` and will not access the value pointed to during drop.

To indicate drop is not doing anything with the inner type, use
`!#[feature(dropck_eyepatch)]` and `unsafe impl<#[may_dangle] T> Drop for
Boks<T>` so compiler would not think the drop uses or drops T. By doing so we
unsafely guarantee the drop impl will not access the inner value T. Then the
below code will compile because `&mut y` could end early because `drop(b)` does
not use `y`, and the immut ref would be allowed.

```
let mut y = 42;
let b = Boks:ny(&mut y);
println!("{}", y);
// b OOS, drop(b)
```

However, after using `#[may_dangle]`, the compiler does not know the drop is
gonna **drop** a T **unless the type owns it**. i.e. the drop glue is not gonna
work. To force the compiler drop check the inner type T during drop of outer
type, pretend we owns T using `_owns_T: PhantomData<T>`. 

e.g. `Box<Inner<&mut i32>>` where `Inner::drop` derefs the `&mut i32`. Then
`Box` must contain a `PhantomData<T>` to enable drop check on `Inner`, even
though we do not use `Inner` during `Box::drop`, we do drop it because we owns
it. (Note that `PhantomData<fn() -> T>` does NOT let the compiler think we own
T.)

When a type has `*mut T` argument, it is invariant, to make it covariant, can
use `NonNull<T>` which is essentially a `*const T` that is covariant.

The above "owning the pointer" paradigm is encapsulated into `Unique<T>` where
it is a `*const T` for covariance, contains `PhantomData<T>`, derives
`Sync/Send`, and marks pointer as `NonZero`.


