## Func Item, Func Pointer, Fn Traits

The type of a function is a **function item** that is unique id of function and
is zero size and only present at compile-time. It's illegal to assign `boo<i32>`
to `boo<u32>`. Merely using the function item would not cause the compiler to
generate the func body.

The func item which has the same signiture could be coerced into a **function
pointer** `fn()`. e.g. `fn baz(f: fn(u32) -> 32)`.

The `Fn()` is a trait. `FnOnce` can only be called once. `FnMut` can only be
called one at a time (requires mut ref). `Fn`and `FnMut` are subtraits of
`FnOnce`. `FnMut` also impls `Fn`. 

`fn` func pointer impls all three traits. Need to pass `fn` by ownership for
`FnOnce`, by mut ref for `FnMut`, by ref for `Fn`.

## Closures

Closures captures enclosing environments. Non-capturing closures are coercible
to func pointers and thus impls the `Fn` traits.

Capturing closures are like defining a struct with all captured vars and impls
`Fn` or `FnMut` or `FnOnce` for the struct, depending on capturing by ref, mut
ref, or move. (`FnMut` requires `&mut self`, `FnOnce` requires `self`).

Capturing by ownership can be inferred by compiler e.g. when calling `drop` on,
but using `move` explicitly moves all captured variable into the closure by
ownership and hence it's lifetime is tied to the closure. All calls to the
closure would refer to the same instance of var moved in.

Can get dynamic dispatch to avoid generating a func for each type of function /
closure passed in by using arg `fn foo(f: Box<dyn Fn()>)`. 

Implementing the func traits for `Box<dyn FnX()>` requires unstable unsized
Rvalue feature, because derefing the `Box` gives a `dyn Fnx()` which is
`!Sized`.

To use the func traits on trait object, need to have the correct indirection:
`FnMut` requires `&mut`, `FnOnce` requires `Box` which is a fat pointer that
provides ownership.

A constant closure can be eval'd at compile time, similar to const fn. e.g. `||
0`.

To specify lifetime for function trait, use syntax 
```
fn foo<F>(f: F)
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    ...
}
```

For async func, which desugars into `fn spawn<F>(f: F) -> impl Future<Output =
()>` that returns a opaque future type, the output future has the same lifetime
as the arg. Therefore often the passed in closure must be `'static` for the
output future to outlive the creating stack. This could be done by `move`
capturing. For other async funcs, often adding `+ 'static` to generic args is
needed.
