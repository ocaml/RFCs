
# Type arguments for modules

This RFC proposes to extend a bit the module language by adding the possibility
of giving types arguments to modules.

The goal of this feature is to simplify replace cases where the user might want
to have a module containing only a single type by directly writing the type.
In those cases we currently force the user to define a module containing that
type.

## Proof of concept

A first draft of this feature is implemented at
https://github.com/samsa1/ocaml/tree/module_type_arg .

It features types as arguments in modules expression and type paths
as arguments in modules paths.

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


## Important restraints

This feature needs some restraints in order to be sound.

### Applicativity of type functors

A functor that takes a type as argument should behave as an applicative functor.

In other words, the following code should type-check

```ocaml
type t1 = M(type int).t
type t2 = M(type int).t

let f (x : t1) = (x : t2)
```

### Paths in paths

Only type paths should be authorized when doing an application inside
a type path.

In other words, `M(type <m: int>).t` should be rejected.

### Type application must be compiled as an empty structure

If we could avoid giving the type arguments at runtime for modules like with
functions we would have soundness issues.

The following example show why it is important to preserve the information of
application all the way to binary code.

```ocaml
module F (type a) = struct
    let r : a option ref = ref None
end

let r1 = let module M = F(type int) in M.r
let r2 = let module M = F(type float) in M.r
```

If `F(type _)` does not launch any computation then `r1` and `r2` will be the
same reference but with incompatible types.

## Possible extensions

This feature could be extended with other similar patterns to be a bit more expressive.

- Implement the same feature with module types,
- Allow using parametric types in paths (for example `int list`).
