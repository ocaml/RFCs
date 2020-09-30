# OCaml compiler support for library linking

This RFC proposes to move library linking support in the core OCaml
system. It is the result of a consensus between François Bobot, Daniel
Bünzli, Jérémie Dimino, Hugo Heuzard, Thomas Refis and Gerd Stolpmann.

With the benefit of [20 years][ocamlfind-init] of experience with
`ocamlfind` it seems we got to a point where we have a good idea of
what is needed to make OCaml library linking effective in the
compiler, namely, a notion of dependency between library archives.

Besides as more systems and development tools are entering the
enlarging OCaml eco-system, these need to be able to make sense of
OCaml library install bases in a simple and uniform manner. To cater
for this eco-systemic need, the proposal relies on a *library install
structure* so that the compiler and third-party tools can have
obvious, efficient and standard file system based lookup procedures
that are easy to understand by humans and easy to (re)implement by
tools.

[ocamlfind-init]: https://inbox.ocaml.org/caml-list/99032602421800.05437@schneemann/ 

## Benefits

The proposal allows to eventually have less concepts, names, tools and
moving parts in the eco-system. This makes the OCaml system more
obvious to understand both for end-users and third-party tools.

Moreover since the proposal is primarily file system and convention
based (vs *metadata* based) it is easy for third-party tools to reuse
its concepts, names and lookup procedures.

A few salient points:

1. The core OCaml system becomes usable as far as using libraries is
   concerned. `ocamlc` and `ocamlopt` are able to link libraries with
   dependencies. The `ocaml` toplevel (REPL) becomes able to load
   them. The `Dynlink` module becomes able to load a `cmxs` and its
   library dependencies without needing build system code generation 
   trickery and a third-party API.
2. Eventually, there are less tools, file formats and kinds of names
   to tame in order to work with OCaml libraries. The resulting
   simplicity benefits both end users and third-party tool
   implementations.
3. `opam` package names, `ocamlfind` package names, library archive
   names and directory install names get unified in the same
   namespace.
4. A convention is provided by upstream for third-party eco-system
   tools to interoperate -- dev. assistants, build systems, package
   systems, documentation systems, etc.
5. The "one library per directory" convention of the proposal
   provides more isolation among libraries and minimizes the chances
   of accidental dependency propagation via `-I` options at compile
   time. 
6. The "one library per directory" convention alongside the library
   naming convention makes it easy for end-users to find out which
   library name must be specified to use a given module.
   
## High-level technical description 

Sadly no deep magic is being introduced in the compiler. We simply do
for linking OCaml code what it has always done for linking C code.

This means we provide the ability to mention, at library archive
creation time, which OCaml library *names* are needed in order to use
that archive and store that in a new metadata field in the archive
objects (`.cma`, `.cmxa` and `cmxs` files) alongside those used for C
linking. 

When an archive gets linked these library names are resolved in an
`OCAMLPATH` library path (see below) and their archive added to the
link invocation -- recursively and sorted in stable dependency order.

Seasoned `ocamlfind` users will simply recognize this as being the
`requires` field of `META` files which we simply pushed into archives.
It is the essential bit of `ocamlfind` that enabled us to work with
libraries for the past 20 years.

Executables are also made to embed the library names that have been
fully linked into them when they are produced with the `-linkall`
flag. This allows the `Dynlink` API to avoid reloading the library
dependencies it already contains.

## Backward compatibility

The proposal is made so that none of the existing compilation workflow
gets changed. In other words using the new dependency support in
library archives is opt-in.

Besides `ocamlfind` is modified (see below) to be able to make sense
of the new mecanism so that library providers can gradually switch to
the proposal while letting users of these libraries using `ocamlfind` 
be undisturbed for the time being.
      
## Libraries and OCAMLPATH

We call a *library* a set of modules designed to be used together. 

Each library is installed in its own isolated *library directory*. A
library directory always defines a single library. It can contain
subdirectories which have other libraries but as far as this proposal
is concerned there's no connection between them except for the name
prefix they share.

The `OCAMLPATH` environment variable defines an ordered list of root
directories in which library directories can be found. The syntax of
`OCAMLPATH` follows the platform convention for `PATH` like
variables. This means they are colon `:` separated paths on POSIX
platforms and semi-colon `;` separated paths on Windows
platforms. Empty paths are allowed and discarded.

The *name* of a library is the relative path up to its root directory
in `OCAMLPATH` in which directory separators are substited by a `.` – for
example the relative directory `foo/bar` becomes `foo.bar`.

Each `.` separated segment (and thus directory name) must be a
non-empty, uncapitalized OCaml compilation unit name, except for the
`-` (U+002D) character which is also allowed. Even though this is not
checked by the compiler, having segment names in the same directory
that differ only by their `_` and `-` characters *must not* be done
(e.g. `dir.foo-bar` and `dir.foo_bar`).

