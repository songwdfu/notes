# The Rustnomican

## Safe and Unsafe

unsafe keyword marks that the interface has additional constraints documented,
or the implementation already checks the documented additional constraints.

Safe code in Rust must never cause UB. In this sense, unsafe code that relies
on specific impl of some trait should verify that it never causes UB, or expose
the interface as unsafe trait, so implementor would provide impl in unsafe
block.

So, safe code trust non-safe code, but unsafe code cannot trust safe code.

## Dyn sized types

In rust, ZST takes 0 bytes. Common ZST include struct with no fields or fields
of Nothing, (), [u8; 0].

DST include dyn Trait and [u8], i.e. trait object and slices. These requires
fat pointer, Trait object is stored via vtable and slices are described with
size at run time. Custom DST has ?Sized trait but can only be coerced from
Sized type.

Empty type is enum with no fields. A Result<T, Void> is always T.

For types that cannot be null, Option<T> is same size as T, null is used to
repr None. This include &, &mut, func pointer, etc.

## Layout

Under #[repr(Rust)], structs are reorderable. For FFI passing, use #[repr(C)].

#[repr(transparent)] is for single-field struct and single-variant enum. This
guarantees same layout as inner and thus FFI to sth that takes inner works.
e.g. UnsafeCell

#[resr(u8)] forces the fieldless enum (can have variants but not data) to be C
equivalent uint or int repr

#[repr(packed)] strips padding, #[repr(align(n)] forces type to be n-byte
aligned. Can be used to avoid false sharing.

## Splitting Borrows

borrowck does not understand disjoint borrow of container.

To impl iterator, impl Iterator for IterMut struct which contains the node of
interest. 

## Coercions

coercion is not performed when matching traits. e.g. `impl<'a> Trait for &'a
i32`, then cannot call fn on `&mut i32`.

The dot operator calling conversion tries T::, <&T>:: or <&mut T>::, then U::
where T: Deref<Target = U>, then unsize (e.g. [i32; 2] to [i32]), recursively

for a snippet that has multiple implementations, the compiler does inference
```
fn func<T:Clone>(value: &T) {
    let cloned = value.clone();
}
```
then cloned is type T, is a clone of T, because it tries <&T>::clone first which
is satisfied by `trait Clone{ fn clone(&T) -> T }`
If T does not impl Clone,
```
fn func<T>(value: &T) {
    let cloned = value.clone();
}
```
then cloned is type &T, is a clone of ref to T, because <&T> does not impl
clone, compiler tries autoref <&&T>, which is Clone, so it calls `fn clone(&&T)
-> T`.

Be careful when calling clone() on a reference &T, make sure T impl Clone to
get a value T, otherwise will get a &T to the same obj. `#[derive(Clone)]` only
works for inner T: Clone. When working with things like `Arc<T>`, better impl
Clone manually `impl<T> Clone for Container<T> { fn clone(&self) -> Self {
Self(Arc::clone(&self.0)) } }` and we'll always get a new `Container<T>`
instead of ref.

## Casting

When casting, length of slice is not adjusted.

`std::mem::transmute<T, U>` is very dangerous. must specify result type.
transmute & to &mut is UB. trans to a ref gives unbounded lifetime. repr(Rust)
has arbitrary layout, even for instances of same generic types. Vec<> can have
diff order of fields. Only works for T, U of same size.
`std::mem::transmute_copy` does not have size check, more dangerous.

raw ptr cast and unions is also has the above problem. Use `repr(C)` when
wanting fixed layout of structs.

## Uninit mem

Can assign once to uninit unmutable var, if legal on every branch.

Assigning by deref `*x = ...` always drop prev x, assigning by let `let x = ...`
always NOT drop prev x

Having conditional init makes Rust infer drop flag at runtime, this is overhead

Array cannot be safely partial init'd. `std::mem::MaybeUninit`  allows `let x =
[const { MaybeUninit::uninit() }; SIZE];` where SIZE is fixed. Then drop on
`MaybeUninit` is no-op. So we can safely assign op `=` instead of using
`std::ptr::write`:
```
for i in 0..SIZE {
    x[i] = MaybeUninit::new(Box::new(i as u32));
    // note *x[i] = Box::new(i as u32) is illegal!
    // because we overwrite an uninit'd Box<u32> and cause drop
    // if MaybeUninit::new cannot be used, use std::ptr::{write. copy,
    // copy_nonoverlapping}. The last is memcpy(src, dest, cnt).
}
```
And transmute this into init'd array (unsafe). `MaybeUninit<T>` has same layout
as `T`.
```
unsafe { std::mem::transmute<_, [Box<u32>; SIZE]> (x) }
```
Note that `Container<MaybeUninit<T>>` maybe not have the same layout as
`Container<T>`! But for Vec it is okay.

`std::mem::{write, copy, copy_nonoverlapping}` src and dest must be aligned.
They do not incur drop so is safe for uninit'd mem or type that does not impl
`Drop`. 

Must check init of vars when drop, for every path including panicking. When
panicking, `MaybeUninit` cannot cause double-free but can cause leak of init'd
values (MaybeUninit::new'd)!

Ref to uninitialized mem is UB. To obtain raw ptr to uninit'd mem, cannot use &.
For array use `base_ptr.add(i)` where `base_ptr` is `* mut T`. For struct use
raw reference:
```
let mut uninit = MaybeUninit::<StructName>::uninit();

