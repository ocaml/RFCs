# Adding relaxed reads to the OCaml memory model

The current OCaml memory model is defined in 

> Bounding Data Races in Space and Time  
> Stephen Dolan, KC Sivaramakrishnan, Anil Madhavapeddy, 2018  
> https://kcsrk.info/papers/pldi18-memory.pdf

The model is described by an operational semantics (with non-deterministic memory operations) and an equivalent axiomatic semantics.

The model distinguishes atomic locations and non-atomic locations. Read/write operations on atomic locations are atomic, they induce a synchronization between a reader and the corresponding writer. Read/write operations on non-atomic locations are non-atomic, they induce no synchronization. This separation corresponds to the current OCaml programming model, where `'a Atomic.t` values are atomic locations and all other mutable data is non-atomic.

@polytypic expressed interest in adding "fenceless" operations to the `Atomic.t` type, and his Multicore-magic library exports `fenceless_{get,set}` operations that are just non-atomic reads and writes. There are many uses of `fenceless_get` in particular in the kcas library; a common pattern is to use a `fenceless_get` in read-compute-cas retry loops: we don't need an atomic read as the following compare-and-set will synchronize with past writers anyway, and reading a stale value does not endanger correctness but just provoke a retry.

One question that came up in discussions with @OlivierNicole, @fabbing, @fpottier and @clef-men is whether we know how to describe the resulting model, where both atomic and non-atomic operations may be performed on a given location.

In #41 I proposed a general mixed-atomicity memory model that supports both relaxed reads and writes on atomic locations, in essence erasing the distinction between atomic and non-atomic locations, based on discussions with @clef-men. @stedolan [explained](https://github.com/ocaml/RFCs/pull/41#issuecomment-2178274266) that relaxed writes are a dangerous addition to the model, as they break desirable properties and cannot be implemented with just the current OCaml compilation scheme.

The current RFC proposes a specification for a simpler extension where only relaxed *reads* are added, but not relaxed writes: only non-atomic accesses are used on non-atomic locations, and atomic locations allow either atomic access or a non-atomic read.

(Warning: we are not memory-model experts, so we are just making stuff up.
cc @stedolan, @kayceesrk, @avsm, the authors of the initial model,
and @gmevel, @jhjourdan and @fpottier as the authors of the Cosmo separation logic)

## A detailed reminder on the current operational memory model

(This section is unchanged from the previous RFC.)

In the current operational semantics for the memory-model, there are separate categories for atomic variables `A` and non-atomic variables `a`. The global store `S` stores a value `x` for each atomic location `A`, but for non-atomic locations `a` it stores a *history* `H`, a map from locations to values such that `H(t) = x` when a write `a := x` happened at timestamp `t`.

A given thread has observed some writes to each non-atomic location `a`, but not necessarily all writes. When the thread reads from `a`, it reads non-determinstically the last-write value that it has observed, or any value more recent than that (written at a later timestamp). We call a *frontier* `F` a mapping from locations to timestamps that represents a time-of-last-observed-write for each location. Each thread has a frontier.

Finally, the global store maps each atomic location `A` to a value `x`, but also to a fronter `Fᴬ`, that represents information propagated by threads that have written into this location. When a thread reads from a location `A`, it updates its own frontier `F` with the frontier of the location `Fᴬ` -- its frontier becomes the maximum of the timestamps, `F ⊔ Fᴬ`. When a thread *writes* to an atomic location, it updates its own frontier to become `F ⊔ Fᴬ`, but it also updates the atomic location's frontier in the same way.

In LaTeX-turned-Unicode, this gives:

```
location              l
atomic location       A
non-atomic location   a
value                 x
timestamp             t ∈ Q (rational number)

Machine     M := <S, P>
Store       S := a ↦ H ⊎ A ↦ (F, x)
History     H := t ↦ x
Frontier    F := a ↦ t
Program     P := i ↦ (F, e)
transition  ϕ := read x | write x

e ~<l:ϕ>→ e'    S(l); F —<l:ϕ>→ C'; F'
———————————————————————————————————————————————— Step
<S, P || (F, e)> —→ <S[l ↦ C'], P || (F', e')>
```

This first judgment above is a machine-reduction rule, it is the rule for memory operation. (There is another rule for silent reductions which I omitted, see the paper.) If a thread has a frontier `F` and its code is `e`, and `e` steps into `e'` by performing a memory operation `ϕ`, the global store is transformed by the same memory operation `ϕ`. The update of the global store captures the memory model.