If the same library name is available in more than one root directory
of `OCAMLPATH` the leftmost one takes over -- the usual `PATH`
convention.

For example for the following `OCAMLPATH` definition:

```
OCAMLPATH=/home/bactrian/opam/lib:/usr/lib/ocaml
```

We get the following map between library directories and library names:

    Library directory                          Library name
    ----------------------------------------------------------------
    /home/bactrian/opam/lib/ptime/clock/jsoo   ptime.clock.jsoo
    /home/bactrian/opam/lib/re/emacs           re.emacs
    /usr/lib/ocaml/ocamlgraph                  ocamlgraph
    /usr/lib/ocaml/ocaml/unix                  ocaml.unix
    /usr/lib/ocaml/re/emacs                    N/A (shadowed)

If you are implementing library object lookups in the `OCAMLPATH`, you
have to stop looking up after finding the first library directory. In
the example above assuming `re.emacs` has no bytecode version of the
library you *MUST NOT* look into `/usr/lib/ocaml/re/emacs` for the
bytecode version.
    
In what follows we use library names and the relative path it
represents interchangeably. When a library name is used to denote a
directory or file path we assume the `.` have been substituted with
the platform specific directory separator.

## Library directory structure

The directory of a library has all the compilation objects of its
modules and a single archive named `lib` with all the implementations
to be used for linking. In other words we have for a library named
`MYLIB` with modules `$(M)` the following files:

    MYLIB/$(M).{cmi,cmx,cmti,cmt}
    MYLIB/lib.{cma,cmxa,cmxs,a}

Shared objects for C stubs usually get installed in their own
dedicated directory `$(STUBLIBS)` (e.g. `$(opam var
lib)/stublibs`). For use the library name `MYLIB` and install the
corresponding DLLs as:

    $(STULIBS)/dllMYLIB.so
    
### Semantics and integrity constraints

The following constraints are assumed to hold on a library directory.

1. If a `m.cmx` has no `m.cmi` then the module is private to the library.
2. If a `m.cmi` has no `m.cmx` and no implementation in the archives then it
   is an `mli`-only module.
3. For any existing `m.cmx` the same `m.cmx` must be present in the `lib.cmxa` 
   archive.
4. If there is a `m.cmx` file there must be a `lib.cmxa`.
5. If there is a `lib.cmxa` there must be a `lib.a` unless `lib.cmxa`
   is empty (since 4.12).
6. FIXME discuss. All `lib.{cma,cmxa,cmxs}` files (as available) contain the 
   same compilation units.
7. FIXME discuss. All `lib.{cma,cmxa,cmxs}` files (as available) contain the 
   same dependency specifications.
8. Empty archives are allowed. They can 
   contain library dependency specifications which are used at link time.
9. A missing library archive means that the library is not available for the 
   given code backend, failures are reported on usage at link time. This 
   entails that a library made from `mli`-only modules must install empty 
   `lib.{cma,cmxa,cmxs}` files in its library directory so that the library
   can be used during the compilation and link phase without users needing 
   to know it has no implementation.

## Library dependency specifications

Library dependencies are needed to indicate that a particular library
`MYLIB` uses other ones for implementing its modules. These
dependencies also need to be provided at link time for producing an
executable that uses `MYLIB`.

We store library dependencies in library archives (`cma`, `cmxa` and
`cmxs` files) as *library names* to be looked up in `OCAMLPATH` at
(dyn)link time. They are specified at library archive creation time
via the repeatable `-require` flag.

It was decided to push library dependencies in the library archives
themselves rather than have yet another compiler managed side-car
file. Here are few points that motivated this decision:

* By doing so there is no metadata file format to agree on and
  no codec for it to be implemented in the compiler.
* It is one file less that can be missing or corrupted at the point
  where an archive has to be used.
* It makes library archives self-contained and allows bare 
  archives to declare their library dependencies (e.g. this is nice 
  for `cmxs` application plugins).
* If needed, it makes it easier to change or migrate the information
  structure internally. The format is allowed to change on each
  version of the compiler.
* Since it's not written as a separate file there's no fight about who 
  gets to write it during parallel `cma`, `cmxa` and `cmxs` archive creation.
* Systems that need to get or store the information separately can
  extract it with an `ocamlobjinfo` which provides a stable output
  accross versions of the OCaml compiler.
  
Executables that are produced with `-linkall` flag also get the name
of the libraries that were used embedded in them so that the `Dynlink`
API can properly load libraries with library dependencies matching
those of the executable without double linking.

## Command line interface commonalites 

### Support for `OCAMLPATH` default value 

A new configuration variable `DEFAULT_OCAMLPATH` is added to OCaml's
`configure` (defaults to empty). This value is used in case the 
`OCAMLPATH` environment variable is undefined.

