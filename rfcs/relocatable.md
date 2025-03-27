# Relocatable OCaml
## Introduction
The OCaml compiler distribution requires the Standard Library to be stored in a fixed location, specified when the compiler itself was compiled. Executables produced by the bytecode compiler (`ocamlc`) by default also require the interpreter (`ocamlrun`) and the Standard Library to be stored in a similarly-fixed location.

A consequence of this is that in order to "move" the compiler to a new location, it is necessary to build a new compiler from scratch, configured with the new location, and then recompile any bytecode executables which were built with the previous compiler.

This can be readily seen in `opam` today:
```console
# Create an opam local switch
dra@Tau:~/work$ opam switch create . ocaml.5.3.0
...

# Report the Standard Library location from both builds of ocamlc
dra@Tau:~/work$ opam exec -- ocamlc.byte -where
/home/dra/work/_opam/lib/ocaml
dra@Tau:~/work$ opam exec -- ocamlc.opt -where
/home/dra/work/_opam/lib/ocaml

# Rename the directory
dra@Tau:~/work$ cd ..
dra@Tau:~$ mv work new-work
dra@Tau:~$ cd new-work

# Repeat the previous test
dra@Tau:~/new-work$ opam exec -- ocamlc.byte -where
Fatal error: exception /usr/local/bin/opam: "execvpe" failed on /home/dra/new-work/_opam/bin/ocamlc.byte: No such file or directory
dra@Tau:~/new-work$ opam exec -- ocamlc.opt -where
/home/dra/work/_opam/lib/ocaml
```

`ocamlc.byte` cannot be run, because the interpreter specified in its "shebang" line (`#!/home/dra/work/_opam/bin/ocamlrun`) no longer exists. `ocamlc.opt` can be run, but returns the original location of the Standard Library, which no longer exists, rather than the new location of the files in `/home/dra/new-work/_opam/lib/ocaml`.

Relocatable OCaml proposes to fix this. An intended consequence of the proposal here is that not only can the directory containing a compiler installation be moved, but it may also be copied, specifically:
```console
# Create an opam local switch using Relocatable OCaml
dra@Tau:~/compiler1$ opam switch create . --repos=relocatable,default ocaml.5.3.0
...

# Clone it
dra@Tau:~/compiler1$ cd ..
dra@Tau:~$ cp -a compiler1 compiler2

# Report the Standard Library location from each directory, without using opam
dra@Tau:~$ compiler1/_opam/bin/ocamlc -where
/home/dra/compiler1/_opam/lib/ocaml
dra~Tau:~$ compiler2/_opam/bin/ocamlc -where
/home/dra/compiler2/_opam/lib/ocaml
```
## What is "Relocatable"
A compiler distribution is _Relocatable_ if its build, for a given system, satisfies the following three properties:
1. The binaries produced are identical regardless of the installation prefix (the `--prefix` argument passed to `configure`) or the working directory in which the compiler was built.[^1]
2. The resulting compiler distribution can be used from any disk location on the system without any further alteration to the binaries.
3. The resulting compiler distribution can be used from any disk location on the system without any further alteration to the user's shell environment.

[^1]: In particular, with reference to the previous demonstration, this means that the compiler produced by creating a local switch in an additional directory `compiler3` will result in _exactly_ the same files being installed.
## Changes required
The specification for being _Relocatable_ is broadly related to being _Reproducible_, and boils down to ensuring that the resulting binaries do not contain either the "prefix" (the common directory to which both the executables and library files have been installed) or the "build directory" (the directory in which the compiler was built).

There are five places where this information is presently stored:
1. `ld.conf`, used to specify the search path for shared bytecode C stubs (prefix directory)
2. The headers/images of "default" bytecode executables, either in the "shebang" header, or in the `RNTM` bytecode section (prefix directory)
3. `OCAML_STDLIB_DIR`, in the build of the runtime, and `Config.standard_library_default`, which underpins the exposed `Config.standard_library` (prefix directory)
4. Debugging information emitted by the bytecode compiler and the assembler (build directory)
5. `Makefile.config` (prefix directory) and `config.status`, which is installed by opam (prefix and build directory)

