# Syntactic function arity

OCaml has dedicated syntax for n-ary (multiple-parameter) functions, written:

    fun a b c -> ...

for lambda expressions, or:

    let f a b c = ...

for let-bound functions.

However, while the OCaml parsetree already contains n-ary function
applications, function definitions are internally always unary. That
is, the function definition `fun P1 P2 P3 -> BODY` is currently
equivalent to:

    fun P1 ->
    fun P2 ->
    fun P3 ->
    BODY

The proposal in this RFC is to introduce n-ary function definitions to
the parsetree, with semantics so that they evaluate their argument
patterns only after all arguments have been received, rather than one
at a time. This makes the above equivalent instead to:

    fun x1 ->
    fun x2 ->
    fun x3 ->
    let P1 = x1 in
    let P2 = x2 in
    let P3 = x3 in
    BODY

The motivation for this change is in two parts, explained in detail
below:

  - **Semantics**: in the relatively rare cases where it makes a
    difference, the proposed semantics is both clearer and closer to
    programmer expectations than the current semantics.

  - **Implementation**: the proposed semantics would allow us to
    simplify some tricky and fragile parts of the compiler.

The precise change to the parsetree is discussed in the final section,
which requires a careful treatment of the `function` keyword (since
the reasonably common pattern `let f a b = function ...` defines a
function of arity 3, not 2).


## Semantic changes

The proposed change is a change to the order of evaluation of function
argument binding, which can make a difference only via side
effects. Such side effects are relatively rare, but do occur in
mutable patterns, optional argument defaults, and patterns that raise
exceptions.


### Mutable patterns

