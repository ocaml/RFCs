# Functor units, functor packs and recursive packs

## Summary

OCaml's module system has great facilities for structuring your code
in the large. It supports encapsulation, parameterisation and even
recursion. Unfortunately not all of these facilities are available
across multiple files. The `-pack` feature allows you to group
multiple files together under an interface, but there isn't really
a way to parameterise a group of files all at once, or to allow
recursion amongst a group of files.

This proposal aims to support parameterisation of groups of files
and recursion amongst groups of files by extending functors and
recursive modules to work on groups of files via the `-pack`
mechanism.

## Motivation

Parameterising a group of top-level modules by another module is a
commonly requested feature. Doing it by hand can be quite painful
because each module must be parameterised by both the main parameter
and the applied forms of all the other parameterised modules that
it depends on. For example, if `B` depends on `A` and they are both
parameterised by `Param` then you get the equivalent of:

```ocaml
module A (Param : S) : T with module Param := S

module B (Param : S) (A : T with module Param := S) :
  R with module Param := S and module A := A
```

which gets more awkward the more modules are included.

Recursion between modules is useful in a number of situations. One
particular use case is constructing mutually recursive types without
exposing the definitions of each type to the other. Recursive modules
allow code of the form:

```ocaml
module rec A : sig
  type t
  val of_b : B.t -> t
end = struct
  type t = B.t
  let of_b b = b
end

and B : sig
  type t
  val of_as : A.t list -> t
end = struct
  type t = A.t list
  let of_as as = as
end
```

This kind of code is currently unavailable between top-level modules.

Whilst there is general uncertainty over what the final form of
recursive modules in OCaml should look like, I think that this use
case is one that should always be supported, and so it is worthwhile
to extend it to top-level modules.

## Details

This proposal is split into 4 sub-proposals:

1. Functor units - allowing compilation units to be functors
2. Functor packs - allowing packed units to be functors
3. Recursive interfaces - allowing groups of module interfaces to
   refer to each other
4. Recursive packs - allowing packed units to be mutually recursive.

Each of these parts can be upstreamed before the later parts are
completed.

### Supporting ml files as functors

The aim for this part is to allow an .ml file to be compiled to a functor
rather than a structure. The idea is that the user:

1. Writes ".mli" files for the parameters of the functor and compiles them
   with a command-line option like "-parameter-of Foo" where "Foo" is the
   name of the functor.

2. Compiles the ".mli" and ".ml" files for the `Foo` module with options
   like "-parameter Arg" where "Arg" is the name of one of the
   parameters. "Arg" will be available as a module in these files, just
   as it would be for the body of a functor. The module type in the
   resulting ".cmi" file is that of a functor taking the given
   parameters. The code in the resulting ".cmo"/".cmx" file is also that
   of a functor taking the given parameters.

Note that the restrictions on what can appear in the body of a functor,
e.g. unwrapping first-class modules must be applied to such modules.

Any attempt to use an interface compiled with "-parameter-of Foo"
outside of `Foo` should give a clear error message.

### Supporting packed functors

The aim for this part is to extend the support in the previous section
to allow packing to be used for creating a functor out of multiple
files. This is achieved by allowing the `-parameter` command-line argument
to be passed when packing a file.

The idea is that the user:

1. Writes ".mli" files for the parameters of the functor and compiles
   them with a command-line option like "-parameter-of Pck".

2. Compiles the ".mli" and ".ml" files for the modules in the body of the
   functor with an option like "-for-pack Pck(Arg)", where `Pck` is the
   name of the packed functor and `Arg` is the name of its parameter.

3. (optional) Compiles the ".mli" of the intended interface for the
   packed functor with "-parameter Arg" as an option. This file
   describes the return type of the functor, and so it has access to the
   parameter modules, but not the component modules of the pack.

3. Runs "ocamlc -pack" passing it "-parameter" options for each
   parameter, and the ".cmi" and ".cmo" files of the pack's
   sub-modules. This then creates a ".cmi" and ".cmo" file for a module
   `Pck` that is a functor from the parameters modules to a module
   containing all the sub-modules.

So if we have a parameter `Arg`, and body modules `Foo` and `Bar`, where
`Bar` depends on `Foo`, then we essentially compile `Foo` to:

```ocaml
module Foo (Arg : ...arg.mli...) : sig
  ...foo.mli...
end = struct
  ...foo.ml...
end
```

and `Bar` to:

```ocaml
module Bar (Arg : ...arg.mli...) (Foo : ...foo.mli...) : sig
  ...bar.mli...
end = struct
  ...bar.ml...
end
```

and finally create `Pck` equivalent to:

```ocaml
module Pck (Arg : ...arg.mli...) : sig
  module Foo : sig ...foo.mli... end
  module Bar : sig ...bar.mli... end
end = struct
  module Foo = Foo(Arg)
  module Bar = Bar(Arg)(Foo)
end
```

Although this will all be implemented at the level of `Lambda` code rather
than elaborating to actual OCaml syntax.

### Supporting mutually recursive interfaces

The aim of this part is to support modules having mutually recursive
interfaces. The idea is for users to compile these mutually recursive
".mli" files with a single call to ocamlc passing an option "-recursive"
along with all the ".mli" files at once. This command will essentially
use `Typemod.transl_recmodule_modtypes` on all the files. When comparing
an implementation with one of these interfaces the compiler would
essentially use `Typemod.check_recmodule_inclusion`.

### Supporting packed recursive modules

The aim for this part is to allow packing to be used for creating a set
of mutually recursive modules out of multiple files. The idea is that
the user:

