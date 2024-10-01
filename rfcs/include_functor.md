# `include functor`

## Proposed change

The `include functor` extension eliminates a common source of boilerplate when
defining modules that include the results of functors.  It adds the module item
form `include functor F`, where `F` must be a functor whose parameter can be
"filled in" with the previous contents of the module.  For example, one can
write this:

```ocaml
module M = struct
  type t = ...
  [@@deriving compare, sexp]

  include functor Comparable.Make
end
```

Traditionally, this would have required defining an inner structure `T` just to
have something the functor can be applied to:

```ocaml
module M = struct
  module T = struct
    type t = ...
    [@@deriving compare, sexp]
  end

  include T
  include Comparable.Make(T)
end
```

These two code fragments behave identically, except that in the first case the
module `M` won't have a submodule `T`.

The feature can also be used in signatures:

```ocaml
module type F = functor (T : sig ... end) -> Comparable.S with type t = T.t

module type S = sig
  type t
  [@@deriving compare, sexp]

  include functor F
end
```

This behaves as if we had written:

```ocaml
module type S = sig
  type t
  [@@deriving compare, sexp]

  include Comparable.S with type t := t
end
```

Currently it's uncommon to define functor module types like `F` (there's no such
module type in
[`Base.Comparable`](https://ocaml.org/p/base/v0.16.3/doc/Base/Comparable/index.html),
for example).  However, you can get the module type of a functor directly with
`module type of`, so the previous signature could equivalently be written:

```ocaml
module type S = sig
  type t
  [@@deriving compare, sexp]

  include functor module type of Comparable.Make
end
```

## Use cases

Occurrences of this pattern are frequent enough across the ecosystem that
`include functor` seems like a useful addition. Below are some examples where
`include functor` could be used (found by doing a quick
[Sherlocode](https://sherlocode.com/?q=include%20%5C%5BA-Z%5C%5D%5C%28%5C%5BA-Za-z0-9_.%5C%5D%5C%29%5C*%20%5C%3F%28%5C%5BA-Z%5C%5D)
search):

- [bap](https://github.com/BinaryAnalysisPlatform/bap/blob/95e81738c440fbc928a627e4b5ab3cccfded66e2/lib/regular/regular_bytes.ml#L62)
- [binsec](https://github.com/binsec/binsec/blob/395b8a8322c48b7d664af05b601261c28765fe0e/src/dba/dba_types.ml#L43)
- [boomerang](https://github.com/SolomonAduolMaina/boomerang/blob/895d5f18b76afbb8275c63ec5d9e71ae7c56e39f/lenssynth/naive_gen.ml#L77)
- [bonsai](https://github.com/janestreet/bonsai/blob/1a229341721c3be02c03ef046ccadabe6a434df7/web_ui/element_size_hooks/src/bulk_size_tracker.ml#L157)
- [dune](https://github.com/ocaml/dune/blob/425743ccaa353a65a36fa86f2551faf3afa6fa3b/src/dune_rules/lib.ml#L476)
- [herdtools7](https://github.com/herd/herdtools7/blob/79e2c831d68410b93bc48528b6195f857e2a1322/jingle/AArch64Arch_jingle.ml#L24)
- [sexp_diff](https://github.com/janestreet/sexp_diff_kernel/blob/7b98745bb6f969568fd5fa8283691fe11755bd5d/src/algo.ml#L59)
- [tezos](https://gitlab.com/tezos/tezos/-/blob/2d6821c6e99368eab442539c4ca6c2245794b0b8/src/proto_alpha/lib_protocol/nonce_hash.ml#L43)
- [tyxml](https://github.com/ocsigen/tyxml/blob/7429ec12b2145cc984d68286e8092a22b151fc38/implem/tyxml_xml.ml#L111)

This feature has already been implemented in Jane Street's
[branch](https://github.com/ocaml-flambda/flambda-backend/) of the OCaml
compiler, and is in active use here. It has proven popular: it is now used more
than 30k times in our internal codebase, and we believe our publicly released
libraries (like `Base` and `Core`) would benefit from it.

## Details and Limitations

To include a functor `F`, it must have a module type of the form:

```ocaml
  F : S1 -> S2
```

or

```ocaml
  F : S1 -> () -> S2
```

where `S1` and `S2` are signatures.

In the current implementation, `include functor` cannot be used in the
signatures of recursive modules.
