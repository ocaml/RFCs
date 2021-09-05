# RFC: Change the stdlib Random implementation to a splittable PRNG from PRINGO (SplitMix or ChaCha)

This RFC proposes to replace the current implementation of the standard library [Random](https://ocaml.org/api/Random.html) module by one of the "splittable" PRNG proposed by @xavierleroy's [pringo](https://github.com/xavierleroy/pringo/) library. The motivation is the move to Multicore, where splittable generators let us provide better behavior for the current global Random interface.

*Note: This RFC is in a Draft state: it needs input from other people (in particular @xavierleroy, js-of-OCaml and mirage people) before it can be considered a final proposal.*


## Motivation: Multicore

### Random and Multicore

`Random` provides a "global" interface that is not explicitly parametrized over the generator state -- the `Random.State` module provides a parametrized version: `Random.bool : unit -> bool`, etc. This "global" interface creates correctness or performance problems in a Multicore world:

- If we keep a single global mutable generator state, it needs to be protected by a lock, which makes the PRNG a concurrency bottleneck.

- If we give an independent random generator to each domain, it is unclear how to initialize the state of a new domain. Reusing the state of the parent domain is wrong (the two domain will generate the same random numbers), and inflexible "seeding" policies will be incompatible against some user choices of seed.

Other approaches tend to have dubious randomness properties, and often require tight coupling between the Random library and the multicore runtime, which would increase implementation complexity.

### Splitting saves the day

Some PRNG algorithms provide a "split" operation that returns two pairs of random-generator states, designed to generate independently-looking sequences.

With such "Splittable PRNGs", creating a new domain simply requires splitting the generator state of the parent domain. This approach is efficient (no synchronization of the generator state), has good randomness properties, and it is simple to implement inside the Random module (using runtime-provided thread-local-storage primitives). The decisions of the user in terms of seeding (OS-random or deterministic) are naturally respected, etc. In short: you want a splittable generator for Multicore.

(For a previous discussion of this problem where the suggestion to move to splittable PRNGs emerged, see  https://github.com/ocaml-multicore/ocaml-multicore/pull/582#pullrequestreview-683676282 )


## PRINGO generators

The [pringo](https://github.com/xavierleroy/pringo/) library provides two splittable PRNG implementations.

| Name     | numeric type | speed wrt. Random | pure OCaml   | secure            | when to reseed?  |
|----------|--------------|-------------------|--------------|-------------------|------------------|
| Random   | int          | =                 | yes          | no                | ?                |
| SplitMix | int64        | slightly faster   | no (C stubs) | no                | after 2^30 draws | 
| ChaCha   | int32        | slightly slower   | no (C stubs) | yes (weak crypto) | after 2^64 draws |

To give a sense of the performance difference, on my machine, on a micro-benchmarking drawing numbers in a tight loop (`make benchmarks` from the pringo directory):

- drawing `int` has the same speed for all generators,
- drawing `int64` takes 55% of the Random time with SplitMix and 93% of the Random time with ChaCha, and
- drawing `float` values in [0;1] takes 66% of the Random time with SplitMix, and 120% of the Random time with ChaCha.

Those differences are unlikely to be noticeable, as drawing random numbers is a neglectible part of most programs. Some very specific PRNG-bound algorithms (Monte-Carlo simulation, etc.) probably use custom PRNGs anyway.


## Requirements for a standard-library PRNG

Here is the list of potential requirements we considered to decide on including a PRNG implementation the standard library.

### Non-concern: 32-bit hosts

SplitMix, using 64-bit integers internally, is going to have lesser performance under 32-bit systems. We have more or less decided that we care less about the performance of 32-bit CPU architectures these days (we specifically discuss Javascript below), so we propose to not consider this in the choice process.

### Concern: good randomness

We want PRNGs that pass statistical tests for good randomness. All algorithms discussed here perform well under standard PRNG-quality test (diehard, etc.).


### Possible concern: lesser portability of C stubs

Random currently has a pure-OCaml implementation (except for the system-specific auto-seeding logic). Moving to C stubs may be an issue, at least for Mirage users, and require extra work from alternative implementations (js_of_ocaml, BuckleScript, etc.).

SplitMix is a very simple algorithm, it is easy to provide a pure-OCaml version that should perform well. Porting Chacha, which is more elaborate, is more work, but it's also harder to predict performance for the pure-OCaml version.


### Possible concern: Javascript performance

js_of_ocaml and Bucklescript/ReScript are important for our users, and it would be nice to ensure that the performance is not disappointing on these alternative implementations.

js_of_ocaml implements int64 with an emulation, suggesting that SplitMix could take a performance hit. (There is no native int64 type under Javascript anyway, "mainstream" approaches such as [long.js](https://github.com/dcodeIO/long.js) emulate them with two numbers.) It may be that JS engines optimize the emulation layer efficiently, so we should evaluate each choice.

(Using C stubs in fact gives more flexibility for alternative implementations to provide their own native implementation of these functions.)


## Summary

What are the requirements for the Random module?

The requirements discussed in the current version of the RFC are:
- implementation being pure OCaml
- performance
- "security" of the PRNG
- reseeding recommendations


Here is a summary of what we understand to be the consensus so far (I'll edit this part of the RFC as discussion progresses):

- pure-OCaml: yes. 

  A pure-OCaml implementation makes people's life simpler would be strongly preferable; we should port SplitMix and Chacha and re-run benchmarks.

- performance: not important, but check that js_of_ocaml does okay.

  A small performance hits are not very important for the standard Random PRNG, so a reasonable decrease (for everyone or just js_of_ocaml) would be perfectly acceptable.
  
  We should still benchmark under js_of_ocaml to have an idea of the performance (once we have pure-OCaml versions). It's bad if some parts of OCaml programs suddenly become much slower there.

- Having a crypto-secure PRNG is not part of our requirement -- Random is not secure in that sense.

- Having to reseed SplitMix every 2^30 draws may be a problem in practice -- loops running 2^30 iterations are perfectly reasonable thse days. All other things being equal, ChaCha should be preferred.

