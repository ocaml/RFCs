This RFC proposes extending `when` guards to allow additional pattern matching.
It also discusses possible extensions: `while` pattern guards; an infix `match`
expression; and multiway matches.

The first case of this match contains an example of the proposed syntax:

```ocaml
match expr with
| Literal x when Map.find x mapped match Some y -> y
         (* ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ *)
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

## Motivation

The example is adapted from ["The Ultimate Conditional Syntax", by Lionel
Parreaux][TUCS].

OCaml supports checking boolean conditions during a pattern match via `when` guards:

```ocaml
match expr with
| Literal x when Map.mem x mapped -> Map.find x mapped
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

However, it doesn't support binding additional variables in the
`when` guard. That means, in the words of [Parreaux][TUCS], there is no "local,
non-disruptive" way to refactor the above example to use `Map.find_opt`, even
though you might want to avoid the brittleness of calling `Map.mem` followed by
`Map.find`.

The most readily available refactor is non-local and involves moving out some
code into a helper function:

```ocaml
let process_value x = print_int x; process x in
match expr with
| Literal x -> begin
    match Map.find_opt x mapped with
    | Some x -> x
    | None -> process_value x
  end
| Add (0, x) | Add (x, 0) -> process_value x
```

This RFC makes a local, non-disruptive refactor available: a *pattern guard*,
which is a `when` guard that involves a possibly non-exhaustive pattern match of
an expression against a single pattern. If the pattern match succeeds, the case
body's environment includes any additional variables bound by the pattern. If
the pattern match fails, the outer pattern match continues to the next case,
just as with the existing boolean `when` guards.  In fact, existing boolean
`when condition` guards are exactly equivalent to `when condition match true`
pattern guards in this proposal.

## Syntax

Recall the example of the proposed syntax from the introduction:
```ocaml
match expr with
| Literal x when Map.find x mapped match Some y -> y
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

Nested `when` guards are permitted, where variables bound in earlier pattern
`when` guards (those with `match`) are available in later guards:

```ocaml
match expr with
| Literal x
    when Map.find x mapped match Some y
    when Map.find y mapped match Some z -> z
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

#### BNF

The old BNF allowed a single optional `when` guard per case:
```
case ::=
  | '|' pattern '->' expression
  | '|' pattern 'when' expression '->' expression
```

The new BNF allows nested `when` guards for a single case:

```
when_guard ::=
  | 'when' expression 'match' pattern
  | 'when' expression

case ::=
  | '|' pattern when_guard* '->' expression
```

#### Exception patterns

Here and throughout the RFC, the pattern on the RHS of the `match`
can be an `exception` pattern. Here's a contrived example of that:

```ocaml
type nat =
  | Zero
  | Succ of nat

exception Even
exception Odd

let rec raise_parity nat =
  match nat with
  | Succ pred when raise_parity pred match exception Even -> raise Odd
  | Succ _ | Zero -> raise Even
```

Allowing exception patterns is more compelling for multiway match, explored
later in the RFC, as there the pattern match can include value patterns
in some cases and exception patterns in other cases.

## Extensions

These extensions are not the main point of this RFC. The main point is the
`when` guards of matches. The extensions are included in this RFC to battle-test
the syntax, showing how it would look for various other constructs.

### Infix `match` expression

