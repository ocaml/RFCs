# Type-directed Array Disambiguation

This RFC describes a way we can use literal syntax for array-like structures
other than the built-in `array` type, hooking into the existing support for
type-directed disambiguation of constructors.

## Motivation

OCaml currently supports two different array-like structures, `array`, and
`floatarray`. There are discussions about adding [immutable
arrays](https://github.com/ocaml/ocaml/pull/13097), and I believe several people
have expressed a desire for compiler support for uniform arrays (which do not
participate in the float-array opimization). (This is just hearsay; I do not
have a reference.) Yet only `array` has first-class syntax, available both for
construction and for pattern-matching. This RFC allows us to use this nice
syntax for these other structures, making it easier to adopt these other types
in code.

We could also imagine extending this syntax to cover `string` and `bytes`, but
this proposal as written does not do so.

## Proposal

During type-checking, when encountering array-literal syntax, in expressions or
in patterns, check to see whether the expected type is known (principally known,
in `-principal` mode). Then:

* If the type is not known, the literal has type `'a array`, as now.
* If the type is known to be ...
  * ... `_ array`, the literal is treated as it is now.
  * ... `floatarray`, the literal is treated as a `floatarray`.
  * any other type, error.
  
If the language begins to support immutable or uniform arrays (or immutable
`floatarray`s or immutable uniform arrays), then disambiguation would work for
these structures, too.

## Discussion

* This disambiguation does *not* extend to lists: list literals are just syntax
for nested calls to `(::)` and `[]`, which can be disambiguated through the
usual mechanisms for constructors.

* This allows us to add new array-like structures without worrying about
  concrete syntax.
  
* The `[| ... |]` syntax behaves very much like a constructor, available both in
  expressions and in patterns. Extending type-directed disambiguation in this
  way seems like a natural step.
  
* This disambiguation does *not* extend to array access operators `.()` and
  `.()<-`. The former is currently an abbreviation for `Array.get` (whichever
  `Array.get` is in scope) and the latter is an abbreviation for `Array.set`
  (whichever `Array.set` is in scope). In addition, the language already
  supports binding new array-access operators, with (mostly) arbitrary symbols
  between the `.` and the brackets. These existing mechanisms serve well to
  control the meaning of access operators, and thus this RFC does not add to
  them.
  
## Implementation

There is currently no implementation of this feature (even in prototype). Jane
Street will either implement or will fund the implementation, if this RFC is
accepted. If implementation / testing reveals unexpected interactions /
complications, we would naturally expect to re-engage with the community for
discussion of whether this is, in practice, a good idea. (That is, we propose
that acceptance of this RFC is conditional on having a smooth implementation
experience.)
