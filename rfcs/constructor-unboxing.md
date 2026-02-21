# Constructor unboxing

### Motivating example: compact rope representation

Data structures defined in OCaml are often less compact than they might be, because of boxing.

For example, here is a type for representing [ropes](https://onlinelibrary.wiley.com/doi/abs/10.1002/spe.4380251203):

```ocaml
type rope = Leaf of string
          | Branch of { llen: int; l:rope; r:rope }
```

With this definition the value `Branch {llen=3; l=Leaf "abc"; r=Leaf "def"}` has the following representation:


```
[--B|-3-|-∘-|-∘-]
         /     \
        /       \
       /         \
      [--L|-∘-]   [--L|-∘-]
           /             \
          /               \
         "abc"             "def"
```

In the general case each part of this representation serves a purpose.  For example, in order to distinguish `Leaf` nodes from `Branch` nodes at run-time, each constructor is represented by a tagged block.  However, for this particular data type the block representing `Leaf` nodes (`[--L|-∘-]`) is unnecessary; since strings are already distinguishable from other blocks, the value could in principle be represented more compactly:


```
[--1|--3|-∘-|-∘-]
         /     \
        /       \
       /         \
       "abc"      "def"
```


### The basic idea 

Adding an `[@@unboxed]` annotation to a variant definition indicates that the representation of unary constructors should not involve an additional block:

```ocaml
type rope = Leaf of string
          | Branch of { llen: int; l:rope; r:rope } [@@unboxed]
```

Only a subset of variant definitions support `[@@unboxed]`.  In particular, it must be possible to distinguish the arguments of unary constructors from each other (and from constant constructors in the same definition) at run-time.  For example, the following definition is not allowed, since the arguments of `X` and `Y` have the same representation:

```ocaml
type strings = X of string | Y of string [@@unboxed] (* Invalid! *)
```

### Performance improvements

Since the `unboxed` variant representation uses less allocation and less indirection, it improves performance in some cases.

For example, here is a simple benchmark for the rope data type.  The benchmark creates rope representations of size `n`, converting the ropes to strings in a final step:

```ocaml
  let rec build n =
    if n = 1 then leaf "a"
    else let llen = succ (Random.int (pred n)) in
         branch (build llen) (build (n - llen))
  in
    string_of_rope (build n)
```

Measurements show substantial improvements for the unboxed representation, especially for larger values of `n`:


| **Size** 	| **Boxed (μs)** 	| **Unboxed (μs)** 	| **Unboxed %** 	|
|---------:	|:-------------:	|:--------------:	|:-------------:	|
|      2^6 	|    3.90       	|     3.81      	|      97.7     	|
|      2^8 	|   15.99       	|    15.38      	|      96.2     	|
|     2^10 	|   64.82       	|    62.32       	|      96.1     	|
|     2^12 	|   281.13      	|    257.30      	|      91.5     	|
|     2^14 	|  1560.99      	|   1220.11      	|      78.1     	|
|     2^16 	|  10089.72     	|   5332.93      	|      52.9     	|
|     2^18 	|  50027.06     	|   35030.16      	|      70.0     	|

(These measurements were taken by building the unboxed representation explicitly using `Obj` rather than actually implementing this proposal in the OCaml compiler.)

### Which types are distinguishable?

Eliminating the block associated with a constructor is safe only when:

1. the representations of distinct constructors remain distinguishable at run-time
2. no type can represent both non-float and float values (to avoid problems with the [float array unboxing optimization](https://www.lexifi.com/ocaml/about-unboxed-float-arrays/))

This proposal currently focuses on concrete types; it may be extended to abstract type constructors and existential variables by building on the notion of *separability* introduced in [Unboxing Mutually Recursive Type Definitions in OCaml](https://arxiv.org/pdf/1811.02300.pdf) (Colin, Lepigre, Scherer, JFLA 2019).

Consider the definition of a data type `t` with unary argument types `t1`...`tn` and constant constructors `C1`...`Cn`:

```ocaml
type t = C1 | ... | Cn | T1 of t1 | ... Tn of tn
```

The simplest case where constructor unboxing is clearly safe is where `t1`...`tn` all have non-immediate and distinguishable representations, and none of them is either `float` or `t`.

However, constructor unboxing is also possible if some of the `t1`...`tn` have immediate representations.  For example, consider the following definition:

```ocaml
type u = C1 | C2 | C3 of bool | C4 of char
```

In OCaml `C1`, `C2`, `bool` and `char` are all represented as immediates, and a single immediate value (i.e. an `int`) can easily represent all of these values.  We can therefore adopt the following representation:

| **Value** 	| **Representation** 	|
|----------:	|:-----------------:	|
| `C1`      	| `0`               	|
| `C2`      	| `1`               	|
| `C3 false` 	| `2`               	|
| `C3 true` 	| `3`               	|
| `C4 c`    	| `4 + Char.code c`  	|

More generally, we can unbox the definition `t` if:

 * the *block-representations* of `t1`...`tn` are all disjoint (and distinct from `float` and from `t` for non-unary constructors)

 * the *immediate-representations* of `t1`...`tn` together with `C1`...`Cn` cover less than the *immediate space*.

(Here *block-representation* indicates the *set* of tags that non-immediate values of a type can have, and *immediate-representation* indicates the *number* of distinct immediate values of a given type.  *Covering less than the immediate space* means that the sum of the immediate representations of `t1`...`tn` is less than `max_int`.)

### How does this relate to the existing [@@unboxed] annotation?

This proposal is a conservative extension of the existing `[@@unboxed]` annotation (PRs [#606](https://github.com/ocaml/ocaml/pull/606), [#2188](https://github.com/ocaml/ocaml/pull/2188)).  It has no effect on existing code, and the extended meaning of `[@@unboxed]` is compatible with the existing meaning.

### How does this relate to the existing proposal for unboxed types?

[Another open proposal](https://github.com/ocaml/RFCs/pull/10) involves unboxing field types into parent types --- for example, producing flat representations of `int32` pairs:

```ocaml
type t = { x : #int32; y : #int32 }
```

The two proposals are distinct, but complementary.  The field-unboxing proposal does not support the compact rope representation, the current proposal does not support pair unboxing, and the combination of the two proposals can support representations that are more compact than either proposal supports in isolation.  In the following example, the `int32` arguments are unboxed into a single block using field unboxing, and the block associated with the `Pair` constructor is eliminated using constructor unboxing:

```ocaml
type t = { x : #int32; y : #int32 }
type topt = Nopair | Pair of t [@@unboxed]
```

Without any support for unboxing, the value `Pair {x=0l; y=0l}` involves 4 blocks; with field unboxing alone it involves 2 blocks; with constructor unboxing alone it involves 3 blocks; with both field and constructor unboxing it involves 1 block.

(There are some additional connections and distinctions between the two proposals.  The current proposal can be seen as a generalization of the *nullable types* mentioned in the field unboxing proposal.  And the field unboxing proposal is more ambitious: it additionally introduces compound unboxed types and changes to the type system to combine unboxing with abstraction.)

### Extension: partial unboxing

The current proposal requires all unary constructor arguments to have distinguishable representations.  It might also be useful to support the case where only some of the arguments are distinguishable by allowing per-constructor unboxing specifications.

### Comment: abstraction

Unboxability (for records) is a property of a *single* type, and is fairly straightforward to abstract.

However, separability is a relation *between* types, and is not so easily abstractable.

One possibility is to optionally expose the *immediate-space* and the *tag-space* of abstract types by extending the [`[@@immediate]` attribute](https://github.com/ocaml/ocaml/pull/188).  For example, we might promise that a type has an immediate representation with *no more than* 256 distinct values:

```ocaml
type t [@@immediate 0..255]
```

We might additionally support setting the tag value associated with particular constructors explicitly to avoid clashes:

```ocaml
type t = { x: int; y: float} [@@tag 240]
```
