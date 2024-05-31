# `val` declarations in structures

## Context

Advanced OCaml features often require writing type
annotations. However, writing type annotations on definitions today
can be awkward, as it forces to write the rest of the definition in
a different style than we would normally use: adding an annotation on
a function definition

```ocaml
let f arg1 arg2 arg3 =
  <body>
```

typically requires adding the annotation after `f`, in the form `:
<type> =` and then adding a `fun`, and then turning the trailing equal
sign into a right-arrow (and making an awkward decision on
indentation):

```ocaml
let f : <type> =
  fun arg1 arg2 arg3 ->
    <body> (* extra indentation *)

(* or possibly *)
let f : <type>
= fun arg1 arg2 arg3 ->
  <body>
```

(Note: we are not talking about per-argument annotations here. They can
be simpler in some cases, but they cannot be used in others, for
example when introducing polymorphic recursion due to GADTs. Generally
their readability is worse than full-declaration annotations in
complex cases.)


## Proposal

We propose to allow using `val` declarations immediately before definitions.

```ocaml
val map : ('a -> 'b) -> 'a list -> 'b list
let rec map f li = ...
```

### Recursive bindings

In a nest of mutually-recursive bindings, each binding may or may not
have a declaration, but the declarations must all come together before
the `let rec` block.

```ocaml
val even : int -> bool
val odd : int -> bool
let rec even n = ...
and odd n = ...
```

(It is not allowed to mix declarations and definitions, because there
is no obvious syntax for this, although `and val` could be
considered. See "Alternative syntax: declarations with `let` blocks"
below for a form that allows this.)

### Local bindings

The `let <structure item> in <expr>` form of
[#14040](https://github.com/ocaml/ocaml/pull/14040) is extended to
cover local declarations:

```ocaml
let rev li =
  let val loop : 'a list -> 'a list -> 'a list in
  let rec loop li acc = ... in
  loop li []
```

## Discussion and extensions

### Locally abstract types in `val` declarations

We propose to extend the form `type a . ...` to work with value
annotations. For example:

```ocaml
val mem : type a . a -> a list -> bool
let mem elt li =
  let rec loop : a list -> bool = function
  | [] -> false
  | x :: xs -> (x = elt) || loop xs
  in loop li
```

Notice that `type a` binds the variable `a` in both the declaration and the definition.

(This extension could be left out of a first implementation of this proposal.)


### Alternative syntax: declarations within `let` blocks

An alternative syntax would be possible where `val` is part of the `let` block, as follows:

```ocaml
let rec
  val map : ('a -> 'b) -> 'a list -> 'b list
  and map f li = ...
```

This syntax is weirder (it feels closer to SML), but it scales
slightly better to mutually-recursive or local definitions:

```ocaml
let rec val f : <type>
and val g : <type>
and f <def>
and g <def>
and val h : <type>
and h <def>
and i <def>
```

```ocaml
let rev li =
  let rec
  val loop : 'a list -> 'a list -> 'a list
  and loop li acc = ...
  in loop li []
```

### Combination with `_` inference from signature

oxcaml has a work-in-progress feature where `_` can be used to elide types and module signatures in structures (in particular `.ml` files), when they are declared in the corresponding signature (in particular the `.mli` file), see https://github.com/oxcaml/oxcaml/pull/2783 . This feature is independent, but it was part of the motivation to revive the current proposal, as it can naturally be combined:

```ocaml
val map : _
let rec map f li = ...
```
