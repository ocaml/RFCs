# Improving the infrastructure around "names of symbols"

Inside the compiler there are many places where we manipulate the names of
_symbols_. Symbols identify statically-allocated values in object files emitted
by the compiler. Their names typically include information about which source
file they came from, which pack was in effect, which variable they correspond to
and some kind of unique stamp. During compilation they may also become
"mangled"---that is to say, encoded according to some OCaml- and/or
target-specific convention (for example to avoid special characters or to
add a particular prefix).

Much of this information is propagated using the type "string", which is
straightforward, but has two disadvantages:

- No extra information (for example, whether the symbol is local or global) can
  be carried along with the name itself without changing many pieces of code.

- It can be unobvious what properties (for example, packing or mangling
  conventions) hold of a given name.

Furthermore:

- Much of the infrastructure related to managing object file symbols in
  Compilenv was made rather complicated when Flambda was merged.

- The various ways in which symbol encoding and mangling are performed can
  be difficult to follow at present (particularly on x86 platforms where the
  x86-specific DSL code deals with some of this).

## Main aims of this proposal

Our main aims are as follows:

1. Provide structured representations of symbols that are appropriate to
   the phase(s) of the compiler where they are used.  By doing this, we gain the
   ability to carry along extra information, and to readily identify what
   variety of symbol is being manipulated simply by looking at its type.

2. Simplify the general area of Compilenv and make it more discoverable.

## User-visible consequences of this work

When building an .ml file with the "-for-pack" option, we introduce a new
requirement that the corresponding .mli file must also be built with the
"-for-pack" option.

This is the only material user-visible change.  However we think this should be
straightforward for users to cope with (especially since Dune doesn't use
packed modules at all, and we suspect ocamlbuild may already satisfy this
requirement).

## Other expected benefits of this work

We expect this work to have benefits across the compiler codebase, including:

- Making the implementation of Leo's namespaces proposal more straightforward.

- Making it clear when unit names with and without packing prefixes
  are being used.

- The ability to propagate information such as whether a symbol is local,
  global or hidden, which can have benefits such as reduced executable size
  (cf. PR#8689).

- The potential to factor out code from the backends that is common across all
  architectures. There is increased commonality here nowadays as some of the
  more esoteric backends are being, or have been, removed.

## People involved

Pierrick Couderc (OCamlPro)
Mark Shinwell (Jane Street)
Leo White (Jane Street)

## Drawbacks of the change and alternatives to the change

Build systems would need to be adapted to specify the "-for-pack" option
when compiling .mli files.

There is a small risk of linking errors if the new pipeline for the naming
of symbols has a bug.  This risk can be made acceptably small by usual
methods of testing, including on the full Jane Street tree, and the INRIA CI
system that has various different kinds of platforms.

There are no known alternative proposals to this change.

## Unresolved questions

None known at present.

# An overview of the basic types on which we build our new abstractions

## Compilation units

We define a "compilation unit" as being the identifier of an OCaml source
file that is being compiled.  This notion already exists in Flambda, but in
this proposal we enhance and extend it, allowing it to be used from the
type checker right through to the translation to Cmm.

These compilation unit identifiers (whose types are abstract) have two parts:
1. Any packing prefix in effect
2. The _name_ of the compilation unit.

Use of these new identifiers in the type checker and when checking validity of
.cmx files being loaded helps to make it clear to the reader of such code what
varieties of names are being handled. At present there are some surprising
subtleties, for example in Compilenv, where some of the values of type "string"
have packing prefixes and in other cases they do not.

## Object files

From the Cmm language onwards, name mangling causes the information provided by
compilation unit identifiers to be subsumed directly into the names of symbols
(see "Backend symbols", below). However it is still useful to track the
provenance of such symbols. This is done by using a straightforward notion of
"object file". The notion of object file is subtly different from that of
compilation unit: at this stage of the compiler, we generate code (e.g. the
startup file) that did not arise from OCaml source files. As such we use a
type that looks like this:
```ocaml
type t =
  | Current_compilation_unit
  | Another_compilation_unit
  | Startup
  | Shared_startup
  | Runtime_and_external_libs
```
## Object file section abstraction

We provide a small module that provides a notion of "object file section", to
correspond to the notion of section in executable file formats. This is used to
track where symbols are defined, thus providing the opportunity for increased
checks at compile time (see "Assembly symbols", below), and reducing the risk
of unusual errors from the assembler.

## Target system abstraction

A small module called Target_system provides distilled, easy-to-use information
about the compilation target (for example which assembler is being used). The
addition of this module should also simplify existing code that tests for such
features.

# An overview of the symbol types themselves

## Middle-end symbols

These symbols name statically-allocated constants and code and are used inside
Closure and Flambda. The various things they may point at are as follows:

- A module block
- A variable found to be constant and lifted
- A closure found to be constant and lifted
- An anonymous lifted constant
- A predefined exception
- The code pointer of a function

The code pointer case is the only one where a middle-end symbol does not
point at a well-formed OCaml value.

With the exception of predefined exception symbols, middle-end symbols are
always associated with a compilation unit.

The names of middle-end symbols are not encoded or mangled in any way.

## Backend symbols

These correspond to names of object file symbols together with knowledge about
whether such symbols refer to code or data. Backend symbols that point at data
always point at well-formed OCaml values. Backend symbols appear from the Cmm
language onwards and are often created from middle-end symbols.

They may represent not only symbols arising from normal OCaml compilation units,
but also symbols referenced via "external" and those living in the startup files
and runtime. Given this, as mentioned above, object file symbols are tied to the
notion of "object file" rather than "compilation unit".

Backend symbols include OCaml-specific, but not target-specific, name mangling.

## Assembly symbols

Assembly symbols represent the names used in the assembly, object and executable
files resulting from compilation. (Unlike _labels_, assembly symbols are named
entities that are potentially accessible from outside an object file; they may
also be seen when an object file is examined, e.g. via objdump.)  Assembly
symbols are created in the emitters, usually from backend symbols.

Assembly symbols identify both which object file they occur in and which section
within such file. This enables certain checks to be performed when constructions
involving the symbols are built. (One example is to ensure that arithmetic
expressions on symbols satisfy the constraints of platform assemblers.)

The code behind the abstraction of assembly symbols knows how to accommodate
target-specific name mangling conventions (including the addition of relocation
information).

Unlike backend symbols, assembly symbols may point anywhere.

# Changes to Compilenv and associated code

We propose one notable change to this area of the compiler, which enables the
code to be significantly simplified:

- Improvements to the handling of packed compilation units, in particular so
  that the full name (i.e. including the pack prefix) of a compilation unit
  can always be known, without having to read the .cmx file.

This has the user-visible consequence described above, namely:

- When building an .ml file with the "-for-pack" option, the corresponding .mli
  file must also be built with the "-for-pack" option.

This change fits well with the other changes that are necessary to incorporate
the new types we describe above. It is appropriate to do both together. The
other changes are:

- Simplification of the existing code that currently lives within Compilenv and
  the provision of improved checks with respect to packed modules.

- Compilenv, part of the middle end, is renamed to Compilation\_state.

- Within Compilation\_state, we introduce two sub-modules, one called
  Closure and one called Flambda.  This provides a clear separation between
  the functionality required by these two different middle-ends, which is not
  the case at the moment.

- Linking-related information that used to live in Compilenv is moved into
  Linking\_state, a new module, which forms part of the backend.  This aims
  to promote improved separation of concerns.

- We provide fully doc-commented interfaces for these modules.  We believe the
  new interface is more discoverable than the old.
