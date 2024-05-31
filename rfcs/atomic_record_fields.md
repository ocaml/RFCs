# Atomic record fields

## Context

The `Atomic` module of the standard library provides
(strongly sequential) atomic references: an `'a Atomic.t` has the same
in-memory representation as a `'a ref` reference, but the read and
write operations `Atomic.get` and `Atomic.set` are synchronized
concurrent accesses. Other useful operations are provided; the full
signature is as follows:

```ocaml
module Atomic : sig
  type !'a t

  val make : 'a -> 'a t
  val make_contended : 'a -> 'a t

  val get : 'a t -> 'a
  val set : 'a t -> 'a -> unit
  
  val exchange : 'a t -> 'a -> 'a
  val compare_and_set : 'a t -> 'a -> 'a -> bool
  val fetch_and_add : int t -> int -> int
  val incr : int t -> unit
  val decr : int t -> unit
end
```

One recognized limitation of this `Atomic` module is that it does not
provided atomic record fields. It is of course possible to have
a record field of type `int Atomic.t` (for example), but the memory
representation is "boxed", there is an extra indirection in
memory. OCaml provides both `mutable x : int` for a mutable field and
`x : int ref` for (boxed) field references, but there is no
corresponding "unboxed" option for `Atomic.t`, limiting the
performance of some applications.

For example, the
[kcas](https://github.com/ocaml-multicore/kcas/blob/240981e0ef9e9d5de6801aa8a15b62f73b7a37af/src/kcas/kcas.ml)
library of @polytypic contains several instances of an unsafe pattern
where a record is unsafely cast into the type `'a Atomic.t`, as a way
to perform atomic accesses on its first field -- this trick only works
for the first field. One instance:

```ocaml
  type _ t =
    | Unset : [> `Unset ] t
    | Elapsed : [> `Elapsed ] t
    | Call : (unit -> unit) -> [> `Call ] t
    | Set : { mutable state : [< `Elapsed | `Call ] t } -> [> `Set ] t

  external as_atomic : [< `Set ] t -> [< `Elapsed | `Call ] t Atomic.t
    = "%identity"
```

Users would want to have fields where some mutable fields are
atomic. This would allow a more efficient memory representation than
the current one, which can make a qualitative difference for low-level
concurrent code implementing important concurrent data structures.


## The easy part

From a distance, this sounds easy: in addition to `mutable`, just add
an `atomic` keyword in field declarations.

```ocaml
type t = {
  id : int;
  atomic mutable state : st;
}
```

Then have `foo.state` perform an atomic read (strong /
sequentially consistent), and `foo.state <- v` perform an atomic
write.

People might not be in favor of adding a new keyword, but we could
handle this with a builtin attribute:

```ocaml
type t = {
  id : int;
  mutable state : st [@atomic];
}
```


## The hard part

The difficulty is how to expose to the users the other atomic
operations : how to support atomic `exchange`, `compare_and_set`,
`fetch_and_add`, etc?

One approach would be to add new langage constructs for all of these:
`foo.state <-> new_state`, `cas foo.state old_state new_state`,
`foo.state += 2`, etc.

But this will not do:

- language designers are reluctant to add new keywords or builtin
  constructs; `atomic` will require a bit of convincing, `cas` is
  a no-go
  
- there is no limit to the amount of builtin constructs we would need
  to add in this situation, if we want to add new atomic operations
  (for example have `add_and_fetch` as well)
  
Even if we accept to make "atomic fields" a builtin construct, we want
to expose atomic operations on them as *library functions* /
primitives, not new language constructs.

Another idea would be to add a magic way to see the field `state` as
an `Atomic.t` value. But this is deeply incompatible with the OCaml
memory representation, which has no support for internal pointers:
except for the very first record field there is no way to expose
a pointer to a specific field as a valid atomic value, exception by
copying the value into a new atomic, which has a completely different
(and useless) semantics.

## The proposal: phantom-typed first-class atomic fields

In addition to the `atomic` keyword (or builtin attribute) on record
fields, we propose to add:

1. a new type `('r, 'a) Atomic.Field.t`, which denotes a field of type
  `'a` within a record of type `'r`. Its representation is just an
  integer, the offset of the field within the record structure.
  
2. Atomic operations on `('r, 'a) t` in this new Atomic.Field module,
   which mimick/duplicate the operations on `Atomic.t`. 
  
3. A single builtin-in construct to built `('r, 'a) Atomic.Field.t`
   values out of atomic record field names: `[%atomic.field foo]`
   
For example, if we have `type 'a t = { atomic state : 'a state }`,
then `[%atomic.field state]` is a polymorphic constant of type
`('a t, 'a state) Atomic.Field.t`.

The `Atomic.Field` submodule looks like this:

```ocaml
module Field = sig
  type ('r, 'a) t (* internally: a field offset *)
  val get : 'r -> ('r, 'a) t -> 'a
  val compare_and_set : 'r -> ('r, 'a) t -> 'a -> 'a -> bool
  val fetch_and_add : 'r -> ('r, int) t -> int -> int
  ... (* other atomic operations *)