1. Compiles all the ".mli" files using "-for-recursive-pack Pck". If the
   interfaces are mutually recursive then they will also need to use
   "-recursive" from the previous section. For now they will also need
   to use "-opaque" to avoid warnings about missing cmx files.

2. Compiles each "ml" file for the sub-modules. The cmo/cmx file
   optionally contains a "module shape" for the module, as computed by
   `Translmod.init_shape`.

3. Runs "ocamlc -pack-recursive" passing it the ".cmo" files for all the
   modules. This should construct the resulting module in a manner
   similar to `Translmod.compile_recmodule`.

### Interface in dune

These features provide more options for how files are compiled. It is
worth considering how such features might be presented to users
through their build systems. I'll describe a potential interface for
dune, but I would expect the presentation of these features in other
build systems to be similar.

#### Exposing `-pack`

Packing could be exposed through dune using a `(pack)` construct
in the list of modules of a library or executable. For example:

```
(library (
   (name foo)
   (modules
     (pack ((name bar) (modules (a b)))))))
```

The user can provide a `bar.mli` file that is the interface of the pack,
but they cannot provide a `bar.ml`.

#### Exposing `-parameter`

Adding a parameter to a module would be exposed using a `parameter`
field in an entry in the `modules` list:

```
(library (
   (name foo)
   (modules (
     a
     ((name b) (parameters (arg)))))))
```

This would expect an `arg.mli` to describe the parameter, which would be
built with `-parameter-of Bar`. `b.mli` would be built with `-parameter
Arg`.

#### Exposing `-pack` with `-parameter`

Packed functors could be exposed using a `(parameters ...)` field in
the pack construct. For example:

```
(library (
   (name foo)
   (modules
     (pack ((name bar) (parameters (arg)) (modules (a b)))))))
```

This would expect an `arg.mli` to describe the parameter, which would be
built with `-parameter-of Bar`.

#### Exposing `-pack-recursive`

Recursive packs could be exposed using a `(recursive true)` field in
the pack construct. For example:

```
(library (
   (name foo)
   (modules
     (pack ((name Foo) (modules (A B)) (recursive true))))))
```

#### Exposing these features directly on libraries

In the long run we might also want to expose these features as part
of the library stanza, for creating a library module as a pack. We
could do that with a `(packed true)` field on libraries as well
as `(parameters ...)` and `(recursive true)` fields -- both of which
would imply `(packed true)`:

```
(library (
  (name foo)
  (parameters (arg))))
```

### Interaction with other tooling

#### Merlin support

For the functor packs I don't think merlin needs any special support
to work fine. The only thing is that the restrictions on first-class
modules when used in the body of a functor would not be reported as
errors. Adding a flag to `.merlin` files would allow that to be
addressed.

For recursive packs merlin would need a couple of small changes so
that it would allow a module to refer to its own .cmi file if that
file is flagged as recursive.

#### Odoc support

This proposal will require some small additions to odoc but they are
pretty easy to handle.

### Full example

As a demonstration, here's the complete source for a library `Lib`
containing a packed functor `Pck` with two submodules `Foo` and `Bar`:

dune
```
(library (
   (name lib)
   (modules (
     (pack ((name pck) (parameters (arg)) (modules foo bar)))))))
```

arg.mli

```
type t

val a : t

val b : t

val print : t -> unit
```

foo.mli

```
type t = Arg.t list

val mk : t -> t

val iter : t -> (Arg.t -> unit) -> unit
```

foo.ml

```
type t = Arg.t list

let mk t = t

let iter = List.iter
```

bar.mli

```
val run : Foo.t -> unit
```

bar.ml

```
let run t =
  Arg.print Arg.a;
  Foo.iter (fun t -> Arg.print t);
  Arg.print Arg.b
```

pck.mli (optional)

```
module Foo : sig

  type t

  val mk : Arg.t list -> t

end

module Bar : sig

  val run : Foo.t -> unit

end
```

lib.ml (optional)

```
module Pck = Pck
```

Here is some code using the functor:

```
module Int = struct
  type t = int
  let a = 1
  let b = 5
  let print = print_int
end

module L = Lib.Pck(Int)

let () = L.Bar.run (L.Foo.mk [2; 3; 4])
```

## Drawbacks

- This builds on the `-pack` mechanism, which has some drawbacks:

  1. Touching any component triggers recompilation of all clients of
     the packed module.

  2. Linking part of a packed module forces linking the whole packed
     module.

  For these reasons most existing uses of `-pack` have been replaced
  by approaches based on module aliases.

  These issues are probably unavoidable for the recursive use case
  anyway since the sub-modules are mutually recursive.

  For the functor use case these issues might be avoidable with some
  other approach. Although it couldn't be based on module aliases,
  since those approaches rely on being able to refer to *the* module
  `M` but thanks to side-effects you cannot talk about *the* module
  `F(X)` only *a* module `F(X)`.

- Recursive modules are basically still an experimental feature. There
  is some consensus that their current form should be replaced by
  something else at some point even if it breaks
  backwards-compatibility.  That might break some uses of the
  `-recursive-pack` and `-recursive` options.

## Alternatives

- A common alternative to this proposal at the moment is `sed`. That
  has many obvious drawbacks, but does avoid adding new compilation
  flags to the compiler.

- Only providing some of the sub-proposals -- the functor and recursion parts
  are fairly independent and could be considered separately.

- Some use cases of recursive packs could be addressed by RFC #2. If
  the mutual recursion is all at the type level then that should be
  sufficient.