Consider a `when` guard like `expr match pat` on its own. This could sensibly be treated as an expression of type `bool`. (It doesn't make sense to talk about "binding patterns" in this context.)

For example, we can define `Option.is_some` as simply:

```ocaml
let is_some o = o match Some _
```

Here is a more contrived example that shows more aspects of the syntax:

```ocaml
type t = Foo of int | Bar of int | Baz of int

let check_conditions x f = x > 3 && f () match Foo _
```

This shows that we *can* treat such a construct as an expression, but should we?
One argument in its favor is that `x match Foo _` is more compact than `match x
with Foo _ -> true | _ -> false` in the common case where you only want to see
whether a value matches a certain constructor, as we saw with `is_some` above.
There are advantages to compactness: for instance, you're more likely to want to
include a compact expression in a compound conditional, and this removes the
need to define `is_CONSTRUCTOR` functions for variants.

Another argument in its favor is that it replaces many uses of polymorphic
equality.  Polymorphic equality is already used as a kind of simple pattern
matching in compound conditionals, where the pattern contains only
literals. (E.g., people can and do write `x = Some Bar`.)  The infix `match`
construct can be used instead of polymorphic equality (`x match Some Bar`), and
has some advantages:
  * Polymorphic comparison is at best a controversial feature of the language, so having a way of avoiding it in more cases is attractive.
  * `match` has some nice features on top of polymorphic equality; e.g., you can put underscores in uninteresting positions in the pattern.
  * Polymorphic comparison to non-immediates, like `Some Baz`, involves a call at runtime to `caml_equal` which is more expensive than a simple constant comparison.

`EXPR match PAT` is a drop-in replacement for many uses of polymorphic comparison, of
which there are [countless](https://github.com/ocaml/ocaml/blob/224c14c41089c12c2f2c1c2f91e88915f62604f5/typing/includemod_errorprinter.ml#L71)
[instances](https://github.com/ocaml/ocaml/blob/224c14c41089c12c2f2c1c2f91e88915f62604f5/typing/oprint.ml#L322) [in](https://github.com/ocaml/ocaml/blob/224c14c41089c12c2f2c1c2f91e88915f62604f5/typing/typedecl_variance.ml#L328) [the](https://github.com/ocaml/ocaml/blob/224c14c41089c12c2f2c1c2f91e88915f62604f5/typing/btype.ml#L157) [compiler](https://github.com/ocaml/ocaml/blob/224c14c41089c12c2f2c1c2f91e88915f62604f5/typing/parmatch.ml#L2035).

#### Warnings or errors

It should trigger an error or warning for `PAT` to bind variables in `EXPR match
PAT`, e.g. `e match Some x` suggests an error. We could use the existing unused variable
warning, introduce a new warning, or make this an error. I suggest the existing unused
variable warning, which means it would not be an error to write `e match Some _x`.

#### Precedence and associativity

`match` binds at a precedence just higher than `&&`. (See [the reference](https://v2.ocaml.org/manual/expr.html).) This lets you write:

```ocaml
foo >>= f match Some _ && bar match Baz _
```
and have it be parsed as:

```ocaml
((foo >>= f) match Some _) && (bar match Baz _)
```

`match` will naturally associate left, as its left operand is an expression and
it produces an expression. So, `3 match 3 match true` should be treated as
`(3 match 3) match true`.

Infix `match` would require parentheses to be used as the direct child of `when` guards (or
similar syntactic extensions):

```ocaml
match e1 with
| Some x when (f x match Some _) -> e2
```

Without parentheses, the `match` is interpreted as being part of the `when` guard,
which allows you to bind variables:

```
match e1 with
| Some x when f x match Some y -> e2
```

### `while` pattern guards

Like `match`, there's currently no local refactor that allows you to bind variables in the condition of a `while`.

Suppose you want to refactor this program to combine `has_next` and `get` into a single function that returns an option, `consume`:
```ocaml
while has_next () do
  let x = get () in
  handle x
done
```

You can't do such a refactor locally, and instead you'd rewrite your program to use a recursive helper:
```ocaml
let consume () = if has_next () then Some (get ()) else None

let rec loop () =
  match consume () with
  | None -> ()
  | Some x -> handle x
in
loop ()
```

`while` patterns guards give a more local refactoring solution:

```ocaml
while consume () match Some x do
  handle x
done
```

#### Simple form

BNF:

```
expression ::=
  ...
  | 'while' expression 'do' expression 'done' (* existing form *)
  | 'while' expression 'match' pattern 'do' expression 'done' (* new form *)
  ...
```

#### Nested form

We could also extend `while` pattern guards to support nested guards, just as we
can for ordinary `when` guards.

BNF, extending the simple form:

```
expression ::=
  ...
  | 'while' expression 'match' pattern when_guard+ 'do' expression 'done'
  | 'while' expression when_guard+ 'do' expression 'done'
  ...
```

There are two productions in order to allow nesting and combining boolean
expressions/pattern guards in any order. This means you could write something
like:

```ocaml
while !should_continue
 when consume () match Some x
 when first_condition x
 when consume () match Some y
 when second_condition x y
do
  handle x y
done
```

### `if` pattern guards

Adding pattern guards to `if` is a logical extension of this RFC. But, as
discussed in [#6685](https://github.com/ocaml/ocaml/issues/6685), it's probably not a good fit for OCaml. It's already
non-disruptive and local to refactor an `if` into something that binds variables
(a `match`). This stands in contrast to the other parts in this RFC, which newly
make a non-disruptive, local refactor available for constructs like `when`
guards and `while`.

Nonetheless, it may be worth comparing how this could look to the other aspects
of this RFC, because it's so similar.  The instantiation of the above scheme for
`if` expressions would be something like:

```ocaml
if consume () match Some x
then handle x
else failwith "No elements"
```

Here is how you could write the same logic using a `match`,
which encourages exhaustive matching in a way that `if` doesn't:

```ocaml
match consume () with
| Some x -> handle x
| None -> failwith "No elements"
```

#### Simple form

BNF:

```
expression ::=
  ...
  | 'if' expression 'then' expression 'else' expression (* existing form *)
  | 'if' expression 'match' pattern 'then' expression 'else' expression (* new form *)
  ...
```

#### Nested form

As for `while`, we could also extend `if` pattern guards to support nested guards.

BNF, extending the simple form:

```
expression ::=
  ...
  | 'if' expression 'match' pattern when_guard+ 'then' expression 'else' expression
  | 'if' expression when_guard+ 'then' expression 'else' expression
  ...
```

For example:

```ocaml
if   consume () match Some x
when consume () match Some y
when consume () match Some z
then
  handle x y z
else
  failwith "Fewer than 3 elements"
```

### Multiway matches

So far, this RFC has limited pattern guards to only allowing one case. But this
leads to some awkwardness:

```ocaml
match foo with
| Foo f when List.find l f match Some true -> E1
| Foo f when List.find l f match Some false -> E2
| Foo f -> (* [List.find l f] is None *) E3
```

The above example is duplicative and wasteful: `List.find l f` runs twice. We
could extend the guard syntax so that you'd be able to write two cases in the
inner `match`, and still fall through to the outer `match` if they didn't match:

```ocaml
match foo with
| Foo f when List.find l f match (
    | Some true -> E1
    | Some false -> E2
  )
| Foo f -> (* [List.find l f] is None *) E3
```

This is a generalization of the single-case `match` guard that's the basis of this RFC:
  * The matched expression in the guard (here, `List.find l f`) is evaluated to a value and
    takes the first case of the inner `match` whose pattern matches that value.
  * If there is no such case, then we consider this case of the outer `match` (`Foo f when List.find l f ...`) to *not* be matched, and so continue on to the next case (`Foo f -> E3`).

BNF:

```ocaml
case ::=
  | pat '->' expr
  | pat 'when' expr '->' expr
  | pat 'when' expr 'match' optional_bar(case)
  | pat 'when' expr 'match' '(' optional_bar(case_list) ')'

(* Something like [optional_bar] already exists; I'm reproducing it for clarity *)
optional_bar(x) ::=
  | x
  | '|' x

(* Something like [case_list] already exists; I'm reproducing it for clarity *)
case_list ::=
  | case
  | case '|' case_list
```

Intuitively, the `match` interior to the guard binds very tightly, and so only grabs the
first case unless the cases are wrapped in parens.

Multiway matches are relevant only for `match` pattern guards and don't interact with
the other extensions (`if`, `while`, infix `match` expression). These other extensions
don't have an immediately-obvious place for branching to occur, and even if they did,
the syntax wouldn't be as familiar as the `->` available for `match`.

## A note on extensions

Here is a dependency tree of the core ideas of this RFC:

  * `match` pattern guards
    * nested `match` pattern guards
    * multiway `match` pattern guards
  * `if` pattern guards
    * nested `if` pattern guards
  * `while` pattern guards
    * nested `while` pattern guards
  * infix `match` expression

Implementing a child demands implementing its parent, but places no such demand
on siblings or children.

This RFC shows how all the pieces could in theory fit together but also claims
independence of these extensions. The community might just want, say, `match` pattern guards.

## Semantics

The below semantics are for non-nested multiway matches, which subsume
single-way matches; in the single-way case, the case list is simply required to
be a singleton.

I'll give the semantics as an interpreter written in OCaml.

A `match` expression now looks like

```
match <expr> with
| <case>_1
| ...
| <case>_n
```

where

```ocaml
type case = pat * guarded_rhs

and guarded_rhs =
  | Simple of expr                   (* <pat> -> <expr> *)
  | Boolean_when of expr * expr      (* <pat> when <expr> -> <expr> *)
  | Pattern_when of expr * case list (* <pat> when <expr> match <case list> *)
```

A value successfully "matching" a case will bind some variables and select an
expression body to run. A value will match against the first matching case of
a case list:

```ocaml
(* [find_matching_case ctx value cases] will return:
    - [None] if no cases match.
    - [Some (ctx', expr)] for the first matching case, where [expr] is the
      body of the case and [ctx'] is formed by adding the variables bound
      by the successful match to [ctx].
*)
val find_matching_case : context -> value -> case list -> (context * expr) option
```

Suppose we're given some simple primitives:

```ocaml
(* Match a value against a pattern. Adds the bound variables to the
   input context to form the output context.
*)
val try_match_pat : context -> value -> pat -> context option

(* Evaluate an expression to a value *)
val eval : context -> expr -> value

(* Check whether a value is the bool [true]. *)
val as_bool : value -> bool
```

An implementation of `find_matching_case` can be given fairly easily:

```ocaml
let rec try_match ctx value ((pat, guarded_rhs) : case) =
  match try_match_pat ctx value pat with
  | None -> None
  | Some ctx ->
    begin match guarded_rhs with
      | Simple body -> Some (ctx, body)
      | Boolean_when (condition, body) ->
          let condition = eval ctx condition in
          if as_bool condition then Some (ctx, body) else None
      | Pattern_when (scrutinee, cases) ->
          let scrutinee = eval ctx scrutinee in
          find_matching_case ctx scrutinee cases
    end

and find_matching_case ctx value (cases : case list) =
  List.find_map (fun case -> try_match ctx value case) cases
```

Or, we can avoid the helper function if we use the multiway version of the new
syntax:

```ocaml
let rec find_matching_case ctx value cases =
  match cases with
  | [] -> None
  | (pat, guarded_rhs) :: _
    when try_match_pat ctx value pat match Some ctx
    when guarded_rhs match (
      | Simple body -> Some (ctx, body)
      | Boolean_when (condition, body)
          when as_bool (eval ctx condition)
        -> Some (ctx, body)
      | Pattern_when (scrutinee, cases)
          when find_matching_case ctx (eval ctx scrutinee) cases match Some _ as result
        -> result
    )
  | _ :: cases -> find_matching_case ctx value cases
```

## Alternative syntaxes

I considered some other syntax options and deemed them either more confusing or
less amenable to future changes than `match`:

  * `'when' 'let' pattern '<-' expression '->' expression`. This is similar to
    Rust's `if let`. But this syntax doesn't work well in some ways: there are
    issues with the obvious alternative to infix `match` (`<pattern> <-
    <expression>` is ambiguous with instance variable update), and there's no
    clear way to extend this syntax to multiway matches.
  * `'when' 'let' pattern '=' expression '->' expression`. Discarded because `=`
    looks like a total match, but we explicitly want to allow partial matches
    here.
  * `'when' pattern '<-' expression '->' expression`. Discarded for the same
    reason as `when let <-`.

I also considered the syntax explored in ["The Ultimate Conditional
Syntax"][TUCS] (TUCS). Here's an example program in the ML-Script programming
language:

```
if name.startsWith("_")
  and name.tailOption is Some(namePostfix)
  and namePostfix.forall(..isDigit)
  and namePostfix.toIntOption is Some(index)
  and index <= arity and index > 0
then Right of (index, name)
else Left(name)
```

and what it would correspond to in this RFC's syntax:

```ocaml
if starts_with name "_" then
  match tail_option name with
  | Some name_postfix
      when String.for_all is_digit name_postfix
      when to_int_option name_postfix match Some index
      when index <= arity && index > 0
      -> Right (index, name)
  | _ -> Left name
else Left name
```

The TUCS syntax is whitespace sensitive (it uses the "off-side" rule) and
more aggressively unifies if-condition syntax and match syntax.

I see the benefit of the proposed `when ...  match ...` syntax
over the TUCS syntax to be that it's a addition to an existing construct
rather than an entirely new construct. In particular, `when ... match ...`
guards can be added incrementally to existing `match` expressions without
needing to change the text of unaffected cases.

## Prior art

  * [Lionel Parreaux's "The Ultimate Conditional Syntax"][TUCS] develops similar ideas with an alternate syntax.
  * Wrapping match cases with some sort of bracket was first proposed in [#715](https://github.com/ocaml/ocaml/pull/715) and PR'ed in [#722](https://github.com/ocaml/ocaml/pull/722). [#722](https://github.com/ocaml/ocaml/pull/722) was closed in favor of [#716](https://github.com/ocaml/ocaml/pull/716), but this PR was also closed. Reception of bracketed cases was mixed &mdash; some people encouraged just wrapping the entire `match` expression in parentheses.
  * [Haskell has pattern guards](https://www.haskell.org/onlinereport/haskell2010/haskellch3.html#x8-460003.13). It uses `,` instead of `when` as a separator for nested guards. Its syntax places patterns to the left of the scrutinized expression &mdash; I think this makes multiway guards much less attractive in this context.
  * [Rust](https://doc.rust-lang.org/rust-by-example/flow_control/if_let.html) and Swift have an "if-let" construct that's similar to the `if` pattern guard proposed here.

[TUCS]: https://www.cse.ust.hk/~parreaux/ucs_v1.2.pdf
