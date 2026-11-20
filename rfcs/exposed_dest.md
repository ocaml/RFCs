# Expose Destinations for PPXs (and audacious programmers)

This RFC proposes to expose some tools to easily manipulate destinations, aka write-once pointers pointing to the inside of allocated blocks.

In [tail modulo cons][tmcdoc], a function is transformed into a "Destination Passing Style" form (or DPS for short).
Such DPS transformation can be represented syntactically. For instance, 
taken directly
from [the TMC paper][tmcpaper]:

```ocaml
let[@tail_mod_cons] rec map f l = match l with
  | [] -> []
  | x :: xs -> f x :: map f xs
```

is turned into (modulo a wrapper function):

```ocaml
let rec map_dps f l dst idx =
  match l with
  | [] -> dst.idx <- []
  | x :: xs ->
    let y = f x in
    let dst' = y ::{mutable} Hole in
    dst.idx <- dst';
    map_dps f xs dst' 1
```

[tmcdoc]: https://ocaml.org/manual/5.4/tail_mod_cons.html
[tmcpaper]: https://arxiv.org/pdf/2411.19397

There are many other uses of destinations besides the tail-modulo cons transformation, notably when working on graphs and trees [1][] [2](https://sv.c.titech.ac.jp/minamide/papers/hole.popl98.pdf) or for more complex tail-based transformations [3](https://dl.acm.org/doi/10.1145/3571233) [4](https://icfp26.sigplan.org/details/icfp-2026-icfp-papers/12/Tail-Modulo-Async-Await).

[1]:https://arxiv.org/pdf/2312.11257

While ensuring safety of manually used destinations require linear types (see [1][]), they are instrumental in many interesting syntactic transformations over programs which could be implemented as "simple" PPXs.
The goal of this RFC is to make this possible.

TMC implements the manipulation of destiny directly in the Lambda IR. 
One of the main reason being that it's actually not possible to implement it
without support from the compiler in OCaml:

```
  Foo (x, HOLE)
```

has different representation if `type t = Foo of int * int` or `type t = Foo of (int * int)` (the latter contains an extra pointer). It's not possible to determine where the pointers are by local syntactical analysis.

Even worse, in the presence of the flat float array optimisation:
  
```
   [| HOLE; HOLE |]
```

the code depends on the type inferred for the holes.

The recent unboxing extensions makes the need to involve the compiler even clearer.

## A compiler-assisted API for destinies

My proposal is to provide a low-level, compiler-assisted API for Destinies.
It would be composed of two parts: a simple API in `Obj`:

```ocaml
type 'a dest
val set_dest : 'a dest -> 'a -> unit
```

and a new extension point understood by the compiler:

```ocaml
type t = Foo of (int, float array, string)

let v : t, (dest1 : float dest, dest2 : string dest) = 
  [%ocaml.value_with_holes Foo (21, [| [%hole] |], [%hole]) ]
```

The expression under `%ocaml.value_with_holes` *must* be a valid 
multi-holed context for TMC, i.e. a nest of constructors which can be allocated
without looking at the content of the holes, and for which holes have known positions.

It returns the allocated value and a tuple (or an heterogeneous list) of the destinations
to all the holes in the expression.
The hole are replaced with a dummy value (following the same convention as TMC).

Typing wise, `[%ocaml.value_with_holes C[[%hole],...,[%hole]]]` is equivalent to

```ocaml
([%ocaml.value_with_holes (C[([%hole] : 'a_1),...,([%hole] : 'a_n)] : 'a)] : 'a * ('a_1 dest * ... 'a_n dest))
```

This expression is not considered pure for the purpose of generalisation (at least while implicit flat float arrays are still a thing, and probably still later because of unboxing).

## Guaranties

Given `x` a variable, `C` a n-ary constructor context, `v_1`, ..., `v_n` arbitrary values,
P an arbitrary program which doesn't read nor write `x` (it can be passed around);
the compiler would guarantee that the following two programs are equivalent:

```ocaml
let x = C[v_1,...,v_n] in
P;
```

```ocaml
let x, (d_1, ..., d_n) = [%ocaml.value_with_hole C[[%hole],...[%hole]] in
P;
Obj.set_dest d_1 v_1;
...
Obj.set_dest d_n v_n;
```

## Implementation

The implementation of destinations is fairly straightforward
```
type 'a dest = {
    pointer : Obj.t ; (* Must be an allocated block *)
    offset : int ;
  }
```

Most of the implementation of `%ocaml.value_with_holes` is already
in the compiler, in `Lambda.Tmc.Constr`.

## Non-goal of the API

The API explicitly doesn't have the following goals:

- Ensure statically that destinations are write-once.
- Ensure values are only read after their holes are filled.

In the case I consider, PPXs emitting code that manipulate the destinations
would ensure these properties by construction.

## Extension: checking destinations dynamically

For now, TMC uses `Lambda.dummy` as hole placeholder, which is a constant integer.
By choosing judiciously the dummy value that is inserted in the hole, 
it should be possible in many cases to check if a hole has been filled (in particular, for the non-floatarray cases).
A simple idea would be to insert a pointer to a constant dummy block, for instance.

This would allow implementing `is_set : 'a dest -> bool` (and/or make `set_dest` fail dynamically if the hole is filled).

A more audacious implementation would be to add some information
in the block header to note the presence of holes, similar to [Mixed Blocks](https://icfp24.sigplan.org/details/ocaml-2024-papers/7/Mixed-Blocks-Storing-More-Fields-Flat).
