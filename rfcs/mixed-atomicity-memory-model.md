# A mixed-atomicity memory model for OCaml

The current OCaml memory model is defined in 

> Bounding Data Races in Space and Time  
> Stephen Dolan, KC Sivaramakrishnan, Anil Madhavapeddy, 2018  
> https://kcsrk.info/papers/pldi18-memory.pdf

The model is described by an operational semantics (with non-deterministic memory operations) and an equivalent axiomatic semantics.

The model distinguishes atomic locations and non-atomic locations. Read/write operations on atomic locations are atomic, they induce a synchronization between a reader and the corresponding writer. Read/write operations on non-atomic locations are non-atomic, they induce no synchronization. This separation corresponds to the current OCaml programming model, where `'a Atomic.t` values are atomic locations and all other mutable data is non-atomic.

@polytypic expressed interest in adding "fenceless" operations to the `Atomic.t` type, and his Multicore-magic library exports `fenceless_{get,set}` operations that are just non-atomic reads and writes. There are many uses of `fenceless_get` in particular in the kcas library; a common pattern is to use a `fenceless_get` in read-compute-cas retry loops: we don't need an atomic read as the following compare-and-set will synchronize with past writers anyway, and reading a stale value does not endanger correctness but just provoke a retry.

One question that came up in discussions with @OlivierNicole, @fabbing, @fpottier and @clef-men is whether we know how to describe the resulting model, where both atomic and non-atomic operations may be performed on a given location.

The current RFC, prompted by a discussion with @clef-men, proposes a formal description of such a mixed-atomicity memory model.

(Warning: we are not memory-model experts, so we are just making stuff up.
cc @stedolan, @kayceesrk, @avsm, the authors of the initial model,
and @gmevel, @jhjourdan and @fpottier as the authors of the Cosmo separation logic)


## A detailed reminder on the current operational memory model

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

We propose a mixed-atomicity system where any location `l` supports both atomic and non-atomic accesses.

```
Machine  M := <S, P>
Store    S := l ↦ (F, H)
History  H := t ↦ x
Frontier F := l ↦ t
Program  P := i ↦ (F, e)
```

Before the store type was `(a ↦ H ⊎ A ↦ (F, x))`: non-atomic locations were not bound to a single value but to a fuzzier history `H`, and atomic locations were bound to a frontier in addition to a value, which expresses synchronization / information transfer. We now get the worst of both worlds, `(l ↦ (F, H))`: we carry a frontier `F` for synchronization, which is the frontier of all past atomic writers, and we carry a history `H` that provides non-deterministic reads after non-atomic writes.

### Minimal change

The Step, Read-NA and Write-NA rules are basically unchanged.
In particular, non-atomic accesses do not interact with the frontier `Fˡ` at the memory location in any way.

```
e ~<l:ϕ>→ e'    S(l); F —<l:ϕ>→ C'; F'
———————————————————————————————————————————————
<S, P || (F, e)> —→ <S[l ↦ C'], P || (F', e')>

F(l) ≤ t      t ∈ dom(H)
——————————————————————————————————————————— Read-NA
(Fˡ, H); F  ——<l:read-na H(t)>—→ (Fˡ, H); F

F(l) < t      t ∉ dom(H)
————————————————————————————————————————————————————————— Write-NA
(Fˡ, H); F  ——<l:write-na x>—→ (Fˡ, H[t ↦ x]); F[l ↦ t]
```

Atomic accesses are similar to the current version, in particular the thread frontier gets updated from `F` to `F' := Fˡ ⊔ F`, the maximum of the current frontier and the frontier stored at the location. Instead of a single value `x` in the store they have to deal with a full history `H`. The rules that determine the timestamps involved are the same as for the non-atomic accesses:

- On read, any timestamp more recent than `F'(l)` may be read. Intuitively (we will make this precise later), the rule becomes non-deterministic exactly when non-atomic writes have been used since the last synchronization.
- On write, a fresh timestamp is created that is more recent than `F'(l)`, but may be older than some unsynchronized non-atomic writes.

