
# Type arguments for modules

This RFC proposes to extend a bit the module language by adding the possibility
of giving types arguments to modules.

The goal of this feature is to simplify cases where the user might want
to have a module containing only a single type by directly writing the type.
In those cases we currently force the user to define a module containing that
type.

## Proposed change

The idea is to extend the syntax of the module language with a new argument to
functors :

```ocaml
module M (type a) = ...

module M2 = functor (type a) -> ...
module type S = functor (type a) -> ...
```

and a new construction for applications.

```ocaml
module M3 = M(type [`A | `B])

let f (x : M(type int).t) = x
```

## Use cases

The motivation for this feature came from people working on modular explicits 
and modular implicits. However, this feature will also benefit OCaml even
without modular implicits.

### Use cases in current OCaml

Is some cases a parametric module requires as argument a type without any
structure attached to it. This is useful when we just want to store the type
but never look into it. Cases where this pattern occurs exists in the OCaml code
base.

#### State monad

For example when implementing a state monad such as the
one in the example available at :
https://github.com/RedPRL/redtt/blob/ae76658873a647eb43d8cf84365a9d68e9a3273c/src/basis/StateMonad.ml.

```ocaml
module type S =
sig
  type state

  include Monad.S

  val get : state m
  val set : state -> unit m

  val run : state -> 'a m -> 'a * state
end

module M (X : sig type t end) : S with type state := X.t =
struct
  type state = X.t

  type 'a m = state -> 'a * state

  let ret a st = a, st

  let bind m k st =
    let a, st' = m st in
    k a st'

  let try_ m kerr st =
    try
      m st
    with exn ->
      kerr exn st

  let get st =
    st, st

  let set st _ =
    (), st

  let run st m =
    m st
end
```

#### In ocamlgraph

Another example of usage that feature in the OCaml would be in
[ocamlgraph](https://github.com/backtracking/ocamlgraph). Where some parametric
modules such as `Imperative.Graph.AbstractLabeled` requires a module containing
a single type as argument (here the types for vertex labels).

### Modular explicits/implicits

This feature becomes very important when taking into account modular explicits
and [modular implicits](https://arxiv.org/pdf/1512.01895). Because interactions
between the code language and the module language becomes much more frequent.

In the case of modular implicits, a signature is given in order to search for a
compatible module. However, basic types are not compatible with parametrized
types.

Thus, the module following module :

```ocaml
module AddList = struct
    type 'a t = 'a list
    let add = List.concat
end
```

does not match the signature of addition.

```ocaml
module type Add = sig
    type t
    val add : t -> t -> t
end
```

A way to fix this is to add a type argument to the `AddList` module.

```ocaml
module AddList (A : Any) = struct
    type t = A.t list
    let add : t -> t -> t = List.concat
```

However, this becomes problematic with modular implicits because you need to
define a sufficient amount of modules to access any possible types. This lead
Patrick Reader and Daniel Vlasits to write a whole file of modules containing
only a type field :
https://github.com/modular-implicits/imp/blob/master/lib/any.mli.

## Proof of concept

A first draft of this feature is implemented at
https://github.com/samsa1/ocaml/tree/module_type_arg .

It features types as arguments in modules expression and type paths
as arguments in modules paths.

## Semantic change

This feature has a slightly different semantic from usual functors. The idea is
to match the non-blocking `(type a)` from the core language.

Thus when defining `module M = functor (type a) -> S[a]`, `S` is directly
computed and `M(type t)` will not launch any computation and allocation.

However in order to preserve the type soundness of the OCaml language a
restriction needs to be applied : we use value restriction on `S`. Thus if `S` is
a structure :
- it cannot contain an expansive expressions (such as `ref None`),
- it cannot define an extensive type or extend an existing one,
- it cannot define an exception,
- it cannot define an object.

The first restriction could be reduced to weak value restriction instead of weak
value restriction if required without impacting soundness.

In order to lift those restriction the user can still write his functors using
a module containing only a single field that defines a type.


Those restrictions comes from the fact that as `F(type _)` does not do any
computation in the example bellow, `r1` and `r2` are the same reference but have
incompatible types.

```ocaml
module F (type a) = struct
    let r : a option ref = ref None
end

let r1 = let module M = F(type int) in M.r
let r2 = let module M = F(type float) in M.r
```


## Important restrictions

This feature needs some restraints in order to be sound.

### Applicativity of type functors

A functor that takes a type as argument should behave as an applicative functor.

In other words, the following code should type-check

```ocaml
type t1 = M(type int).t
type t2 = M(type int).t

let f (x : t1) = (x : t2)
```

This also mean that when defining a type-functor `functor (type a) -> ME1`,
the same restrictions on `ME1` apply than on `ME2` in `functor (M : S) -> ME2`.

### Paths in paths

Only [type constructors](https://ocaml.org/manual/5.2/names.html#typeconstr)
should be authorized inside a module path. The reason for this restriction is
that adding too much complexity inside paths would add a circular dependency
between path comparison and type unification.

For example, `M(type <m: int>).t` should be rejected.

## Possible extensions

This feature could be extended with other similar patterns to be a bit more expressive.

- Implement the same feature with module types,
- Replace value restriction in type-parametrized modules to weak value restriction.
- Allow using parametric types in paths (for example `M(type int list).t`).
