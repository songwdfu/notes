## Lifetime

## Misc

### ref keyword

`ref` keyword let you take ref in pattern matching. It is the inverse of `&mut`
```
if let Some(ref mut foo) = bar /* Option<T> */
```
Then foo will be a &mut T

### working with `Option`

`Option<T>::take` will set the original `Option` to `None` and return the `Some`
if there was one.

`?` works for `Option<T>` as well. But it's better write
```
if let Some(ref mut foo) = self.opt {
    ...
} else {
    ...
}
```

For the below snippet, if `T impl Copy`, then `foo` will be pointing to a
**Copy** of the `T` inside `self.opt`'s `Some(T)`, but not `self.opt`.
``` 
let ref mut foo = self.opt? /* Option<T> */ 
```
To do that, use `Option<T>::as_mut() -> Option<& mut T>`
```
let ref mut foo = self.opt.as_mut()?
```

## Lifetime annotation

Annot after impl makes impl block generic, the 'a after MyStruct is the annot
```
impl<'a> Iterator for MyStruct<'a> {
    type Item = &'a str;
    fn next(&mut self) -> Option<Self::Item> {...}
}
```

The `'_` notation is either some other lifetime or inferred lifetime. The below
snippet, return value has lifetime of **x**!!! Because `y: '_ u8` has some other
lifetime and the return type has inferred lifetime that could only be `x`'s
```
fn mul(x: u8, y: '_ u8) -> '_ u8 {...}
```

Only have multiple lifetimes when storing multiple refs who have different
lifetime. If only one lifetime is annotated, when using, it will be interpreted
as the shorter lifetime of the args passed in. 
In the below snippet, if `StrSplit<'a>` has one annot, `s` and `c` will have the
same lifetime, that is `c`'s bc it is shorter. So it should really be
`StrSplit<'a, 'b>` and the impl should specify `type Item = &'a str;`
```
fn match_char(s: &str, c: char) -> &str {
    StrSplit::new(s, &format!("{}", c))...
}
```

Can specify relationship between lifetimes by 
```
struct Foo<'a, 'b> {
    x: &'a str,
    y: &'b str,
}

impl<'a, 'b> Foo<'a, 'b> 
where
    'a: 'b
{
    ...
}
``` 
This means `'a` impls `'b`, so `'a` is at least `'b` (then a > b).

If something we don't care we can elide the lifetime
```
impl <'a> Foo<'a, '_> { ... }
```

## String

`str` is similar to `[char]`, a seq of chars. `&str` is a fat pointer that
stores both the start and len, same as slice
`String` is similar to `Vec<char>`, heap alloc, can shrink and grow and Vec.

`String -> &str` is trivial (`AsRef`). 
`&str -> String` can only heap alloc and memcpy

storing struct field as `String` is really bad, as it would require an
allocator! Should store `&str` instead

Better yet, we can store some generics instead of `&str` so we don't need to
alloc using `format!` to turn char into `String` and then `&str`.

Interesting snippet for finding char in string
```
impl Delimiter for char {
    fn find_next(&self, s: &str) -> Option<(usize, usize)> {
        s.char_indices()
            .find(|(_, c)| c == self) /* find indice and char where char == self */
            .map(|(start, _)| (start, start + self.len_utf8())) /* map indice to tuple */
    }
}
```
`String::find(s: &str)` returns first indice of appearence of `s`
