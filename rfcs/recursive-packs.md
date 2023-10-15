# Recursive packs

## Summary

OCaml's module system has great facilities for structuring your code
in the large. It supports encapsulation, parameterisation and even
recursion. Unfortunately not all of these facilities are available
across multiple files. The `-pack` feature allows you to group
multiple files together under an interface, but there isn't really
a way to allow recursion amongst a group of files.

This proposal aims to support recursion amongst groups of files by
extending recursive modules to work on groups of files via the `-pack`
mechanism. It is an extension of the [Functor packs](https://github.com/ocaml/RFCs/pull/11) proposal.

## Motivation

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

This proposal is split into 2 sub-proposals:

1. Recursive interfaces - allowing groups of module interfaces to
   refer to each other
2. Recursive packs - allowing packed units to be mutually recursive.

The first part can be upstreamed before the second part is
completed.

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
build systems to be similar. This is based on top of the proposed
interface for functor packs.

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
could do that with a `(recursive true)` field -- which
would imply `(packed true)`:

```
(library (
  (name foo)
  (parameters (arg))))
```

### Interaction with other tooling

#### Merlin support

Merlin would need a couple of small changes so that it would allow a
module to refer to its own .cmi file if that file is flagged as
recursive.

#### Odoc support

This proposal will require some small additions to odoc but they are
pretty easy to handle.

### Full example

As a demonstration, here's the complete source for a library `Lib`
containing a recursive pack `Pck` with two submodules `Foo` and `Bar`:

dune
```
(library (
   (name lib)
   (modules (
     (pack ((name pck) (recursive true) (modules foo bar)))))))
```

foo.mli

```
type t

val other : t
val bar : Bar.t -> t

val to_string : t -> string
```

foo.ml

```
type t =
  | Other
  | Bar of Bar.t

let other = Other
let bar b = Bar b

let to_string = function
  | Other -> "other"
  | Bar b -> Bar.to_string b
```

bar.mli

```
type t

val string : string -> t
val foo : Foo.t -> t

val to_string : t -> string
```

bar.ml

```
type t =
  | String of string
  | Foo of Foo.t

let to_string = function
  | String s -> s
  | Foo f -> Foo.to_string f
```

pck.mli (optional)

```
module Foo : sig

  type t

  val other : t
  val bar : Bar.t -> t

end

and Bar : sig

  type t

  val string : string -> t
  val foo : Foo.t -> t

  val to_string : t -> string

end
```

lib.ml (optional)

```
module Pck = Pck
```

Here is some code using the module:

```
let s = Lib.Pck.Bar.to_string (Lib.Pck.Bar.foo Lib.Pck.Foo.other)
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

- Recursive modules are basically still an experimental feature. There
  is some consensus that their current form should be replaced by
  something else at some point even if it breaks
  backwards-compatibility.  That might break some uses of the
  `-recursive-pack` and `-recursive` options.

## Alternatives

- A common alternative to this proposal at the moment is `sed`. That
  has many obvious drawbacks, but does avoid adding new compilation
  flags to the compiler.

- Some use cases of recursive packs could be addressed by RFC #2. If
  the mutual recursion is all at the type level then that should be
  sufficient.
