# Unit headers for OCaml source files

## Context

In OCaml, source files play a double role:

- They are interpreted inside the language as modules, formed by
  sequence of structure items. (Modules can be nested, but a file
  always acts as a toplevel module.)

- They are interpreted by the compilation tools as "compilation
  units", the primary units of compilation and linking, whose
  dependencies on other compilation units are tracked and whose
  linking order determines the program semantics.

  (Technically a compilation unit is formed by a pair of a .ml and
  a .mli file, but sometimes only one of them when the other does not
  exist.)


Some things that OCaml programmers can express only make sense for
compilation units, not modules. Currently they can only be expressed
through compiler command-line options, typically stored in the build
system. For example:

- Dependencies on other compilation units or (in general)
  archives/libraries/packages.
- Global compilation options (-safe-string, -rectypes).

Sometimes it would be convenient, even important, to specify those
aspects in the source code itself, but there is no place in the syntax
to specify them: they are not valid structure items as they don't make
sense inside an arbitrary (nested) module.

### One example of the problem

One example use-case is `[@@@warning "-missing-mli"]`: we would like to
let users explicitly disable the new `missing-mli` warning
(introduced by [#9407](https://github.com/ocaml/ocaml/pull/9407) in
4.13~dev) inside a particular .ml file, indicating that it
intentionally does not have a corresponding .mli file.

This warning is implemented at the level of compilation units, not
during the checking/compilation of the module code, so the current
implementation of `[@@@warning ..]` does not support disabling it: it
only enables/disable warnings for the following structure items in the
current module.

A proposal exists to change the semantics of toplevel `@@@warning`
attributes to remain in scope for the whole checking/compilation of
the compilation unit, see
[#10319](https://github.com/ocaml/ocaml/pull/10319).  This is
a special case of one the two options discussed in this RFC, and it
led to the present discussion.


## Proposals

Two proposals to address this issue, one "implicit" and one
"explicit".

The `[@@@warning "-missing-mli"]` PR implements the implicit proposal.

I prefer the explicit proposal.


### Implicit proposal: handle toplevel attributes/extensions at the compilation unit level

We could consider that floating attributes and extensions that are at
the toplevel of a file are not interpreted as "normal" structure
items, at the level of the module, but instead as "unit"
attributes/extensions at the level of the unit.

```ocaml
let foo = ...

[@@@warning "-missing-mli"] (* warning setting for the whole compilation unit *)

module Foo = struct
  [@@@warning "..."] (* warning setting for a submodule only
end
```

Pros:

1. Reasonably easy to implement, no syntax change (we reinterpret
   syntax differently).

2. This is consistent with the way toplevel directives `#foo ;;` are
   handled today: toplevel directives are only valid at the toplevel,
   but can be mixed with other structure items.

Cons:

1. Confuses two notions.

2 We lose the current property that any OCaml code can be moved inside
  a submodule, preserving its meaning.

3. We cannot hope to extend this idea in the future to support global
   settings, such as `-rectypes`, because it would be a mess to allow
   those to change in the middle of other structure items.

#### Variant

One possible variant of this proposal would be to specify certain
attributes/extensions as "header attributes", that have the same
syntax as floating structure/signature-level attributes/extensions,
but can only be used at the beginning of the file (before any
non-header construct). This solves Cons.3, but aggravates Cons.1 by
creating more surprises for users (certain toplevel floating
attributes can be moved around and other not, etc.).


### Explicit proposal: create a "header" extension for compilation-unit configuration

Instead of implicitly treating toplevel attributes/extensions as
scoping over the whole compilation-unit, we propose a builtin
`ocaml.unit_header` extension whose content should be understood as scoping
over the whole compilation unit, not just a module.

```ocaml
[%%unit_header
  [@@@warning "-missing-mli"]
  [@@@rectypes]
]

let foo = ...
```

"unit headers" must be before any other structure/signature items
(comments are allowed before headers). They are the only component of
the .ml syntax that cannot be moved into a submodule (doing so results
in an error).

Note: this RFC does *not* propose a new `@@@rectypes` attribute to be
supported here, it is an example of the sort of feature that could,
over time, become available in unit headers. `[@@@warning
"-missing-mli"]` would be immediately adapted to work (only) in unit
headers, but the RFC itself proposes the "header" notion itself, and
not any specific item to be part of it.


#### Future extensions

In the future, certain toplevel directives could be allowed in the
unit header. This is not proposed here.

We could imagine certain tools querying the unit header of source
files for configuration (or querying the compiler to ask for them),
for example to support a "#require ..." directive integrated in the
build system. This is not proposed here, and in fact not necessarily
the best approach.
I think it probably makes more sense to reserve the header for aspects
of OCaml programs that the compiler knows about (that correspond to
command-line options), so that header interpretation is left entirely
in the compiler. If someday we have a compiler that handles
dependencies (and/or ppx resolution, etc.) by itself, then those
aspects would become naturally specifiable in unit headers.
