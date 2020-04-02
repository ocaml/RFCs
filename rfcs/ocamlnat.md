# Fast native toplevel using JIT

# Overview

We (Jane Street + OCL/Tarides) would like to make the native toplevel
faster and more self-contained.

At the moment, the native toplevel works by calling the assembler and
linker for each phrase. This makes it slow and dependent on an
external toolchain which is not great for deployment.

To reach this goal, we would simply like to bring [this work][paper]
up to date and merge it in the compiler.

# Motivation

This work would provide a simple way to compile and execute OCaml code
at runtime, which would unlock a lot of new possibilities to develop
great new tools.

Coupled with the fact that we can already embed cmi files into an
executable, this work would make it possible to distribute a
self-contained binary that can evaluate OCaml code at runtime. This
would make it simple and straightforward to use OCaml as an extension
language.

## Verified  examples in documentation comment

We are particularly interested in this feature for the [mdx][mdx]
tool. More precisely, we are currently working on a feature allowing
verified toplevel snippets in mli files. For instance:

```ocaml
(** [chop_prefix s ~prefix] returns [s] without the leading [prefix]. *)

    {[
      # chop_prefix "abc" ~prefix:"a";
      - : string option = Some "bc"
      # chop_prefix "abc" ~prefix:"x";
      - : string option = None
    ]}
*)
val chop_prefix : string -> prefix:string -> string
```

In the above example, the `{[ ... ]}` would be kept up to date by
`mdx` to ensure that the document stays in sync when the code
changes. In fact, the user would initially only write the `#` lines
and `mdx` would insert the results just as with expectation tests.

# The change in detail

This change would add JIT code generation for x86 architectures as
described in [the paper][paper]. For other architectures, we would
still rely on the portable method of calling the assembler and linker.
The main additions to the compiler code base would be:

- some code in the backend to do the assembly in process
- a few more C functions to glue things together

The paper mentions that it adds 2300 lines of OCaml+C code to the
compiler code base.

One detail to mention: IIUC the [JIT ocamlnat][ocamlnat] from [the
paper][paper] goes directly from the linear form to binary
assembly. Now that we have a symbolic representation of the assembly
we could also go from the symbolic assembly in order to share more
logic between normal compilation and JIT.

We discussed with @alainfrisch and @nobj since LexiFi has been using
an in-memory assembler in production for a while. They mentioned that
they would be happy to open-source the code if they can, which means
that we could be using code that has been running in production for a
long time and is likely to be well tested and correct.

LexiFi's binary emitter is about 1800 lines of code including comments
and newlines. This looks a bit smaller than the JIT part of the JIT
ocamlnat, so we would still be adding approximately the same amount of
code if we went this way.

# Drawback

This is one more feature to maintain in the compiler and it comes with
a non-negligible amount of code. However, and especially if we can
reuse LexiFi in-memory assembler, most of the additions would come from
well tested code. @alainfrisch and @nobj also mentioned that this code
was very low-maintenance and had pretty much not changed in 5 years.

# Alternatives

For the mdx case, we considered a few alternatives.

## Using a bytecode toplevel

Mdx currently uses a bytecode toplevel where everything is compiled
and executed in byte-code. This includes:

- code coming from user libraries
- the full compilation of the toplevel phrases

as a result, mdx is currently very slow and the round-trip time
between the user saving a file and seeing the result easily climbs in
the tens of seconds.

In the case of Jane Street, we have one more difficulty with this
method: a lot of our code doesn't work at all in bytecode because we
never use bytecode programs.

## Staging the build

Given that mdx is a build tool, one alternative is to redesign the
interaction between mdx and the build system. For instance, it could
done in stages with a first step where mdx generates some code that is
then compiled and executed normally by the build system. This is how
the [cinaps][cinaps] tool works for instance.

However, it is difficult to faithfully reproduce the behavior of the
toplevel with this method. What is more, such a design is tedious and
requires complex collaboration between the tool and the build system.

Going through this amount of complexity for every build tool that wants
to compile OCaml code on the fly doesn't feel right.

## Using a mixed native/bytecode mode

One idea we considered is using a mixed mode where a native
application can execute bytecode. This would work well for us as the
snippets of code we evaluate on the fly are always small and fast.

However, it is completely new work while the native JIT has already
been done.  What is more, while it would work for us it might not work
for people who care about the performance of the code evaluated on the
fly.

A native JIT would likely benefit more people.

[mdx]: https://github.com/realworldocaml/mdx
[paper]: https://arxiv.org/pdf/1110.1029.pdf
[ocamlnat]: https://github.com/bmeurer/ocaml-experimental/
[cinaps]: https://github.com/ocaml-ppx/cinaps
