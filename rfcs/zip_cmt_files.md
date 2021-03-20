
# Compressed cmt files

Cmt and Cmti files are currently made of marshalled typedtree ASTs. They tend
to become pretty big for non trivial codebases (often > 500kb, or more, for a single module).

A quick experiment on a few repositories shows
the following reduction after using `gzip` to compress all `.cmt` and `.cmti` files,
for the content of the opam `lib/<project>` directory:

| project | normal | compressed |
| --- | --- | --- |
| containers | 14M | 9.2M |
| base | 40M | 27M |
| dolmen | 14M | 9M |
| logtk | 71M | 49M |
| lwt | 11M | 8.1M |
| camomille | 12M | 8.5M |
| oseq | 1.1M | 820k |

The proposal is thus to bundle zlib (some C implementation of it, which weights as little
as 98kb on my system as a .so file) in the compiler, and always compress cmt/cmti files
with it, on the fly, during their generation.

A potential side benefit would be to be able to add zlib to the standard library,
which could really use such a pervasive format. Of course it can remain a compiler-only
dependency if preferred.