With the current semantics of n-ary function definitions, the
following program prints `0` both times (code from issue
[#7789](https://github.com/ocaml/ocaml/issues/7789)):

```ocaml
type data = { mutable n  : int; }

let f { n } t = Printf.printf "n=%d\n%!" n

let go udata g =
  g 1.0;
  udata.n <- 10;
  g 1.0
;;

let udata = { n = 0 } in
go udata (f udata);;
```

The read of the field `n` by `f` happens at the time of the creation
of the partial application `(f udata)`, rather than at the time of the
eventual call `g 1.0`, and the value `0` is cached.

This behaviour was changed to match the specification in 4.06. The current
behaviour is sufficiently confusing that it was repeatedly reported as
a bug in 4.06 (
[#7675](https://github.com/ocaml/ocaml/issues/7675),
[#7789](https://github.com/ocaml/ocaml/issues/7789),
[#10032](https://github.com/ocaml/ocaml/issues/10032),
[#10641](https://github.com/ocaml/ocaml/issues/10641)),
and in 4.12 a new warning (68, `match-on-mutable-state-prevent-uncurry`)
was introduced which fires if this behaviour ever occurs.

The semantics proposed here amount to returning to the pre-4.06
behaviour, but this time with a specification to match. Warning 68
would also become redundant and be removed.

### Optional argument defaults

Optional arguments can have default values, which are evaluated if the
optional argument is not specified, e.g.:

```ocaml
let f ?(x = default ()) y z = x + y + z
```

Due to an intentional loophole in the current specification, the exact
point at which `default ()` is evaluated is not specified. The
behaviour of the current implementation is to try to delay evaluation
of defaults until the function body. However, this sometimes results
in evaluation at an unexpected time (cf. issues
[7531](https://github.com/ocaml/ocaml/issues/7531),
[5975](https://github.com/ocaml/ocaml/issues/5975)).

First, since OCaml 4.13, defaults may be evaluated earlier than the
function body. This was changed after a number of soundness bugs were
discovered in the delaying heuristic in prior versions, as defaults
were being delayed past patterns that may depend on their results. In
the current behaviour, the rules for exactly when a default is
evaluated are subtle. For instance, evaluation order in the above
example of `default ()` changes if the variable pattern `y` above
changes to the record pattern `{y}` (regardless of mutability).

Second, since functions in the parsetree are always unary, it is not
always clear where the function body is, and defaults may be evaluated
later than expected. For instance, consider:

```ocaml
type distinct_counter = string -> int

let make_counter_1 ?(tbl = Hashtbl.create 10) () : distinct_counter =
  Hashtbl.(fun s -> replace tbl s (); length tbl)

let make_counter_2 ?(tbl = Hashtbl.create 10) () : distinct_counter =
  fun s -> Hashtbl.replace tbl s (); Hashtbl.length tbl
```

The function returned by `make_counter_1` returns the number of
distinct strings it has seen. However, the function returned by
`make_counter_2` always returns 1. The difference is that the module
open construct `Hashtbl.(...)` in the former is sufficient to block
delaying of default evaluation, while in the latter the creation of
the hashtable is delayed past `fun s -> ...`.

The semantics proposed here are that default arguments will always be
evaluated at the start of the function body, which in most cases
matches the current behaviour. The issues that arose with reordering do
not affect the proposed semantics, as all arguments are equally delayed.

### Patterns that raise

Nonexhaustive patterns and lazy patterns can raise exceptions during
evaluation (as can certain edge cases of unboxed floats, when
interrupted by signal handlers). The proposed change to the semantics
of n-ary functions will affect functions with such patterns.
Specifically, the following example will no longer raise a match
failure at definition time:

```ocaml
let f (Some x) y = x + y
let g = f None
```

Instead, the match failure will only be raised once `g` is called.
This is an observable change in behaviour, but it's hard to imagine a
program relying on it.

## Implementation considerations

The current logic to deal with the interleaving of patterns and
lambdas is quite subtle in parts, and these parts could be simplified
with the proposed semantics. In particular, the logic for delaying
default arguments (described above) would be simplified, as would the
logic for first-class module patterns (which for reasons to do with
the structure of Lambda, already partially delay binding). Finally,
the currying optimisations would become more reliable, as described
below.

The counterpoint is that some new work would need to be done to plumb
the arity of function expressions from parsetree through to Lambda, as
these are currently erased by the parser and approximately recovered
by Lambda (and are already preserved from Lambda onwards).

### Currying detection

Function definitions in OCaml are curried by default, in principle
accepting one argument at a time and returning a closure accepting the
rest. Since most function applications are *saturated* (that is, they
pass the same number of arguments as the function definition expects),
the implementation of functions is optimised to make saturated
applications fast.

However, this optimisation requires knowing how many arguments a
function definition expects. Since this information is not currently
tracked in the parse tree, it must be guessed later, and sometimes
this guess is wrong. For example:

```ocaml
let hello ~person ~greeting ppf =
  Printf.fprintf ppf "%s, %s!\n" greeting person

type greeter = { greet : person:string -> unit } [@@unboxed]

let make_greeter ~greeting ppf =
  { greet = hello ~greeting ppf }
```

Here `make_greeter` is detected as a 3-argument function rather than a
2-argument one, because the partial application of `hello` results in
construction of a `Lambda.Lfunction` which is picked up by the
currying optimisation as though it were a third argument to
`make_greeter`. Saturated two-argument applications of this function
are therefore not optimised.

N-ary lambdas would make this issue go away, because the number of
parameters of a function is known rather than inferred.



## Parsetree changes

As well as the `fun a b c -> ...` syntax, OCaml additionally supports
matching on the final argument of a function using the `function`
keyword, allowing for instance the definition of a three-argument
function as:

```ocaml
let f a b = function
  | None -> a + b
  | Some c -> a + b + c
```

Additionally, a return type annotation may appear in a function:

```ocaml
fun (a : int) (b : int) (c : int) : int -> a + b + c
```

If the `function` keyword is being used to match on the final
argument, then the only place the return type annotation may legally
appear is just before the final argument:

```ocaml
fun a b : int option -> int = function
  | None -> a + b
  | Some c -> a + b + c
```

Finally, newtypes may appear interspersed with function arguments, as
in `fun a (type t) b -> ...`

So, the proposed representation for functions contains a sequence of
arguments (either function parameters or newtypes), followed by an
optional type annotation, followed by a body (either a single
expression or a case list introduced by `function`). The type
annotation types the body in either case.

The proposed addition to the parsetree is as follows:

```ocaml
type fun_body =
  | Fun_body of expression
  | Fun_cases of case list

and fun_argument =
  | Arg_simple of pattern
  | Arg_labelled of string * pattern
  | Arg_optional of string * pattern * expression option
  | Arg_newtype of string loc

and expression_desc =
  ...
  | Pexp_function of argument list * core_type option * function_body
  ...

```
