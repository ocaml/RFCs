# The problem

OCaml's module system does not allow splitting mutually recursive type
definitions into several compilation units.  This encourage to put all
definitions in the same place, exposing all structural
definitions. One would sometimes prefer to have multiple modules, each
exposing one type, perhaps in an abstract or private from (together
with functions operating on that type).

This RFC attempts to propose a pragmatic work-around for this
situation, taking inspiration from the widely used work-around for the
similar problem in the core language (references to implement forward
references to functions across compiltation units).


# Proposed solution

- Extend type expressions with "global type-constructors":

       ('a1, ..., 'an) "foobar"

  where "foobar" is an arbitrary string literal (or restricted in some way).

- Such type constructors behave as purely abstract ones (unification/equality
  on arguments; equality on the name).

- The type-checker checks that in a given compilation unit, a given
  name is never used with two different arities, and that mapping
  from name to arity is stored in .cmi files. Whenever a .cmi file
  is loaded, a check is performed on arities (between two .cmi files,
  and between an external .cmi file and the current unit).

- An implementation can define a global type-constructor with a concrete
  definition (a type expression), with a syntax to be determined, let's say:

      type ('a1, ..., 'an) "foobar" := <TYPEXPR>

  This is not allowed in local modules or inside a functor's body,
  only at toplevel and in nested modules.

  The rest of the compilation unit is type-checked with an environment
  that remember how to unroll the "foobar" global type-constructor.

- To be discussed: can we expose such definitions in the module interface
  and "import" them from external units?

- .cmo and .cmx files remember which global type-constructors are defined
  in the corresponding implementation, and the linker checks that at most
  one implementation linked into the program provides the definition
  for each name.

- We arrange so that Dynlink enforces a similar invariant: the main program
  could use an undefined global type-constructor, whose definition is
  provided at runtime by some dynlinked unit (but only one!).