```
F' := Fˡ ⊔ F   F'(l) ≤ t    t ∈ dom(H)
——————————————————————————————————————————— Read-AT
(Fˡ, H); F ——<l:read-at H(t)>—→ (Fˡ, H); F'

F' = (Fˡ ⊔ F)   F'(l) < t  t ∉ dom(H)  F" := F'[l ↦ t]
——————————————————————————————————————————————————————— Write-AT
(Fˡ, H); F ——<l:write-at x>—→ (F", H[t ↦ x]); F"
```

### Factored rules

Now that the rules are very similar, it is possible to present a factorized version where we decompose our event labels `ϕ` into a pair `ϕ at?`, where the label `ϕ` indicates the read/write status and the value read or written, and `at?` is a boolean indicating whether the operation is atomic. Then the rules use a condition `if at? then ... else ...` to manipulate different objects in the atomic or non-atomic case.

```
e ~<l:ϕ at?>→ e'    S(l); F —<l:ϕ at?>→ C'; F'
———————————————————————————————————————————————
<S, P || (F, e)> —→ <S[l ↦ C'], P || (F', e')>

F' := if at? then Fˡ ⊔ F else F
F'(l) ≤ t    t ∈ dom(H)
———————————————————————————————————————————— Read
(Fˡ, H); F ——<l:read at? H(t)>—→ (Fˡ, H); F'

F' := if at? then Fˡ ⊔ F else F
F'(l) < t  t ∉ dom(H)  F" := F'[l ↦ t]
—————————————————————————————————————————————————–———————————————————— Write
(Fˡ, H); F ——<l:write at? x>—→ (if at? then F" else Fˡ, H[t ↦ x]); F"
```

In the `Read` rule, the only difference between the atomic and the non-atomic version is that the atomic version updates the thread frontier.

In the `Write` rule, the thread frontier used depends on the `at?` test, and the location frontier only gets updated in the atomic version.

One can check that these "factored rules" are equivalent to the "minimal change" presentation above. We are not sure which of the two versions people will find easier to understand.


## Implementation

We *assume* that the current compilation strategy is enough to realize this memory model, by simply compiling all accesses as we are currently doing.
(This is equivalent to assuming that unsafe-magicking an Atomic.t into a reference and performing normal `get` and `set` is a valid implementation of `{get,set}_relaxed`.)


## Reasoning about the model

### When are operations race-free? Relating to the previous model.

The memory-model paper calls a transition "weak" when it is not sequentially-consistent, and "strong" otherwise. Here we say that a store `S` is "strong" at a location `l`, written `Strong(l ↦ S(l)), when its frontier determines a unique readable timestamp.

```
Strong(l ↦ Fˡ,H) :=
  {t | Fˡ(l) ≤ t ∧ t ∈ dom(H)} = {Fˡ(l)}
```

This corresponds to the usual situation for atomic locations in the previous non-mixed model, which were bound to a single/unique value in the store.
Formally, given a partition of our locations `l` into atomic locations `A` and non-atomic locations `a`, there is a one-to-one correspondence (bijection) between:
- stores `Sₚ` in the previous, non-mixed model
- stores `S` in the current model such that all atomic locations `A` are strong in `S`
(the bijection sends `A ↦ (Fˡ, H)` in the current model to `A ↦ (Fˡ, H(Fˡ(l)))` in the previous model)

We can extend this notion to a full machine: a machine `<S, P>` is "strong" when all its store locations are strong, and furthermore all its processes see the most recent timestamp for each location.

```
Strong(<S, P>) :=
  ∀ (F, e) ∈ P,
    ∀ (l ↦ (Fˡ,H)) ∈ S,
      Strong(l ↦ (Fˡ,H))
      ∧ F(l) = Fˡ(l)
