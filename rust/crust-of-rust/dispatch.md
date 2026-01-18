## Monomorphomization

monomorphomization: fill generics with the specific type to compile, rust
generates the func / struct def for each specific type used. This might be
problematic if this is a dynamically linked lib where the source doesn't contain
some of the types used for the generic. Another downside is increased binary
size and hence larger region needed for caching the instructions.

## Sized

`Sized` trait: types with const size known at compile time. A generic template
implicitly requires T to be `Sized`. Bc the compiler need to alloc space on
stack to place the var. It is always an autotrait that is not impl'd by us.
Common `!Sized` are `str` and `[u8]`.

## Dispatch and Trait Object

**static dispatch**: rust does code specialization by generating versions of generic
func, then it's trivial for the function to be called according to the specific
type.

**dynamic dispatch**: 

Is essentially type erasure. This avoids generating specialized code for each
used specific types, and allows collections of trait objects that could be in
fact different types.

`pub fn bar(s: &[impl Hei])` is same as `pub fn bar<H: Hei>(s: &[H])`, then all
elements must be the same type. `&[dyn Hei]` allows different types that impls
`Hei` but the size of `dyn Hei` is unknown.

To make `dyn X` sized, we can make it a ref or `Box` that is itself `Sized` for
any inner that is `?Sized`. A trait object is something that only behave as some
underlying trait, e.g. `Box<dyn AsRef<str>>`.

## VTable

The compiler uses a vtable for dispatch of trait objects.

The ptr types to DSTs (wide pointers) are twice as large as ptr types to `Sized`
objects. It contains a ptr to the mem location as usual, but also, ptr types to
slices contain num of elements, ptr types to trait objects contain ptr to
vtable.

vtable is a struct whose fields are pointers to methods of the trait for each
type turned into trait object. e.g. `method: &<Type as Trait>::method`.
vtables are built at compile time.

vtable also contains pointer to `Drop` impl for every concrete type, so the
concrete types turned into trait objs will be properly `drop`'d when OOS. i.e.
any `dyn X` is actually `dyn (X + Drop)`. vtable also contains the size and
alignment of the type.

## Trait-object-safe

Trait objects are limited compared to generics

- It's illegal to create a trait object for traitA + traitB where traitB is not
autotrait (because otherwise two traits require 2 vtables), but you can define
trait C: traitA + traitB and use `dyn C`.

- It's illegal to use undeclared associated type with `dyn`, must declare `dyn
Trait<Type = ()>`. Bc the type is not in the vtable.

- It's illegal to call a method that does not take `self` over a trait object.
Bc it cannot be put into a vtable. e.g. `fn weird()`. But can opt this function
out of the vtable for trait object by `fn weird () where Self: Sized`, which
requires the caller to be `Sized`. When the method takes `self` instead of
`&self`, it also cannot be called via trait object for the same reason.

- It's illegal to create trait object where the trait method is generic. The
generic method would be specialized into many functions and the vtable cannot be
constructed because it does not know which specialized version to point to. 
e.g. `trait Extend<T>`, `fn extend<I>(&mut self, iter: I) where I:
IntoIterator<T>`, then `dyn Extend<bool>` is illegal, because it doesn't know
which `I` version to point to for the vtable.

- It is illegal to create trait object where the trait method returns `Self` .
Bc when calling this via trait object, `Self` is `!Sized`. But for this and the
above case, you can opt out the method by marking `where Self: Sized`. 

## Manually constructing vtable at runtime

`ptr-meta` feature allows inspecting and manipulating metadata of fat pointers.

`std::task::RawWaker` contains a `RawWakerVTable` that is contructed by passing
pointers to `wake` functions, etc., so it is generic.

## fn, impl Fn, dyn Fn

`fn` is a function pointer, `dyn Fn` allows closure as well (which captures
environment as well). `impl Fn` is generic over `T: Fn` and would incur
specialization for every type of closure passed in. 

Using `dyn Fn` avoids the wrapper to be generic and thus makes the wrapper trait
trait-object-safe (trait that wraps `fn foo(f: impl Fn())` is generic `foo<T:
Fn()>` and not trait-object-safe).

## std::any::Any

contains `pub fn type_id(&self) -> TypeId` that gives the unique id of the
concrete type of the trait object. `pub fn downcast_ref<T>(&self) -> Option<&T>`
allows going from the trait object `dyn Any` to the known concrete type safely.
