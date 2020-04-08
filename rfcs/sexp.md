# Add a `Sexp` module to the standard library

I propose to add a `Sexp` module to the standard library.
The goal is twofold:

1. Provide a common type definition that can be shared
   among libraries that deal with s-expressions.

2. Promote the use of s-expressions as a lightweight
   human-readable serialization format.

The module would include at least the following definitions:

```
type t =
  | Atom of string
  | List of t list

val equal: t -> t -> bool

val compare: t -> t -> int

val pp: Format.formatter -> t -> unit

val to_string: t -> string

val of_string: string -> t option
```

**Question:** The above functions will pretty-print their arguments.  Should
"compact" variants (minimum amount of whitespace) also be included?

## Support in other modules of the standard library

In addition, functions

- `to_sexp : t -> Sexp.t`
- `of_sexp_opt: Sexp.t -> t option`
- `of_sexp: Sexp.t -> t`

would be added to all existing modules of the stdlib for which it makes sense:

- `Unit`
- `Char`
- `Uchar`
- `Bool`
- `Int`
- `Int32`
- `Int64`
- `Float`
- `String`
- `Bytes`

Container types would have suitable functions added to them:

- `Lazy`, `Seq`

```
val to_sexp: ('a -> Sexp.t) -> 'a t -> Sexp.t (* forces the argument *)
val of_sexp_opt: (Sexp.t -> 'a option) -> Sexp.t -> 'a t option (* returns a strict collection *)
val of_sexp: (Sexp.t -> 'a) -> Sexp.t -> 'a t (* returns a lazy collection *)
```

- `Array`, `List`, `Option`

```
val to_sexp: ('a -> Sexp.t) -> 'a t -> Sexp.t
val of_sexp_opt: (Sexp.t -> 'a option) -> Sexp.t -> 'a t option
val of_sexp: (Sexp.t -> 'a) -> Sexp.t -> 'a t
```

- `Hashtbl`, `Result`

```
val to_sexp: ('a -> Sexp.t) -> ('b -> Sexp.t) -> ('a, 'b) t -> Sexp.t
val of_sexp_opt: (Sexp.t -> 'a option) -> (Sexp.t -> 'b option) -> Sexp.t -> ('a, 'b) t option
val of_sexp: (Sexp.t -> 'a) -> (Sexp.t -> 'b) -> Sexp.t -> ('a, 'b) t
```

**Question:** Should we include just one variant of `of_sexp` ?

## Support in functor modules (`Map.Make`, `Hashtbl.Make`)

Unfortunately, it is not possible to change the signature of the module
arguments of `Map.Make` and `Hashtbl.Make` to require `sexp`-conversion
functions. Thus no explicit support is proposed for these modules.

## Support for I/O on stdlib channels

The following two functions would be added to `Stdlib`:

```
val input_sexp: in_channel -> Sexp.t
val output_sexp: out_channel -> Sexp.t -> unit
```

## Compatibility module

A compatibility module can also be added to OPAM to work with older OCaml
versions.

Unfortunately, a package `sexp` already
[exists](https://github.com/janestreet/sexp), so an alternative name must be
found.