### 1. `ld.conf`
> **TL;DR**
> - Lines beginning `./` or `../` in `ld.conf` now interpreted relative to `ld.conf`
> -  `$OCAMLLIB/ld.conf`, `$CAMLLIB/ld.conf` and the default `ld.conf` are all processed
>
>_New behaviour is backwards-compatible and so always enabled._

By default, `ld.conf` contains the full path to the Standard Library (which is the directory containing `ld.conf`) and then the full path to the `stublibs` subdirectory.

At present, the runtime blindly follows each line, even if the lines are not absolute paths. This interpretation of such lines is not useful and it is exceedingly unlikely that they are being used this way. The proposal is to interpret "explicit-relative" paths (that is paths beginning `./` or `../`) as being relative to the directory containing `ld.conf` (i.e. relative to the Standard Library directory). This means that the default `ld.conf` just contains the lines `./stublibs` and `.`.

This change can be made unconditionally. The only other OCaml ecosystem tool known to manage `ld.conf` is [findlib](http://projects.camlcity.org/projects/findlib.html). However, the way opam packages findlib means that `ocamlfind` never changes `ld.conf` and, owing to a misconfiguration, `ocamlfind` has been emitting a warning about `ld.conf` for years without anyone noticing!

The final change for `ld.conf` is to harden the _runtime_ against the setting of `OCAMLLIB` (and the ancient `CAMLLIB`) environment variables. At present, OCaml processes the first `ld.conf` found in the directory pointed to by either `$OCAMLLIB` or `$CAMLLIB` and only if neither is set does it look for `ld.conf` in the configured Standard Library location. The proposal is to read each of these files, simply adding the lines to the search path. The only breaking change here is that potentially a bytecode program which could _not_ start previously may now actually run - but in any environment where a program previously started successfully, it would continue to load in the same way.
### 2. Bytecode headers
> **TL;DR**
> - New option `-runtime-search` control the search mode for finding `ocamlrun`
> - `--enable-runtime-search[=always]` configuration option to control whether the search mode for the compiler's executables
> - Name mangling scheme for DLLs and for the `ocamlrun` binaries to reduce the chances of different compiler distributions interfering with each other. Mangling is accessed via `-suffixed` option for `ocamlmklib` and `-dllib-suffixed` for `ocamlc`.
>
_Name mangling always enabled because it's not user-visible; new behaviour for the header search is opt-in._

The proposal for dealing with the embedded absolute path in either "shebang" line of a bytecode executable or the `RNTM` section of a bytecode image is to provide two additional mechanisms for locating `ocamlrun`:
1. Looking in the directory containing the bytecode executable (which corresponds neatly with an opam switch's `bin` directory)
2. Searching the `PATH` environment variable for `ocamlrun` (which is what Windows presently does by default already)

The mechanism used is selected using a new `-runtime-search` option to `ocamlc` which takes three possible values:
- `disable` - the current behaviour of the compiler, which causes an absolute path to be used for the shebang/`RNTM`
- `enable` - the header will first attempt to use the absolute path, but fallback to the two searches above if the interpreter no longer exists at that location
- `always` - no absolute path is used, and only the two searches above are carried out

There is a `configure` option `--enable-runtime-search[=always]` which enables/disables this option for the build of the compiler itself and `--enable-runtime-search-target[=always]` which controls the default value that the compiler which has been built will use for executables it creates. For opam switches, the proposal is that the compiler be built with `--enable-runtime-search=always --enable-runtime-search-target` - i.e. the compiler binaries are fully relocatable, and bytecode executables produced by that compiler (i.e. the user's binaries) have the fallback ability to search for the runtime.

Both the `PATH` search proposed here and the increased number of places from which `ld.conf` is now read increase the chances of the "wrong" runtime being located, and the proposal therefore includes the adoption of a name mangling scheme to be used for _all_ the DLLs produced by the distribution (both dynamically-loaded bytecode C stub libraries and the "shared" versions of the runtime libraries) and also for the `ocamlrun` binary.

The `runtime-launch-info` configuration file in the compiler also contains the location of the binary directories. The proposal is to add an interpretation of `.` in this file to refer to the directory containing the compiler (i.e. instead of writing `/usr/local/bin` to `runtime-launch-info`, it becomes possible just to write `.` and have `/usr/local/bin/ocamlc` interpret that as meaning `/usr/local/bin`).

The mangling scheme introduces a two-part suffix to DLL file names. The first part is the host-triplet (e.g. `x86_64-pc-linux-gnu`) and the second is an encoded bitmask identifying the runtime. This second part is formed from the version of OCaml and active settings which affect the configuration of the runtime. The scheme is designed such that DLLs for different architectures/platforms have fundamentally different names, as do DLLs targeting different versions of OCaml, as do DLLs compiled for OCaml with flat-float-array or without flat-float-array. For example, where before the DLL stubs for `unix.cma` were always in `dllunixbyt.so`, under this scheme they instead are put in a DLL named `dllunixbyt-x86_64-pc-linux-gnu-0018.so` (the `0018` indicating a development release of OCaml 5.4).

This mangling scheme is opted into by using a new `-suffixed` option to `ocamlmklib` which generates files with the appropriate names (these names can also be reliably predicted using values displayed by `ocamlc -config`). The list of libraries to be loaded by `.cma` files is augmented with an additional flag to indicate if the name is supposed to be mangled.

`unix.cma`, however, still just records that it requires `-lunixbyt`, but a new suffixed flag is set (and recorded in the .cma file), which signals to the _runtime_ to mangle the name to the correct `dllunixbyt-x86_64-pc-linux-gnu-0018.so` when `unix.cma` is dynamically loaded. This is simply an extension of the existing mechanism in the runtime for dealing with the `.so` (Unix) or `.dll` (Windows) extension.

Since there are some runtime options which only affect native code, there are different identifiers for both bytecode and native code, although the native code identifier is only needed for `libasmrun_shared.so`. To maintain simplicity, the same bit numbers are used for settings - if they only affect, say, native code, then that bit is simply always 0 when calculating the bytecode identifier.

`ocamlrun` itself is also mangled in the same way and the binary is installed as, say, `x86_64-pc-linux-gnu-ocamlrund-0018`. However, symlinks are also created pointing to this new name for both `ocamlrun` and `ocamlrun-0018`. Bytecode executables therefore do not search for `ocamlrun` (which could be any version of OCaml), but instead search for `ocamlrun-0018`. Thus a bytecode executable using the Unix library now:
1. Searches for a runtime called `ocamlrun-0018`
2. That runtime knows its host triplet is `x86_64-pc-linux-gnu` applies both that and the runtime identifier to search for `dllunixbyt-x86_64-pc-linux-gnu-0018.so`
Crucially, both the runtime, and stubs DLLs, of other versions of OCaml on the same system and potentially also in `PATH` will be "automatically" ignored. The _same_ bytecode image can also be run on a different system - for example, if copied to an `aarch64` system, then the `ocamlrun-0018` binary, being configured with a different host triplet, will then search for its appropriate DLLs.
### 3. `caml_standard_library_default`
> **TL;DR**
> - New configure option `--with-relative-libdir=../lib/ocaml` allowing the compiler to be configured to locate the Standard Library relative to the running binary
> - New primitive `%standard_library_default` accesses linker support in both `ocamlc` and `ocamlopt` for emitting the _computed_ location of the Standard Library to user programs (i.e. programs _compiled_ with the compiler still use an absolute path)
> - New option `-set-runtime-default standard_library_default=../lib/ocaml` overrides the value computed by the linker and allows a program to opt-in to the relative search
> 
> _Existing programs work exactly as before - relocatable binaries are entirely opt-in and the compiler by default still builds in "absolute" mode._
 
Both the compiler and user programs need to be able to rely on `Config.standard_library` giving the current location of the Standard Library. Fundamentally, for the compiler distribution to be relocatable, this value cannot be stored as a constant in the module, but must be somehow derived when the compiler is called.

At present, when the compiler is configured, a commitment is made that the Standard Library remains at a fixed location on the system. The proposal adds a new configure option `--with-relative-libdir` which instead allows a different commitment to be adopted: the compilers will always be executed at the same location relative to the Standard Library. For example, the default build of OCaml (with just `--prefix`) is made relative by specifying `--with-relative-libdir=../lib/ocaml` and now, instead of committing to having the Standard Library at a fixed location on disk, the user instead commits to having `ocamlc`, `ocamlopt` etc. be executed from a directory such that `../lib/ocaml` from that directory will be the Standard Library.

This works perfectly for the compiler, but user programs which link against `ocamlcommon.cma` and may need `Config.standard_library` have not opted in to any guarantee about being executed in a specific directory _relative_ to the Standard Library.

The proposal, therefore, is to add linker support which is provided by the addition of a new primitive `%standard_library_default` and a new compiler option `-set-runtime-default`. The `%standard_library_default` primitive has type `unit -> string`. When called, this primitive returns the location calculated at link-time and embedded in the executable. The `-set-runtime-default` allows the value calculated by the linker instead to be replaced by a user-provided location.

When built with `--with-relative-libdir`, both `ocamlc` and `ocamlopt` calculate the absolute location of the Standard Library when linking executables and ensure that _this_ value is written to executables they link. The `-set-runtime-default` option for both compilers then allows this absolute value to be overridden, and this provides the mechanism for creating Relocatable executables. Thus `ocamlc` and `ocamlopt` themselves are compiled with `-set-runtime-default standard_library_default=../lib/ocaml` (when OCaml has been configured with `--with-relative-libdir`). This _relative_ path is then returned by the `%standard_library_default` primitive and additional code in the `Config` module _at runtime_ calculates the actual value for `Config.standard_library` using the directory of the running compiler executable itself. A user program, compiled without specifying `-set-runtime-default` simply receives the absolute path from the `%standard_library_default` primitive and the `Config` module does no further processing to it.

Bytecode executables always need to know the Standard Library location for loading shared libraries for C stubs, but in native code, the value is only required if `Config.standard_library` is actually used. The proposal therefore uses a `%` primitive, which is converted to a C call in bytecode but which is directly compiled to a pointer in native code (using a very similar mechanism to the constants in the `Sys` module).
### 4. Debugging information
OCaml already has capabilities for dealing with this, since support for `BUILD_PATH_PREFIX_MAP` was added in [ocaml/ocaml#1515](https://github.com/ocaml/ocaml/pull/1515) and released in OCaml 4.08.0. This support already detects C compiler support (via `-fdebug-prefix-map`), and this detection can be further extended into to the C portions of the OCaml distribution to create fully reproducible artefacts.

The proposal is to set `BUILD_PATH_PREFIX_MAP` when the compiler is configured with `--with-relative-libdir` in order to map the build directory to `.` (i.e. instead of embedding `/home/dra/relocatable/ocaml/stdlib/stdlib.ml` as a source location, the location is mapped to `./stdlib/stdlib.ml`). In addition to this, when available, the C compiler option `-fdebug-prefix-map` or `-ffile-prefix-map` is used internally by the build so that the objects in `libcamlrun.a`, etc. and binaries with debugging information also don't contain these absolute paths.

It's important to note that in the context of opam (where this setting will principally be used) that these source locations are already bogus, since they refer to the source tree from when the compiler was packaged, which opam deletes after the compiler is installed. Making installed compilers have correct debugging source locations is left as future work - the key things for this proposal is that mapping these paths away to `.` leaves something which is already broken still broken and, because it's only activated with `--with-relative-libdir`, even in an opam switch, the previous behaviour can be restored for debugging purposes (e.g. when opam is instructed to keep the source tree of compiled packages after installation).
### 5. `Makefile.config` & `config.status`
While GNU make potentially allows `Makefile.config` to be updated to calculate the values for `$(prefix)` and so forth, `Makefile.config` is presently able to be read by other implementations of `make`, and it seems unnecessary to lose that. Packages which are using `$(LIBDIR)` from `Makefile.config` would be much better off calling `ocamlopt -where` instead. The proposal is therefore to leave these values in `Makefile.config`, and to use the bulk testing tools for opam-repository to see if they can instead be permanently removed in the future.

`config.status` is installed by the compiler's packages in `opam` to allow the source tree to be rapidly re-configured. The fact this file includes some references to the original build directory are innocuous, and can be eliminated by post-processing the file in the build to remove them.