For example with: 
```
./configure DEFAULT_OCAMLPATH=/usr/lib/ocaml:/usr/local/lib/ocaml
```
the default `OCAMLPATH` value, when the variable is undefined, will be: 
```
/usr/lib/ocaml:/usr/local/lib/ocaml
```
A new option `-ocamlpath` is added to `ocaml{c,opt}` which simply 
prints the contents of the `OCAMLPATH` environment variable or the 
default value if undefined.

This allows end user to extend the `OCAMLPATH` with for example:
```
export OCAMLPATH=~/.local/lib/ocaml:$(ocamlc -ocamlpath)
```
With this system, reading the value `OCAMLPATH` variable, if defined, always 
gives a total picture of what is going on.

FIXME Ask system packagers if that works for them.

### Support for `OCAMLPATH` extension

Every tool from the toolchain that interprets the `OCAMLPATH`
environment variable gets a new repeateable and and ordered `-L PATH`
option that allows to prepend `PATH` to `OCAMLPATH` from left to
right. For example with:

```
ocamlc -L dir1 -L dir2 ...
```

The effective `OCAMLPATH` for the invocation becomes `dir1:dir2:$(OCAMLPATH)`.

### Distinction between library names and file paths

The `-require ARG` option argument `ARG` allows, in certain cases, to specify
either a library name `LIB` or a file path `PATH` to an object. A
`PATH` is unconditionally detected by the presence of the directory
separator (`/` on Unix, `/` or `\` on Windows), otherwise the argument
is parsed as library name. To specify a file in the current directory 
you must thus use `./FILE`.

## `ocamlc` support 

The following behaviours are added to `ocamlc`.

### Archive creation

A repeatable and *ordered* `-require LIB` option with `LIB` a library name 
is added to archive creation mode. For example:
```
ocamlc -a -o ar.cma -require LIB
```
This adds the library name `LIB` to the library dependencies of 
archive `ar.cma`. The information is stored in a new field 
`lib_requires : string list` of the `cma` 
[library descriptor][cma-lib-descriptor]. 

`LIB` does not need to exist in the current `OCAMLPATH` when the command
is invoked.

[cma-lib-descriptor]: https://github.com/ocaml/ocaml/blob/e18564d8393475acd9e36f02fe3b0927fe8a0f5c/file_formats/cmo_format.mli#L53-L58

### Compilation phase

A repeatable and *ordered* `-require LIB` option with `LIB` a library 
name is added to compilation phase. For example:

```
ocamlc -c -I bla -require LIB src.ml
```
This resolves `LIB` in the `OCAMLPATH` and adds its library directory 
to the includes after `-I bla`. Effectively `-require LIB` is replaced
by `-I /path/to/LIB`.

Errors if `LIB` does not resolve to an existing library. Note that
type checking does not need to access library archives.

### Link phase 

A repeatable and *ordered* `-require LIB|PATH.cma` option is
added to the link phase. For example:

```
ocamlc -o a.out -require ARG src.ml
```

If `ARG` is: 

1. A library name `LIB`. The library is resolved in `OCAMLPATH` and its archive
   file `lib.cma` is added to the link sequence. The `lib_requires` of 
   `lib.cma` is read and these names are resolved in `OCAMLPATH` to their
   `lib.cma` archive, added to the link sequence and recursively in 
   stable dependency order. Errors if any library name resolution fails.
2. A direct path to an archive `PATH/ar.cma` file. The archive is added to 
   the link sequence. The `lib_requires` of `ar.cma` is read and 
   the dependencies resolved as is done in the previous point.
   
A repeateable and unordered `-assume-library LIB` option is added to
the link phase. `LIB` must be a library name and it does not need to
exist in the current `OCAMLPATH` when the command is invoked. Using
this flag requires library name `LIB` and assumes it was already
resolved. The user is in charge in providing its functionality on 
the cli as bare objects.

### Dynlink API support on `-linkall`

Support is provided to record which libraries are fully statically
linked in bytecode executables.

This happens whenever `-linkall` flag is specified. In this case the name
of any library resolved in `OCAMLPATH` via `-require LIB|PATH.cma` and
those specified via `-assume-library LIB` is embedded in the bytecode
executable in a new section called `LIBS`.

## `ocamlopt` support

The following behaviours are added to `ocamlopt`

### Static archive creation

A repeatable and *ordered* `-require LIB` option with `LIB` a library name 
is added to archive creation mode. For example:
```
ocamlopt -a -o ar.cmxa -require LIB
```
This adds the library name `LIB` to the library dependencies of 
archive `ar.cmxa`. The information is stored in a new field 
`lib_requires : string list` of the `cmxa` 
[library descriptor][cmxa-lib-descriptor]. 

`LIB` does not need to exist in the current `OCAMLPATH` when the command
is invoked.

[cmxa-lib-descriptor]: https://github.com/ocaml/ocaml/blob/8947a38b61c9de1f95bcd2f5ec8292987211bf4b/file_formats/cmx_format.mli#L53-L56

### Shared archive creation (plugins)

A repeatable and *ordered* `-require LIB|PATH.cmxa` option with `LIB` a library 
name is added to shared archive creation mode. For example:
```
ocamlopt -shared -a -o ar.cmxs -require LIB
```

If `ARG` is: 

1. A library name `LIB`, this adds the library name `LIB` to the library
   dependencies of the archive `ar.cmxs`. This information is stored in 
   a new field `dynu_requires : string list` of the `cmxs` 
   [plugin descriptor][cmxs-plugin-descriptor]. `LIB` does not need to 
   exist in the current `OCAMLPATH`. 
2. A direct path to an archive `PATH/ar.cmxa` file. The archive 
   is added to the link sequence *and* its `lib_requires` field is 
   added to to the `dynu_requires` field of the `cmxs` plugin 
   descriptor.

[cmxs-plugin-descriptor]: https://github.com/ocaml/ocaml/blob/8947a38b61c9de1f95bcd2f5ec8292987211bf4b/file_formats/cmxs_format.mli#L32-L35

### Compilation phase

A repeatable and *ordered* `-require LIB` option with `LIB` a library 
name is added to compilation phase. For example:

```
ocamlopt -c -I bla -require LIB src.ml
```

This resolves `LIB` in the `OCAMLPATH` and adds its library directory 
to the includes after `-I bla`. Effectively `-require LIB` is replaced
by `-I /path/to/LIB`.

Errors if `LIB` does not resolve to an existing library. Note that
type checking does not need to access library archives.

### Link phase 

A repeateable and *ordered* `-require LIB|PATH.cma` option is
added to the link phase. For example:

```
ocamlopt -o a.out -require ARG src.ml
```

If `ARG` is: 

1. A library name `LIB`. The library is resolved in `OCAMLPATH` and its archive
   file `lib.cmxa` is added to the link sequence. The `lib_requires` of 
   `lib.cmxa` is read and these names are resolved in `OCAMLPATH` to their
   `lib.cmxa` archive, added to the link sequence and recursively in 
   stable dependency order. Errors if any library name resolution fails.
2. A direct path to an archive `PATH/ar.cmxa` file. The archive is added to 
   the link sequence. The `lib_requires` of `ar.cmxa` is read and 
   the dependencies resolved as is done in the previous point.
   
A repeateable and unordered `-assume-library LIB` option is added to
the link phase. `LIB` must be a library name and it does not need to
exist in the current `OCAMLPATH` when the command is invoked. Using
this flag requires library name `LIB` and assumes it was already
resolved. The user is in charge in providing its functionality on the
cli as bare objects.

### Dynlink API support on `-linkall`

Support is provided to record which libraries are fully statically
linked in bytecode executables.

This happens whenever the `-linkall` flag is specified. In this case
the name of any library resolved in `OCAMLPATH` via `-require LIB|PATH.cma`
and those specified via `-assume-library LIB` is embedded in the native
code exectuable in a new `caml_imported_libs` symbol.
  
## `ocamlobjinfo` support

The following behaviours are added to `ocamlobjinfo`.

* The regular `ocamlobjinfo` output on `cma`, `cmxa` and `cmxs` (if
  BDF library is available) is extended according to the current
  format to output the new `lib_requires` and `dynu_requires` field.s
* The output on bytecode executable is extended to report the new 
  `LIBS` section in an "Imported Libraries" section.

## `ocamldep`, `ocamldebug`, `ocamlmktop` and `ocamlmklib` support 

The following behaviours are added to `ocamldep` and `ocamldebug`

* `(ocamldep |ocamldebug) -require LIB`. The repeatable and ordered
  `-require LIB` option resolves `LIB` in `OCAMLPATH` and adds its
  library directory to includes. Errors if `LIB` does not resolve to
  an existing library.

The following behaviours are added to `ocamlmktop`

* The repeatable and ordered `-require LIB` option resolves and links
  `LIB`'s dependencies as per `ocamlc` support. So goes for direct
  `cma` arguments.

The following behaviours are added to `ocamlmklib`

* The repeateable and ordered `-require LIB` argument is added 
  and propogated to the resulting `cma`, `cmxa` and `cmxs` files.

## `ocaml` and `ocamlnat` support

The following behaviours are added to `ocaml` and `ocamlnat`.

### `-require` option

The repeatable and ordered `-require LIB|PATH.cm(a|xs)` is added 
to `ocaml` and `ocamlnat`. For example:

```
ocaml -require ARG 
```

If `ARG` is: 

1. A library name `LIB`. The library is resolved in `OCAMLPATH`, this 
   adds its directory to includes and loads the library archive and its 
   dependencies according to `cma` linking (for `ocaml`, see `ocamlc` support) 
   or to `cmxa` linking (for `ocamlnat` see `ocamlopt` support). Note that 
   directories of dependent libraries are *not* added to the includes.
2. A direct path to an archive `PATH/ar.cma` file. `PATH` is added to the 
   include directories and `ar.cma` and its dependencies are loaded like
   in the preceeding point.

Note that in the above if any of the names resolved in the `OCAMLPATH` is 
already embedded in the `ocaml` or `ocamlnat` executable, the names are 
not resolved (see `ocamlc` and `ocamlopt` Dynlink API support).

### `#require` directive

A new `#require "ARG"` directive is added to the toplevel. If `ARG` is 

1. A library name `LIB`. The library is resolved in `OCAMLPATH`, its
   directory added to the includes, its library archive and its
   dependencies are loaded according to `cma` linking 
   (for `ocaml`, see `ocamlc` support) or to `cmxa` linking 
   (for `ocamlnat`, see `ocamlopt` support)). Note that directories 
   of dependent libraries are *not* added to the includes.
