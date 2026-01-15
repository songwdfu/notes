T: U (T is subtype of U) if T is at least as useful as U

'static is subtype of any 'a.

function arguments is the only contravariant in Rust. a `Fn(&'a str) -> ()` is
more "useful" than `Fn(&'static str) -> ()` as it is less strict in its
arguments.

invariant requires exact type

Mutable ref is invariant to the thing they reference `&mut T` is invariant in
`T`. Cannot pass `&mut &'static str` to `&mut &'a str`. 

Mutable ref is covariant to the lifetime `&'a mut T` is covariant in `'a`. Can
pass a `&'static mut T` to `&'a mut T`.

If the function signiture is `fn foo<'a> (_: &'a mut &'a str) -> &'a str`, it
forces the lifetime of the mut ref and the str to be the same. This causes
trouble because the thing mut ref'd is invariant. So the str will be borrowed
longer than needed if the arg is sth like `&'static str`.

A fix is to introduce another lifetime `fn foo<'a, 'b> (_: &'a mut &'b str) ->
&'b str`, then the passed in type could be something like `&'x mut &'static
str`, and the mut ref will be dropped after x OOS. Another syntax is `fn foo<'s>
(_: &mut &'s str) -> &'s str`. Compiler infers a unique lifetime for unmarked
`&mut` to `&'_ mut`.

Using `PhantomData<fn() -> T>` in a struct would not incur dropcheck on T while
giving covariant so the compiler can auto shorten lifetime. `PhantomData<T>`
will cause dropcheck while `PhantomData<fn(T)>` will be contravariant and be
annoying to use. `PhantomData<*mut T>` will cause the struct to be invariant.

Be careful when manually correcting the variance of the type, be careful of
invariance attack like 

```
let mut x: &'static str = "hello world";
let z: &'a str = String::new();

// x: &mut &'static str, z: &'a str, 
// if &mut T is not invariant over T, this would have compiled!
let x = z;

drop(z);
println!("{}", x); // UB, should not compile!

```

It's very rare that compiler concludes covariant while you require invariant.
Only case is `&mut T` to `*const T` to `&mut T` cast, which is not UB but evil.
Casting `&T` to `&mut T` is always UB.

`NonNull<T>` is a `*mut T` that is non-null and covariant. This is correct for
most containers like `Rc` and `LinkedList`. Add `PhantomData<Cell<T>>` if
invariant is needed.


