# Persistent compiler processes

Build systems typically perform each compiler invocation as a separate process.
However, starting the compiler has a cost: starting a process has a cost, and
the compiler typically repeats some of the same actions in each invocation (eg
reading and unmarshalling `.cmi` files). That is why some compilers have a
**persistent** or **server** mode (see eg
https://per.bothner.com/papers/GccSummit03/gcc-server.pdf for GCC). Keeping a
single process for longer and passing multiple individual requests to the same
server can significantly reduce the amount of duplicate work and hence of
compilation times.

This RFC proposes the addition of such a "persistent" mode to the compiler tools
(`ocamlc`, `ocamlopt`, `ocamldep`, etc).

Each of these tools is extended with a new `-server` flag. When this flag is
passed, upon being launched the tool waits for requests on `stdin`. Whenever a
request arrives, the tool services the request and replies with a response on
`stdout`, and waits for the next request.

Each request encodes a command-line invocation of the tool and consits of:

1. a request id
2. a directory (the current working directory for the invocation)
3. an array of arguments (ie the `argv` of the invocation)

Each response consists of:

1. a request id (that of the corresponding request)
2. an exit code
3. two strings, containing the stdout and stderr of the invocation

The objective of this RFC is to gather feedback, decide if this is a direction
we want to go into, identify any blockers, etc.

## Prototype

I implemented a prototype in order to do some preliminary benchmarking:

https://github.com/ocaml/ocaml/compare/ocaml:ocaml:5.4...nojb:ocaml:server_mode_540?expand=1

Request:
```
REQ request-id number-of-arguments
current-directory
argument-1
...
argument-N
```
Response:
```
RES request-id exit-code out-length err-length
out-blob err-blob
```

Note that the prototype branch has a Python script that can be used as a shim to
simulate the integration with a build system. Roughly one can invoke the shim as
one would the compiler today, but the shim starts a background process that
maintains a pool of workers to service incoming requests. I am using this shim
to test by build existing codebases which do not know anything about the
`-server` mode.

## Benchmark

TL;DR: the `-server` version is around 10-20% faster (depending on the size of
the file being compiled). The speedup is similar on Windows and Linux (I had
expected the speedup to be more pronounced on Windows, but in my tests that was
not the case).

I did the measurements using the following script `bench.sh` which repeats the
same compilation command (with flags given on the command line) some number of
times, once using separate process invocations and once using a single server
process.

```sh
#!/bin/bash

# Usage: bench.sh <numiter> <tool> [args...]

numiter=$1
shift

tool=$1
shift

time {
    for i in $(seq 1 $numiter); do
        $tool "$@"
    done
}

time {
    for i in $(seq 1 $numiter); do
        echo "REQ t$i $#"
        echo "$PWD"
        for arg in "$@"; do
            echo "$arg"
        done
    done
} | $tool -server >/dev/null
```

**Linux**

- `typecore.cmx` (30 times)
```
$ ./bench.sh 30 [...] -c typing/typecore.ml

real    0m31.926s
user    0m28.119s
sys     0m3.797s

real    0m29.009s
user    0m27.264s
sys     0m1.731s

# => 10% faster
```
- `clflags.cmx` (100 times)
```
$ ./bench.sh 100 [...] -c utils/clflags.ml

real    0m11.399s
user    0m8.854s
sys     0m2.616s

real    0m9.026s
user    0m8.010s
sys     0m0.894s

# => 20% faster
```

**Windows**

- `typecore.cmx` (30 times)
```
$ ./bench.sh 30 local/bin/ocamlopt.opt [...] -c typing/typecore.ml

real    0m35.535s
user    0m0.304s
sys     0m0.319s

real    0m32.102s
user    0m0.094s
sys     0m0.046s

# => 10% faster
```
- `clflags.cmx` (100 times)
```
$ ./bench.sh 100 local/bin/ocamlopt.opt [...] -c utils/clflags.ml

real    0m26.032s
user    0m0.940s
sys     0m0.990s

real    0m20.317s
user    0m0.154s
sys     0m0.124s

# => 20% faster
```

## Some technical details

- To avoid depending on `unix`, the simplest is to use `stdin` and `stdout` to
  communicate with clients. In particular clients will need to guarantee not to
  interleave requests (however, pipelining requests, ie have more than one
  in-flight request at a time, presents no problem). The compiler will handle
  each incoming request in a strictly sequential manner.

- One needs to reset all compiler state between requests. Luckily, we already
  have some infrastructure to help with this: `Local_store`. For example, in the
  prototype above, all top-level references (notably in `Clflags`) have been
  switched from using `ref` to using `Local_store.s_ref`.

- All output (mostly error messages and diagnostic information) needs to be
  saved to a buffer when in server-mode. In the prototype this is achieved by
  replacing calls to `Stdlib.print_string` by a dedicated function which
  captures the output so that it can be sent back to the client when the request
  is complete.

- Similarly, naked calls to `Stdlib.exit` must be replaced by an exception or a
  similar mechanism to avoid terminating the process, and instead just return a
  response to the client. Luckily we already had an exception for this purpose:
  `Compenv.Exit_with_status`.

## Integration with Dune and other build systems

Some build systems define generic protocols for persistent worker processes (see
eg [Bazel](https://bazel.build/remote/persistent) and
[Buck2](https://buck2.build/docs/prelude/rules/core/worker_tool/)). Dune may
want to define its own generic protocol which we would then use in the
compiler side. Or we could define an ad-hoc protocol just for use of the
compiler (as I did in my prototype).

Preliminary discussion with @rgrinberg confirms that if this feature existed in
the compiler, there is apetite for it to be supported in Dune.

Technically, to integrate this feature, Dune would have to maintain a pool of
server processes, and dispatch with an RPC call each compilation command
(instead of spawning a new process as today).

## Future directions

Of course, saving on the process startup cost as in this proposal is only the
beginning. Once that is done, it opens the door to caching certain data between
requests, for example unmarshalled `.cmi` files, which is likely to further
reduce compilation times.

## Some references

- Bazel persistent worker protocol: https://blog.bazel.build/2015/12/10/java-workers.html (see also https://bazel.build/remote/persistent)
- Buck2 persistent worker protocol: https://buck2.build/docs/prelude/rules/core/worker_tool/
- GHC persistent worker plugin: https://github.com/MercuryTechnologies/ghc-persistent-worker (see also https://www.tweag.io/blog/2019-09-25-bazel-ghc-persistent-worker-internship/)
