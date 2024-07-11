# Primitive aliases

## Context

OCaml supports both a number of built-in primitives as well as the ability to call
functions written in C. Both of these are introduced via the `external` keyword, e.g.

```ocaml
external magic : 'a -> 'b = "%identity"
external reachable_words : t -> int = "caml_obj_reachable_words"
```

Simple examples such as those above behave much like regular OCaml functions, with some
special handling in the compiler. However, as discussed in
[manual section 11](https://ocaml.org/manual/5.2/intfc.html#s:C-cheaper-call) "cheaper C calls", C call
primitives may have more than one symbol associated with them, as well as a number of
attributes on argument and return types, such as with `Float.ldexp`:

```ocaml
external ldexp
  :  (float[@unboxed])
  -> (int[@untagged])
  -> (float[@unboxed])
  = "caml_ldexp_float" "caml_ldexp_float_unboxed"
[@@noalloc]
```

In order for the compiler to utilize this information and call the unboxed version, the
primitive must be exposed with all attributes and C symbols in both structures and
signatures, though this can be overcome with sufficient inlining.

## Motivation

It's often useful to simply re-bind a value (e.g. with a different name), but there is
currently no way to do re-bind a _primitive_ without wrapping it in an OCaml value and
destroying any additional information. Consider a module such as:

```ocaml
module Float : sig
  external modfp :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
  val ( % ) : float -> float -> float
end = struct
  external modfp :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
  let ( % ) = modfp
end
```

We can only rely on the compiler to pick the unboxed C call (when applicable) for
`Float.( % )`, and not for `Float.O.( % )`, because it is not exposed as a primitive.
Thus, users may prefer to explicitly write out the entire `external` description for every
binding, like so:

```ocaml
module Float : sig
  external modfp :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
  external ( % ) :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
end = struct
  external modfp :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
  external ( % ) :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
end
```

This is obviously duplicative, verbose, and prone to inconsistency.

## Proposal

We will extend the syntax for `external` to support _aliases_, allowing one to re-bind a
primitive without syntactic overhead:

```ocaml
module Float : sig
  external modfp :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
  external ( % ) = modfp
end = struct
  external modfp :  float -> float -> float = "modfp" "modfp_unboxed" [@@unboxed]
  external ( % ) = modfp
end
```

We do not anticipate any difficulties in parsing this construct. We propose extending the
AST with a new `primitive_description` record, rather than continuing to represent
primitives via `value_description`, and adding a `Psig_primitive` constructor to
`signature_item_desc`.

Typing such aliases seems similarly straightforward: we can simply look up the primitive
bound to the symbol on the right-hand side in the current environment, recovering all the
same information as if were written out in full.
