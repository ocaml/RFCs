# Generalize documentation comments with several syntactic categories

## High-level description

In the current implementation of the compiler, there is one kind of
documentation comments. Normal comments (of the form `(* ... *)`) are dropped
after lexing, and documentation comments (of the form `(** ... *)`) are
collected and attached to the parsetree as attributes identified with
`ocaml.doc`.

This RFC proposes to add new kinds of documentation comments, distinguished
syntactically from the usual documentation comments available today.

The concrete syntax would be as follows:

```
(** ... *)  : a usual doc string
(*@ ... *)  : a doc string of kind "@"
(*! ... *)  : a doc string of kind "!"
...
```

i.e. allow a (reasonable) range of predefined characters as a "kind" marker
after the opening comment tag `(*`.

Then, different kinds of documentation comments would be elaborated to different
attributes in the parsetree. Since `(** ... *)` produces attributes `ocaml.doc`,
`(*@ ... *)` would produce attributes identified with e.g. `ocaml.doc_at`, etc.

## Motivation

This proposal is motivated by work as part of the [Vocal
project](https://vocal.lri.fr/) ([github
repo](https://github.com/vocal-project/vocal)).

One goal of the project is to design a specification language for OCaml
libraries, named Gospel, and taking inspiration in particular from
[JML](https://en.wikipedia.org/wiki/Java_Modeling_Language).

For that purpose, we want to attach to the items of a signature not only
human-readable docstrings (between `(** ... *)`, to be processed by ocamldoc),
but also machine-readable formal specifications, to be processed by a dedicated
tool including a dedicated parser. It is convenient to be able to write these
specifications in a dedicated comment block, between `(*@ ... *)`.

Note that we *do not want* our specification-comment-blocks to be picked up by
ocamldoc/odoc as standard docstrings (initially they would be simply ignored;
and then we could imagine implementing dedicated support for them in odoc).

One example from the vocal repo, specifying (both formally and informally) the
`create` function of a
[Vector](https://github.com/vocal-project/vocal/blob/master/src/Vector.mli)
module.

```ocaml
val create: ?capacity:int -> dummy:'a -> 'a t
(** [create] returns a fresh vector of length [0].
   All the elements of this new vector are initially
   physically equal to [dummy] (in the sense of the [==] predicate).
   When [capacity] is omitted, it defaults to 0. *)
(*@ a = create ?capacity ~dummy
      requires let capacity = match capacity with
                 | None -> 0 | Some c -> c in
               0 <= capacity <= Sys.max_array_length
      ensures  length a.view = 0 *)
```

We believe that it makes sense to distinguish syntactically between docstrings
"for humans", and formal specifications: they correspond to two different levels
of abstraction, and will be processed by different tools.

Instead of introducing new syntax, we could require the use to directly use the
syntax of attributes, and instead write something like (note that the user now
needs to worry about proper escaping between the `"`):

```ocaml
[@@gospel "a = create ?capacity ~dummy
      requires let capacity = match capacity with
                 | None -> 0 | Some c -> c in
               0 <= capacity <= Sys.max_array_length
      ensures  length a.view = 0"]
```

This is somewhat doable but relatively heavyweight and inconvenient, which is
why we would prefer having a dedicated lightweight comment-like syntax.

## Technical details

In our current prototype implementation (as a small patch on top of the ocaml
parser), @-docs have dedicated parsing rules, and at the moment are only allowed
in top-level and declaration-level position, since Gospel has no use for those
in other places. @-docs are desugared in the parser to attributes.

The issue I can see with that approach is that parsing rules of normal
docstrings and @-docstrings end up being duplicated, with the risk of being
parsed in subtly inconsistent ways. From a user perspective, I believe that
normal docstrings and @-docstrings should be interchangeable from a parsing
perspective.

This weights in favor of extending the existing parsing mechanism for docstrings
to handle docstrings of several kinds.

## Unresolved questions

There seems to be a major issue implementation-wise due to the current
attachement rules for docstrings. As far as I can tell, those essentially only
allow one docstring to be attached to a declaration. For instance, when writing:

```
type t
(** foo *)
(** bar *)
```

the second docstring `(** bar *)` is simply dropped by the compiler (triggering
warning 50 "unattached documentation comment" if it is enabled).

I suspect this has to do with the fact that docstrings can be placed before and
after a declaration, and cases such as `type t (** foo *) (** bar *) type u`
need to be disambiguated (currently, this results in "foo" being attached to `t`
and "bar" attached to `u`).

To handle the style illustrated above, where several docstrings (of possibly
different kinds) are attached to a single toplevel declaration, one would need
to modify the rules for attaching docstrings to AST nodes.

I am not familiar with this part of the compiler, so I am interested in getting
some input about whether that would be possible, or even make sense.

## Drawbacks / limitations

If the "kind identifier" of a docstring is set to a single character (`@`, `!`,
...) belonging to a fixed set, as proposed previously, then we have a finite,
fixed number of different kinds of docstrings available.

This might not be very modular: what if both tool A and tool B decide that they
handle docstrings of the form `(*@ ... *)`, and one wants to use both at the
same time? How do we do if we have more tools than docstring kinds?

It is not clear whether that would be an issue in practice. One solution
(outside of the scope of the compiler) would be make the "docstring kind" an
optional parameter of the gospel tool (so that if can be set to use e.g. `(*!
... *)` if `(*@ ... *)` is already taken). And as a last resort it would still
be possible to use the attribute-based syntax directly.

## Alternative approach

An alternative approach, as a middle ground (syntactically heavier than `(*@ ..
*)` but less troublesome to implement), would be to provide a lightweight syntax
for writing string attributes, along the lines of the lightweight syntax for
string extensions, proposed by @Drup in
[#8820](https://github.com/ocaml/ocaml/pull/8820).

This would allow writing attributes using the following syntax (where the "..."
may contain quotes without having to escape them, as with "quoted strings" `{| .. |}`):
```
{@gospel| ... |}    as a shorthand for [@gospel {| ... |}]
{@@gospel| ... |}   as a shorthand for [@@gospel {| ... |}]
etc
```

Independently of the availability of this syntax, it seems a few modifications
to the attachment rules for docstrings would still be useful. Indeed, as
illustrated above on the spec for `create`, it is more natural to write the
human-readable docstring first, and the specification second. However, this does
not work currently. Writing:
```
type t
(** foo *)
[@@gospel bar]
```
results in the docstring "foo" being discarded, even though it is sandwiched
between the declaration and an attribute attached to this declaration. The
reverse order (attribute first, docstring second) works, but it would be nice if
this way was also supported.

Assuming that both the lightweight syntax and the modified attachment rules are
implemented, the snippet with `create` becomes:
```ocaml
val create: ?capacity:int -> dummy:'a -> 'a t
(** [create] returns a fresh vector of length [0].
   All the elements of this new vector are initially
   physically equal to [dummy] (in the sense of the [==] predicate).
   When [capacity] is omitted, it defaults to 0. *)
{@@gospel|
  a = create ?capacity ~dummy
      requires let capacity = match capacity with
                 | None -> 0 | Some c -> c in
               0 <= capacity <= Sys.max_array_length
      ensures  length a.view = 0
|}
```
