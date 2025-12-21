# Declarative Macros

```
vec![1; 3]; /* heap allocated vector of 3 ones */
```

declaring macros
```
macro_rules! vec {
    ($ elem : expr ; $ n : expr) => { ... };
    /* pattern */ => /* expansion */
}
```

pattern: $ name : ty / expr / path / ident ...

"The little book of rust macros" a book on this

The delimiter can be () or [] or {}

Ident declared in macro field is distinct form program world.

macro that matches multiple expressions
```
macro_rules! avec {
    /* first case, empty vec, {} is expansion */
    () => {
        Vec::new()
    }; 

    /* + means multiple times, ? means 0 or 1 times */
    /* $(,)? allows trailing commas */
    ($( $element : expr ),+ $(,)?) => {{ 
        let mut vs = Vec::new();
        /* for each expression that contains $element, do this. 
         * The plus means as many time as num elements */
        $(vs.push($element);)+ 
    }}
}
```

macro rules are useful when impl trait for multiple things
```
macro_rules! max_impl {
    ($t:ty) => {
        impl $crate::MaxValue for $t {
            fn max_value() -> Self {
                <$t>::MAX    
            }
        }
    }
}
```
`$crate` is the crate that the macro was defined in.

When testing macro can compile, write doc tests, it will check the code in the
comment block can compile
```
/// ```compile_fail
/// let x: Vec<u32> = mymodule::avec![42; "foo"];
/// ```
#[allow(dead_code)]
struct CompileFailTest;
```
```
 
Optimization for `vec![n, x]` creation: instead of initing a `Vec::new()` and
push to it many times, reserve capacity, avoid ptr increment and hence bound
check by
```
let mut vs = Vec::with_capacity(count);
vs.extend(std::iter::repeat(element).take(count));
```
Here `std::iter::repeat` creates an iterator that gives clone of element for each
time the iterator it `take`n.

Careful for not evaling expr multiple times when defining

Hack for counting: avoid evaluating expression multiple times by using private
macros to substitute expr with unit.
```
macro_rules! avec {
    /* case 1, exprs delimited by comma */
    ($( $element : expr ),*) => {{ 
        let mut vs = Vec::with_capacity($crate::avec![@COUNT; $(element),*]);
        /* for each expression that contains $element, do this. 
         * The plus means as many time as num elements */
        $(vs.push($element);)*
    }};

    /* case 2, trailing comma, convert to case 1 */
    ($( $element: expr, )*) => {{
        $crate::avec![$(element),*]
    }};

    /* case 3, [num; elem] syntax */
    ($element: expr; $count: expr) => {{
        let mut vs = Vec::new();
        vs.resize($count, $element);
        vs
    }};

    /* private macro patterns for counting */

    (@COUNT; $(element: expr),*) => {
        // [$($element),*].len() this is bad, element eval'd multi times.
        // Below, <[()]> is the type "slice of units".
        // The expr in the len() substitutes each element with a unit so 
        // expr is not eval'd and the resulting [()] is zero-size and wouldn't 
        // be allocated anywhere. But rust still recognizes which input to 
        // count because $element is used in the below line.
        <[()]>::len([$($crate::avec![@SUBST; $element]),*])
    };

    (@SUBST; $element: expr) => {
        ()
    };
}
```
