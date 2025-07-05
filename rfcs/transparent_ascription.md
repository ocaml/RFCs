# Transparent ascription and static aliases

This RFC proposes to change the mechanism of aliasing in the module language by
introducing *static aliases* and *transparent signatures*. Its content is
organized as follows.

* Some context around the current state of the feature of *module aliases* is
  given in section 1.
* In section 2, we give an high-level view of our proposed change: *transparent
  signatures* and *static aliases*
* In section 3, we discuss the technical details
* In section 4, we briefly discuss a concurrent design: *transparent paths*.
* Finally, in section 5, we list unresolved questions.

# 1. Context

In the following, we use the following conventions:

* "*module expressions*", written `M`, for the module language
* "*signatures*", written `S`, for the types of modules, including `sig ... end`, `functor ()
  -> S`, `functor (Y:S) -> S` and named module types.
* "*declarations*", written `D`, for fields of signatures (inside `sig ... end`)
* "*binding*", written `B`, for fields of structures (inside `struct ... end`)

## 1.a. Aliasing

The module system currently offers a notion of [*module alias*](https://ocaml.org/manual/5.3/modulealias.html#start-section): bindings of the form:

```ocaml
module X = P
```

can be typed with the declaration:

```ocaml
module X = P
```

This feature is only available at the module-declaration level, but internally,
it is represented as a special signature case `Mty_alias`.

While historically introduced as a linking and namespace mechanism, module
aliasing turned out to have subtle semantics and interactions with other parts
of the language (applicativity, recursivity, signature constraints, etc).  The
feature was restricted to prevent issues (no functor application, no functor
parameters, no recursive modules), but those restrictions rely on syntactic
checks that are unstable through substitution and prevent some interesting code
patterns.

## 1.b. Presence

In addition, the module system infers a `presence` flag (`Mp_present`,
`Mp_absent`) on module declarations to indicate if the field is actually present
at runtime or not. The invariant is the following : absent fields *must be*
aliases, but not all aliases are absent. This flag is not accessible to the
user, but inferred by the typechecker, which tries to make aliases absent as
much as possible.

### Present aliases

Currently, most aliases are absent. User-written aliases will always be
absent. However, strengthening and avoidance can make aliases present :

1. Using strengthening:

   ```ocaml
   module M  = struct module X = struct end end (* sig module X : sig end end *)

   (* present alias inferred ! *)
   module N  = struct include M end (* sig module X = M.X end (present) *)

   (* absent alias inferred ! *)
   module N' = struct module X = M.X end (* sig module X = M.X end (absent) *)
   ```
2. Using avoidance:

   ```ocaml
   module X0 = struct end
   module N = struct
     open (struct module X1 = X0 end)
     module X = X1
   end (* sig module X = X0 end (present) *)
   ```

### No linking of absent aliases

The `-no-alias-deps` flag instructs the compiler to not link modules that are
referred as absent aliases, when the alias is not accessed in the rest of the
file. For instance, in the following (where both are absent):

```ocaml
(* top.ml compiled with -no-alias-deps *)
module L = LongModuleName1
module M = LongModuleName2
type t = M.t
```

Here, `LongModuleName1` will not get linked, as it is just aliased but not
accessed. By contrast, `LongModuleName2` will get linked, as it accessed (when
doing `M.t`).

## 1.c Transparent ascription / Module identity

### Transparent signatures

Present aliases can be seen as a special case of *transparent ascription in
signatures* (also called *transparent signatures*), a new feature of the
signature language where both the aliasing information `(= P)` and the
underlying signature `S` are stored in the type:

```ocaml
module X : (= P :> S)
```

We say that `X` has the *identity* `P` and the *associated signature*
`S`. Transparent signatures do not suffer from the same issue as aliases: they
can support functor parameters, functor applications or recursive modules, and
are stable through substitutions.

Aliasing can be seen as a special case of transparent signatures, where the path
is associated with its own signature. `module X = P` can be seen as

```ocaml
module X : (= P :> module type of P)
```

As an optimization (and a way to make signatures more concise), the associated
signature can be omitted, writing `(= P :> _)`, when its known to be exactly
`module type of P`.

However, it's worth noting that substituting a functor parameter by a path
during functor application might break this invariant and might require to
re-expose the signature:

```ocaml
module type S = sig type t end
module F(Y:S) = Y (* functor (Y:S1) -> (= Y :> _) *)
module X0 = struct type t = int type u = bool end (* ... *)
module X1 = F(X0) (* (= Y :> S) *)
```

Here, the signature of `X1` is not the same as `X0` and therefore, it must be
made explicit.


### Transparent ascription in module expressions

Transparent signatures are linked with the module expression construct of
*transparent ascription*, written `M :> S`. It is a form of ascription where all
type equalities (including module identities) are preserved, as in:

```ocaml
module type T = sig type t end

(* opaque *)
module Opaque = (struct type t = int end : T)
(* module Opaque : sig type t end *)

(* transparent *)
module Transparent = (struct type t = int end :> T)
(* module Transparent : sig type t = int end *)
```

With transparent ascription of paths `P :> S`, module identities are also
persevered :

```ocaml
module X = struct type t = int end

(* opaque *)
module X_opaque = (X : T)
(* module X_opaque : sig type t end *)
module X_transparent = (X :> T)
(* module X_transparent : (= X :> T) *)
```

It is already implemented in Coq/Rocq and SML (with the reverse syntax!),
although it does not have the same consequences on the type system (SML does not
have applicative functors, and Coq/Rocq does not use module identities).

Transparent ascription is technically not present in OCaml, but can be somewhat
simulated in two ways :

1. using a normal (opaque) ascription where all the type sharing has been made
   explicit in the signature (which can quickly become cumbersome):

   ```ocaml
   module X_transparent = (X : T with type t = int) (* module X : sig type t = int end *)
   ```

2. using a functor call, where avoidance is used to preserve type equalities:

   ```ocaml
   module X = (functor (Y:T) -> Y)(X) (* module X : sig type t = int end *)
   ```

   However, equalities between abstract types and module identities can be lost
   using this technique:

   ```ocaml
   module X' = (functor (Y:T) -> Y)(struct type t type u = t end);;
   (* module X' : sig type t type u end *)
   (* loss of type sharing ! *)
   ```

## 1.d Resources

List of some relevant discussions (PRs and issues):

* [Allow functor application and functor arguments as module aliases #10435](https://github.com/ocaml/ocaml/pull/10435)
* [Transparent ascription of modules #10612](https://github.com/ocaml/ocaml/pull/10612)
* [Use Mp_present for module aliases #10673](https://github.com/ocaml/ocaml/pull/10673)
* [Disallow aliases of functor arguments in module types #11460](https://github.com/ocaml/ocaml/pull/11460)
* [Remove positions from paths #1610](https://github.com/ocaml/ocaml/pull/1610)

List of relevant papers (chronological order):

* [Type-level module aliases: independent and equal](https://www.math.nagoya-u.ac.jp/~garrigue/papers/modalias.pdf) - description of the feature in OCaml
* [F-ing modfules](https://dl.acm.org/doi/abs/10.1145/1708016.1708028) - Section
  8.2 presents identity sharing of modules
* [Fulfilling OCaml Modules with
  Transparency](https://dl.acm.org/doi/abs/10.1145/3649818) - Section 2.4
  presents the soundness issue with aliases and functor calls, and present
  transparent signatures.
* [Avoiding Signature Avoidance in ML Modules with
  Zippers](https://dl.acm.org/doi/10.1145/3704902) - While the paper focuses on
  avoidance, the type system supports transparent signatures and applicative
  functors. Section 1.1.4 and 3.2 explain how transparent signatures can be used
  for delayed strengthening.
* [Retrofitting and strengthening the ML module system (Blaudeau's PhD
  thesis)](https://clement.blaudeau.net/assets/pdf/thesis.pdf) - see sections
  2.2.3 and 2.2.4 for a high-level introduction of module aliasing; see section
  4.2 for the key intuition behind module identities.

Manual:

* [Type-level module aliases](https://ocaml.org/manual/5.3/modulealias.html#start-section)

# 2. High-level summary of the change

As of today, the aliasing mechanism merges several features in an unsatisfactory
way. Splitting the features into user-controlled, well-behaved and
well-understood ones will improve usability and expressivity, and simplify the
typechecker. We propose the following changes:

1. **Feature split**:
   * Split aliases into *static aliases* (absent ones) and *dynamic aliases*
     (present ones) in the internal representation of the typechecker.
   * Provide attributes (`static_alias`, `dynamic_alias`) for each construction.
   * Do not change yet the "aliasable" criterion, i.e. keep the restriction that
     dynamic aliases cannot contain functor parameter, functor application, etc.
   * Remove the `presence` flag (which would coincide exactly with being a
     static alias).
   * Have dynamic aliases be a subtype of static aliases (a dynamic alias can be
     coerced into a static one, provided that the static alias was wellformed).
   * Maintain the current inference of presence (which alias is static and which
     is dynamic) and the behavior of `-no-alias-deps`.

2. **Feature introduction 1/2**:
   * Introduce *transparent signatures* by extending dynamic aliases with an
     optional signature, making dynamic aliases a special case of transparent
     signatures.
   * Introduce new syntax

3. **Feature activation**:
   * Allow transparent signatures of any path (functor parameter, functor
     application, etc.).

4. **Feature introduction 2/2**:
   * Introduce transparent ascription in module expressions, with the syntax `(M
     :> S)` (and `module X :> S = M`).

5. **Depreciation**:
   * Change the inference of ambiguous aliases to infer static aliases only when
     a persistent module is aliased in a covariant position. Otherwise, infer
     dynamic ones. In the former case, emit a warning stating that the special
     syntax (or attribute) should be used.
   * Restrict all static aliases to persistent modules paths

# 3. Technical details

## 3.a. No linking of absent aliases

The non-linking behavior of `-no-alias-deps` on absent aliases is critical to
the dune-style builds and should be preserved. As a consequence, the typechecker
has to infer *absent* aliases for some of the module bindings of the form
`module X = P`, at least on the top-level ones generated by dune.

## 3.b Backward compatibility and separate syntax

As the move towards separate syntax is made, backwards compatibility must be
taken into account. The proposed planning for the transition (starting in 1 and
ending in 4) is the following:

1. Introduce attributes `static_alias`, `dynamic_alias` that can be safely
   ignored by previous versions of the compiler:
   ```ocaml
   module X_static = Y [@@static_alias]
   module X_dynamic = Y [@@dynamic_alias]
   ```
   Attributes can be used both in bindings and declarations (in `.ml` and `.mli`)

2. Introduce new syntax (syntax options are discussed in Section 4):

   - `module X == P` for static aliases fields, both in bindings and declarations
   - `(= P :> S)` for transparent signatures
   - `M :> S` and `module X :> S = M` for transparent ascription expressions.
   - `(= P)` for dynamic aliases signatures
   - `P :> _` and `module X :> _ = P` for dynamic aliases expressions

3. For the ambiguous `module X = P` (without attribute), when

   - the path `P` is *persistent* (reachable from the top-level) [1]
   - in the strictly positive part (top-level and submodules, not inside functors)

   then, infer a static alias, but emit a warning stating that an explicit
   `static_alias` attribute should be added, or that the new syntax `==` should
   be used. In other cases, change the inference to infer a dynamic alias.

[1] "Persistent" modules cannot change through substitutions (functor
applications, signature constraints, etc.) and therefore do not risk being
scrapped/dropped. This restriction, which makes sense for a name-space
mechanism, should fix issues such as
[#13997](https://github.com/ocaml/ocaml/issues/13997). Non persistent modules
can still be aliased with transparent signatures.

## 3.c Properties of transparent signatures

In this subsection, we clarify some wrong intuitions regarding the semantic
model: what does it really implies for a module to be an alias of another.  We
consider a situation where the typechecker returns :

```ocaml
module X1 : (= P :> S1)
module X2 : (= P :> S2)
```

Here, `X1` and `X2` share the same identity, which means that they come from
coercions of the same definition `P`. The type fields that they have in common
(say `X1.t` and `X2.t`) will always be deemed equal by the typechecker. However,
this intuition does not extend to value fields. If `P` contains (applicative)
functor applications, `X1` and `X2` might not come from the same *instance*! If
the applicative functor is impure, as in:

```ocaml
module F (_:sig end) = struct let x = ref (Random.int 10) end
module X0 = struct end
module X1 = F(X0) (* module X1 : (= F(X0)) *)
module X2 = F(X0) (* module X1 : (= F(X0)) *)
```

Then, even though `X1` and `X2` have the same identity, `X1.x` and `X2.x` might
be different. We stress that this behavior is not a bug of transparent
signatures, but rather a consequence of the missing tracking of effects in
applicative functors. Currently, it's the user responsibility to use generative
functors for impure code. By contrast, in the following:

```ocaml
module X0 = struct let x = ref(Random.int 10) end
module X1 = X0
module X2 = X0
```

The modules `X1` and `X2` are obtained by a coercion of `X0`, and will therefore
refer to the same value (`X1.x = X2.x`).

Overall, the compiler does not use module identity for optimization: value
fields sharing for modules with the same identity may only come from normal code
optimization of definitions. Impure applicative functors are considered an
orthogonal issue.  However, sharing the same identity guarantees equality of
type fields.

## 3.d Avoidance

This RFC is *not* about solving signature avoidance issues, and therefore will
not significantly improve the computation of supertypes when trying to avoid
paths that appear in transparent signatures or in static aliases. For example,
such code might fail:

```ocaml
module M = (struct
  open (struct module X0 = struct end end)
  module X1 = X0
  module X2 = X0
end : sig
  module X1 : sig end
  module X2 : (= X1)
end)
```

Indeed, when computing the signature of the structure, the identity information
is lost, and `X1` and `X2` are no longer aliases of each other, making the
subtyping fail.

This should not be surprising, but addressed latter by a redesign of the
avoidance mechanism.

# 4. Drawbacks and alternatives

## 4.a Choices

We identify the following options:

* Terminology: *static/dynamic*, *absent/present*, *namespace/runtime*
* Syntax:
  * token for transparent ascription: "`:>`", "`>`", "`<:`", "`<`", "`:~`", "`::`"
  * for dynamic aliases: `(= P)`, `(= P :> _)`
  * for transparent signatures: `(= P :> S)`, `(P :> S)`, `(= P, S)`
* Inference of static aliases: the criterion for inference of absent aliases
  during the transition period can be adapted (restrict to top-level only,
  restrict to *unaccessed* aliases, etc.)
* End state for the ambiguous syntax. Instead of ultimately depreciating the
  inference of absent aliases, the proposed transition mechanism can be kept.


## 4.b Transparent paths

A variant of the design of transparent signatures `(= P :> S)` is to only have
dynamic aliases, but attach signatures to path, as in:

```ocaml
module X = (P < S)
module X' = (P < S).Y
module X'' = (F < SF).G(Z).Y
```

Doing so, the system would only need a construct for dynamic aliases, and
functor application on a path would just be a substitution by the transparent
path:

```ocaml
module type S = sig val x : int end
module F (Y:S) = struct module Z = Y end
module X = struct let y = 42 let x = y + 1 end
module Res = F(X)
(* -> *) module Res : sig module Z = (X :> S)
```

While this substitution semantics is appealing, we reject this design because:
* It would require signatures used in ascriptions to only be named module types
  (not any signature)
* As a consequence, inlining a module type might make paths invalid, as it is
  the case for first class modules, but it would happen more often.
* It might clutter paths that appear in core-language types.
* Changing the notion of paths is much more invasive.
* The meta-theory is more uncertain.


# 5. Unresolved questions

1. The identity sharing of functor applications, while underlying values are
   different (discussed in section 3.c) can be surprising. An option is to emit
   a warning when the applicative functor is not explicitly marked as pure (see
   [#13905](https://github.com/ocaml/ocaml/pull/13905))
2. The semantics of module signature constraints `with module X = ...` will have
   to be updated to support both kinds of aliases. The main use case (replacing
   a non-alias by an alias) should be persevered, and the different cases
   documented properly.
3. Transparent signatures can be used to delay strengthening: accessing a field
   inside `(= P :> S)` should return the field of `S` strengthened by `P` (with
   some care around abstract module types). There are different possible
   heuristics that might have different performance trade-offs: compute the
   strengthened signature and keep it on the side, memoïzed strengthened
   accesses, etc. Those would need to be investigated.
