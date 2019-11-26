# Unboxed types for OCaml

Suppose you define this type:

    type t = { x : Int32.t; y : Int32.t }

Values of this type carry 8 bytes of actual data. However, in OCaml,
values of this type are represented as:

  - an 8-byte pointer

  - to a 24-byte block  
    (8B header + 2x 8B fields)

  - pointing at two more 24-byte blocks  
    (8B header + 8B `caml_int32_ops` + 4B data + 4B padding)

Below is a proposed extension to OCaml to allow the layout of data
types to be controlled by the programmer. This proposal won't affect
the type defined above, but will allow you to define more compact
layouts.  There are two main aspects. The first part, "unboxed types",
lets you write the following:

    type t = { x : #int32; y : #int32 }

Here, the x and y fields have the unboxed type `#int32`, which
consumes only 4 bytes. Values of type `t` are still boxed: they are
represented as 8-byte pointers to a 16-byte block, consisting of an
8-byte header and two 4-byte fields. Despite containing unboxed
fields, the type t is an ordinary OCaml type: it can be passed to
normal functions, stored in normal data structures, and so on.

The second part, "unboxed records", additionally lets you write:

    type u = #{ x : #int32; y : #int32 }

This is an unboxed record of unboxed fields. Values of this type are
represented as a pair of 32-bit integers. No allocations are made when
creating values of this type. If one of these is passed to a function,
it is passed in two registers.

The price is that unboxed records are more awkward to use: you cannot
have, for example, a `u list`.

## Layouts

The central issue with adding unboxed types to OCaml is that
operations that used to work for all possible types don't necessarily
work for unboxed types. Here's an example:

    let diag x = (x, x)

This function is polymorphic, having type:

    val diag : 'a -> 'a * 'a

However, if you try to use it with `'a = #int32`, then you end up
creating a tuple containing unboxed 32-bit integers where the garbage
collector expected tagged or boxed values. Segfaults ensue.

So, unboxed types mean that polymorphic definitions need to be
polymorphic over only a subset of the possible types (say, the boxed
ones). We do this by introducing *layouts*, which classify types in
the way that types classify values. (The technical term for such
things is *kinds*, but we're avoiding that word because (a) it's
unfamiliar to programmers who aren't type theorists and (b) it doesn't
describe what we're using kinds for). The layout of a type determines
how many bytes of memory values of that type consume, how many and
what kind of registers are needed to pass such values to functions,
and whether locations holding such values need to be registered with
the GC.

Once we have layouts, the type `'a -> 'a * 'a` is a short form of:

    val diag : ('a : value) . 'a -> 'a * 'a

The layout `value` is the layout of ordinary OCaml values (boxed or
immediate). This means that trying to use diag at type `#int32 ->
#(int32 * int32)` causes a compilation error, as `#int32` is not of
layout `value`.

The available layouts include:

    value
    immediate
    int32
    int64
    float
    any_layout

As well as the above primitive layouts, layouts may be combined using
the `*` operator. Unlike types, layouts are 'flat'. That is, the
following are all equivalent:

    (value * value) * value
    value * (value * value)
    value * value * value

There is a subtyping relation between these layouts, where `immediate
<= value`, and `L <= any_layout` for all layouts `L`. This means that
all types of layout `immediate` also have layout `value`, and every
type has layout `any_layout`. (The subtyping relationship between
product layouts is pointwise, and only between products of the same
length).


## Unboxed types and records with unboxed fields

All of the currently-existing types in OCaml have layout `value`. Some
also have layout `immediate`, such as `int` or `unit`. We're planning
to add unboxed versions of a few builtin types, such as `#int32` and
`#float`. This gives:

    type float : value
    type int32 : value
    type int : immediate
    type #float : float
    type #int32 : int32

All of these can be passed and returned from functions, but only the
`value` types (including `immediate` ones) can be stored in, say,
lists. Note that the existing `+.`, etc. primitives won't work on
`#float`, because they have the wrong type. New ones for `#float` will
be made available. (Same for `#int32`, etc.)

Unboxed types will be allowed in record fields:

    type t = {
      foo : int;
      bar : #float;
      ...
    }

Because of how the OCaml runtime works, there are some restrictions on
how boxed and unboxed types can be mixed in a record. If there are any
fields which might contain pointers (that is, are of layout `value`
but are not `immediate`), then all fields must be of layout
`value`. So, you can mix `int` and `#float` in a single record, but
not `string` and `#float`.

## Unboxed tuples and records

The second part of this proposal is to allow user-defined unboxed
types (by contrast to what's above: user-defined boxed types
containing unboxed fields). The simplest of these is unboxed tuples:

    #( int * float * string )

This is an unboxed tuple, and is passed around as three separate
values.  Passing it to a function requires three registers.
The layout of an unboxed tuple is the product of the layout of its
components. In particular, the layout of `#( string * string )` is
`value * value`, not `value`, and so it cannot be stored in, say, a
list.

Unboxed records are in theory just as straightforward, but have some
extra complexity just to handle the various ways that types are
defined and used. Suppose you write:

    type t = { x : int32; y : int32 }

Sometimes we want to refer to `t` as a boxed type, to store it in
lists and other standard data structures. Sometimes we want to refer
to `t` as an unboxed type, to avoid extra allocations (in particular,
to inline it into other records). The language never infers which is
which (boxing and allocation is always explicit in this design), so we
need different syntax for both.  So, `type t = { ... }` defines *two*
types, the boxed record `t : value` and the unboxed record `#t :
int32 * int32`. They have different syntax for introduction and
elimination:

    let p : t = { x = a; y = b }
    let px = p.x
    let q : #t = #{ x = a; y = b }
    let qx = q.#x

An unboxed record like `q` cannot itself contain mutable
fields. However, a boxed record can contain a mutable field of type
`q`. This can be used to inline one record inside another, e.g.:

    type s = { foo : int; mutable bar : #t }

This defines a boxed record, layed out as three fields. We will
support efficient updates of sub-parts of `bar`, at least with the
following syntax:

    x.bar <- { x.bar with y = y' }

It would be nice to also allow the syntax `x.bar.y <- y'`, although
this is a little trickier to understand and implement, since it is
`x.bar` rather than `x.bar.y` which is the mutable field.

Type aliases do not automatically bind an unboxed variant, because
the scoping of type aliases should not depend on whether they expand
to records or not. So, if you write:

    type u = t

then you don't get `#u`. You can get around this by re-exporting the
definition:

    type u = t = { x : int32; y : int32 }

which gets you both `u = t` and `#u = #t`. You can also define an
alias directly to the unboxed type:

    type u = #t

Similarly, an abstract type abstracts only the boxed type:

    module M : sig
      type t
    end = struct
      type t = { x : int32; y : int32 }
    end

defines `M.t` but not `M.#t`. (It may be useful to allow exporting
`M.#t` despite `t` being abstract, perhaps with syntax like `type t :
value with #t : int32 * int32`).


## Polymorphism, abstraction and type inference

The major change to the internals of the type system is that type
variables need to be annotated with their layout. This affects all
uses of type variables, both in type definitions and in value
descriptions. The syntax is extended to allow these to be given
explicitly, e.g.:

    type ('a : value, 'b : immediate) t = Foo of 'a * 'b
    val fn : ('a : immediate) . 'a ref -> 'a -> unit

In signatures, type variables that are not bound with either of the
above syntaxes are given layout `value`.

Layout annotations also appear on abstract types:

    type ('a : value, 'b : int32) t : immediate

Again, if this annotation is omitted it's assumed to be `value`.

It may seem a little weird here to allow abstraction over types of
layout `int32`. After all, there's only one such type (`#int32`).
However, this ability becomes valuable when combined with abstract
types: two different modules can expose two different abstract types
`M.id`, `N.id`, both representing opaque 32-bit identifiers. By
exposing them as abstract types of layout `#int32`, these modules can
advertise the fact that the types are represented in four bytes
(allowing clients of these modules to store them in a memory-efficient
way), yet still prevent clients from mixing up IDs from different
modules or doing arithmetic on IDs.

### Unification

As above, type variables are annotated with the layout of the types
over which they range. During unification, newly-created type
variables initially range over `any_layout`. Two interesting things
can then happen: unification of two variables or unification of a
variable and a non-variable. (Unification of two non-variables works
as before).

**Two variables** If we have two type variables `'a : L`, `'b : K`,
then their unifier is a type variable that ranges over only types that
could be assigned to both `'a` and `'b'`. Its layout is therefore the
intersection of the layouts `L` and `K`, that is, the layout of types
that have both layouts. This can be done in general, as layouts form a
partial semilattice: either there is some layout which is exactly the
intersection of `L` and `K`, or else the two layouts have no types in
common (causing a unification error).

**One variable, one non-variable** To unify `'a : L` with a
non-variable type `t`, we must check first that `t : L`. The
interesting case here is when `t` is of the form `(p, q) r`, for some
`r` in the type environment.

There is a subtlety here: the layout of `(p, q) r` can in general depend on
the layouts of `p` and `q`. For instance:

    type ('a : value) id = 'a

Since `'a : value`, we know that `'a id` is always of layout `value`,
but if `'a = int` it may also have the more specific layout
`immediate`.

For each type constructor `(_, _) r` in the environment, we store
along with it a layout `K` which is an upper bound for the layout of
any instance of `r`. (For `'a id` above, this is `value`). When
checking `(p, q) r : L`, we compare `L` and `K`. If `K` is a subtype
of `L`, then the check passes. However, if the check fails and `r` is
an alias, we must expand `r` and try again. We only report a
unification failure once the check fails on a type which is not an
alias.

For example, suppose we have the unification variables `'a : value`
and `'b : immediate`, and we're trying to unify `'a id = 'b`. We need
to check `'a id : immediate`, but the environment tells us that the
layout of `_ id` is `value`. So, we expand `'a id` to `'a` and retry
unification, which succeeds after unifying `'a` and `'b` (the
resulting type variable has layout `immediate`). The effect here is
that after unifying `'a id` with `'b`, we leave `'a` as an
uninstantiated unification variable, but its layout is restricted to
`immediate`, to match `'b`.

### Type well-formedness

Thanks to how types are constructed in OCaml, we need no separate
well-formedness check. Suppose we're given the following type:

    type ('a : immediate) t = ...

and the user attempts to write `string t`. The syntax `string t` is
converted to a type expression by first constructing a template with a
fresh unification variable `'b t`, and then unifying `'b` with
`string`. Here, since `'b` ranges over layout `immediate`, the
unification with `string` will fail.

### Generalisation

In OCaml, type variables which are not constrained are *generalised*
at the nearest enclosing `let`, and can be used polymorphically from
then on. Done naively, this means that the function:

    let diag x = (x, x)

would have the generalised type:

    val diag : ('a : any_layout) . 'a -> 'a * 'a

In many ways, this type makes sense: this definition of `diag` does
really work for any type of any layout. But a polymorphic function of
this type is not compilable, as we cannot allocate storage for a type of
`any_layout`: we don't know how many registers it needs, how many
bytes of memory it should take when stored, or whether it needs to be
registered with the GC!

So, when generalising a type variable, if its layout is `any_layout`
then we change the layout to `value` instead, giving `diag` the
compilable type:

    val diag : ('a : value) . 'a -> 'a * 'a

We retain principal *types*, but we don't have principal *layouts*:
for any expression, there's a best type for any given layout, but
there's no best layout (or at least, the best layout isn't always
compilable).

We only need to do this when the polymorphic type variable is used as
an input to the function (i.e. in a contravariant position). If the
polymorphic type variable is only used positively, then the function
can never manipulate any values of this type, and remains compilable
(similar reasoning justifies the relaxed value restriction). For
instance, for the function `raise`, we can actually compile the
following type:

    val raise : ('a : any_layout) . exn -> 'a

The function's type only mentions `'a` in positive positions, so it
never manipulates any values of type `'a`. This is useful: it means
that functions like `raise` that never return can be used where
e.g. an `#int32` is expected.

## Beyond records

As well as the basic unboxed types and unboxed records, we'd like to
have some subset of the following.

**Variants**

So far, we've only discussed records, when unboxed variants are
equally useful. The general case is quite tricky, but several special
cases are easy and useful: variants containing only constant
constructors, or variants whose non-constant constructors have the
same types.

The tricky part of the general case is that the layout of a variant
(including number of fields and whether those fields need to be
registered with the GC) varies according to the tag. One simple way
around this difficulty is to always allocate enough space for all
fields: this might make sense for small variants like `result`, where
the savings from unboxing outweigh the extra word used to represent them.

**Nullable types**

Unboxing the `'a option` variant would improve efficiency, but an
unboxed `t option` still requires two words: one to store the `t` (if
`Some`), and one to store the tag (`Some` or `None`). Picking a
special magic value for `None` isn't enough to cram options into one
word, since we need to be able to distinguish `None` from `Some None`,
`Some (Some None)`, etc.

In many cases, we know statically that the argument to option cannot
be another option (e.g. `string option`, `int option`, `float
option`...). In these cases, we can use *nullable types* instead. In a
nullable type, the type parameter `'a` has layout `non_nullable`,
which is a subtype of `value`. This means that it is represented as an
ordinary OCaml value (a pointer or tagged integer), but that it cannot
be another nullable type, so the equivalent of `t option option` gives
a compile error. The nullable type then only needs one word of
storage.

**Padding**

As well as being able to control boxing of types, it's sometimes also
useful to control the bytes of padding between unboxed elements. (This
is sometimes useful for optimisation, but the main application is
being able to directly map externally-specified formats like protocols
or C structs).

**Bitpacking**

Related to byte-level padding, but quite different in implementation,
bitpacking allows multiple unboxed fields (say, boolean flags) to
occupy the same byte. In particular, a record containing <32 booleans
could be given an `int32` layout rather than a product layout (and
thus be passed around in just a single register).

**GADTs**

As well as refining types, GADTs should be able to refine layouts as
well. It would be nice to be able to define the following:

    type _ is_immediate =
    | Immediate : ('a : immediate) -> 'a is_immediate
    | Other : ('a : value) -> 'a is_immediate


## Future projects

**Monomorphisation**

"Uncompilable" types like `diag : ('a : any_value) . 'a -> 'a * 'a`
can actually be compiled, by compiling a separate version of `diag`
for each layout. In fact, we need fewer layouts than that: we can
share the same version for `value` and `immediate`. Generally,
we need one version per 'sort' (roughly, register width). Within a
sort, there may be multiple layouts with subtyping, but there is no
subtyping between sorts. Then, we could type the general `diag` as:

    val diag : ('s : sort) ('l : 's layout) ('a : 'l) . 'a -> 'a * 'a

This means that typechecking needs to support unknown sorts and
unification of sort variables. This shouldn't be too hard: since
there's no subtyping between sorts, standard unification works fine.

As well as allowing definitions that get monomorphised, this approach
can also support "weakly layout-polymorphic" definitions. For
instance, we can compile `diag` at only a single sort, but use
unification sort variables to delay the choice of that sort until we
see how it's used, rather than defaulting to the sort `value`.

**Macros**

The most interesting use of a sort system is to support, rather than
automatic monomorphisation, manual instantiation of sorts and layouts
using macros, to allow well-typed well-laid-out code generation at
compile time. This sounds all sorts of fun, but it's a different
project entirely.