2. A direct path to an archive `PATH/ar.cma` file. `PATH` is added 
   to the include directories.
       

Note that in the above if any of the names resolved in the `OCAMLPATH` is 
already embedded in the `ocaml` or `ocamlnat` executable, the names are
not resolved (see `ocamlc` and `ocamlopt` Dynlink API support).

A nice side effect of the new `#require` is that if used with library
names, it doesn't mention file extensions, this allows to use it in
`.ocamlinit` and have it work both for `ocaml` and `ocamlnat`. We will 
also make sure to make `ocamlnat` use `Dynlink.adapt_filename` when file
paths are specified to `#require` so that a mention of `cma` get turns into
a mention of a `cmxs`.


## `Dynlink` library support

We add the following entry points:

```
Dynlink.require : ocamlpath:string list -> string -> unit 
Dynlink.assume_library : string -> unit
```
In `Dynlink.require ~ocamlpath arg` if `arg` is:

1. A library name `LIB`, then `LIB` is looked up in the `OCAMLPATH`, its
   library archive gets loaded and its recursive dependencies in the correct 
   order. 
2. An archive `PATH/ar.cm(a|xs)`, then the archive and its recursive 
   dependencies in the correct order.
   
In both cases if any resolution requested in the `OCAMLPATH` is a library
name that already exists in the executable (see `ocamlc` and `ocamlopt` Dynlink
API support), the requested name is not looked up and assumed to be already
loaded. 

