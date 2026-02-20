# Labeled Tuple Projections 

## Overview 

This RFC proposes a quality-of-life improvement to OCaml's tuples, adding
support for labeled tuple projections. 

## Proposed change 

The idea is to allow users to directly project elements from a labeled tuple 
using labels, as opposed to patterns: 

```ocaml 
# let x = (~tuple:42, ~proj:1337, "is", 'c', 00, 1);;
val x : (tuple: int * proj:int * string * char * int * int) = 
  (~tuple:42, ~proj:1337, "is", 'c', 0, 1)
# x.~tuple;; 
- : int = 42
```

Here, we're able to project out of a 6-tuple (containing both labeled and
unlabeled components) simply by writing `x.~l` (for a label `l`).  

This is useful for a couple reasons: 
- Clarity: `x.~label` is more readable than `let (~label, ..) = x in ...`
- Parity with records: complements record field projection 

**A historical note**: an earlier proposal also explored projections 
for *unlabeled* tuples, along with an empirical analysis of projection 
patterns within the ecosystem. Since labeled tuples are a recent addition,
a similar analysis is not yet useful for motivating this feature. 

## Previous work 

Many other strongly-typed languages support built-in tuple projections.

### SML 

SML models tuples as records with integer field names (1-indexed), so projection uses
record selection syntax:
```sml 
> val x = (1, "hi", 42);;
val x = (1, "hi", 42): int * string * int
> val y = #1 x;;
val y = 1: int
> val z = #3 x 
val z = 42: int
```

### Rust 

Rust supports tuple projections (0-indexed) for ordinary tuples (and tuple structs): 
```rust 
let x = (42, "is", 'c'); 
let y = x.0; // 42
let z = x.1; // "is"

struct Point(i32, i32);
let p = Point(3, 4);
let x_coord = p.0; // 3
```

Record structs also use the same syntax for projection: 
```rust 
struct Point { x : i32, y : i32 };
let p = Point { x = 3, y = 4 }; 
let x_coord = p.x; // 3 
```


### Swift 

Swift supports tuple projections via both positional indicies and labels (as in this proposal): 
```swift 
import Foundation 

let x = (tuple: 42, proj: 1337, "is", 'c', 00, 1); 

print(x.tuple) // 42
print(x.5) // 1
```

## Implementation 

An experimental implementation is available at [PR 14257](https://github.com/ocaml/ocaml/pull/14257).

### Parsetree changes 

Labeled tuple projections would introduce a new `Pexp_proj (e, l)`
former for `expression` to represent `e.~l`.

### Typechecking

While typechecking, when encountering a labeled tuple projection in expressions 
`e.~l`, check to see whether the expected type is known: 
- If the type is known to be `(?l0:ty0 * ... * l:tyl * ... * ?ln:tyn)`: type the projection 
as `tyl`. 
- Otherwise, raise a type error.

## Considerations

### Limitations of type-based disambiguation

OCaml's current type-based disambiguation mechanism is relatively weak. As a result, 
many of the patterns that tuple projections are intended to replace would be ill-typed under 
today's implementation. For instance: 
```ocaml 
# List.map (fun x -> x.~num) [(~num:42, "Hello"); (~num:1337, "World")];;
Error: The type of the tuple expression is ambiguous.
       Could not determine the type of the tuple projection.
```

That said, this limitation does not arise from the feature itself, but from the
weaknesses in OCaml's type propagation. Improving type propagation (separately)
would benefit not only tuple projections, but other features that rely on
type-based disambiguation (e.g. constructors and record fields). As such, we
argue that tuple projections should not be rejected on this point alone, and
that the broader issues of type propagation and disambiguation be addressed
separately. 

### Syntactic overloading

An alternative proposal could reuse the existing projection syntax `e.l` for 
both record fields and labeled tuples. The downside is that it increases reliance 
on type-based disambiguation and introduces particularly nasty error messages.

