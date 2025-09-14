# `for...in`

This RFC proposes to add a `for...in` construct in OCaml, similar to [Python's for-in](https://docs.python.org/3/reference/compound_stmts.html#for), [JavaScript's for-of](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of), [Rust's for-in](https://doc.rust-lang.org/reference/expressions/loop-expr.html#iterator-loops), or [Lean's for-in](https://lean-lang.org/doc/reference/latest/Functors___-Monads-and--do--Notation/Syntax/#monad-iteration-syntax) (though that last one is more complex).


## Overview

The feature is straightforward, and would allow users to write something like[^1]:

```ocaml
for x in Iter.(1 -- 1000) do
   Printf.printf "Number: %d" x
done
```

The iterator must be of type `('a -> unit) -> unit`, corresponding to [popular implementations](https://ocaml.org/p/iter/1.6) of the iterator pattern in OCaml and to what the standard library offers -- modulo flipped arguments (see drawbacks).


## Motivation

The motivation is twofold: 1) improve readability, and 2) allow for a style that newcomers would be familiar with.

The OCaml language has nicely evolved in the last few years to allow more imperative-but-somewhat-functional features. This includes the addition of let-binding operators and effect handlers. In particular, while let-binding operators offer only nicer syntax for something already possible, they have been [very popular](https://sourcegraph.com/search?q=context:global+%22let*%22+lang:OCaml+&patternType=keyword&sm=0) in the OCaml ecosystem. This feature has a similar motivation.


## Technical implementation

### Proof of concept

While this feature doesn't currently have a proof of concept, a first version of this feature seems light to implement. It requires adding the following construct to the syntax:
```
expr += for pattern in expr do expr done
                       ----    ----        
                        |       |- body
                        |
                        |-- iterator
```

and the following desugaring:
```ocaml
for p in iter do e done

(* desugars into *)

(iter : ('a -> unit) -> unit)
  (fun (p : 'a) -> (e : unit))
```

The type ascriptions are there to ensure that the for-in syntax is not abused for other purposes than iteration

### Additional considerations

#### Error messages

It would be nice to have a specific type error for an invalid iterator used in a for loop. Rather than
> This has type {...} but was expected to have type `('a -> unit) -> unit`

something like the following would make the feature slightly more usable:

> for loops are expected to iterate on an iterator, see {doc}

Though this requires modifying the type checker, which is a much deeper change than a desugaring.

#### Future: monadic for-loops

If this feature turns out to be of interest, future extensions could involve the definition of custom `for` operators (i.e. `let ( for* ) = ...`) for "monadic iteration", à la [Lean](https://lean-lang.org/doc/reference/latest/Functors___-Monads-and--do--Notation/Syntax/#ForIn___mk)

#### Future: modular implicits 

One of the features that makes this feature popular in other languages is the ability to iterate on any "iterable data structure", which is, "any data structure that implements Iterable" or some equivalent. This is already expressible in any of the languages mentioned in intro: Rust and Lean do this through typeclasses, and Python/JavaScript through duck typing.

Similarly, this feature would greatly benefit from the eventual addition of modular implicits in OCaml, to avoid manually specifying the iteration function. When modular implicits are added to the language, this feature should be straightforward to extend without breaking compatibility. The following would be added to the standard library:

```ocaml
module Into_iter = struct
   type 'a t
   val into_iter = 'a t -> ('a -> unit) -> unit
end

(* Using syntax from the 2014 paper *)
let into_iter {S: Into_iter} x = S.into_iter x
```

The desugaring would become the following:

```ocaml
for p in iter do e done

(* desugars into *)

(Stdlib.into_iter iter) (fun p -> e)
```

and retro-compatibility would be preserved by having `'a Iter.t` implement `Into_iter`:

```ocaml
implicit module Iter_into_iter = struct
   type 'a t = ('a -> unit) -> unit
   val into_iter x = x
end
```

## Drawbacks

#### No typeclass/modular implicit

While half of the motivation for this feature is wanting to import a widely popular construct into OCaml, newcomers could be surprised of the need to manually specify the way to iterate on the structure while the other languages mentioned previously do not need to.

#### No external prototyping

Custom let-bindings were first prototyped and gained adoption as JaneStreet's [ppx_let](https://ocaml.org/p/ppx_let/latest). This was possible because the syntax extensions for doing so already existed. Unfortunately, even with syntax extensions, OCaml does not have a `for...in` syntax, and bending around `for ... = ... (down)to ...` would be difficult.


#### Mismatch with the standard library's iterators

The standard library currently implements an `iter : ('a -> unit) -> 'a t -> unit` function for most of its collections. While this makes sense for e.g. piping, it also makes it unfortunately impossible to write the following (should the feature be implemented):
```ocaml
for x in List.iter [1; 2; 3] do ...
```

This can be overcome by adding an `Iter` module to the standard library, and a function for each structure of the standard library, such as `Iter.of_list` or just `Iter.list`.

#### Adds more imperativeness

This feature does add more "imperativeness" to the language, which is questionable. That being said, so do effect handlers and they turned out to work beautifully with the language.

#### Redundancy

It was argued in [this discussion](https://caml.zulipchat.com/#narrow/channel/527805-compiler/topic/.60for.20.2E.2E.20in.60.20in.20OCaml/with/539113499) that a `for-in` construct would be redundant with what is currently possible with let-bindings:

```ocaml
let ( let- ) = Iter.of_list
let ( let@ ) = ( @@ )

let () = 
   let- i = [1; 2; 3] in
   Printf.printf "%d" i
   
let () =
   let@ i = Iter.of_list [1; 2; 3] in
   Printf.printf "%d" i
```

Both of the above unit expressions would be equivalent to:
```ocaml
let () =
  for i in Iter.of_list [1; 2; 3] do
    Printf.printf "%d" i
  done
```

However, first it is not immediately clear to a newcomer that either of these let-bindings performs iteration over the list, and makes simple code somewhat harder to read.

In addition, the same argument would have held against the addition of let-binding operators, where
```ocaml
let* x = e in ...
```
is equivalent to
```ocaml
e >>= fun x ->
...
```

There is also an argument of authority to make in saying that the same feature is widely popular in the other languages mentioned in the introduction, even though they also all have the ability to express it functionally.

[^1]: For lack of creativity in finding a more interesting example