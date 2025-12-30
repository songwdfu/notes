# The Rustnomican

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

Be careful when calling clone() on a reference &T, make sure T impl Clone to get
a value T, otherwise will get a &T to the same obj. `#[derive(Clone)]` only
works for inner T: Clone. When working with things like `Arc<T>`, better impl
Clone manually `impl<T> Clone for Container<T> { fn clone(&self) -> Self {
Self(Arc::clone(&self.0)) } }` and we'll always get a new `Container<T>` instead
of ref.

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

To obtain raw ptr to uninit'd mem, cannot use &. For array use `base_ptr.add(i)`
where `base_ptr` is `* mut T`. For struct use raw reference:
```
let mut uninit = MaybeUninit::<StructName>::uninit();
// raw pointer to the field of struct without intermediate ref, which is UB
let f1_ptr = unsafe { &raw mut (*uninit.as_mut_ptr()).field };
unsafe{ f1_ptr.write(true); }
let init = unsafe { uninit.assume_init() }
```

