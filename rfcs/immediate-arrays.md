# A custom type of 'immediate arrays' not scanned by the GC

In the last few days I have heard from several users who would like to
define potentially-large arrays of integers, but are concerned about
the (small) continued performance cost of having the GC scan those
arrays in the major heap.

My understanding is that this is a situation where:

- the actual performance cost is probably very small, neglectible
  except for some very rare workflows

- ... but the *perception of cost* is troubling our users and
  encouraging them to consider sub-par implementation choices
  (see the 'Alternatives' section below).

I propose to solve this situation by adding specific support in the
Array module for arrays of immediate values, at a *different type*
than the usual type `'a array`, using a restricted submodule in
a style reminiscent of Array.Float (but documented).


## Proposed interface

An `Array.MakeImmediate` functor with the following signature:

```ocaml
MakeImmediate :
functor
  (Elt : sig type t [@@immediate] end)
-> sig
  type elt = Elt.t
  type t (** arrays of immediate values *)
  
  val make : int -> elt -> t
  val length : t -> int

  val get : t -> int -> elt
  val unsafe_get : t -> int -> elt
  
  val set : t -> int -> elt -> unit
  val unsafe_set : t -> int -> elt -> unit  
end
```

Proposed implementation: a custom block.

The intention is to provide a minimal interface for a low-level
module, rather than trying (like Float.Array) of copying the whole
interface of Array. I would propose to restrict the scope of this
submodule with this intention, but accept proposals/requests for
extensions when they concern primitives that can be implemented
significantly more efficiently in C -- for example `fill`,
`blit` and `sub`.

#### Design choice: a functor on immediate types

This could be specialized to just an `IntArray` module, but I believe
that a functor over any immediate type is significantly more
convenient for end-users. In particular, this allows its usage on
types such as

```ocaml
type id = Id of int [@@unboxed]
```

which improve type-abstraction and safety.

Note: one could consider a design where we accept an `immediate64`
type as parameter, with an implementation that reverts to a standard
array type on 32-bit machines. `immediate64` is obscure and inconvient
to use from the stdlib, so I dropped the idea; we could provide an
additional `MakeImmediate64` version in the future if we really wanted
to.

#### Design note: asking for an initial element

We could skip the demand for an initial element and 0-initialize the
array. I decided against this for two reasons:

- zero-initialization is not substantially more efficient than our API,
  in particular because `memset` cannot be used (the OCaml
  representation of 0 is actually a full-word 1, which is not periodic
  at the byte level)
  
- in the future we could have immediate types for which the value 0 is
  invalid, if we allow to "shift" the initial tag of a constructor.


## Alternatives

### Bigarrays

One can use `(int, int_elt, c_layout) Bigarray.Array1.t` for this purpose.

Downsides:

- working with Bigarrays is a lot more code/ceremony (but this could be hidden
  by a dedicated library)

- bigarray access is slightly slower due to performing an extra read
  to get the layout

- it is harder to reason about the pacing of the GC, I would be
  slightly worried about allocating lots of small-to-medium immediate
  arrays with this implementation


### Bytes

One can use `Bytes.t` with the primitives `{get,set}_int64_ne` for access.

Downsides:

- getting an `Int64.t` is more cumbersome if we want to work with `int` values
  (again, a library layer can hide this)
  
- The stdlib does not expose `unsafe` accessors for 64-wide reads.
  (a shortcoming we could fix)
  
- Checked accesses are slower and generate more code due to the extra
  work to compute the "real" size of the string.


### Fake arrays with an abstract tag

Behold:

```ocaml
external unsafe_fill_bytes : Bytes.t -> offset:int -> len:int -> char -> unit = "caml_fill_bytes"

let create_int_array size : int array =
  (* We are trying to create an [int array] that is not scanned by the GC. *)
  let arr =
    (Obj.obj (Obj.new_block Obj.abstract_tag size) : int array)
  in
  (* At this point the array elements are uninitialized memory,
     so they may be unsound as integers. Fill them with odd bytes
     to make them valid immediates. *)
  unsafe_fill_bytes (Obj.magic arr : Bytes.t)
    ~offset:0
    ~len:(size * (Sys.word_size / 8))
    '\001';
  arr
```

Downsides:

- This uses a "fake array", something that we pretend to the runtime is an array
  but has a tag different from `0` or `Double_array_tag`. This currently works
  but I am unconvinced that this is a good idea.
  
- Using `Abstract_tag` means that comparison, serialization and hashing are unsupported.
  This may be fine for some specialized use-cases, but is likely to come back to bite
  users in practice.
  
- Exposing our array-of-immediate as a fake array severely restricts
  our implementation choices. For example it becomes impossible to
  switch to a `Custom_tag` block later, because their layout is
  incompatible with arrays -- they store non-element data in the first
  argument word.
  
Upside:

- Thanks to the use of a fake array, the whole Array module is available to users,
  including convenience functions (`for_all`, `map`, `iter2` and what not) and
  optimized runtime primitives (`blit`, `fill`, `concat` etc.)

In this RFC I made the choice to give up on the convenience of reusing
the `'a array` type and all its available functions and primitives,
for the benefits of comparison/hashing/serialization and of avoiding
introducing a non-standard representation in the runtime -- in
comparison, custom blocks are precisely designed for this purpose.


## Previous work

### IntArray in Domainslib

Domainslib is suffering from the sequential bottleneck of initializing
large arrays, which hurts scalability. They have promising experiments
with a specialized IntArray type that does not need initialization:

- https://github.com/ocaml-multicore/domainslib/pull/7
- https://github.com/ocaml-multicore/domainslib/pull/10

Our proposed API does not help with this problem as it does not allow
parallel initialization. We could consider exposing an unsafe creation
primitive

    val unsafe_create_uninitialized : int -> t
    
which does not initialize the array elements. This is unsafe, as
reading an uninitialized element returns a type-unsound value that
will crash the GC.

Or: we could change the runtime to use a pool of worker threads to
parallelize operations on large arrays, so that users don't have to do
this. I would actually be more in favor of that approach.


### `Obj.set_tag` to disable GC scanning in automated provers

Some OCaml 4 codebase would use `Obj.set_tag` on arrays after the fact
to disable GC scanning. This is explicitly mentioned in
https://github.com/ocaml/ocaml/pull/1725 which deprecated `set_tag`;
the alternative `Obj.with_tag` is less convincing (it requires
allocating the array twice; above I used `Obj.new_block` instead). In
that PR @stedolan explicit mentions that we could add builtin support
for a `private int array` type of fake arrays using Abstract_tag.

The use-case mentioned in that PR was in the Zipperposition prover:
https://github.com/sneeuwballen/zipperposition/blob/2e6e100d5334d8e1e742432310171605305dee0e/src/core/Ordering.ml#L103

There are also instances of this trick in the Cubicle model-checker
( https://github.com/cubicle-model-checker/cubicle/pull/5#issuecomment-1044548566 ),
and probably some other automated theorem provers.
  
Note: these users do not add Obj.set_tag just for fun, so they
certainly *did* measure a noticeable performance benefit to doing
so. (In the intro I said that the scanning by the GC probably does not
matter; it apparently does for automated theorem provers.)