// raw pointer to the field of struct without intermediate ref, which is UB
let f1_ptr = unsafe { &raw mut (*uninit.as_mut_ptr()).field };

// assign to uninit'd field
unsafe{ f1_ptr.write(true); }

// cast to init'd obj
let init = unsafe { uninit.assume_init() }
```

## Ownership Based Resource Management

The only constructor is Rust is to name the object and init any fields.

In rust, types must be able to be memcpy'd anywhere, so move constructor is
meaningless. `Clone` is equivalent to C++ copy but has to be explicitly invoked,
`Copy` is merely copying the bits and is implicit.

By default, after `fn drop(&mut self)` is **run**, rust drops fields
recursively, applies to struct and enum. So when impl custom drop, be careful
about use-after-free and double-free.

To prevent auto recursive drop, use a `Option<T>` field, drop T manually, and
assign a `None`.

## Lifetime

Sometimes the inferred lifetime of local will be longer than expected, since
destructor uses the variable as well. One can call drop(x) to get rid of x
early.

For Fn trait there’s High-Rank Trait Bound like impl<F> Closure<F> where for
<‘a> F: Fn(&‘a (u8, u16)) -> &’a u8 {…}

This defines the lifetime for the Fn trait object F

## Subtyping and Variance

A is subtype of B if A satisfies B

F is covariant if F<A> is subtype of F<B> where A is subtype of B

&’a T is covariant over both ‘a and T. Bc you can downgrade &‘static T to &’a
T, also if A: B, a func requires &B, you pass an A and A gets downgraded. 

&’a mut T is covariant over ‘a because you can downgrade &’static mut T to &’a
mut T safely. But &mut T is invariant over T. When the function asks for a &mut
&’a str, if you pass a &mut &’static str, it could be modified to point to some
obj that lives shorter. On the other hand if a func requires &mut &’static str,
you also cannot pass a &mut &’a str because it does not live long enough.  

Anyways, &mut T, *mut T, UnsafeCell<T>, and other interior mutability types are
invariant over T, but owning type such as Box<T>, Vec<T>, also &T, *const T are
covariant over T.

Anything that’s interior mutable cannot be passed to a func that requires
shorter lifetime, otherwise inner could die earlier than the wrapper. Anything
that’s owned would have move semantic and the lifetime of the owned obj is tied
to the variable, so it’s similar to how everything is covariant over ‘a. 

Fn(T)->U is contravariant over T and covariant over U. The former because a
func that requires any ‘a to do the job satisfies a requirement for a function
that is given a ‘static to do the job. 

## Drop check

Sound generic drop need to be enforced when impl Drop for struct that has
generics. The generic fields must outlive the struct. 

In unsafe code, if `struct <‘a, T: ‘a> MyStruct` only contain *const T has
unbounded lifetime. When defining struct, add PhantomData<&’a T> to reference
the lifetime for correct variance and drop check. 

Traditionally rust does not drop check * T fields, now it’s not the case if we
have impl<T> Drop for MyStruct<T>. Rust will think we own T in MyStruct even if
we only maintain pointer / ref to it.

When drop order matters, use ManuallyDrop wrapper. The #[may_dangle] label
makes drop unsafe and guarantee all expired fields are not accessed

## Exception Safety

Option is for reasonably absent value, Result is for resolvable error, panic is
for program error or extremely bad conditions. Panic kills the thread and
unwind it as if all functions return immediately. 

Unwinding across lang is UB. Must catch every exception at the boundary of
FFI!!!

Unsafe code should prepare for unsound safe code. If create transient unsound
state, should not run panickable code during that. This is exception safety.
For example, when extending vec with uninitialized mem, if we inc the Len
first, watch out for clone() which could be customized impl that can panic!
After that the uninit’d vec will be read when being dropped during unwind. To
fix, inc Len later so uninit’d state wouldn’t be observed.

Things to watch out for are customizable code, including clone(), comparison, 

To make it exception safe, could use a struct to record state and impl Drop to
correct the unsound state

