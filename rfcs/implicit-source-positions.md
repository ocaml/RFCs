# Implicit source positions

A new type of arrow is introduced. Syntactically, it looks like:

```ocaml
val f : loc:[%call_pos] -> ...
```

in a signature, and

```ocaml
let f = fun ~(loc : [%call_pos]) ... ->
  ...
```

in an implementation.

The label name doesn’t matter, but there must be a label. The function parameter thus defined has type `Lexing.position` and, if not specified at a call site, is automatically filled with the position of the call site.

This can serve several purposes:

- When chaining monadic binds, the usual stack backtraces are very noisy and less useful. You can replace `(>>=) : 'a t -> ('a -> 'b t) -> 'b t` with `(>>=) : loc:[%call_pos] -> 'a t -> ('a -> 'b t) -> 'b t` to automatically track binding locations (and then implement a tracing facility).
- It can be used to improve the usefulness of testing frameworks (see e.g. https://github.com/mirage/alcotest/pull/366) or tracing systems (such as meio[^1]), or error reporting in embedded languages.

## History

A first PR[^2] has been proposed ten years ago by @let-def. After some design amendments, that PR was stalled for a while, until renewed interested was expressed in 2023 by @TheLortex. The feature was then discussed during a developer meeting and gathered consensus on the principle[^3]. It was subsequently implemented at Jane Street and has been in use there for several months.

The goal of this RFC is therefore to present the details of the design in use at Jane Street to make sure there is agreement to port the feature upstream. Jane Street has asked Tarides for help in the summarizing work that led to this RFC.

## Design

### Typing

Implicit position arguments behave sort of like optional arguments: if not provided, some default value (the call site’s position) is passed. But their semantics is neither that of optional arguments, nor of the usual labelled arguments. So they are a new kind of function parameter (or, theoretically speaking, a new class of function arrow types).

At callsites, you can specify that an argument is `[%call_pos]` to help type inference infer the right type for a function. Usually this is only relevant in higher-order functions:

```ocaml
let wrap_function b f g ~here =
  if b
  then f ~(here : [%call_pos]) ()
  else g ~(here : [%call_pos]) ()
```

### Controlling file paths in source positions

When implementing this feature, Jane Street also introduced a new compiler flag, `-directory`. When specified, `-directory some/path` causes the filename in automatically inserted positions to be prepended with `some/path`. Jane Street’s build system uses that flag to control the file paths reported by `[%call_pos]`. For example, this can be used to give more informative file paths than just a single `./filename.ml`, all the while ensuring a reproducible result and avoiding to include machine-specific paths like `/home/user/build/.../filename.ml` in the compiled program.

When discussing this flag with @MisterDA, he informed me that the compiler supports using the [`BUILD_PATH_PREFIX_MAP`](https://reproducible-builds.org/specs/build-path-prefix-map/) environment variable to achieve reproducible builds, and it looks like maybe the function achieved by `-directory` flag could be implemented with that, instead. But it’s not entirely clear how well `BUILD_PATH_PREFIX_MAP` is supported across the ecosystem, for example, whether Dune uses it or supports it. There also seems to be [some caveats](https://github.com/ocaml/ocaml/issues/8665) regarding absolute paths.



## References

- [PR on the Jane Street compiler](https://github.com/ocaml-flambda/flambda-backend/pull/2401)


[^1]: https://github.com/ocaml-multicore/meio
[^2]: https://github.com/ocaml/ocaml/pull/126
[^3]: https://github.com/ocaml/ocaml/pull/126#issuecomment-1582519924
