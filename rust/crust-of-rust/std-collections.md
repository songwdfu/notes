## Vec

`RawVec` is a contiguous chunk of memory, `Vec` on top of that tracks the actual
number of els in it. Use `Vec::with_capacity()` to init to avoid frequent
resizing. 

`Vec::swap_remove` removes the vector and put the last el in place, efficient but
does not preserve ordering

`Vec::set_len` is unsafe way to extend vec into maybe uninit'd memory, useful
when working with C FFIs where C might write into the raw ptr and we can set_len
to turn it into rust vec.

`Vec::retain` removes items that doesn't match the passed in closure. It's more
efficient than looping and removing.

`Vec::leak` will give a ref to a slice that is convenient to be passed around
and read in multiple places. 

`Vec` impls `Extend`, which is smart about the number of els to be extended
given a vector or an iter with known num els.

`small_vec` crate allows some els to be kept on the stack to sometimes avoids
heap alloc. However the stack els will be copied when vec passed by ownership.

Rust `Vec` layout is C compatible.

## VecDeque

Double ended queue ring buf.

`VecDeque::as_slices` return two slices (start to physical end, wrapped around).

There are `binary_search` and `binary_search_by` methods that are handy.

`rotate` for `VecDeque` is more efficient than `Vec`

`make_contiguous` ensures no wrap-around part after that.

## LinkedList

It's more efficient to embed the links in the underlying struct. However it's
annoying to make the borrow checkers not complain when implementing doubly ll by
one self. The prev and next ptrs are private in LinkedList, which is
inconvenient. The only interface exposed are `cursor_front` methods that allows
walking.

For the above reason this is rarely used in the collections.

## HashSet, BTreeSet

Are maps where the values are ZSTs. 

## HashMap, BTreeMap

`shrink_to_fit` will make the physical size fit to the logical size. Hashing
function might change.

Also have `with_capacity` to predetermine the number of buckets.

`entry` interface gives the bucket for the entry that could be vacant or
occupied. This avoids two rounds of bucket lookup when conditional insert. This
is doable with `map.entry(key).or_insert(value)`.

BTreeMap doesn't need to resize to fill factor so is generally more mem
efficient. It has good lookup interfaces like `push_first`, `pop_last`,
`split_at`, `range_mut`, `cursor_front` etc.

## Binary Heap

A priority queue. Is more efficiently implemented than `BTreeMap` for getting
min and max. It also allows duplicates.

## Misc

`Option` is a vec of capacity 1. `Result` is a map from bool to bigeneous type.
They are like collections.

`collect` method allows creating a collection from something. Can also collect
into `Option` and `Result`.

It's legal to do `let x: Option<Vec<usize>> = (0..10).map(Some).collect()` where
x will be `None` if any of the els are `None`. Similar applies to `Result`. This
is related to the `FromIterator` trait.

collecting results into a `Result<()>` will discard the `Ok`'s inner values but
produce a `Ok` iff all collected results are `Ok`. Useful when the inner value
is not needed but still need to check whether ops all succeeded.

For concurrent collections impl, `dashmap` `evmap` `hashbrown` provides
hashmaps, `crossbeam` contains some. There's also `indexmap`.