A call to `Dynlink.assume_library n` add library name `n` to the set of 
loaded libraries names without loading anything.

Mirroring the existing support for compilation units, we also add the following 
functions to query the state of the program w.r.t. to loaded libraries.

```
val Dynlink.main_program_libraries : unit -> string list
(** [main_program_libraries ()] is the list of library names statically linked
    in the program. *)

val Dynlink.public_dynamically_loaded_libraries : unit -> string list
(** [public_dynamically_loaded_libraries ()] is the list of library names
    that were dynamically loaded via {!require}. *)

val Dynlink.all_libraries : unit -> string list
(** [all_libraries ()] is the union of {!main_program_libraries} and
    {!public_dynamically_loaded_libraries}. *)
    
val Dynlink.has_library : string -> bool
(** [Dynlink.has_library l] is [List.mem l (Dynlink.all_libraries ())]. *)
```

Keeping track of library names present in the executable and those
loaded by calls to `Dynlink.require` is a matter of extending the
existing Dynlink API's [state structure][dynlink-state] which
currently keeps track of these things at the compilation unit level.

[dynlink-state]: https://github.com/ocaml/ocaml/blob/03c33f500563f3e12355694f1add98e7bd1096ae/otherlibs/dynlink/dynlink_common.ml#L45-L63

## OCaml library install support 

In order to support the convention the OCaml library install has to be
shuffled a bit so that libraries are contained into proper
directories. Here's a proposal (can be bikesheded to wish though):

* `ocaml`, only has the stdlib (could be moved `ocaml/stdlib` if there's 
  the wish).
* `ocaml/bigarray`, the `bigarray` library (do we really still need it?)
* `ocaml/dynlink` has the `dynlink` library.
* `ocaml/compiler-libs/common`, has the `ocamlcommon` library
* `ocaml/compiler-libs/bytecomp`, has the `ocamlbytecomp` library
* `ocaml/compiler-libs/optcomp`, has the `ocamloptcomp` library 
* `ocaml/compiler-libs/toplevel`, has the `ocamltoplevel` library 
* `ocaml/raw_space_time_lib` has the `raw_spacetime_lib` library. 
* `ocaml/str` has the `str` library
* `ocaml/threads` has the `threads` library (unchanged).
* `ocaml/unix` has the `unix` library. 

