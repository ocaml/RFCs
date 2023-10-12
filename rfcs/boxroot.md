# Movable OCaml roots for a more efficient and more flexible FFI

On behalf of the [ocaml-rust](https://gitlab.com/ocaml-rust/) team:
- Bruno Deferrari,
- Jacques-Henri Jourdan,
- Gabriel Scherer,
- myself.

## Summary

We propose a new API for registering roots with the OCaml GC, where the
user asks for a root that is allocated by the GC, instead of the
current API where the user decides the root address and asks the GC to
remember it. This complementary API has two strong benefits:

- It can be noticeably faster in many scenarios of heavy root usage,
  as the runtime can use efficient allocation techniques instead of
  having to track arbitrary addresses provided by the user.

- The root has ownership semantics: it can be passed, returned and
  stored in data structures, and its ownership can be transferred.
  (This can be implemented on top of the current API by allocating the
  root on the heap with malloc, an extra indirection with additional
  costs.)

We have a prototype implementation,
[ocaml-boxroot](https://gitlab.com/ocaml-rust/ocaml-boxroot/), that
demonstrates feasibility of the idea as an external library using the
GC scanning hooks. It brings significant performance benefits over the
current global root-registration APIs, in addition to the extra
expressiveness provided by movability.

The use-case we are currently interested in is to use it within OCaml
bindings to the Rust programming language -- this is part of a
collaborative work on the
[ocaml-interop](https://github.com/simplestaking/ocaml-interop/)
library. It seems feasible to use a boxroot-style API for all OCaml
roots in Rust code, which results in efficient and idiomatic Rust
code. In this approach, the Rust programmer would use a “caller-roots”
calling convention (see below), in contrast with the current
“callee-roots” convention used by the OCaml-C API. It allows us to
entirely dispense with local roots, which are not a good fit for Rust
idiom. Similar use-cases may apply for C++ bindings.

We also expect many plain old C-bindings to potentially benefit, if
they use global roots but don't need the extra control on the
location. As a matter of fact, a quick glance at opam packages shows
that a common use of global roots is to malloc a value-sized cell and
register it as a root, in order to store it inside a foreign data
structure.

Integrating this API into the OCaml runtime, rather than a separate
library, would make it easier for users to discover and adopt this API
if it fits their needs. It would also allow for tighter integration
with the garbage collector than the current scanning hooks allow. In
particular, currently the major-heap boxroots cannot be scanned
incrementally.


## Proposed API

When manipulating OCaml values from a foreign language, any newly
created value must be registered as a root; otherwise the OCaml
garbage collector would not know of this value, and possibly collect
OCaml values that are reachable from it.

The current root-registration API assumes that those "roots" are
pinned: they at a fixed place in memory, decided by the FFI
programmer, and never move.

```c
    /* [caml_register_generational_global_root] registers a global C
       variable as a memory root for the duration of the program, or until
       [caml_remove_generational_global_root] is called.

       If the program needs to change the value of this variable, it
       must do so by calling [caml_modify_generational_global_root].

       The [value *] pointer passed to [caml_register_generational_global_root]
       must contain a valid OCaml value before the call. */
    void caml_register_generational_global_root(value *r)

    /* [caml_remove_generational_global_root] removes a memory root
       registered on a global C variable with
       [caml_register_generational_global_root]. */
    void caml_remove_generational_global_root(value *r)

    /* [caml_modify_generational_global_root(r, newval)]
       modifies the value contained in [r], storing [newval] inside.
       In other words, the assignment [*r = newval] is performed,
       but in a way that is compatible with the optimized scanning of
       generational global roots. [r] must be a global memory root
       previously registered with [caml_register_generational_global_root]. */
    void caml_modify_generational_global_root(value *r, value newval)
```

We propose an alternative API where the FFI user provides a value to
initialize the root, but the runtime decides where to place it and
returns a pointer.

```c
    /* `boxroot_create(v)` allocates a new boxroot initialised to the
       value `v`. This value will be considered as a root by the OCaml GC
       as long as the boxroot lives or until it is modified. A return
       value of `NULL` indicates a failure of allocation of the backing
       store. */
    boxroot boxroot_create(value);

    /* `boxroot_get(r)` returns the contained value, subject to the usual
       discipline for non-rooted values. `boxroot_get_ref(r)` returns a
       pointer to a memory cell containing the value kept alive by `r`,
       that gets updated whenever its block is moved by the OCaml GC. The
       pointer becomes invalid after any call to `boxroot_delete(r)` or
       `boxroot_modify(&r,v)`. The argument must be non-null. */
    value boxroot_get(boxroot);
    value const * boxroot_get_ref(boxroot);

    /* `boxroot_delete(r)` desallocates the boxroot `r`. The value is no
       longer considered as a root by the OCaml GC. The argument must be
       non-null. */
    void boxroot_delete(boxroot);

    /* `boxroot_modify(&r,v)` changes the value kept alive by the boxroot
       `r` to `v`. It is equivalent to the following:

       boxroot_delete(r);
       r = boxroot_create(v);

       In particular, the root can be reallocated. However, unlike
       `boxroot_create`, `boxroot_modify` never fails, so `r` is
       guaranteed to be non-NULL afterwards. In addition, `boxroot_modify`
       is more efficient. Indeed, the reallocation, if needed, occurs at
       most once between two minor collections. */
    void boxroot_modify(boxroot *, value);
```

Would the OCaml maintainers be open to the idea of adding another
root-registration API in the runtime, along these lines? If so, we
would be happy to work on a Pull Request by adapting our prototype
implementation.

## Drawbacks

The drawbacks are those of adding something to the language. OCaml
would commit to supporting boxroots with normal maintainance and
backwards-compatibility costs. The prototype implementation is longer
than generational global roots, and this does not yet include making
it multi-threaded and low-latency.

## Alternatives

### Not doing anything

`boxroot` would remain supported as an external library. We will not
be able to make it incremental for intensive root users, and making it
efficiently multithreaded with multicore will be more complicated.
Note that this can still cause compatibility issues when OCaml
evolves, since we currently use hooks in ways that might not have been
intended. For instance, we use minor timing hooks to record when a
minor collection is taking place, and when this is the case we make
the assumption that we do not need to scan everything (generational
optimisation). We might make even more brittle assumptions later with
multicore if we want to efficiently make it multithreaded.

### Evolving the scanning hooks

It would be possible to keep maintaining `boxroot` as a library in the
long term if new scanning hooks were improved to support our use-case:

- Generational: let us distinguish minor from major scanning.
- Incremental: let us interrupt scanning when a requested amount of
  work has been done.
- Multicore: let us receive as argument the domain for which scanning
  must be performed in parallel, for which we hold the domain lock.
  (And please make the hook-registration API thread-safe.)

One drawback is that one likely beneficiary of `boxroot`, the OCaml
runtime itself (witness `caml_root` in multicore), will be unable to
use it.

## Implementation

A description of the prototype (a custom allocator similar to
Streamflow, arranged for efficient scanning) is available in the
[README](https://gitlab.com/ocaml-rust/ocaml-boxroot/#implementation).

## Discussion

### Caller-roots vs. callee-roots

In Rust, this API allows us to treat GC roots as resources, enabling
the idiomatic programming style where a function registers the roots,
and then pass those roots to its callees (passing ownership, or using
a borrow). This corresponds to a “caller-roots” convention, where
callees receive already-rooted values.

The current root-registration API (with the efficient local roots) is
designed for a “callee-roots” convention where functions receive
unrooted values, and must make sure to root them before calling the GC
using `CAMLparam`. One cannot easily use a caller-root style, because
values that we know are rooted cannot be passed around by value -- we
would need to add an indirection to pass pointers to rooted values,
which is incidentally what the
[CAMLroots](https://github.com/let-def/camlroot) proposal by Frédéric
Bour does in C, to ease debugging rather than for performance reasons.

`boxroot`s are pointers to values rooted by the GC. They are more
efficient than generational global roots, and seamlessly support a
caller-roots calling convention. In fact they are close enough to the
performance of local roots that one could use only boxroots in a FFI,
without having to deal with two kind of GC roots.

One API constraint is worth noting: it is not possible to assume that
the deletion of `boxroot`s necessarily happens while the OCaml runtime
lock is held (due to limitations of the Rust type system, and in the
end for practicality reasons too, when giving ownership of roots to
foreign libraries). This places constraints on the implementation,
which effectively has to handle the multi-threaded case even for
current OCaml. It is also a problem that one would encounter with the
`malloc` + global root approach, that will be solved with `boxroot`.

### A root-heavy benchmark

We implemented a prototype and various preliminary benchmarks, to
demonstrate potential performance gains.

In general the idea is that:

- Current global roots may be anywhere in memory, so the runtime has
  to use generic data structures (currently a skiplist); this makes
  insertion/deletion slower, and it also reduces the memory locality
  of scanning all roots.

- Boxroots are allocated in tightly-packed arrays, allocation/deletion
  can be faster, and scanning has better memory locality.

We implemented various benchmarks to test our prototype in the
[ocaml-boxroot](https://gitlab.com/ocaml-rust/ocaml-boxroot/)
repository. See there for the details.

Our benchmark `perm_count` takes a toy program that allocates a lot,
and replaces one large source of allocation by external root
allocations done in C code -- the code creates an abstract block to a
value registered as a root). This means the original version that uses
the GC act as a baseline, and abstract-block + root versions add
overhead, more or less depending on the root-registration
implementation.

Running this benchmark with various implementations, we get:

- 3.51s for `ocaml` (pure OCaml implementation)
- 3.53s for `gc` (C implementation using the OCaml GC)
- 3.46s for `boxroot` (abstract blocks containing a boxroot)
- 8.28s for `generational` (abstract blocks containing a generational global root)
- 43.94s for `global` (abstract blocks containing a non-generational global root)

We see that either of the current root-registration APIs add a large
overhead, while boxroots add none. (Boxroots is currently faster in
some micro-benchmarks than alternatives that do not use roots, which
we attribute to greater cache locality during scanning.)


### Multicore

The multicore developers are currently working on adapting the current
root-registration APIs to a multi-domain runtime. The issue is
discussed in
[ocaml-multicore#424](https://github.com/ocaml-multicore/ocaml-multicore/issues/424).
They introduced an intermediary abstraction called `caml_root`, whose
API seems very close to our `boxroot` proposal, on top of which they
implemented the current APIs. `caml_root` is in the process of being
removed, in favour of global roots. There is also the question of
making global roots efficiently multi-threaded, and whether this
should include support cross-thread unregistration (due to the
malloc + global root idiom being already used in the wild).

Our belief is that the `boxroot` API is actually easier to implement
efficiently in a parallel/concurrent setting than (generational)
global roots, since it is based on a well-studied allocation technique
originally meant for multi-threaded allocators. So adding `boxroot`
may not add a lot of work, and provide stronger performance benefits
(relative to generational global roots) in a multicore setting, and
ultimately reduces time trying to make multi-threaded global roots
efficient. (But we have not worked on parallel/concurrent
implementations yet.)


### Future possibilities: caller-root externals

Our calling convention is based on the passing of references to rooted
values. `boxroot` makes it convenient to adopt such a caller-root
convention in foreign code manipulating OCaml values. It can be used
by transferring ownership of a root but also by borrowing roots. When
a root is borrowed, we get a polymorphic type `ValueRef`, which has a
representation equivalent to the C type `value const *`.

But the convenience does not extend to the actual binding with the
OCaml side: the OCaml `external` declaration itself assumes a
callee-root discipline, so foreign code has to start by rooting the
passed values before calling the caller-root version:

```ocaml
(* in OCaml *)
external foo : t -> int = "foreign_foo_wrapper"
```

```c
/* in foreign code */
value foreign_foo_wrapper(value v) {
  CAMLparam(v);
  CAMLreturn(foreign_foo(&v));
}

value foreign_foo(value const *v) {
  // in caller-roots style
  ...
}
```

It would be convenient to provide a variant of `external` that follows
the caller-root discipline (the runtime would pass a pointer to the
OCaml stack):

```ocaml
external foo : t -> int = "foreign_foo" [@@caller_roots]
```