It is described by the judgment `S(l); F -<l:ϕ>→ C'; F'`. This judgment takes the current data `S(l)` for the location `l` in the store, and the frontier `F` of the current thread, and returns new data `C'` to put in the store and an updated frontier `F'` for the thread.  `S(l)` and `C'` are elements in the store, that is either a history `H` for non-atomic locations `a` or a frontier-and-value pair `(F, x)` for atomic locations `A`.

```
if F(a) ≤ t, t ∈ dom(H)
———————————————————————————— Read-NA
H; F  ——<a:read H(t)>—→ H; F
```

Read-NA is the rule for a non-atomic read on `a`, which may non-deterministically read any value in the history `H` of `a` that is at least as recent as `F(a)`, the last write on `a` observed by the current thread.


```
if F(a) < t, t ∉ dom(H)
—————————————————————————————————————————— Write-NA
H; F  ——<a:write x>—→ H[t ↦ x]; F[a ↦ t]
```

Write-NA on a non-atomic location `a` creates a new timestamp that is fresh, and more recent than the time-of-last-observed-write `F(a)` – but there may still be more recent timestamps in the history. The frontier of the current thread is updated, this new timestamp `t` becomes the time of the last observed write.

```
————————————————————————————————————————— Read-AT
(Fᴬ, x); F ——<A:read x>—→ (Fᴬ, x); Fᴬ ⊔ F
```

A read on an atomic location `A` returns the value read from the location (and does not change the frontier `Fᴬ` stored in the store), but it updates the frontier of the current thread to become `Fᴬ ⊔ F`: for each location `a`, the new time-of-last-known-write is the maximum between the previous observed time `F(a)` and the time `Fᴬ(a)` known to past writers of `A`.

```
—————————————————————————————————————————————— Write-AT
(Fᴬ, y); F ——<A:write x>—→ (Fᴬ ⊔ F, x); Fᴬ ⊔ F
```

A write on an atomic location `A` updates the value in the store, *and* the frontier in the store, *and* the frontier of the current thread. This is where synchronization happens.

One higher-level way to think of this is that, for an atomic location `A`, the store tracks the maximum frontier of all threads that wrote to `A`. Reading from an atomic location transfers this synchronization information from all previous writers.

## New proposed version

We propose to keep all definitions and rules in the current model unchanged, in particular the rule for atomic reads which we recall for reference:

```
————————————————————————————————————————— Read-AT
(Fᴬ, x); F ——<A:read x>—→ (Fᴬ, x); Fᴬ ⊔ F
```


and to add a new rule for relaxed/non-atomic reads to atomic locations.

```
——————————————————————————————————————— Read-relaxed
(Fˡ, x); F ——<A:read-at x>—→ (Fˡ, x); F
```

The only difference with atomic reads is that the thread frontier is not updated. Such a relaxed read gets the "current" value of the store (as defined by the sequential ordering between atomic writes), but without any synchronization with the atomic writers.

## Implementation

We *assume* that the current compilation strategy is enough to realize this memory model, where relaxed reads are compiled exactly like non-atomic reads.

## Reasoning about the model

### Reasoning about programs

One easy way to reason about relaxed reads is when there is an atomic access on the same location right before or right after the relaxed read:

- If there is an atomic access after the relaxed read, the thread frontier gets updated at this point,
  and then the execution behaves as if the read had been atomic.

- If there is an atomic access before the relaxed read, then the thread frontier has already been updated,
  and some executions of the version where the read is atomic would have resulted in the same frontier
  after the read.

This is an example from `kcas` with a relaxed read after an atomic access:

https://github.com/ocaml-multicore/kcas/blob/240981e0ef9e9d5de6801aa8a15b62f73b7a37af/src/kcas/kcas.ml#L307-L312

This is an example from `saturn_benchmarks` with a relaxed read before an atomic access:

https://github.com/lyrm/saturn_benchmarks/blob/23755624cb35eabb8cd341008ebcb8582cc24d82/datastructures/two_stack_queue/two_stack_queue.ml#L83-L89

(reasoning about this requires checking that `push` always eventually performs an atomic access on the same location)

We also found plenty of examples in the wild where `fenceless_get` is used in more complex situations, for example cases where the atomic update only happens conditionally, based on the result of the relaxed read. Those will require more complex reasoning.

### Meta-theory

Our non-expert intuition is that there are no races between atomic writes and relaxed reads on the same location: either the read reads-from the write, and it comes after, or it reads-from an older write and it comes before.

In particular, unlike the richer proposal in #41, there is no new notion of race / conflict. If this intuition holds, it should be easier to establish that the "Local DRF" theorem of the memory-model paper is preserved.

## Remaining questions

1. Is there a nice axiomatic formulation of this model?

2. Are people reassured by this description enough to consider adding `Atomic.get_relaxed` to OCaml?

3. Do we indeed still get a "Local DRF" theorem in this setting?

