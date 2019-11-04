# Nominal (abstract) types

## Summary

GADTs have revealed some subtle problems relative to the injectivity and identity of types [#5985](https://github.com/ocaml/ocaml/issues/5985).
This RFC proposes to solve both problems by allowing one to mark some abstract types as nominal, implying injectivity for all type parameters, and with an optional name, allowing to know for sure that two types are different.

```ocaml
type 'a u [@@nominal]
type 'a t [@@nominal "M.t"]
```
Here `u` is injective, and `t` is both injective and incompatible with all nominal types whose name is not `"M.t"`. (But not the converse, i.e. two types with the same name `"M.t"` not be equal.)
The name _nominal_ comes from the fact these types behave like nominal types, which in ocaml stands from variant and record types.

This is useful for two things:
* Allowing to use (parametric) abstract index types in GADT definitions. Currently such definitions are rejected as non-injective.
* Making the the exhaustiveness check more precise, bit having a way to deduce that some abstract types are incompatible with other types. Currently this is only allowed for predefined types (hard-coded in the compiler) and types defined in the current module (arguably a bad design choice).

An experimental implementation is available as PR [#9042](https://github.com/ocaml/ocaml/pull/9042).

## Motivation

Here are some common problems with GADTs and abstract types:

```ocaml
module M : sig type +'a t end = ...
type _ ty = M : 'a ty -> 'a M.t ty;;
Error: In this definition, a type variable cannot be deduced
       from the type parameters.
```
Since abstract types are not injective, one cannot have a type variable appear under an abstract type inside a type parameter. This would allow unsound uses.

```ocaml
module M : sig type u end = ...
type _ t = Int : int t | U : M.u t
let f : int t -> int = function Int -> 3;;
Warning 8: this pattern-matching is not exhaustive.
Here is an example of a case that is not matched:
U
val f : int t -> int = <fun>
```
Here the definition of `u` is unknown, so there is no way to prove that it is incompatible with `int`, hence the warning.

```ocaml
module Vec = struct
  type zero
  type 'n succ = Succ
  type ('a,'n) vec =
    | Nil : ('a, zero) vec
    | Cons : 'a * ('a,'n) vec -> ('a,'n succ) vec 
  let head : ('a,_ succ) vec -> 'a = function Cons (h,t) -> h
end;;
let head : ('a,_ Vec.succ) Vec.vec -> 'a = function Vec.Cons (h,t) -> h;;
Warning 8: this pattern-matching is not exhaustive.
Here is an example of a case that is not matched:
Nil
val head : ('a, 'b Vec.succ) Vec.vec -> 'a = <fun>
```
Here, while the definition of `head` inside `Vec` is seen as exhaustive, the one outside of it is not exhaustive, because `zero` becomes compatible with `succ`.
Moreover, it was necessary to turn `succ` into a pseudo variant types to make it injective.

There exists workarounds, of various costs. For index types representing values, like in the last case, the fix is easy: one could just turn them into variant types (`type zero = Zero`), since their value constructors do not matter. This ensures both injectivity and a form of identity, since constructor names allow to distinguish types.

If the type used as index has some real contents, as in the first case, and possibly the second one, this is much heavier, and not very natural:
```ocaml
module M : sig type +'a t0 type 'a t = M_t of 'a t0 end = ...
```
I.e., one has to explicity wrap the internal `t0` inside a new type `t`, and manually cast between the two types in the body of `M`. 

There used to be branch of the compiler [abstract-new](https://github.com/ocaml/ocaml/tree/abstract-new), implementing a variation of this called _new types_, and relying on implicit coercions to avoid manual casts.
Here are some [examples](https://github.com/ocaml/ocaml/blob/abstract-new/testsuite/tests/typing-private/new.ml).
This attempts to solve the same problem, but seems over-engineered.

The present RFC could solve all these problem using nominal annotations alone.

## Specification

While the goal of this extension is to annotate abstract types, for soundness we also need to annotate their implementations, which may be either purely abstract (i.e. external custom types) or variant or record types.

This gives us the following forms of nominal types, both in interfaces and implementation.
```ocaml
type 'a u [@@nominal]
type 'a t [@@nominal "M.t"]
type 'a v = A of t1 | B of t2 [@@nominal "M.v"]
type 'a v = 'a Z.t = A of t1 | B of t2 [@@nominal "M.v"]
type 'a r = {f1:t2; f2:t2} [@@nominal "M.r"]
type 'a r = 'a Z.u = {f1:t2; f2:t2} [@@nominal "M.r"]
type 'a e = .. [@@nominal "M.e"]
type 'a e = 'a Z.v = .. [@@nominal "M.e"]
```
While it is allowed to annotate non-abstract types with just `[@@nominal]`, this is not particularly useful, as they are already nominal.

Two relations are important for nominal types: (module level) subtyping, and compatibility.

### Subtyping
Subtyping is reflexitive and transitive, and includes the following rules:
```
type 'a u [@@nominal] <: type 'a u
type 'a u [@@nominal "s"] <: type 'a u [@@nominal]
type 'a u = 'a t <: type 'a u [@@nominal "s"] if type 'a t <: type 'a u [@@nominal "s"]
type 'a u = 'a t <: type 'a u [@@nominal] if type 'a t <: type 'a u [@@nominal]
type 'a v = A [@@nominal "s"] <: type 'a v [@@nominal]
type 'a v = A <: type 'a v [@@nominal]
type 'a v = 'a w = A [@@nominal "s"] <: type 'a v = A [@@nominal s]
```
(rules for records and extension types are identical)
Note that in all cases arity and parameter order must be preserved.

In particular, note that the following are _not_ allowed:
```
type 'a u <: type 'a u [@@nominal]                     (* introduction of [@@nominal] *)
type 'a u [@@nominal "s"] <: type 'a u [@@nominal "t"] (* change of name *)
type 'a v = A [@@nominal "s"] <: type 'a v = A         (* forget name of datatype *)
```

### Compatibility
Nominal types with different names are incompatible:
if `type t [@@nominal "t"]` and `type u [@@nominal "u"]` then `t` and `u` are incompatible.
Nominal types with an explicit name are incompatible with variant, records, or extension types without a name:
if `type t [@@nominal "t"]` and `type v = A` then `t` and `v` are incompatible.

The latter one is the reason we cannot forget a name on a variant type, without forgetting the definition.

It is also sound to say that two nominal types with different arities are incompatible. (The current implementation has a special case to allow keeping the name when exporting a nullary abbreviation; it is unclear whether this is worth the extra complexity.)

### Injectivity
All nominal types are injective.
I.e., if `type 'a t [@@nominal]`, then `t1 t = t2 t` implies `t1 = t2`.

Moreover, it is sound to extend this to types with the same name:
if `type 'a t [@@nominal "t"]` and `type 'a u [@@nominal "t"]`,
then `t1 t = t2 u` implies `t1 = t2`.
(Either `t = u`, and the implication is valid, or `t != u`, and the assumption is a contradiction)

### Abstract types

Unnannotated abstract types become compatible independently of their location.
Predefined abstract types would be made nominal explicitly in the initial environment, without using a special case during unification:
```ocaml
type 'a array [@@nominal "array"]
```

## Examples

### Injectivity
```ocaml
module M : sig type +'a t [@@nominal "M.t"] end =
  struct type 'a t = Nil | Cons of 'a * 'a t [@@nominal "M.t"] end
type _ ty = M : 'a ty -> 'a M.t ty;;
```

### Incompatibility
```ocaml
module M : sig type +'a t [@@nominal "M.t"] val create : 'a list -> 'a t end = struct
  type 'a t = Nil | Cons of 'a * 'a t [@@nominal "M.t"]
  let rec create = function [] -> Nil | a::l -> Cons (a, create l)
end
type _ exp = M : 'a list -> 'a M.t exp | Int : int -> int exp
let eval_int : int exp -> int = function Int x -> x;;
```

### Index types
```ocaml
type zero [@@nominal "zero"]
type 'n succ [@@nominal "succ"]
type ('a,'n) vec =
  | Nil : ('a, zero) vec
  | Cons : 'a * ('a,'n) vec -> ('a,'n succ) vec 
let head : ('a,_ succ) vec -> 'a = function Cons (h,t) -> h
```

### Functors
A functor can require that one of its arguments is nominal:
```ocaml
type (_,_) eq = Eq : ('a,'a) eq
module F(X : sig type 'a t [@@nominal]) = struct
  type _ exp = Int : int | T : 'a exp -> 'a X.t exp
end
```

While functors cannot return distinct nominal types according to their input, one can use a hidden constraint to make the result type depend on the input.
```ocaml
module type S = sig
  type elt
  type t
  val create : elt list -> t
end
module Set(X : sig type t end) : sig
  type 'a t1 constraint 'a = X.t [@@nominal "Set.t"]
  include S with type elt = X.t and type t = X.t t1
end = struct
  type elt = X.t
  type 'a t1 = {elems: 'a list} constraint 'a = elt [@@nominal "Set.t"]
  type t = elt t1
  let create l = {elems=l}
end;;
module Set :
  functor (X : sig type t end) ->
    sig
      type 'a t1 constraint 'a = X.t [@@nominal "Set.t"]
      type elt = X.t
      type t = X.t t1
      val create : elt list -> t
    end

module Int = struct type t = int end
module Bool = struct type t = bool end;;

let f : (Set(Int).t,Set(Bool).t) eq option -> int = fun None -> 1;;
```

## Design issues

* The syntax is not clear yet. The annotation approach is non-intrusive, but it is a bit verbose.
* How should we write names? The names are only here for the incompatibility relation, so they could be anything. Restricting to longidents could be more intuitive.
* How far incompatibility should go? It is sound to make nominal types of different arities incompatible (even without name).
* How far injectivity should go? It is sound to assume injectivity between nominal types of same arity (even without name).

## Implementation

An experimental implementation is available as PR [#9042](https://github.com/ocaml/ocaml/pull/9042).
Currently the syntax is `[@@unique "s"]` for `[@@nominal "s"]`, and `[@@nominal]` is not implemented (`[@@unique]` has a different meaning).
