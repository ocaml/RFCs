# Cmt files - both as input and output

This RFC asks for two separate additions to the compiler front end. They are
technically independent, but mostly useful together. So I decided to bundle
them.

The first request is to type check compilation units and produce `.cmt` files:

```
$ ocamlc -c foo.ml -output-cmt
```

The second request is to take `.cmt` files as input to produce object files:

```
$ ocamlc -c foo.cmt
$ ocamlopt -c foo.cmt
```

How does this help? The motivation is mainly to simplify some rough edges of
build tools and improve compilation speed in some use cases. Some concrete
examples where this is useful:

## Fast "check" mode

Dune's check mode (`$ dune build @check`) attempts to verify the user's code
type checks in the fastest way possible. Currently, this is done by building all
`.cmi`, `.cmt`, `.cmti` targets. Targets such as `.cma`, `.cmx`, `.cmxa`, files
are omitted as an optimization. Obviously producing `.cmt` files requires
building bytecode, which slows things down somewhat.

## Duplicate warnings

When a project is built in both bytecode and native mode concurrently,
compilation warnings are displayed twice (once per mode). This is bad user
experience. Build systems such as dune need to add hacks to filter duplicate
errors to make things more bearable. A separate type checking phase would now be
responsible for these errors and such warnings would only appear once.

## Speed up compilation when type checking is a bottle neck

This likely doesn't occur very often, but there are cases where type checking is
slow. This proposal should improve compilation speed in such projects, because
type checking will be done only once per compilation unit.

## More flexible build rules for users

This one is kind of dune specific, but I'm sure it would help other build
systems as well. Since dune uses bytecode targets to produce `.cmt` files, users
cannot have full editor integration (since it requires `.cmt` files) and disable
bytecode builds. It could be argued that dune can be improved to produce `.cmt`
files via the native rules, but this a needless complication.