end
```

The library functions would be implemented as primitives, and they can
be implemented efficiently with this interface.


## Advanced topics

### Type disambiguation

One remaining design question is: if `[%atomic.field foo]` refers to
the field name `foo` at a type in the context, how can we disambiguate
if there are different record types with such a field in scope? One
can use module prefixing `[%atomic.field M.foo]`, but even this may be
ambiguous.

Broadly our intuition is that this should be checkable like a field
access, using propagated type information. In particular, it should
always be possible to write explicitly:
`([%atomic.field M.foo] : (my_record, _) Atomic.Field.t`.

It may turn out necessary in practice to support a more concise syntax
such as `[%atomic.field (M.foo : my_record -> _)]` where the type
annotation must be of the form `<record type> -> <element type>`. But
we would suggest trying without it first.

### Bonus fetaure: atomic arrays

We can expose atomic arrays with the same mechanism: a builtin type
`'a Atomic.Array.t`, with a primitive function to build an index from
an integer:

```ocaml
module Atomic : sig
  ....
  type 'a array
  val index : int -> ('a t, 'a) Field.t
end
```

We can reuse the `Field.t` type and its atomic-operation API, but note
that the index here may be outside the bounds of the array, and will
have to be checked on access. (The index is not bound-checked by the
`index` function, as it may be called on different arrays later.) For
`Field.t` values at record type, no such bound checking is necessary.

Note: we could also simply duplicate the atomic-operation API in an
`Array` submodule instead of reusing the `Atomic.Field` module for
this purpose.

### No support for cache-line padding

The experiments of @polytypic suggest that for values shared between
domains with contention, false-sharing avoidance by cache-line padding
as provided by `Atomic.make_contended` is important to reliable
performance.

We propose to consider a first version of this proposition *without*
any support for padding of atomic fields or their containing
record. imagine supporting. For the time being, the best solution if
padding is necessary for performance is to use a non-atomic field with
a (padded) `Atomic.t` value, or unsafe solutions such as
`Multicore_magic.copy_as_padded : 'a -> 'a`.

It is possible to extend the syntax we propose to indicate that an
atomic field is "contended":

```ocaml
type t = {
  id : int;
  mutable state : st [@atomic contended];
}
```

but this implies an invasive change to the OCaml value representation,
that may be more difficult to implement than the rest of the proposal.

Supporting padding of the record value itself, rather than its fields,
could also useful in some cases where several atomic fields are always
accessed in lockstep by each domain. We propose to leave this out of
scope for this proposal. It could probably be seen as a special case
of a more general language feature, something like: allocation
attributes on block literals.



## Notes

This RFC was inspired by a discussion with @clef-men.
