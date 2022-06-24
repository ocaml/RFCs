# Hidden includes

This RFC proposes to add to the compiler command line a new kind of includes
(`-I`) called hidden includes (`-Ihidden`). Currently `-I` has two impacts:
* User: All modules corresponding to `.cmi` files found in included directories
  are added in the initial scope.
* Typer: Type definition present in the `.cmi` files found in included
  directories can be used by the typer.

So if `Foo` defines `val v: Bar.t` and `Bar` defines `type t = int`. The user
can type `Foo.v + 1`. Currently if the user only depend on the library `foo` the
build system also needs to include the library `bar` in the include directories.

The problem arise when the user uses directly the `Bar` module for example by
writting `(Foo.v + 1 : Bar.t)`. If the library `foo` stop depending on `bar`,
for example `val v: int` the compilation of the user code will fail because
`bar` is not anymore a transitive dependency. We would like the user code to be
forced to directly depend on `bar` if it uses `Bar`.

In order to do that we propose to add a variant of `-I` for example called
`-Ihidden` that is seen by the typer but not by the user, i.e. not added in the
initial scope. If `foo` is a direct dependency of the user code and `bar` only a
dependency of `foo` a build system can make the following call to the compiler:

```
ocamlopt -I <foo>/ -Ihidden <bar>/
```

If both `foo` and `bar` are dependencies of the user code it would be the usual:

```
ocamlopt -I <foo>/ -I <bar>/
```

## Proof of concept

There is currently no proof of concept. Yet the idea would be for `Load_path` to
consider `-I` and `-Ihidden` in the same way except for a new a function
`Load_path.get_not_hidden` that would only give directories from `-I`. It
would be used during the creation of the initial environment in `Typemod`:

```
  let units =
    List.map Env.persistent_structures_of_dir (Load_path.get_not_hidden ())
  in
```

## Extensibility

 * Private modules are not in the scope of this RFC. However it is not clear if
 removing the `.cmi` of the private modules, is supported by the typer. Perhaps
 another type of include `-Iprivate` could be used to give the location of the
 `.cmi` of the private modules. The typer should not use the type equalities,
 since it is the goal of a private module, but perhaps it could simplify the
 support of private modules.

 * It has been proposed to not use directories for including modules but
   directly the list of `.cmi` that should be made accessible. It could help to
   avoid creating specific directories for sorting `.cmi` files and be very
   precise on what the compilation depends. However such feature is orthogonal,
   could be generalized to any io made by the compiler and is not required for
   correct dependencies.

```
scope /dir/foo.cmi\000
hidden /dir/bar.cmi\000
```

### Summary

This RFC proposes to add a variant `-Ihidden` of `-I`, which don't modify the
initial scope of the environment but allows the typer to lookup type definition:

| option  | Initial scope  | Type Lookup  |
|:-:|:-:|:-:|
| -I  | X  | X  |
| -Ihidden  |   | X  |


## Related

- Dune issue with `(implicit_transitive_deps false)`: https://github.com/ocaml/dune/issues/2733
