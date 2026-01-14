# Iterator

e.g. usage of iterator
```
while let Some(e) = iter.next() { }
```

`trait Iterator`, impl fn `next`, etc.
`trait IntoIterator`, impl fn `into_iter`. 

Iterator has associated `type Item`, it is not generic. Because iterator only
have one impl that make sense for this type. Only use generics when there are multiple
possible implementations for each type.

Watch out borrow or owned when using for
```
let vs = vec![1, 2, 3];

for v in vs {
    // consumes vs! gives owned v
}

for v in &vs {
    // borrors vs, & to v
}

for v in vs.iter() {
    // borrors vs, & to v
}
```

`Iterator::flatten` works for when `Item impl IntoIterator` and it will flatten
the nested items into a `Flatten<T>`

```
/// An extention trait allows object of extended trait to call this extra method
pub trait IteratorExt: Iterator {
    // this is the fn name for anything that impl Iterator and hence IteratorExt
    fn flatten(self) -> Flatten<Self>
    where 
        // bc Flatten<Self> stores self in the struct, Self has to be Sized
        Self: Sized,
        Self::Item: IntoIterator;
}

/// Default impl of IteratorExt for obj that already impl IntoIterator
impl<T> IteratorExt for T
where
    T: Iterator,
{
    fn flatten(self) -> Flatten<Self>
    where 
        Self: Sized,
        Self::Item: IntoIterator,
    {
        flatten(self)
    }
}

/// The IntoIter type is what iter is turned into
pub fn flatten<I>(iter: I) -> Flatten<I::IntoIter>
where
    I: IntoIterator,
    I::Item: IntoIterator,
{
    Flatten::new(iter.into_iter())
}

struct Flatten<O> 
where 
    O: Iterator,
    O::Item: IntoIterator,
{
    outer: O,
    inner: Option<<O::Item as IntoIterator>::IntoIter>,
    /* the iterator type turned into */
}

impl<O> Flatten<O> 
where 
    O: Iterator,
    O::Item: IntoIterator,
{
    fn new(iter: O) -> Self {
        Flatten{outer: iter, inner: None}
    }
}

// Here O::Item is matrix[i]'s type, should impl IntoIterator
// <O::Item as IntoIterator>::Item is matrix[i][j]'s type
impl<O> Iterator for Flatten<O> 
where
    O: Iterator,
    O::Item: IntoIterator,
{
    type Item = <O::Item as IntoIterator>::Item;
    fn next(&mut self) -> Option<Self::Item> {
        loop {
            if let Some(ref mut inner_iter) = self.inner {
                if let Some(i) = inner_iter.next() {
                    return Some(i);
                }
                self.inner = None
            }
            
            /* None if outer.next() is None, else outer.next().into_iter() */
            let next_inner_iter = self.outer.next()?.into_iter();
            self.inner = Some(next_inner_iter);
        }
    }
}
```

Extension trait extends the base trait with one that only contains the extra
method to be added and has a default impl for any type that impls base trait.

Watch out when something is stored in the struct it might have to be `Sized` so
the compiler knows. 
