# Unboxed types in OCaml

This proposal is discussed at this [pull request](https://github.com/ocaml/RFCs/pull/34).

OCaml currently has two attributes that this proposal may easily be confused
with:

* We can mark arguments to external calls as `[@unboxed]`, as [documented in the
  manual](https://v2.ocaml.org/manual/intfc.html#s%3AC-cheaper-call). This
  proposal essentially takes this idea and expands it to be usable within OCaml
  itself, instead of just at the foreign function interface boundary.
* We can mark some types as `[@@unboxed]`, as briefly described in a bullet
  point in [this manual
  section](https://v2.ocaml.org/manual/attributes.html#ss:builtin-attributes).
  This attribute, applicable to a type holding one field of payload (either a
  one-field record or a one-constructor variant with one field), means that no
  extra box is allocated when building values of that type. It is notionally
  separate from this proposal, though there is an interaction: see the section
  mentioning `[@@unboxed]` in its title.

## Motivation

Suppose you define this type in OCaml:

```ocaml
type t = { x : Int32.t; y : Int32.t }
```

Values of this type carry 8 bytes of actual data. However, in OCaml,
values of this type are represented as:

  - an 8-byte pointer

  - to a 24-byte block
    (8B header + 2x 8B fields)

  - pointing at two more 24-byte blocks
    (8B header + 8B `caml_int32_ops` + 4B data + 4B padding)

There are some efficiency gains to be had here.

## Principles

In this design, we are guided by the following high-level principles.
These principles are not incontrovertible, but any design that fails
to uphold these principles should be considered carefully.

**Backward compatibility**: The extension does not change the
correctness or performance of existing programs.

**Availability**: Unboxed types should be easy to use. That is,
it should not take much change to code to use unboxed types and
values.

## Unboxed primitives

This is a proposed extension to OCaml to allow the layout of data
types to be controlled by the programmer. This proposal won't affect
the type defined above, but will allow you to define better layouts.
With unboxed types, we can write the following:

```ocaml
type t = { x : #int32; y : #int32 }
```

Here, the `x` and `y` fields have the unboxed type `#int32`, which
consumes only 4 bytes. Values of type `t` are still boxed: they are
represented as 8-byte pointers to a 16-byte block, consisting of an
8-byte header and two 4-byte fields. Despite containing unboxed
fields, the type t is an ordinary OCaml type: it can be passed to
normal functions, stored in normal data structures, and so on.

On the other hand, the new type `#int32` is unboxed. This means that
it can't be used to instantiate ordinary type parameters.  For
example, `fun (x : #int32) -> Fn.id x` will not work: `Fn.id` doesn't
work on types that are not represented by exactly one word. Similarly,
you can't have a `#int32 list`, because a list expects its contents to
be represented by exactly one word. Getting this right is important
for, e.g., the garbage collector, which expects values in memory to
have a certain format.

Unboxed types, thus, require a trade-off: they are more compact,
but do not work as smoothly as normal boxed types.

The following types will be available in the initial environment, given with
example literals of that type:

```ocaml
#()  : #unit
#42b : #int8
#42s : #int16
#42l : #int32
#42L : #int64
#42n : #nativeint
#42. : #float
```

## Layouts

So, unboxed types mean that polymorphic definitions need to be
polymorphic over only a subset of the possible types (say, the boxed
ones). We do this by introducing *layouts*, which classify types in
the way that types classify values. (The technical term for such
things is *kinds*, but we're avoiding that word because (a) it's
unfamiliar to programmers who aren't type theorists and (b) it doesn't
describe what we're using kinds for.) The layout of a type determines
how many bytes of memory values of that type consume, how many and
what kind of registers are needed to pass such values to functions,
and whether locations holding such values need to be registered with
the GC.

Once we have layouts, the type of, say, `diag : 'a -> 'a * 'a` is a short form
of:

```ocaml
val diag : ('a : value) . 'a -> 'a * 'a
```

The layout `value` is the layout of ordinary OCaml values (boxed or
immediate). This means that trying to use diag at type `#int32 ->
#int32 * #int32` causes a compilation error, as `#int32` is not of
layout `value`.

The following layouts are available, given with example types of that layout:

```ocaml
(* concrete ones *)
string     : value
int        : immediate
#unit      : void
#int8      : bits8
#int16     : bits16
#int32     : bits32
#int64     : bits64
#nativeint : word
#float     : float64

(* indeterminate ones *)
any
gc_friendly
gc_ignorable
```

(On 64-bit machines, `word` has the same representation as `bits64`, but
we don't want to bake this coincidence into the type system.)
As well as the above primitive layouts, layouts may be combined using
the `*` and `+` operators. Unlike in types, `*` and `+` are associative
in layouts. That is, the
following are all equivalent at runtime:

```
(value * value) * value
value * (value * value)
value * value * value
```

There is a (transitive) subtyping relation between these layouts, as pictured
here:

```
           ----any-----
          /            \
      gc_friendly   gc_ignorable-------------
         |    \    /   |     \               \
       value   void   word   float64     bits_8..64
         |
     immediate
```

(It would be safe to add `immediate <= word`, as well, but we don't for now. See
an Extension toward the end of this document.)  The subtyping relationship
between product and sum layouts is pointwise, and only between products/sums of
the same length. A product/sum is a subtype of `gc_friendly`
(resp. `gc_ignorable`) iff its components are subtypes `gc_friendly`
(resp. `gc_ignorable`). All products/sums are subtypes of `any`.

A `gc_friendly` layout is one that the garbage collector knows how to treat:
these are pointers to gc-managed memory and tagged immediates. A `gc_ignorable`
layout is one that the garbage collector is free to ignore: these values are not
pointers to gc-managed memory (and contain no such pointers).

We might imagine that we could make, e.g. `(value * value) * value`
a subtype of `value * (value * value)` (and vice versa), but this
would play poorly with type inference. For example, if we had
`'l1 * 'l2 = value * value * value` (where `'l1` and `'l2` are unification
variables), how could we know how to proceed? Perhaps we could make
a subtyping relationship here, but only when there is an explicit check;
we leave this off the design for now.

Names used in layouts (both the alphanumeric names listed above and the infix
binary operators) live in a new layout namespace (distinct from all existing
namespaces) and are in scope in the initial environment. Though this proposal
does not introduce syntax to do so, we can imagine constructs defining new names
in the layout namespace that could shadow these existing names.

### Layouts and the `[@@unboxed]` attribute

Because a layout describes how a type is stored in memory and passed to and from
functions, an `[@@unboxed]` type must have the same layout as its field
type. That is, in both of the following declarations, `outer` is assigned the
same layout as `inner`:

```ocaml
type outer = { f : inner } [@@unboxed]
type outer = K of inner [@@unboxed]
```

Without the `[@@unboxed]` attribute, both `outer`s would be `value`s. It is
possible that recursion prevents us from finding the layout of the right-hand
side:

```ocaml
type loopy = { f : loopy } [@@unboxed]
```

In this case, we default the layout of `loopy` to `value`, although a programmer
can also write a layout annotation to choose a different layout, as follows:

```ocaml
type immediate_loopy : immediate = { f : immediate_loopy } [@@unboxed]
```

Both `loopy` and `immediate_loopy` are uninhabited.

## Unboxed types and records with unboxed fields

All of the currently-existing types in OCaml have layout `value`. Some
also have layout `immediate`, such as `int` or `unit`. We're planning
to add unboxed versions of a few builtin types, such as `#int32` and
`#float`. This gives:

```ocaml
type float : value
type int32 : value
type int : immediate
type #float : float64
type #int32 : bits32
```

All of these can be passed and returned from functions, but only the
`value` types (including `immediate` ones) can be stored in, say,
lists. Note that the existing `+.`, etc. primitives won't work on
`#float`, because they have the wrong type. New ones for `#float` will
be made available. (Same for `#int32`, etc.)

Unboxed types will be allowed in record fields:

```
type t = {
  foo : int;
  bar : #float;
  ...
}
```

The OCaml runtime forbids mixing safe-for-GC fields and unsafe-for-GC
fields in a single block. See the mixed-block restriction, below.

## Restrictions around known representations

A key aspect of the design of unboxed types is that we must know
the representation of a type before we can ever manipulate a value
of that type. For example, check out `wrong` here:

```ocaml
let wrong : ('a : any). 'a -> 'a = fun x -> x
```

This function is impossible to define; it would get rejected at
compile time. The reason is that the compiled `wrong` function must
take an argument somehow and then return it somehow. Yet, if we don't
know the representation of that argument, we cannot do this! Does the
argument fit in one register or two? Is it a pointer (that the GC would
have to be aware of) or not? We cannot know: the argument's layout is
`any`.

Now consider this function:

```ocaml
let strange : ('a : any). string -> 'a = fun msg -> raise (Msg msg)
```

This one is OK. This function never has to manipulate the value
of type `'a`: that value is produced by a tail call, and so the code for
`strange` never has to move it. (`raise` is magical, allowed to compile
by virtue of the fact that it doesn't actually return to its caller.) In
practice, a function like `strange` would always be called with a known
layout (such as `value` or `bits32 * bits32`), and so the caller would know
where and how to expect the result. The key aspect of this, though, is that
we can compile `strange` to executable code without knowing the representation
of `'a`.

Here's one last example:

```ocaml
let rec silly : ('a : any). unit -> 'a -> 'a =
  fun () -> silly ()
```

(The `unit` argument is just to prevent
the definition from falling into a loop right away.) This definition
is also accepted. Note that, even though the type of `silly` has a
type of layout `any` to the left of an arrow, it does not
manipulate a value of that type. Thus, this is fine.

Abstracting from these examples, there are precisely two places we
require a concrete layout -- no `any` here.

1. When binding a term variable of type `t`, `t` must have a concrete
layout.
2. When passing a term of type `t` to a function, `t` must have a
concrete layout.

There is no restriction on returning a value from a function.
See "Kinds are calling conventions" for more exploration of these
restrictions.

There is also no restriction on knowing the representation of type variables in
type declarations. For example, the following declaration is fine:

```ocaml
type ('a : any) pair = { first : 'a; second : 'a }
```

The above restrictions imply that any construction of `pair` or pattern-match on
`pair` would work only for instantiations that have known representations, but
we can suffice with just one, flexible type declaration.

## Adding a box

Because of the restrictions around the usage of unboxed types
(both that they cannot work with standard polymorphism and the
restriction just above about mixing representations in record
types), we sometimes want to box an unboxed type. The types
we have seen so far have obvious boxed cousins, but we will soon
see the ability to make custom unboxed types. We thus introduce

```
type ('a : any) box

val box : ('a : any). 'a -> 'a box
val unbox : ('a : any). 'a box -> 'a
```

These definitions are magical in a number of ways:

* It is best to think of the type `box` as describing an infinite
family of types, where the member of the family is chosen
by the layout of the boxed type. (The family is infinite because of
the `*` and `+` layouts, which can be used to create new layouts.)

* It would be impossible to define the `box` function without magic, because
it runs afoul of the restriction around function arguments.
A consequence of this design is that the `box` function cannot
be abstracted over: a definition like `let mybox (x : ('a : any)) = box x`
runs into trouble, because it binds a variable of a type whose layout
is unknown.

* Because the function `box` is really a family of functions, keyed
by the layout of its argument, it is not allowed to use `box` without
choosing a layout. The simplest way to explain this is to say that `box`
must always appear fully applied. In this way, it might be helpful to
think of `box` more as a keyword than a function. But we could, in theory,
allow something like `let box_float : ('a : float64). 'a -> 'a box = box`,
where `box` is unapplied (but specialized). However, there is little
advantage to doing so, and it's harder to explain to users, so let's
not: just say `box` must always appear applied. (Note: GHC has a similar
restriction around similar constructs.)

* There is additional magic around various forms of unboxed types,
as described below.

## Arrays

Going beyond storing just one unboxed value in a memory box, we also
want to be able to store many unboxed values in an array. Accordingly,
we extend the existing `array` to take a type argument of `any`,
as if `array` were declared with

```ocaml
type ('a : any) array
```

Just like we need magical value-level primitives `box` and `unbox` to deal
with `box`, we will also need similar primitives to deal with arrays
of unboxed values. This proposal does not spell out the entire API for this
feature, but we will work it out during the implementation. Regardless
of the API details, the `array` type must have a similar layout-indexed
magic as `box`, though `array` could conceivably use a different memory
layout as `box` does. In particular, the memory format of a boxed variant
(as described below in the section on "Unboxed variants") has a variable length,
making it impossible to pack into an `array`. Thus `array` may choose to use
a different memory format as `box` in order to allow for indexing.

Note that extraction of an element from an array of unboxed values (e.g.
with `get`) requires
copying the element. There are two ways a user might want to get an
unboxed value, via the following hypothetical API:

    get : 'a array -> int -> 'a
    get_boxed : 'a array -> int -> 'a box

If the type `'a` is unboxed, the `get` function copies the value from
the memory allocated as part of the array into registers (or other
destination). To avoid this copying, one might imagine `get_boxed`, intended
to return a pointer to the existing allocated memory within the array.
However, `get_boxed` cannot do this: an `'a box` is expected to be a pointer
to a block with a header, and an element within an array lacks this header.
Instead, we might imagine

```ocaml
get_element : 'a array -> int -> 'a element
```

where `element` is a new type that represented a 1-element slice of an
allocated array. The `get_element` function would indeed just return
a pointer, but it requires yet another magical type `element`, represented
by a pointer to the middle of a block. (This would pose a challenge
for garbage collection, though presumably not an insurmountable one.)

We do not explore this aspect of the design further in this document.
Instead, this consideration of `get_boxed` and `get_element` serves to
explain why there is no loss of run-time efficiency if the memory format used
by `box` and the memory format used by `array` are completely different.

## Unboxed tuples

In addition to new primitive types, like `#int32`, this proposal also
includes user-defined unboxed
types (by contrast to what's above: user-defined boxed types
containing unboxed fields). The simplest of these is unboxed tuples:

```ocaml
#( int * float * string )
```

This is an unboxed tuple, and is passed around as three separate
values.  Passing it to a function requires three registers.
The layout of an unboxed tuple is the product of the layout of its
components. In particular, the layout of `#( string * string )` is
`value * value`, not `value`, and so it cannot be stored in, say, a
list.

Constructing and pattern-matching against an unboxed tuple requires
a `#`: `let ubx : #(int * #float) = #(5, #4.)`. Note that parentheses
are required for construction and pattern-matching unboxed tuples.

Naturally, boxing an unboxed tuple yields a boxed tuple. We thus add
the following rules:

* For all unboxed tuples `#(ty1 * ty2 * ...)`,
`#(ty1 * ty2 * ...) box = (ty1 * ty2 * ...)`. The `...` is meant to
denote that this works for any arity, including 0.

* The syntax `(e1, e2, ...)` continues to work for boxed tuples.

### The mixed-block restriction

The current OCaml runtime has the following restriction:

* For every block of memory, one of the following must hold:

    1. All words in the block (other than the header) may be inspected
    by the garbage collector. That is, every word in the block is either
    a pointer to GC-managed memory or has its bottom bit tagged. This is a
    gc-friendly block.

    2. No words in the block are pointers to GC-managed memory. This is a
    gc-ignorable block.

This is also described at in [Real World OCaml](https://dev.realworldocaml.org/runtime-memory-layout.html).

We call this rule the *mixed-block restriction*, and we call types
whose memory layout is mixed as *mixed-block types*. We imagine that
we will be able to lift this restriction in a future release of the
runtime, but we do not expect to do this as part of our initial rollout
of unboxed types. We thus refer to this restriction in several places
in this specification so that readers might imagine a future improvement.

Note that a mixed-block type has a layout that is neither a subtype of
`gc_friendly` nor a subtype of `gc_ignorable`.

The first consequence of the mixed-block restriction is already apparent:
we cannot store, say, an unboxed tuple `#(string * #float)` in a memory
block, as that block would be mixed.

We thus have the following restriction on mixed-block types: `box` may
never be called on a mixed-block type. Equivalently, the layout of the argument
to `box` must be either a subtype of `gc_friendly` or `gc_ignorable`.
(This is indeed the essense of
the mixed-block restriction.) However, because `box` is implicitly
used by constructors of boxed types -- such as a boxed tuple -- this
restriction applies more widely than code literally calling `box`. For
example, the following would be rejected:

```ocaml
let bad : (string * #float) = "hi", #4.
```

The problem is the implicit call to `box` in the definition, which would
expand to

```ocaml
box #("hi", #4.)
```

Note that the mixed-block restriction never requires looking directly at
*types*; instead, a mixed-block type can be identified by looking only at
its *layout*. This is in keeping with our understanding of `box` as a
family of functions, indexed by layout. The members of the family corresponding
to mixed-block types simply do not exist.

## Unboxed records

Unboxed records are in theory just as straightforward as unboxed tuples, but
have some extra complexity just to handle the various ways that types are
defined and used. Suppose you write:

```ocaml
type t = { x : #int32; y : #int32 }
```

Sometimes we want to refer to `t` as a boxed type, to store it in
lists and other standard data structures. Sometimes we want to refer
to `t` as an unboxed type, to avoid extra allocations (in particular,
to inline it into other records). The language never infers which is
which (boxing and allocation is always explicit in this design), so we
need different syntax for both.

To make this all work, we consider a declaration like the one for
`t` above to actually declare *two* types. To wit, we desugar the
above syntax to these declarations:

```ocaml
type #t = #{ x : #int32; y : #int32 }
type t = #t box
```

That is, we have a primitive notion of an unboxed record, called
`#t`, and then we define `t` just as `#t box`. (The `#{` syntax is
also available to users, if they want to just directly declare an
unboxed record.)

To make this backward compatible (and at all pleasant to program in),
we add a little more magic to `box`:

* In the syntax `e.x` (where `e` is an arbitrary expression and `x`
is a path), if `e` has type `ty box` (for some `ty`), then we treat
the expression as if it were projecting from `ty` after unboxing.

* Along similar lines, we allow record-creation and pattern-matching
syntax `{ ... }` to create and match against `box`.

We might imagine an alternative design without automatically creating
unboxed record definitions in this way, but that design does not uphold
the **Availability** principle as well as the current design.

### Construction

We use a `#` in the syntax for creating unboxed records:

```ocaml
type q = #{ x : int; y : #int32 }
let g : q -> q = fun #{ x; y } as r -> #{ x = x + r.x; y }
```

(If we didn't have the special treatment for `box`, we wouldn't need to
differentiate construction syntax, as we can infer that the record is unboxed
from the field labels, as we do for other record types. We could also
release a first version that requires `#{` and then see if users want to
be able to avoid the `#`. If we drop the `#` requirement, we could simply
default to boxed, unless the type system disambiguation mechanism specifically
selects for unboxed.)

Type aliases do not automatically bind an unboxed version; rather
it is a property of the record type declaration that creates the
extra unboxed definition. However, if you create a transparent alias
like

```ocaml
type u = t = { x : int32; y : int32 }
```

then we translate this to become

```ocaml
type #u = #t = #{ x : int32; y : int32 }
type u = #u box   (* equivalent to type u = #t box *)
```

Naturally, you can also directly alias an unboxed type:

```ocaml
type u = #t
```

Similarly, an abstract type abstracts only the boxed type:

```ocaml
module M : sig
  type t
end = struct
  type t = { x : int32; y : int32 }
end
```

defines `M.t` but not `M.#t`. If you want to export both
an unboxed and boxed version of an abstract type, you can with

```ocaml
module M : sig
  type unboxed_t : value * value
  type t = unboxed_t box
end = struct
  type t = { x : int32; y : int32 }
  type unboxed_t = #t
end
```

We might imagine allowing module signatures to include `type #t`
when they also include `type t`, but as we see here, this feature
is not strictly necessary.

### Field names and disambiguation

If we have a

```ocaml
type t = { x : int; y : int }
```

and write a function

```ocaml
let f t = t.x + t.y
```

what type do we infer for `f`? Of course, we want to infer `t -> int`, but in
the presence of unboxed types, we might imagine inferring `#t -> int`, as `t.x`
is a valid way of projecting out the `x` field of both `t` and `#t`. Yet
inferring `#t -> int` would be terribly non-backward-compatible.

We thus effectively have the boxed projection shadow the unboxed one. That is,
when we spot `t.x` (for a `t` of as-yet-unknown type), we assume that the
projection is from the boxed type `t`, not the unboxed type `#t`. If a user
wants to project from the unboxed type, they can disambiguate using a prefix
`#`, like `t.#x`. Thus, if we have

```ocaml
let g t = t.#x + t.y
```

we infer `g : #t -> int`, though inference is non-principal. Users could also
naturally write

```ocaml
let g t = t.#x + t.#y
```

which supports principled type inference.

To be clear, the `#` mark is just used for disambiguation. The following also
works:

```ocaml
let g (t : #t) = t.x + t.y
```

There is no ambiguity here, and thus the `#` mark is not needed on record
projections.

Although this section discusses only record projections, the same idea applies
to record construction and pattern matches: the field of the boxed record
shadows the field of the unboxed record, though the latter can be written with a
`#` prefix (or can be discovered by type-directed disambiguation).

### Mutation

There are several concerns that arise when thinking about mutable fields and
unboxed types; this section lays out the scenarios and how they are treated.
We assume the following declarations:

```ocaml
type point = { x : #int32; y : #int32 }
type mut_point = { mutable x : #int32; mutable y : #int32; identity : int }
```

1. **Mutable unboxed field in a boxed record**. Example:

    ```ocaml
    type t = { radius : #int32; mutable center : #point }
    ```

    A mutable unboxed field can be updated with `<-`, analogously to a mutable
    boxed field. However, updating an unboxed field might take multiple separate
    writes to memory. Accordingly, there is the possibility of data races when
    doing so: one thread might be writing to a field word-by-word while another
    thread is reading it, and the reading thread might witness an inconsistent
    state. (For records with existential variables and unboxed sums, this
    inconsistent state might lead to a segmentation fault; for other types, the
    problem might arise only as an unexpected result.) We call this undesirable
    action *tearing*.

    To help the programmer avoid tears, an update of a mutable
    unboxed field is sometimes disallowed. We surely want to disallow such an
    update if it is not type safe. We also want to disallow such an update if it
    could violate abstraction: perhaps some abstract record type internally
    maintains an invariant, and a torn record might not support the invariant.

    Some cases are easy: we definitely want to support mutation of `#float`s, and
    we definitely want to disallow mutation of wide unboxed variants (where we
    might update the tag and the fields in separate writes). What about wide
    unboxed records? It depends. Consider the type `t` in this section, and
    imagine `type s = { mutable circle : #t }`. Should we allow
    `s.circle <- ...`? Our answer, in essence: if a type is concrete, then allow
    the update. After all, a concrete type cannot be maintaining any invariants
    (or if it is, the programmer is responsible). So, if we have the definition
    of `#t`, then `s.circle <- ...` is allowed. However, if we do not have the
    definition of `#t`, then `s.circle <- ...` is disallowed.

    Yet this design -- based on the concreteness of a type -- is disappointing,
    as it bases the choice of allowing mutable updates on whether or not an
    interface exposes a representation. Instead, we would want to be able to
    write an abstract type yet which allows a potentially-tearing update.

    We thus introduce a new layout `*w` (the "w" stands for "writable"). (We
    will use `*w` in our notation here, but the surface syntax is different and
    defined below.) `*w` is like `*` (and `l1 *w l2 <= l1' * l2'` iff `l1 <=
    l1'` and `l2 <= l2'`), but a product built with `*w` is allowed in a mutable
    update, even if it is potentially non-atomic. The inferred layout of a most
    (see below) product types (like `#t`) will be built with `*w`, not `*`. Yet
    if a type is exported abstractly from a module, the module signature will
    have to specify the layout of that type; most users will write a layout
    using `*`, thus protecting values of that type from getting torn.

    If a user wants to write a layout with `*w`, they can do so like this:

    ```ocaml
    type t : bits32 * (bits32 * bits32) [@writable]
    ```

    The `[@writable]` attribute transforms all `*`s in a layout to be `*w`s.

    Despite most product types being inferred to have a `*w` layout, product
    types with existential variables are inferred to have a `*` layout, as
    tearing an existential could be type-unsafe. Here is an example:

    ```ocaml
    type ex_int = K : 'a * ('a -> int) -> ex_int
    ```

    The layout of `#ex_int` is `value * value` (the first components is `value`
    because there is no annotation on `'a` suggesting otherwise), with the
    non-writable `*`.

    Putting this all together, we define the following predicate to test for
    writeability:

    ```
    writable : concrete_layout -> bool
    writable(l1 * l2) = false
    writable(l1 + l2) = false
    writable(l1 *w l2) = writable(l1) && writable(l2)
    writable(_) = true
    ```

    The update of a mutable field is thus allowed if *either*:

    1. The layout of the field fits in a 64-bit word, *or*
    2. The layout of the field satisfies the `writable` predicate above.

    The first possibility -- for word-sized layouts -- is because a type whose
    values fit in a 64-bit word have no threat of getting torn, and thus
    can be updated safely. We choose *64-bit* words (not 32-bit ones) because
    32-bit OCaml does not support multicore, and so no possibility of tearing
    exists.

    The restriction described here is called the *tearing restriction*.

    The tearing restriction allows updates of our `mutable center` field,
    for two reasons: the type `#point` is concrete and thus has layout
    `bits32 *w bits32`, with `*w`, and a `#point` fits in a 64-bit word.

    A user who wishes to update a non-atomic mutable field (with their own plan
    for synchronization between threads) may do so with the `[@non_atomic]`
    attribute, like so:

    ```ocaml
    type point = { x : #int32; y : #int32 }
    module Circle : sig
      type t : bits32 * (bits32 * bits32)
    end = struct
      type t = #{ radius : #int32; mutable center : #point }
    end

    type t = { mutable circle : #Circle.t; color : Color.t }
    let f (t : t) = t.circle <- #{ ... } [@non_atomic]
    ```

    The design in this section upholds the following principle:

    * Even in the presence of data races, a programmer can never observe an
      intermediate state of a single assignment (unless they write `[@non_atomic]`).

    **Optional.** It may be convenient for users to get a warning when they
    declare a mutable field that falls afoul of the tearing restriction and
    thus requires `[@non_atomic]` on *every* update.

2. **Mutable field in an unboxed record**. Example:

    ```ocaml
    fun (pt : #mut_point) -> (pt.x <- pt.x + #1l); pt
    ```

    Mutation within an unboxed record would be  unusual, because unboxed values have no
    real notion of identity. That is, an update-in-place appears to be
    indistinguishable from a functional update (that is, `{pt with x = pt.x +
    #1l}`). By its unboxed nature, an unboxed value cannot be aliased, and so the
    mutation will not propagate.

    Here is an example:

    ```ocaml
    fun (pt : #mut_point) ->
      let boxed = box pt in
      pt.x <- pt.x + #1l;
      (unbox boxed).x = pt.x
    ```

    This function would always return `false`, because the point stored in the
    box is a *copy* of `pt`, not a reference to `pt`. In other words, unboxed
    values are passed by *value*, not by *reference*. Accordingly, every time an
    unboxed value is passed to or returned from a function, it is copied.

    Because of the potential for bugs around mutability within unboxed records,
    this proposal disallows it: in an unboxed record, any `mutable` keywords are
    ignored. Accordingly, the example above is rejected: `pt.x <- ...` is
    disallowed, because `pt` is an unboxed record.

    This section can be summarized easily:

    * Fields of unboxed records are never mutable.

3. **Fields within a mutable unboxed record within a boxed record.** Example:
    if we have `t : t` (where the type `t` is from mutation case 1, above), then
    `t.center.x` is a field within a mutable unboxed record within a boxed
    record.

    The case of such a field is interesting in that an unboxed record placed
    within a boxed record *does* have a notion of identity: it lives at a
    particular place in the heap, and the surrounding boxed record might indeed
    be aliased. Thus mutation in this nested case does make good sense, and we
    surely want to support it.

    We thus introduce a new syntax for updates of individual fields of a
    mutable unboxed record: We write e.g. `t.(.center.x) <- ...`. Note the
    parentheses: they denote that we're updating the `x` sub-field within the
    mutable `center`. The parentheses are required here, as is the leading
    `.`. (The leading `.` within the parentheses disambiguates the syntax with
    array access.)

    The rule is this: Consider the list of field accesses in a parenthesized
    field assignment. The first field must be mutable, and all fields in the
    list must be unboxed.

    By the first field being mutable, we know that the record directly before
    the open-parenthesis is boxed, because unboxed records do not have mutable
    fields.

    This allows `t.(.center.x) <- #3l` even though `x` is not
    declared as `mutable`. This does not break abstraction: the type of `center`
    must already be concrete in order to write this, and so a user can always
    write `t.center <- {t.center with x = #3l}` instead (modulo the
    tearing restriction). In effect, `t.(.center.x) <- #3l` can be seen as
    syntactic sugar for `t.center <- #{ t.center with x = #3l }`, but allowed
    even if `t.center` is too wide for the tearing restriction.

    Beyond just mutable fields in boxed records, this new syntax form extends to
    elements of mutable arrays. So if we have a `#pt array` named `arr`, users
    could write `arr.(.(3).x) <- #4l` to update one field of an element of the
    array.

### Nesting

According to the descriptions above, the `#` prefix on types is
*not* recursive. That is, the contents of a `#t` are the same as
the contents of a `t` -- it's just that the `#t` is laid out in
memory differently. Let's explore the consequences of this aspect
of the design by example:

Consider the definitions (let's assume a 64-bit machine)

```ocaml
type t0 = { x : int32; y : int16 }
type t1 = { x : #int32; y : #int16 }
type t2 = { t1 : t1; z : #int16 }
type t3 = { t1 : #t1; z : #int16 }
type t4 = #{ t1 : #t1; z : #int16 }
```

The points below suggest a calling convention around passing arguments
in registers. These comments are meant to apply both to passing arguments
and returning values, and are up to discussion with back-end folks.

1. A value of type `t0` will be a pointer to a block of memory with 3 words: a
header, a pointer to a boxed `int32` for `x`, and a word-sized immediate for
`y`. It is passed in one register (the pointer).

2. A value of type `#t0` will comprise two words of memory: a pointer for `x`,
and an immediate for `y`; it is passed in two registers to a function.

3. A value of type `t1` will be a pointer to a block of memory with 2 words:
a header and a word containing `x` and `y` packed together (precise layout
within the word to be determined later, and likely machine-dependent).
It is passed in one register (the pointer).

4. A value of type `#t1` will comprise one word of memory, containing
`x` and `y` packed together. When passed to a function, it will be
passed as *two* registers, one for each component.

5. A value of type `t2` does not exist: it would be pointer to a *mixed* block,
containing a pointer to a `t1` structure and a non-GC word for `z`.

   **Alternative:** (We do not plan to do this.) A value of type `t2` is a
pointer to a block of memory with three words: a header, a pointer to a `t1`
structure (as described above), and a tagged immediate word, 16 bits of which
store `z`. (In this design, a `z : #int64` would be rejected because there is no
room for the tag.)

6. A value of type `#t2` comprises two words: one is a pointer to a `t1`
structure, and one contains `z`. (The `z` word does *not* need to be tagged,
as the GC will know not to look at it.) It may *not* be stored in
memory; it is passed in two registers.

7. A value of type `t3` is a pointer to a block of memory with 2 words:
a header and one packed word containing 48 bits of `t1` and 16 bits of
`z`. It is passed in one register.

8. A value of type `#t3` comprises one packed word in memory, containing
all of `x`, `y`, and `z`, with no tag bit. It is passed in three registers,
one each for `x`, `y`, and `z`.

9. The type `t4` is utterly identical to the type `#t3`.

10. The type `#t4` does not exist; it is an error, as the name `#t4` is
unbound.

The key takeaway here is that if a programmer wants tight packing, they have to
specify the `#` in the definition of their types. Note that there is no way to
get from the definition of `t0` or `t2` to a tightly packed representation; use
`t1` or `t3` instead!

A separate takeaway here is that memory layout in our design is *associative*:
the memory layout does not depend on how the type definitions are
structured. This is in contrast to C `struct`s, where each sub-`struct` must be
appropriately aligned and padded. For example, the C translation of this example
is

```c
struct t3 {
  struct t1 { int32 x; int16 y; } t1;
  int16 z;
};
```

Yet this would take 2 words in memory, as `t1` would be padded in order to be
32-bit aligned.

### Further memory-layout concerns

We wish for unboxed records (and tuples, to which this discussion equally
applies) to be packed as tightly as possible when stored in memory. (This dense
packing does not apply when an unboxed record is stored in local variables, as
it may be more efficient to store components in separate registers.)

This section of the proposal describes a possible approach to tight memory
packing that supports reordering. This section is more hypothetical than others,
and we type theorists defer to back-end experts if they disagree here. The
section ends with a few user-facing design conclusions.

The key example is

```ocaml
type t5 = { t1 : #t1; a : int; z : #int16 }
```

Note that now, we have an `a` between the `#t1` and the `#int16`.

A bad idea would be to tightly pack the `int` against the `#t1`, like this (not
to scale):

    63       31        0 63       31       0
    xxxxxxxxxyyyyyaaaaaa aaaaaaaaaaaaaaazzzz

Note that the 64 bits of `a` are spread across *two* words. This would make
operations on `a` very expensive. Just say "no"!

Instead, we insist that `a` is word-aligned. We might thus imagine

    63       31        0 63       31       0 63        31        0
    xxxxxxxxxyyyyy000000 aaaaaaaaaaaaaaaaaaa zzzzz0000000000000000

where `0` denotes padding. That works, but it's wasteful. Instead, we do

    63       31        0 63       31       0
    xxxxxxxxxyyyyyzzzzzz aaaaaaaaaaaaaaaaaaa

Everything is aligned nicely, and there's no waste. The only drawback is that
we have *reordered*.

In general, we reserve the right to reorder components in an unboxed tuple or
record in order to reduce padding. With demand, we could imagine introducing
a way for programmers to request that we do not reorder. (An unboxed type that
does not participate in reordering would have a different layout from one that
does participate in reordering, meaning essentially a new layout former, such
as `&`, analogous to `*`.)

However, the reordering is *not* encoded in the layout system. Imagine now

```ocaml
type t6 = { t1 : #t1; z : #int16; a : int }
```

This `t6` is the same as `t5`, but with fields in a different order. We have

```ocaml
t5 : (bits32 * bits16) * immediate * bits16
t6 : (bits32 * bits16) * bits16 * immediate
```

Accordingly, a type variable that ranges over `t5` would not also be able to
range over `t6`: the two have *different* layouts. We could imagine a very
fancy layout equivalence relation that would detect the reordering here and
say that `t5`'s layout equals `t6`'s; we choose not to do this, for several
reasons:

* Encoding reordering in the layout system potentially constrains us in the
  future, in case we wish to do fewer reorderings.
* This significantly complicates layouts, for little perceived benefit.
* Different architectures may benefit from different reorderings; we do not want
  the type system to depend on the architecture.

Accordingly, the reordering of physical layout is mostly undetectable by users:
they just get a more compact running program. The way to detect the reordering
is by either inspecting memory manually (of course) or by sending unboxed
records through a foreign function. In order to support foreign functions, we
will add an interaction with `ocamlc` that produces a C header that offers
functions that extract and set the various fields of an unboxed tuple. Foreign
code could then use this header to be sure that it interacts with our reordering
algorithm correctly.

#### Backward compatibility

Existing OCaml programs may have foreign interfaces that rely on a certain
ordering of record fields. The reordering story need not disrupt this. To wit,
we promise that, as we work out the details of reordering, we *never* reorder
fields in records (or tuples) where all fields have layout `value`. (This
includes *all* types expressible before our change.)

Going further, and imagining possibly lifting the mixed-block restriction, we
can imagine the following rule:

* (Hypothetical) In any record or tuple type, its fields are ordered in memory
  with all `value`s first, in the order given in the declaration, followed by
  non-`value`s, in an implementation-defined order.

This rule actually contradicts the layout of `t5` above, which would put `a` (a
`value`, because `immediate`s are `value`s) first. However, we make no promises
about the hypothetical rule and include it here just as a possible way forward.

## Unboxed variants

Unboxed variants pose a particular challenge, because the boxed
representation is quite different than the unboxed representation.

Consider

```ocaml
type t = K1 of int * bool | K2 of #float * #int32 | K3 of #int32 | K4
```

An unboxed representation of this would have to essentially be like a C
union with a tag. This particular `#t` would use registers as follows for
argument passing:

```ocaml
bits2         (* tag information *)
immediate     (* for the int *)
immediate     (* for the bool *)
float64       (* for the #float *)
bits32        (* for both #int32s, shared across constructors *)
```

No matter which variant is chosen, this register format works. This is important
because the layout of an unboxed type must describe its register
requirements, regardless of what constructor is used. We thus use a new
layout former to describe unboxed variants, `+`. That is, the layout of `t` would
be `(immediate * immediate) + (float64 * bits32) + bits32 + void`.
By specifying only the
layouts of the variants' fields -- not the actual register format itself -- we leave
abstract the actual algorithm for determining the register format.
(See "Aside" below for discussion on this point.)

One challenge in working with unboxed variants is that it may be hard for the
programmer to predict the size of the unboxed variant. That is, a programmer
might have a variant `v` and then think that `#v` will be more performant than
`v`. However, `#v` might be wider than even the widest single variant of `v` and
thus actually be *less* efficient. (Of course, this is always true: we should
always test before assuming that e.g. unboxing improves performance.) As we
implement, we should keep in mind producing a mechanism where programmers can
discover the memory layout of an unboxed variant, so they can make an informed
decision as to its usage.

### Boxing

In contrast to the fixed register format above, a boxed variant needs only
enough memory to store the fields for one particular constructor. That's because
boxed variants get allocated specifically for one constructor -- there is no
requirement that all variants have the same size and layout.

Despite the challenges here, `box` can still work to convert an unboxed variant
to a boxed one: `box` simply understands the `+` layout form to mean
alternatives,
just as variants have always worked.

We thus extend the treatment of unboxed records to work analogously
for unboxed variants. That is, we treat a definition of a boxed variant

```ocaml
type t = K1 of ty1 | K2 of ty2 | ...
```

to really mean

```ocaml
type #t = #( K1 of ty1 | K2 of ty2 | ... )
type t = #t box
```

We then additionally add magic to `box` to make this change transparent to users:

* A boxed unboxed variant may be constructed and pattern-matched against as
if the box were not there. Following our example, `K1` and `K2` could construct
values of type `t`, and values of type `t` could be matched against constructors
`K1` and `K2`.

Depending on how an unboxed variant ends up in memory, it has one of three possible
representations:

* If an unboxed variant is `box`ed, then it has the same representation as
  today's variants: a block including a tag and the fields of
  the particular constructor (only).

* If an unboxed variant is part of a boxed product (i.e. record, tuple, or array),
  then it takes up exactly as much space as needed to store the tag and the
  widest constructor. Here is the example:

    ```ocaml
    type t = K1 of int * string * string | K2 of bool * bool | K3 of string
    type t2 = { f1 : float; f2 : #t; f3 : string }
    ```

    A value of type `t2` will be a pointer to a block arranged as follows:

    ```
    word 0: header
    word 1: f1, a pointer to a boxed float
    word 2: tag for f2
    word 3: K1's int (immediate) or K2's bool (immediate) or K3's string (pointer)
    word 4: K1's string (pointer) or K2's bool (immediate)
    word 5: K1's string (pointer)
    word 6: f3, a pointer to a string
    ```

    Note that the `#t` takes up 4 words, regardless of the choice of
    constructor. This fixed-width representation is needed in order to give `f3`
    a fixed offset from the beginning of the record, which makes accesses of
    `f3` more efficient. Imagining a variable-width encoding requires examining
    the tag of the variant in order to address later fields; this becomes
    untenable if a record has multiple inlined variants. (We can think of `#t`
    here more as an inlined variant than an unboxed one.)

* If an unboxed variant is part of an outer variant, we essentially inline the
  inner unboxed variant. The section on "Nesting", below, covers this case.

### Constructor names and disambiguation

Just as we have done for record fields, we consider the constructor for the
boxed variant to shadow the constructor for the unboxed one. That is, writing
`K1 blah` will construct a `t`. However, if we already know that the expected
type of `K1 blah` is `#t`, then we use type-directed disambiguation to discover
that the `K1` is meant to refer to the unboxed variant, not the boxed
one. Echoing the design for record fields, we can use a `#` prefix to
disambiguate manually: `#K1 blah` unambiguously creates a `#t`.

### Nesting

We can naturally nest unboxed variants, just like we did with unboxed records.
Just as before, the `#` mark is *not* recursive. Also just as before, we can
gain extra efficiency by cleverly packing one variant inside of another. Let's
explore an example:

```ocaml
type t = K1 of int * bool | K2 of #float * #int32 | K3 of #int32 | K4
  (* as above *)
type t2 = K2_1 of #int32 | K2_2 of #float * #float | K2_3 of #t
```

The `t` is unedited from above, but the `t2` definition contains a `#t`.

We imagine the following register format for `t2`:

```ocaml
bits3          (* for the combined tag *)
immediate      (* for K1's int *)
immediate      (* for K1's bool *)
float64        (* for K2's #float and K2_2's first #float *)
float64        (* for K2_2's second #float *)
bits32         (* for K2's, K3's, and K2_1's #int32s #)
```

A few observations to note about this format:

* There is no part of this arrangement that matches the arrangement for `t`;
indeed, it is not necessary to efficiently get from a `t2` to a `t`: when the
user asked to unbox `t` in the definition of `t2`, they gave up on this
possibility.
* If the `#t` component of `K2_3` is, say, passed to another function, its
  components will have to be copied into new registers, so that the function can
  extract the pieces it needs.
* If the `#t` component of `K2_3` is, say, matched against, the matcher can
  simply remember an offset when looking at the tag. In our case, we might
  imagine that tags `0` and `1` correspond to `K2_1` and `K2_2`, with tags `2`
  through `5` corresponding to the constructors of `t`. Then, a match on `t`
  would simply subtract 2 from the tag before comparing against `t`'s
  constructors.

Note that the description above around boxing applies to nested variants, too,
because the layout of `#t2` is

```ocaml
bits32 + (float64 * float64) + (immediate * immediate) +
  (float64 * bits32) + bits32 + void
  ```

. Each summand is interpreted as a variant by `box`, and thus the in-memory
representation does not need to take any extra space. Spelling this out, a
`#t2 box` would be either be the tagged immediate for `0` (representing
`K2_3 K4`) or a pointer to a block containing a header and the following fields

    tag  |  fields                         | represents
    ===================================================
      0  |  bits32                         |  K2_1 x
      1  |  float64 ; float64              |  K2_2 (x, y)
      2  |  immediate ; immediate          |  K2_3 (K1 (x, y))
      3  |  float64 ; bits32               |  K2_3 (K2 (x, y))
      4  |  bits32                         |  K2_3 (K3 x)

As in the discussions above about low-level details: it's all
subject to change. The key observation here is that unboxing a variant
effectively inlines it within another variant. Put another way, we want the
concrete use of memory and registers for a type to be unaffected by the choice
of how the type is structured (that is, ignore the associativity choices of `+`,
much like we have ignored the associativity choices of `*`).

### Aside: Why we have a fresh layout for unboxed variants

The section above describes a fresh layout constructor `+` for unboxed
variants. However, in memory, an unboxed variant is laid out just like
an unboxed tuple, so we could, in theory, use the same layout constructor
for both. Under this idea, the `t` example above would have layout
`immediate * immediate * float * int32 * int2`.

However, we do not adopt this design for (at least) these reasons:

1. Encoding the unboxed-variant layout algorithm in the type system
seems fragile in the face of possible future improvements/changes. Maybe
we can come up with a better way of packing certain fields in the future,
and it would thus be a shame if such a change broke layout checking.

2. An unboxed product type of that kind can reasonably be the
type of a mutable record field. However, the variant case
can't. That's because racing writes of an unboxed variant type
risk memory safety -- the fields from one constructor might
win the race, whilst the tag from the other constructor wins.

3. The most natural representation for including an
unboxed sum type within a boxed sum type is actually to add
aditional constructors to the boxed sum type -- dual to how
unboxing a product into a product adds additional fields. In
other words, we would store the tag of the nested unboxed sum
type as part of the tag of the enclosing boxed sum type.

That representation also gives an additional excuse for not
allowing mutable unboxed variant fields.

Note that the kind of representation described above is still
available using GADTs to make the tag fields explicit. For example:

```ocaml
type foo = { a : int;
             unused1 : unit;
             unused2 : unit; }

type bar = { b : float;
             c : string;
             unused : unit; }

type baz = { d : string;
             e : string;
             f : string; }

type ('a : value * value * value) tag =
  | Foo : #foo tag
  | Bar : #bar tag
  | Baz : #baz tag

type t =
   T : { g : float;
         tag : 'a tag
         data : 'a } -> t
```

## Polymorphism, abstraction and type inference

The design notes below here are about the details of making unboxed
types play nicely with the rest of the type system.

The key problem is that we must now somehow figure out the layouts
of all type variables, including those in type declarations and
in value descriptions. The **Availability** principle suggests that
we should infer layouts as far as is possible. Yet the **Backward
compatibility** principle tells us that we should prefer the
`value` layout over other layouts when there is a choice.

We thus operate with the following general design:

* The layout of a *rigid* variable is itself rigid: it must be supplied
at the type variable's binding site.

* The layout of a *flexible* (unification) variable is itself flexible:
it can be inferred from usage sites.
If we still do not know the layout of a variable when we have finished
processing a compilation unit, we default it to `value`.

(Why wait for the whole compilation unit? Because we might get critical
information in an mli file, seen only at the very end of an entire file.)

We retain principal *types*, but we don't have principal *layouts*:
for any expression, there's a best type for any given layout, but
there's no best layout (or at least, the best layout isn't always
compilable).

Programmers may also specify a layout, using the familiar `:` syntax
for type ascriptions. A layout may be given wherever a type appears.
For example, these declarations are accepted:

```ocaml
type ('a : value, 'b : immediate) t = Foo of 'a * 'b
val fn : ('a : immediate) . 'a ref -> 'a -> unit
val fn2 : 'a ref -> ('a : immediate) -> unit   (* NB: not on declaration of 'a *)
val fn3 : (unit : immediate) -> unit           (* NB: layout ascription on type, not variable *)
type ('a : bits32 * bits32) t2 = #{ x : #float; stuff : 'a }
val fn4 : 'a t2 -> 'a t2    (* layout of fn4 is inferred *)

type t5 : bits32   (* layout ascription on an abstract type *)

val fn5 : 'a -> 'a    (* NB: 'a is defaulted to have layout `value` *)
```

It may seem a little weird here to allow abstraction over types of
layout `bits32`. After all, there's only one such type (`#int32`).
However, this ability becomes valuable when combined with abstract
types: two different modules can expose two different abstract types
`M.id`, `N.id`, both representing opaque 32-bit identifiers. By
exposing them as abstract types of layout `#int32`, these modules can
advertise the fact that the types are represented in four bytes
(allowing clients of these modules to store them in a memory-efficient
way), yet still prevent clients from mixing up IDs from different
modules or doing arithmetic on IDs.

### Multiple aspects of a layout

A concrete layout actually encodes three different properties of a type: how
the type is represented in memory (how many bits the type needs
and what kind of register should hold the value), whether the GC will
be able to successfully inspect values of the type (is the type `gc_friendly`?),
and whether the GC can ignore values of the type (is the type `gc_ignorable`?).

Here are the possible representations:

```
value_rep
word_rep
void_rep
bits8_rep
bits16_rep
bits32_rep
bits64_rep
float_rep
rep1 * rep2
rep1 + rep2
```

We now define several meta-functions, defined only over *concrete* layouts:

```
rep : concrete_layout -> representation
rep(value) = value_rep
rep(immediate) = value_rep
rep(bits_n) = bits_n_rep
rep(word) = word_rep
rep(float64) = float_rep
rep(lay1 * lay2) = rep(lay1) * rep(lay2)
rep(lay1 + lay2) = rep(lay1) + rep(lay2)

gc_friendly : concrete_layout -> bool
gc_friendly(value) = true
gc_friendly(immediate) = true
gc_friendly(void) = true
gc_friendly(lay1 * lay2) = gc_friendly(lay1) && gc_friendly(lay2)
gc_friendly(lay1 + lay2) = gc_friendly(lay1) && gc_friendly(lay2)
gc_friendly(_) = false

gc_ignorable : concrete_layout -> bool
gc_ignorable(value) = false
gc_ignorable(lay1 * lay2) = gc_ignorable(lay1) && gc_ignorable(lay2)
gc_ignorable(lay1 + lay2) = gc_ignorable(lay1) && gc_ignorable(lay2)
gc_ignorable(_) = true
```

These meta-functions respect subtyping among concrete layouts, as stated in the
following properties:

* If `l1 <= l2`, then `rep(l1) = rep(l2)`.
* If `l1 <= l2`, then `gc_friendly(l2) ⇒ gc_friendly(l1)` (where `⇒` denotes
  logical implication).
* If `l1 <= l2`, then `gc_ignorable(l2) ⇒ gc_ignorable(l1)`.

In addition, these definitions allow us to derive the mixed-block restriction.
It would definitely be problematic to have a layout that
is both not gc-friendly and not gc-ignorable: the GC can't ignore it, but also
can't look at it. No primitive layouts have this undesirable property. But mixed
blocks do! And so it had better be the case that we never build them.

### Unification of two types with different layouts

Every concrete type has a most accurate (i.e. lowest) layout. I
think it is reasonable in this system to consider this as its
"true" layout. A statement of the form:

```
t : l
```

Does not say that type `t` has `l` as its true layout, merely
that `l` is an upper bound on the true layout of `t`. In
particular, a unification variable with kind `l`:

```
'a : l
```

stands for some type that we haven't worked out yet, whose true
layout has `l` as an upper bound.

That's why, if we have:

```
'a : la
'b : lb
'a = 'b
```

then, even though the true layout of `'a` must be equal to the
true layout of `'b`, it is not the case that `la` must be equal
to `lb`. What is true is that the true layout of `'a` and `'b` must
be a sublayout of both `la` and `lb`. That means it must be a
sublayout of the greatest lower bound of `la` and `lb` --
written `la ⊓ lb`. Thus a most general solution for these
constraints is to have:

```
'c : la ⊓ lb
```

and substitute `'c` for both `'a` and `'b`.

(Equivalently, we can substitute `'a` for `'b` and replace `'a`s
 layout with `lb ⊓ la`, which is how we actually implement
 it. But it is simpler to think of the layout of a
 variable as immutable, and think of variable-variable
 unification as using a third variable.)

`lb ⊓ la` is commutative and defined in the following cases (omitting symmetric
cases):
  * If `lb` is `any`, then `lb ⊓ la = la`.
  * If `lb` is `gc_friendly` and `la <= gc_friendly`, then `lb ⊓ la = la`.
  * If `lb` is `gc_ignorable` and `la <= gc_ignorable`, then `lb ⊓ la = la`.
  * If `lb` is `gc_friendly` and `la` is `gc_ignorable`, then `lb ⊓ la = void`.
  * Otherwise, we require `rep(lb) = rep(la)`. Assuming `rep(lb) = rep(la)`:
    * If `lb = la`, then `lb ⊓ la = la`.
    * If `lb = value`, then `lb ⊓ la = la`.

(Correctness condition: for all layouts `lb` and `la`, `lb ⊓ la <= lb`, `lb ⊓ la
<= la`, and for all layouts `lc` such that `lc <= lb` and `lc <= la`, `lc <= lb
⊓ la`.)

(Note that if `immediate <= word`, `gc_friendly ⊓ gc_ignorable` would not
exist and this algorithm would need to change. Furthermore, the definition would
become incomplete, because `value ⊓ word` *would* exist -- it would be
`immediate` -- and yet the definition above would not compute this.)

Because of the `rep(lb) = rep(la)` restriction, we must track this as a new form
of constraint, arising whenever we must compute `lb ⊓ la`.

Similarly, if we have:

```
'a : la
'a = t
```

We can solve that by substituting `t` for `'a`, but this generates
an additional constraint that:

```
t : la
```

This is equivalent to:

```
l <= la
```

where `l` is the best (lowest) layout we have available for `t`.

The key to handling these additional forms of constraints is the
observation that: When creating a unification variable for a type we
are inferring, there is no requirement that we also infer its true
layout. We are already inferring the type, and the type determines
what the true layout is anyway. Instead we only need to provide an
appropriate upper bound on that layout.

This means that we do not need unification variables that range
over indeterminate layouts. Instead
we can just assign fresh type unification variables the upper bound
`any`:

```
'a : any
```

Note that, given such an `'a`, we can solve any constraint of the
form

```
'a = t
```

by substituting `t` for `'a` because

```
t : any
```

holds for all `t`.

Similarly, we do not actually need unification variables that
range over concrete layouts
either. Instead we can use unification variables that range over
*representations*. When we know that a type needs to have a concrete layout
it is sufficient to create a type unification variable the
following upper bound:

```
'a : upper_bound('r)
```

where 'r is a representation variable and `upper_bound(r)` is a function that
maps a representation to the layout that is the upper bound of all the layouts
with that representation:

```
upper_bound(value_rep) = value
upper_bound(word_rep) = word
upper_bound(bits_n) = bits_n
upper_bound(float_rep) = float64
upper_bound(rep1 * rep2) = upper_bound(rep1) * upper_bound(rep2)
upper_bound(rep1 + rep2) = upper_bound(rep1) + upper_bound(rep2)
```

(Correctness property: for all concrete layouts `l`, `l <= upper_bound(rep(l))`.)

This restricts the type to have a concrete layout, whilst still allow
its representation to be discovered by inference.

With all this in place, we
can now trivially solve all those additional constraints.

```
rep(upper_bound(ra)) = rep(upper_bound(rb))
```

is solved by just:

```
ra = rb
```

and

```
l <= upper_bound(r)
```

is solved by:

```
l ≠ any /\ rep(l) = r
```

meaning that the only layout related non-trivial constraints in
the system are equality constraints on representation variables, which can
be handled by ordinary undirected unification.

### Examples

When we invent a new unification variable with an unknown layout,
how should we discover its layout? We must be careful, because some
unification variables will be required to have known, concrete layouts;
while others will be allowed to have `any`. Let's look at
some examples:

```ocaml
let f x = x
let rec g : unit -> 'a -> 'a = fun () -> g ()
```

In `f`, we guess the type for `x` to be some type `'a`. We also
must create a *representation unification variable* `'r`, where
`upper_bound('r)`
is the layout of `'a`. The codomain of the `upper_bound` meta-function
is only the concrete layouts, so we are guaranteed to infer a concrete
layout.

In `g`, we must also infer the layout for `'a`. However, here,
because there is no place whether the layout is restricted to be
concrete, we just say `'a : any`. If something were to constrain
the layout, we would end up unifying `'a` with a type with a more
restrictive layout.

Let's now return to `f` and `g` and see how this all plays out:

`f`:

1. We spot that `f` is a function of one variable. We thus say
that `f : 'a -> 'b` for fresh variables `'a` and `'b`. We assign
layout `upper_bound('ra)` to `'a` and `any` to `'b`,
for a fresh unification variable `'ra`. Note that, because
`'b` is the return type, we do not require a concrete layout for it.

2. We see that the return value is `x`. This forces us to unify
`'b` with `'a`. We now have this unification problem:

    ```
    ('b : any)  ~=~   ('a : upper_bound('ra))
    ```

As described above, we solve this by inventing
`'c : any ⊓ upper_bound('ra)` and filling both `'b` and
`'a` with `'c`. The layout here quickly simplifies to `upper_bound('ra)`.

3. At this point there is no more information about `f` we can
use. We see it has type `'a -> 'a`, where `'a : upper_bound('ra)`.
Because `'a`
is not free in the environment (or, equivalently, it has a level
higher than the surrounding context) we can generalize it. But
we still have a problem with `'ra` and `'ga`; our type system does *not*
support layout polymorphism, so we cannot generalize these variables. Instead,
we make them into weakly polymorphic variables. When we finish
processing the top-level definition containing `f` (which might just
be `f` itself), we will default `'ra := word` and `'ga := must_gc` if we have not already
unified them.

The final type for `f` is thus `('a : value). 'a -> 'a`, just as it
always has been.

`g`:

1. We have a type signature for `g`, but it mentions a unification
variable `'a`. We choose `'a : any`.

2. Processing the body of `g` doesn't yield anything interesting,
and so we assign `g` the type `('a : any). unit -> 'a -> 'a`,
which is strictly more general than the type `g` is assigned today.

### Layout checking

Thanks to how types are constructed in OCaml, we need no separate
well-formedness check. Suppose we're given the following type:

```ocaml
type ('a : immediate) t = ...
```

and the user attempts to write `string t`. The syntax `string t` is
converted to a type expression by first constructing a template with a
fresh unification variable `'b t`, and then unifying `'b` with
`string`. Here, since `'b` ranges over layout `immediate`, the
unification with `string` will fail.

# Extensions and other addenda

## Alternative idea: `#` as an operator

One alternative idea we considered was to make `#` a (partial)
operator on the type algebra, computing an unboxed version of
a type, given its declaration. This ran aground fairly quickly,
though.

One problem is that OCaml doesn't have the ability to extract
a type argument from a composite type (like to get the `int`
from `int list`). So viewing `#` as the inverse of `box` doesn't
quite work. (Richard thought for a little bit that
`type 'a unlist = 'b constraint 'a = 'b list` did this unwrapping,
but it doesn't: it just requires that the argument to `unlist` be
statically a `list` type; this is not powerful enough for our needs,
because it would reject `'a. ... # 'a ...`, which would be part
of this more general operator's purview.)

Another problem is that `#` would have to have an elaborate type:
it would need to be something like `# : Pi (t : value) -> t layout_of_unboxed_version_of`,
necessitating the definition of `layout_of_unboxed_version_of`.
This is a dependent type. Dependent types are cool and all, but
unboxed types seem like a poor motivation for them.

## Extension: Restrictions around unexpected copy operations

Today, OCaml users expect a line like `let x = y` to be quick, copying a word of
data and requiring ~1 instruction. However, if `y` is a wide unboxed record,
this assignment might take many individual copy operations and be unexpectedly
slow. The same problem exists when an unboxed record is passed as a function
argument. Perhaps worse, this kind of copy can happen invisibly, as part of
closure capture. Example:

```ocaml
type t = { a : int; b : int; c : int; d : int; e : int }

let f (t : #t) =
  let g n = frob (n+1) t in
  ...
```

Building the closure for `g` requires copying the five fields of `t`.

We could imagine a restriction in the language stopping such copies from
happening, or requiring extra syntax to signal them. However, we hope that there
will not be too many wide unboxed records, and copying a few fields really isn't
so slow. So, on balance, we do not think this is necessary.

## Extension: limited layout polymorphism.

We can imagine an extension where
the layout of `(p, q) r` could in general depend on
the layouts of `p` and `q`. For instance:

```ocaml
type ('a : value) id = 'a
```

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

## Extension: more layout inference

The current design describes that we require users to tell us the
layout of rigid variables but that we will infer the layout of
flexible variables. However, it is possible to do better, inferring
the layouts even of rigid variables. For example:

```ocaml
let f1 : ('a : float). 'a -> 'a = fun x -> x
let f2 : 'a. 'a -> 'a = fun y -> f1 y   (* 'a inferred to be a float *)

module type T = sig
  type ('a : float) t1
  type t2
  type t3 = t2 t1   (* this forces t2 : float *)
end
```

This can be done simply by using unification variables for the layouts
of even rigid type variables; if we don't learn anything interesting
about the layout within the top-level definition, we default to
`value`, as elsewhere.

## Extension: more equalities for `box`

It would be nice to add this rule to the `box` magic:

* If `ty : value`, then `ty box = ty`. There is no reason to box something
already boxed. Note that this equality depends only on the *layout* of
`ty`, not the `ty` itself. For a given layout of the argument, `box` is
parametric: it treats two types of the same layout identically.

This is hard, because at the time we see `ty box`, it might be the case
that we haven't yet figured out the layout for `ty`, meaning we'd have
to continue inference, perhaps figure out the layout, and then return
to `ty box` and perhaps simplify it to `ty`. This is possible (GHC
does it), but it would be breaking new ground for OCaml, and so should
be strongly motivated. So we hold off for now, noting that adding this
feature later will be backward compatible.

## Extension: disabling magic for `box`

The main proposal describes that a declaration like

```ocaml
type t = { x : int; y : float }
```

would really be shorthand for

```ocaml
type #t = #{ x : int; y : float }
type t = #t box
```

and that the expression e.g. `{ x = 5; y = 3.14 }` is shorthand
for `box #{ x = 5; y = 3.14 }`, along with similar shorthand for
record projection and variants.

However, we could have a flag, say `-strict-boxes`, that removes
the auto-boxing and auto-unboxing magic. This would make record-construction
syntax like `{ x = 5; y = 3.14 }` an error, requiring a manual call
to `box` to package up an unboxed record. The advantage to this
is that programmers who want to be aware of allocations get more
compiler support. A downside (other than the verbosity) is that `-strict-boxes`
would still not prevent all allocation: any feature that doesn't go
via `box` might still allocate, including closures.

## Extension: `immediate <= word`

Semantically, `immediate <= word` makes good sense. To see why,
we start by observing we can collapse the `value_rep` and `word_rep`
representations: after all, both `value`s and `word`s take up the
same space in memory. (Their relationship to the garbage collector
is different, but we track that separately, not in the representation.)
We can
see this in the following table:

```
              | representation | gc_friendly | gc_ignorable
    ----------+----------------+-------------+-------------
    immediate | word_rep       | true        | true
    value     | word_rep       | true        | false
    word      | word_rep       | false       | true
```

Just as `immediate <= value` works, `immediate <= word` works:
`immediate` is really the combination of `value` (with its
gc-friendliness) and `word` (with its gc-ignorableness).

However, we hold off on adding `immediate <= word` to our type
system for now, because it adds complication without anyone
asking for it.

Here are some complications introduced:

* It is very convenient to reduce computing
`la ⊓ lb` to a constraint `rep(la) = rep(lb)`. In order to support `value ⊓ word
= immediate`, we would thus drop the `value_rep` representation, setting
`rep(value) = word_rep`.

* The `upper_bound` function we use in type inference then becomes impossible to
define: neither `value` nor `word` is an upper bound for the other, and so
`upper_bound(word_rep)` has no definition.  We thus have to design a different
type inference algorithm than the one described above, likely tracking
representation, gc-friendliness, and gc-ignorability separately. This would not
be all that hard, but it's definitely more complicated than the algorithm
proposed here.

* We sometimes might infer `immediate` even when no function
nearby has anything of layout `immediate`:

    ```ocaml
    val f1 : ('a : value). 'a -> ...
    val f2 : ('a : word). 'a -> ...

    let f3 x = (f1 x, f2 x)
    ```

    Here, we need to infer that the `x` in `f3` has layout
    `immediate`, even though neither `f1` nor `f2` mentions
    `immediate`. This might be unexpected by users.

## Further discussion: mutability within unboxed records

This proposal forbids mutability within an unboxed record (unless that
record is itself contained within a boxed record). This decision is made to
reduce the possibility of confusion and bugs, not because mutability within
unboxed records is unsound. Here, we consider some reasons that push against the
decision above and include some examples illustrating the challenges.

Reasons we might want mutability within unboxed types:

* The designer of a type may choose to have some fields be mutable while
  other fields would not be. For example, our `#mut_point` type allows the
  location of the point to change, while the identity of the point must
  remain the same.

* OCaml programmers expect to be able to update their mutable fields, and it
  may cause confusion if a programmer cannot, say, update the `x` field of a
  `#mut_point`.

The reason we decided not to allow mutability within unboxed types is because of
examples like this:

```ocaml
type mut_point = { mutable x : #int32; mutable y : #int32; identity : int }
let () =
  let p : #mut_point = #{ x = #4l; y = #5l; identity = 0 } in
  let foo () = print_uint32 p.x in
  foo ();
  p.x <- #10l;
  foo ()
```

Because the closure for `foo` captures that unboxed record `p` (by copying `p`),
this code prints 4 twice, rather confusingly. Yet the current design does not
fully rule such a confusion out, as we see here:

```ocaml
type point = { x : int; y : int }
type t = { mutable pt : #point }

let () =
  let t : t = { pt = #{ x = 5; y = 6 } } in
  let pt = t.pt in
  let foo () = print_int pt.x in
  foo ();
  t.(.pt.x) <- 7;
  foo ()
```

This will print 5 twice, because the unboxed `t.pt` is copied into the local
variable `pt`. Yet the boxed version of this (with a `mutable x` and no hash
marks) prints 5 and then 7. This is a key motivation for putting the parentheses
in the `t.(.pt.x)` syntax: it forces the programmer to think about the fact that
they are mutating `pt.x`, not just `x`. Accordingly, they should expect `pt` not
to be changed (even if `pt` were an alias -- which it's not). Without the
parentheses there, a user might think they're mutating `x` and be surprised.

Here's yet another example:

```ocaml
type t2 = { x : #int32; y : #int32 }
type t1 = { mutable t2 : #t2; other : int }

fun (t1 : t1) ->
  let other_t1 : t1 = {t1 with other = 5} in
  t1.(.t2.x) <- #10l;
  other_t1.t2.x = t1.t2.x
```

Assuming `t1.t2.x` doesn't start as `#10l`, this will return `false`. But this
should not come as a surprise: `t1.t2` is copied as part of the functional
record update, and thus mutating `t1.t2` (as part of `t1.(.t2.x) <- #10l`) changes
it. If, on the other hand, we did not have the parentheses, users might expect
`other_t1`'s `x` to be changed when `t1`'s `x` gets changed.

## Bad idea: width subtyping

We wondered at one point whether we could support `int8 <= int64` in
the subtyping relation. After all, a function expecting an `int64`
argument can indeed deal with an `int8` one. The problem would be
collections. That is, if we have an `#int8 array` (for a magical
`array` type that can deal with multiple layouts, like `box`), how
would we index into it? Is it a `#int8`-as-64-bits or a plain `#int8`
in there? The way to tell is to have a second argument to `array`
that denotes the layout. Richard thinks this could work out in the
end, but probably would still be a bad idea.

## Bad idea: splitting `box`

The mixed-block restriction tells us that there are really two forms of `box`:
one for gc-friendly boxes and one for gc-ignorable boxes. We briefly thought
about having two different primitives `box_gc` and `box_non_gc` (and no `box`
function), where users would choose the one they wanted. The type system would
ensure that the layout of the argument is appropriate for the type of box
created.

This plan would be good because it would simplify inference: currently, every
use of `box` imposes a requirement that the layout of the argument is *either*
gc-friendly or gc-ignorable. But "or" requirements are annoying: we can't
simplify or usefully propagate the requirement until we know which branch of the
"or" we wish to take. If the user specified which kind of box they want, then
we don't have the "or" any more.

However, this plan doesn't work very well, because of auto-boxing. That is, if
we have something like

```ocaml
let x = ... in
let y = ... in
x, y
```

what kind of box should we use when constructing the tuple? It depends on the
type of `x` and `y` -- or rather on the layouts of those types. At the point
where our boxing magic inserts the call to `box`, we haven't yet completed type
inference, and so we can't know which kind of `box` to insert. In the end, we've
just re-created the challenge with "or" described above. So there seems to be
little incentive to have the two different boxes explicitly anywhere.