## `ocamlfind` support 

The `ocamlfind` compatibility support works as follows. For a package
request `foo.bar` if the package name is described by a `META` file,
`ocamlfind` follows its instructions as usual.

If there is no such package but `foo/bar/lib.{cma,cmxa,cmxs}` (as
needed) exists in a directory `R` of the `OCAMLPATH`. Then it behaves
as if the following `R/foo/META` file exists:

```
package "bar" (
requires = ... # contents of `ocamlobjinfo's require of R/foo/bar/lib.cma` 
directory = "bar"
archive(byte) = "lib.cma"
archive(native) = "lib.cmxa"
plugin(byte) = "lib.cma"
plugin(native) = "lib.cmxs"
)
```

In general given an OCaml library name `foo.bar` it is assumed to
correspond to an `ocamlfind` package name `foo.bar`.

Since `ocamlfind` is in charge to put the recursive archive
dependencies on the compiler cli at the link phase and that there may
be a mix of archive using `META` files and others using the new
mecanism it must take care to resolve the dependencies of the latter
according to the `OCAMLPATH` semantics and specify them as bare archives
on the command line so that no double linking occurs. 

## `dune` support

Currently, Dune reads `META` files of installed libraries that were
not built by Dune and generates `META` files for all the libraries it
installs.

Once the compiler supports this proposal, Dune will start installing
library artifacts using the convention described in this document. It
will also pass the `-require` field when assembling library
archives. This will be a non-breaking change since from the point of
view of users the naming of artifacts is a implementation detail of
Dune. Dune will only enforce the new convention for the libraries it
installs but will not assume it. This way, users of Dune have nothing
to do and projects that do not use Dune do not have to be upgraded all
at once.

Starting from Dune 3, Dune will assume that all OCaml libraries follow
the convention described in this document and will stop reading `META`
files. This means that it will no longer be possible to consume legacy
libraries in Dune. On the plus side, it will simplify a bit the Dune
code base.

Starting from Dune 4, Dune will no longer generate `META` files.

## Examples

### Compiling and using libraries locally in a build

The following shows how to compile and layout two libraries 
to use locally in a build. We have:

* Library `a` made of source `a.ml` depending on the `ocaml.str` 
  library.
* Library `b` made of source `b.ml` depending on the library `a`.
* An executable `exec.ml` which depends on `b`

Here are our sources:

```
.
├── a.ml
├── b.ml
└── exec.ml
```
We create directories for the libraries. The `libs` directory 
will be added to the `OCAMLPATH`.

```
mkdir -p libs libs/a libs/b
```

We compile and layout library `a`:

```
ocamlopt -require ocaml.str -c -o libs/a/a.cmx a.ml
ocamlopt -require ocaml.str -a -o libs/a/lib.cmxa libs/a/a.cmx
```

We compile and layout library `b`. We extend the OCAMLPATH on the 
command line via the `-L` option.

```
ocamlopt -L libs -require a -c -o libs/b/b.cmx b.ml
ocamlopt -L libs -require a -a -o libs/b/lib.cmxa libs/b/b.cmx
```

We compile our executable:
```
ocamlopt -L libs -require b exec.ml
```

## Unresolved issues 

### Compilation phase includes

At the moment the proposal indicates that during the compilation phase
use of a library via `-lib LIB` simply adds `LIB`'s directory library
to the includes for the compilation. The include directories of the
libraries `LIB` depends on are not added. The reason is that doing the
latter leads to dependency underspecification (see
[here](https://github.com/ocaml/opam-repository/pull/13803) for a real
example).

The rationale is that if the library `L` does not make its usage of
another library `LD` abstract and that it leaks in `L`'s interface
then concretly the client of `L` also need to depend on `LD`.

The problem of this approach is that this can lead to errors that are
difficult to diagnose and that this notion seems to be currently ill
defined.

Here are two approaches that have been proposed to alleviate the problem: 

1. Make `cmi` files self-contained by expanding type aliases, module 
   aliases and module type aliases.
2. Add a `--lib-hidden LIB` option that tells the compiler to also 
   look into `LIB` for `cmis` but without directly exposing the 
   declarations to the source files that are being compiled.
   
It is perceived that 1. will make the `cmi` files too large. The
problem with 2. is that it breaks the idea that the compilation phase
need not be aware of the dependencies of a library which are currently
only stored in library archives.


## Implementation 

Work on the proposal has started thanks to a grant of the 
[OCaml software foundation](http://ocaml-sf.org/).

### OCaml status

Partial implementation for the compiler support is available
[here](https://github.com/ocaml/ocaml/compare/trunk...dbuenzli:ll)
for the previous iteration of this RFC.

For interested reviewers, the patches are meant to be read
individually and in sequence.

To create an opam switch with a compiler that has that support in you can use:
```
opam switch create ocaml-ll --empty 
opam pin add ocaml-variants.4.11.0+ll git+https://github.com/dbuenzli/ocaml#ll
```

A bit of testing can be found in [this repository](https://github.com/dbuenzli/ocaml-lltest/)

Here are a few notes about what is implemented and deviations of what
was proposed above.

1. `ocamlopt` and `ocamlc` support are done.
2. `ocamlmktop` support is done (the tool simply shells out to `ocamlc`).
3. `ocamldebug` support is done.
4. `ocamlmklib` support is done. Note that to compile a library say 
  `mylib` for the library proposal one should use `-o mylib/lib` 
  and `-oc mylib` or `-oc mylib_stubs` otherwise the static library and dll for 
  stubs will by called `liblib.a` and `dlllib.so` which will likely 
  lead to lookup problems.
5. `ocamlobjinfo` support is done only for outputing the `lib_requires` field 
   of `cm{a,xa,xs}` files alongside the rest of the information. The plan 
   to make `ocamlobjinfo` work on `cmxs` files without `bfd` was thwarted by 
   reality see discussion [here](https://github.com/ocaml/ocaml/issues/9306).
   For now the new `-lib-requires` flag mentioned above is not implemented
   it's a bit unclear whether there's an advantage to add a new format 
   for now.
6. `ocaml` support is done. There's quite a bit of churn around 
   load path initialisation. The support was done so that `-required`
   libraries do not add their library directory to lookup objects 
   on specified on the cli (i.e. `ocaml find.cmo -require lib` will 
   not try to find `find.cmo` in the library directory of `lib`). This 
   can be changed. In custom toplevels if a library is embedded in the 
   the toplevel executable it is not loaded if required again. 
7. There is one thing that `ocamlfind` does that the proposal missed is
   that during linking it also add library directories as `-I` includes
   to OCaml. 

   Most of the time this is has no effect; but may influence unprefixed, so
   called "implicit" (see `Filename.is_implicit`) positional arguments
   lookups. It is however important for libraries that have C stubs since 
   this eventually it adds a `-L` to the C linker for the library directory and
   allows to find the static C stub library archive usually recorded as 
   `-lmylib_stub` arguments in cma (for `-custom`)and cmxa files. 
   
   It is a little bit unclear what to do here. Here are a few solutions:
   
   1. Do it the `ocamlfind` way and simply add the library directory of all
      resolved libraries to the includes as is done during compilation.
   2. Automatically add the directory of cm[x]as as `-L` arguments
      to the C linker invocation.
   3. There is somehow already partial support for point 2. under 
      the form of the undocumented `$CAMLORIGIN` substitution 
      for `-ccopt` arguments (rationale is [here](https://github.com/ocaml/ocaml/issues/6642)). This means that compiling a library with C stubs as follows, 
      is sufficient to make bytecode -custom and native code linking work:
      ```
      ocamlmklib -o mylib/lib -oc mylib/mylib_stubs ... -ccopt '-L\$CAMLORIGIN'
      ```
      See for example [this test](https://github.com/dbuenzli/ocaml-lltest/blob/fba2fe1fc01fc57415a52879efb428b28067cd78/Makefile#L46-L53).
      But it feels a bit silly to have to do it explicitely.
      
   It should be noted that 1. has an additional side effect is that 
   it adds the library directory to the DLL path used to check 
   DLLs during non-custom bytecode compilation. This means if your
   library has not installed it's `dllmy_stub.so` in a directory 
   accessible via `CAML_LD_LIBRARY_PATH` and you try to link a byte
   code executable using it, it won't fail at compile time (however
   the executable it will fail at runtime if the `.so` is not 
   accessible).
8. The proposal had both a `-lib` and `-lib-require` flag. The former was
   meant to be used to denote library usage and the latter was meant to
   be used for recording library usage in library archives. 

   During implementation it became clear that these two usages are
   mutually exclusive. So currently there is a single flag that is used
   which is `-require` (whose name also corresponds to the toplevel
   directive for loading libraries) and whose semantics depends on
   context: during archive creation it means record library usage,
   otherwise it means use the library.

   Case could be made that both in `ocamlc -a` and `ocamlopt -shared
   -a` you would like to have the difference since these two allow to
   repack multiple cmas and cmxa into a single archive (note that
   `ocamlopt -a` does not allow this). In this case we argue that if
   you want to repack multiple archives you can simply lookup the cma
   and cmxas yourself according to the library convention (it's not
   something you'd do in the eco-system anyways as this would lead to
   library lookup problems).
9. Requires and ordering. To simplify the implementation we follow the way
   ocamlfind does when `-packages` are specified. For compilation all 
   `-require`d lib include directory are added at the end, in the given order. 
   For linking all `-require`d library archive are put before the other 
   files in the given order. This means:
   ```
     ocamlc -c -I inc0 -require lib1 -I inc2
   = ocamlc -c -I inc0 -I inc2 -I /path/to/lib1
   ≠ ocamlc -c -I inc0 -I /path/to/lib1 -I inc2

     ocamlc -o a.out obj0.cmo -require lib1 obj1.cmo
   = ocamlc -o a.out obj0.cmo /path/to/lib1/lib.cma obj1.cmo 
   ≠ ocamlc -o a.out /path/to/lib1/lib.cma obj0.cmo obj1.cmo
   ```

   Initially we had intended to respect potential interleaving with
   includes and compilation objects. This could be done but is likely
   to be quite an invasive and delicate refactoring procedure for the
   compiler.  The problem is that the cli parsing sets references in
   `Clflags` which are used throughout the code base (with some
   parts of the compiler actually mutating these flags...). And we
   would need to merge what are currently separated references into a
   single `[ I of dir |  File of string | Require of Lib.Name.t] list`
   to be able to get the relative orderings in different contexts.
10. The `-linkall` support to embed library names in byte and native
    code executables is done. Respectively by adding a new `LIBS`
    section in the byte code and a new `caml_imported_libs` symbol in
    native code with the set of library names that were fully linked
    via `-require` and recursively. An `-assume-library LIB` was also
    added that allows to simply add `LIB` to the set of libraries that
    are supposed to be embedded in the executable. This is useful for
    handling `-linkall` with uninstalled libraries or for build
    systems that perform lookups and put library archives themselves
    on the cli (cf. the `-noautoliblink` flag).    
11. In theory `-assume-library a` (see 10.) and `-require b` could
    always be used together in particular if library `b` requires `a`,
    it won't be looked up. In practice however due to
    point 9. problems will arise at link time since the archive you
    specify for `a` on the cli will come after `b`'s one and lead to a
    link error. So at the moment having dependencies from `-require`
    to `assume-library` libraries is not really supported but that is
    not usually the case. If that happens though you should translate
    all `-require` to `-assume-library`, do all the lookups and
    sorting yourself and use `-noautoliblink`.
12. Dynlink API: support for library loading was added. Works only in
    bytecode for now. Both `loadfile` and `loadfile_private` take a new
    optional `ocamlpath` argument in which, if provided, libraries
    mentioned in objects are looked up and loaded (if needed and if
    they are not part of the executable, see `-linkall`). It is a
    little bit unclear how libraries should be loaded by
    `loadfile_private`, for now it was decided that the libraries that
    a private object needs are loaded publicly. A `loadlib` function was also
    added to load a library from a specified `OCAMLPATH` aswell as a few
    functions to enquire about which libraries are loaded.
13. Dynlink native API: there is currently a problem with loading the
    libraries required by a `.cmxs` file. More time than was allocated
    for now is needed to fully investigate this to find the right solution. The 
    problem is that the libraries to load before the `cmxs` are mentioned in the
    `caml_plugin_header` symbol of the `cmxs` but the `dlopen(2)`
    complains before about missing symbols from the dependencies. A
    [quick hack](https://github.com/dbuenzli/ocaml/commit/f15a16c234808712e4bbd5bd3cd3d5bad76de16c) 
    was tried to load the symbols lazily using
    [`RTLD_LAZY`](https://linux.die.net/man/3/dlopen) but it didn't
    seem to have an effect on macOS and on linux that change makes the
    compiler build fail at some point where it dynlinks `unix.cma` with 
    `invalid mode for dlopen(): Invalid argument`. 
    Here are a few options to solve that problem:
    1. We manage to make `RTLD_LAZY` work reliably, this means we can lookup 
       `caml_plugin_header` and load the libraries it depends on before 
       proceeding to run the OCaml init bits.
    2. Add seperate metadata for `cmxs` files which would be a bit sad. 
    3. See how expensive (time and code-wise) it is to devise ad-hoc
       executable data section symbol readers for `ELF`, `Mach-O`, and `PE`
       executable. This would allow to lookup the `caml_plugin_header`
       ourselves before we try do `dlopen` the `cmxs` (and allow us to
       get rid of the `bfd` library dependency see point 5.). The problem 
       of course is the day someone invents a new executable format the 
       system will not be ready.
14. It's unclear whether defaulting `OCAMLPATH` to `$(ocamlc -where)/..` is a 
   good idea or not. 
15. This implementation has no support for `OCAMLPARAM`. 

## Old supporting work

### OCAMLPATH

There is a [PR](https://github.com/ocaml/ocaml/pull/8946) for making
`OCAMLPATH` meaningful to the OCaml toolchain. For now its only effect
is to redefine the `-I +DIR` notation so that distributions can start
reshuffling their install structure to make it easier to extend a
system OCaml package install prefix with an opam package install
prefix.