```

Claims:

- If a reference `l` is strong in the current store, an atomic access on `l` (preserves strongness of `l` and) has exactly the same behavior as in the previous, non-mixed model:
  if we execute them in input stores that are in the bijection, the output stores are still in the bijection

- For a given reference `l`, if we start from a strong initial configuration and only perform atomic operations on `l`, then `l` is strong in the current store.

These claims imply that programs that do *not* mix atomicities on the same location (in particular all current OCaml programs) have the same behavior in the previous model and in the new model. 

Bonus claim, a remark on single-threaded executions:

- If we start from a strong initial configuration and only ever perform reductions in one thread, the reduced machines are all strong.


### Only mixed reads

One subset of this model that is easy to reason about is the subset that has `Atomic.get_relaxed` / `Atomic.fenceless_get` (they mean the same thing and I'm not sure which name is best), but not `Atomic.set_relaxed` / `fenceless_set`. In this subset, there are no mixed-atomicity writes, only mixed-atomicity reads. A location is "write-atomic" if it only gets atomic writes, and "non-atomic" if it only gets non-atomic writes.

In this subset, non-atomic locations `a` behave exactly as in the previous model. write-atomic locations `A` have the nice property that the store is always strong for them, just like atomics in the non-mixed model – only non-atomic writes may make locations weak.

We believe that this subset could be formulated equivalently as follows:

```
if F(a) ≤ t, t ∈ dom(H)
———————————————————————————— Read-NA
H; F  ——<a:read H(t)>—→ H; F

if F(a) < t, t ∉ dom(H)
—————————————————————————————————————————— Write-NA
H; F  ——<a:write x>—→ H[t ↦ x]; F[a ↦ t]

———————————————————————————————————————————— Read-AT
(Fˡ, x); F ——<A:read-at x>—→ (Fˡ, x); Fˡ ⊔ F

———————————————————————————————————— Read-AT-NA
(Fˡ, x); F ——<A:read x>—→ (Fˡ, x); F

F' = (Fˡ ⊔ F)   F" := F'[l ↦ t]
————————————————————————————————————————— Write-AT
(Fˡ, y); F ——<A:write-at x>—→ (F", x); F"
```

In this presentation, (write-)atomic locations are back to storing a single value. There are two non-atomic read rules, one for non-atomic location (identical to the non-mixed model) and one for atomic locations, which reads the value without updating the current thread's frontier.

This sounds fairly simple, and we note that there are much more uses of `Atomic.fenceless_get` in the wild than uses of `Atomic.fenceless_set`, so maybe we should consider this seriously as a first step.

(Question for relaxed-atomic experts: what are typical use-cases for `fenceless_set`? The only use we found so far is https://github.com/ocaml-multicore/multicore-bench/blob/3ea8cafaf9f36d5c425683d63e646401c97330d3/lib/times.ml#L119 , which does not look essential.)


### Mixed writes

On a first non-atomic write to a location, the location becomes weak: there are two timestamps at which the location can be observed, except for the thread that just did the write which can only see the most recent timestamp -- its own frontier was updated by the write. Furthermore, this write communicated no information into the location frontier.

When performing a read on this location, *including* an atomic read, one may observe any of the values written after the last atomic write (included) into the location. Performing an atomic read synchronizes with all past atomic writers, but not with any non-atomic writers.

The only way to make the location strong again is to perform an atomic write on the location.

(Note: global barrier such as stop-the-world events are not described in the model, but it is an admissible rule to consider that they make all locations strong again: they update the frontier of all threads to the maximal / most recent frontier, and at this point the current frontier of the locations becomes irrelevant.)


## Remaining questions

1. Do we need `fenceless_set`?

2. Is there a nice axiomatic formulation of this model (the full mixed-atomicity, or the mixed-read-only version)?

3. Are people reassured by this description enough to consider mixed-atomicity operations in OCaml?

4. I have not tried to prove the key "Local DRF" theorem of the memory-model paper. The notion of L-stable and L-sequential transitions is unchanged, happens-before is trivial to adapt, but the notion of data race requires just a bit of thought: is it racy if we have a non-atomic read and an atomic write, or an atomic read and a non-atomic write? ("Yes to both" sounds like a good starting point.)

