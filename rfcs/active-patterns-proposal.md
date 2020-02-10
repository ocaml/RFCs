# Active patterns

- [Active patterns](#active-patterns)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Abstraction and encapsulation support](#abstraction-and-encapsulation-support)
    - [Parameterized patterns](#parameterized-patterns)
  - [Design discussion](#design-discussion)
  - [Related works and alternatives](#related-works-and-alternatives)

## Summary

Pattern matching is a great convenience for inspecting and decomposing values. Moreover, in concise functional languages like ML family residents it is made as a foundation stone: it is used in nearly every declaration and there are actions which are impossible to express without its usage. But in its basic form, which comes from far 1980s, it is seriously limited. As [P. Wadler noted in 1987](https://www.cs.tufts.edu/~nr/cs257/archive/phil-wadler/views.pdf) pattern matching does not go well with abstraction and encapsulation, the key principles of the software design. In particular, it is almost impossible to use pattern matching with abstract types and patterns are not first class values of the language.

A dozens of extensions have been suggested since that time. Some of them were rejected, some adopted. Anyway nowadays almost every leading functional language has its own extension reconciling pattern matching and abstraction.

## Motivation

### Abstraction and encapsulation support

In the current OCaml there are 3 ways of including type in module signature, each of which supports different set of properties:

|                           | Pattern matching | Abstraction | Encapsulation |
| ------------------------- |:----------------:|:-----------:|:-------------:|
| To add it with definition | +                | -           | -             |
| To make it private        | +                | +           | -             |
| To make it abstract       | -                | +           | +             |

But as [P. Wadler noted in 1987](https://www.cs.tufts.edu/~nr/cs257/archive/phil-wadler/views.pdf) it is possible to support all 3 desired properties: to support abstraction and encapsulation one needs make the type abstract and to support pattern matching on it one can add additional data type, which is called *view*, and add one of or both transformation functions (from type instance to the view and vice versa). The only addon which the language needs to support this idea is to allow the usage of this transformation functions inside patterns (explicitly as [Haskell view patterns](https://gitlab.haskell.org/ghc/ghc/wikis/view-patterns), or implicitly as [F# active patterns](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.542.4243&rep=rep1&type=pdf)). There are lots of different designs (see [related works](#related-works-and-alternatives)) but essentially they all are just flavours of this central idea.

The one classical example of the pattern matching ability losing is the list module signature:

```ocaml
module type AbsList =
sig
  type 'a t
  val empty : 'a t
  val cons  : 'a * 'a t -> 'a t
  val head  : 'a t -> 'a option
end
```

Modules implementing this signature can have absolutely different implementations of the `t` type: plain lists, join lists, lazy lists, balanced trees etc. That is why `t` is abstract and that is why we can not use pattern matching on its instances and have to use `head` function. This can lead to the significant increase of the code size. For example:

```ocaml
(* Standard list version *)
let rec destutter1 = function
  | [] | [_] as l -> l
  | hd :: (hd' :: _ as tl) when hd = hd' -> destutter1 tl
  | hd :: tl -> hd :: destutter1 tl
```

```ocaml
(* AbsList version *)
let rec destutter2 l =
  match head l with
  | Some(hd, tl) ->
      (match head tl with
      | Some(hd', _) ->
          if hd = hd'
          then destutter2 tl
          else cons hd (destutter2 tl)
      | None -> l)
  | None -> l
```

As we can see, though they both express the same logic the second version is bigger and less declarative, because all matches in it are with type `option`. To emulate the first version we are to use `AbsList.head` function, which provides "view" on the first element of the list. The problem with it is that to use it we always must introduce new nested match, so firstly, we loose the opportunity of structural nesting of patterns, what decreases expressiveness, and secondly, not all logic can be implemented in this way (for instance, after transition to the nested match we can not return to the outer evaluation, i.e there is no "backtracking" ability).

The solution is deadly simple: just to allow the compiler to insert these transformation functions invocations before nested patterns evaluation:

```haskell
-- Haskell version
{-# LANGUAGE ViewPatterns #-}

destutter (head -> Just(hd, tl@(head -> Just(hd', _))))
                      | hd = hd' = destutter tl
destutter (head -> Just(hd, tl)) = cons hd (destutter tl)
destutter l = l

-- Proposed Haskell version with implicit Maybe and tuple decompositions
destutter (head => hd tl@(head => hd' _))
               | hd = hd' = destutter tl
destutter (head => hd tl) = cons hd (destutter tl)
destutter l = l

-- Proposed Haskell version with implicit transformations
destutter (=> hd tl@(=> hd' _)) | hd = hd' = destutter tl
destutter (=> hd tl) = cons hd (destutter tl)
destutter l = l
```

```fsharp
// F# version
let (|Cons|Nil|) l =
  match head l with
  | Some(hd, tl) -> Cons(hd, tl)
  | None         -> Nil

let rec destutter = function
  | Nil() | Cons(_, Nil()) as l -> l
  | Cons(hd, Cons(hd', _) as tl) when hd = hd' -> destutter tl
  | Cons(hd, tl) -> cons hd (destutter tl)
```

The first Haskell version solves the problem but it is too verbose because of `Maybe` and tuples decompositions. There is a [proposed version of syntax](https://gitlab.haskell.org/ghc/ghc/wikis/view-patterns#implicit-maybe) which allows to make it implicit. And finally there is even a version which allows to make the transformation function implicit.

The F# version is even simpler and exactly expresses the standard lists `destutter`. The compiler automatically inserts the transformation function by the used name of one of the alternatives, listed in the name of the transformation function.

### Parameterized patterns

Another interesting opportunity is to allow the parameterization of the pattern. It has lots of different applications:

- More precise data inspection:

    ```fsharp
    // F# code
    match xml_data with
    | (Attr "x" (Float x) &
        Attr "y" (Float y) &
        Attr "z" (Float z)) -> Some(x,y,z)
    | _ -> None
    ```

    This example also uses and-patterns (`&`), which are not supported by the OCaml now, because only matches with data constructors are allowed which are always mutually exclusive. But as [noted by Don Syme et al.](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.542.4243&rep=rep1&type=pdf) and-patterns become more handy with support of active or view patterns, because they can overlap as in this example: the first branch matches only when XML node has all three attributes "x", "y" and "z", which values are extracted by the active pattern `Attr` parameterized by the needed attribute names. Result expression can use bindings from any conjuncts, there is no limitation on the bindings names (comparing to or-patterns, where all disjuncts must have exactly the same set of bindings). Actually active or view patterns can easily emulate and-patterns:

    ```fsharp
    // F# code
    let (|All3|) x = (x, x, x)

    match xml_data with
    | All3(Attr "x" (Float x), Attr "y" (Float y), Attr "z" (Float z))
        -> Some(x,y,z)
    | _ -> None
    ```

- Regex expressions:

    ```fsharp
    // F# code
    let swap s =
      match s with
      | ParseRegex "(\w+)-(\w+)" [l; r] -> r ^ "-" ^ l
      | _ -> s
    ```

- [Ranges expressions](https://github.com/ocaml/ocaml/issues/8504):

    ```fsharp
    // F# code
    let clump signal =
      match signal with
      | IntMoreThan 256   _  -> 255
      | IntBetween  0 255 it -> it
      | IntLessThan 0     _  -> 0
      | _ -> assert false
    ```

- [Additional bindings introduction](https://github.com/ocaml/ocaml/issues/6901):

    ```ocaml
    (* OCaml version with code duplication *)
    let rec unif x y =
      match x, y with
      | Var r           , _ when !r != None -> unif (deref f   []) y
      | App(Var r, args), _ when !r != None -> unif (deref f args) y
      | ...
    ```

    ```fsharp
    // F# version
    let (|bind|) binding_value scrutinee = (binding_value, scrutinee)

    let rec unif x y =
      match x, y with
      | bind [] (args, Var r) | App(Var r, args), _ when !r != None ->
          unif (deref f args) y
      | ...
    ```

## Design discussion

Both, Haskell view patterns and F# active patterns, can be easily expressed by each other. The difference between them is syntactic only. We personally prefer active patterns over view patterns, as it syntactically more clear and probably requires less modifications in the language grammar.

All opinions about concrete design preferences or drawbacks are welcomed to the pull request discussion.

## Related works and alternatives

- [Active patterns](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.542.4243&rep=rep1&type=pdf) in F#.
- [Pattern guards](https://www.researchgate.net/publication/2646388_Pattern_Guards_and_Transformational_Patterns), [view patterns](https://gitlab.haskell.org/ghc/ghc/wikis/view-patterns) and [pattern synonyms](https://www.researchgate.net/publication/307090856_Pattern_synonyms) in Haskell.
- [Extractors](https://www.researchgate.net/publication/37442102_Matching_Objects_with_Patterns) in Scala.
- [Views](https://www.cs.tufts.edu/~nr/cs257/archive/chris-okasaki/views.pdf) for Standard ML ([candidates](https://people.mpi-sws.org/~rossberg/hamlet/README-succ.txt) for Successor ML).
