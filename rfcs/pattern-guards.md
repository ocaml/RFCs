This RFC proposes extending `when` guards to allow additional pattern matching.

The first case of this match contains an example of the proposed syntax:

```ocaml
match foo with
| Foo f when List.find l f match (
    | Some true -> E1
    | Some false -> E2
  )
| Foo f -> (* [List.find l f] is None *) E3
```

It's worth highlighting an example of the syntax in the common
situation where the pattern guard has only has a single case, as this
doesn't require parentheses around the cases:

```ocaml
match expr with
| Literal x when Map.find x mapped match Some y -> y
         (* ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ *)
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

## Motivation

The example is adapted from ["The Ultimate Conditional Syntax", by Lionel
Parreaux][UCS].

OCaml supports checking boolean conditions during a pattern match via `when` guards:

```ocaml
match expr with
| Literal x when Map.mem x mapped -> Map.find x mapped
        (*  ^^^^^^^^^^^^^^^^^^^^^
              when guard
         *)
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

However, it doesn't support binding additional variables in the
`when` guard. That means, in the words of [Parreaux][UCS], there is no "local,
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
which is a `when` guard that involves a (usually) non-exhaustive pattern match of
an expression against a series of cases. If the pattern match succeeds against a
case's pattern, the case's body is executed in an environment that includes any
additional variables bound by the pattern. If the pattern match fails against
all cases' patterns, then fallthrough occurs: the outer pattern match continues
to the next case, just as with the existing boolean `when` guards. In fact,
existing `p when condition -> e` cases are exactly equivalent to `p when
condition match true -> e` cases in this proposal.

## Terminology

`when e` is an example of a guard. `when e1 -> e2` is a guard plus a case body.
We'll call this latter construct a "guarded right-hand side", or "guarded RHS".

It's most correct to think of this RFC as proposing pattern-guarded
right-hand sides ("pattern-guarded RHSes") and not "pattern guards".
That's because the construct in this proposal includes (possibly multiple)
case bodies.

## Syntax

The old BNF allowed a single optional `when` guard per case:

```
expression ::=
  ...
  | 'match' expression 'with' optional_bar(case_list)
  | 'function' optional_bar(case_list)

case ::=
  | pattern '->' expression
  | pattern 'when' expression '->' expression

(* Something like [optional_bar] already exists; I'm reproducing it for clarity *)
optional_bar(x) ::=
  | x
  | '|' x

(* Something like [case_list] already exists; I'm reproducing it for clarity *)
case_list ::=
  | case
  | case '|' case_list
```

The new BNF replaces the definition of `case` to allow pattern matching of a
scrutinee against still more `case`s:

```
case ::=
  | pat guarded_rhs

guarded_rhs ::=
  | '->' expr
  | 'when' expr '->' expr
  | 'when' expr 'match' optional_bar(case)
  | 'when' expr 'match' '(' optional_bar(case_list) ')'
```

### Example: single-case pattern guards

In the circumstance where the pattern-guarded RHS has a single case, the
parentheses may be omitted. This means that, without needing any additional
syntax beyond what's outlined in the BNF, users can effectively write a "pattern
guard" that's analogous to the existing `when e` "boolean guard" construct:

```ocaml
match expr with
| Literal x when Map.find x mapped match Some y -> y
              (* ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                      "pattern guard"
              *)
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

It similarly doesn't require additional machinery to permit nested
pattern guards, where variables bound in earlier pattern
guards are available in later guards:

```ocaml
match expr with
| Literal x
    when Map.find x mapped match Some y
    when Map.find y mapped match Some z -> z
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

To make it clearer how this corresponds to the BNF in the earlier section,
I'll write the equivalent program, but adding the optional vertical bar and
surrounding the case with parens:

```ocaml
match expr with
| Literal x when Map.find x mapped match (
    | Some y when Map.find y mapped match (
        | Some z -> z
      )
  )
| Literal x | Add (0, x) | Add (x, 0) -> print_int x; process x
```

### Example: Exception patterns

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

### Example: Why multiple cases?

You could imagine a version of this RFC that omits the `'when' expr 'match' '(' optional_bar(case_list) ')'` production of the `guarded_rhs` rule. This would be
tantamount to allowing only a single case in guarded RHS. You might be tempted
to view this as a simplification or as a proving ground for a basic version of
the feature before we consider adding the multiple-case version. However, my
stance is that the multiple-case version is essential to this feature.

A major pitfall of the single-case version is that it would encourage a
programming style where the user would write the same expression multiple times
to match it against different patterns:

```ocaml
(* Bad example; don't write this! *)
match foo with
| Foo f when List.find l f match Some true -> E1
| Foo f when List.find l f match Some false -> E2
| Foo f -> (* [List.find l f] is None *) E3
```

The above example is duplicative and wasteful: in the most obvious semantics,
`List.find l f` would run twice.

That's why this RFC centers the ability to write multiple cases for an
expression scrutinized in a pattern-guarded RHS:

```ocaml
match foo with
| Foo f when List.find l f match (
    | Some true -> E1
    | Some false -> E2
  )
| Foo f -> (* [List.find l f] is None *) E3
```

## Semantics

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

Or we can define it in the new syntax (sorry, we couldn't help ourselves):

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

## Prior art

  * [Lionel Parreaux's "The Ultimate Conditional Syntax"][UCS] develops similar ideas with an alternate syntax.
  * Gabriel Scherer started a discussion on related ideas in a [2019 blog post][gasche]. This RFC is influenced by ideas developed there.
  * Work has been done on pattern guards for Successor ML, [see here](https://github.com/JohnReppy/compiling-pattern-guards). The repo has some detail about a compilation scheme.
  * Wrapping match cases with some sort of bracket was first proposed in [#715][715] and PR'ed in [#722][722]. [#722][722] was closed in favor of [#716][716], but this PR was also closed. Reception of bracketed cases was mixed &mdash; some people encouraged just wrapping the entire `match` expression in parentheses.
  * [Haskell has pattern guards](https://www.haskell.org/onlinereport/haskell2010/haskellch3.html#x8-460003.13). It uses `,` instead of `when` as a separator for nested guards. Its syntax places patterns to the left of the scrutinized expression.

[UCS]: https://www.cse.ust.hk/~parreaux/ucs_v1.2.pdf
[gasche]: https://discuss.ocaml.org/t/musings-on-extended-pattern-matching-syntaxes/3600
[722]: https://github.com/ocaml/ocaml/pull/722
[715]: https://github.com/ocaml/ocaml/pull/715
[716]: https://github.com/ocaml/ocaml/pull/716

## Exhaustivity and fallthrough

In this RFC, the cases in a pattern-guarded RHS can be non-exhaustive (and, in
fact, in all of the examples so far, they are non-exhaustive). In the case that
the scrutinee matches none of the cases, pattern matching "falls through" to the
outer match.

There is a question of whether a warning should be raised in either of these cases:
  - the inner cases are exhaustive.
  - the inner cases are *non*-exhaustive.

The discussion on [gasche's blog post][gasche] considers some arguments. I expect
this is something we will continue to discuss in the RFC comments. In the meantime,
I'll summarize the existing arguments:

  - (In favor of warning for exhaustive matches.) The essential feature of pattern
    guards is that their cases are *non*-exhaustive; otherwise, you could just write
    a normal `match` expression. Writing an exhaustive pattern guard is using *too*
    expressive of a feature than is required, which might be suggestive of a programming
    error. There is analogy between this and the warning that's raised on `let
    rec f () = ...` where `f` is used nowhere in its definition.
  - (In favor of warning for *non*-exhaustive matches.) It's a common programming error
    to write a non-exhaustive match where you intend to match exhaustively.

## Alternative syntaxes

As always, there are multiple options for concrete syntax. We weighed
several options against the one ultimately proposed in this RFC.

### `'when' 'let' pattern '<-' expression '->' expression`

Here's what this syntax would look like:

```ocaml
let add_lookup env val0_opt var1 var2 =
  match val0_opt with
  | Some val0
      when let Some val1 <- lookup env var1
      when let Some val2 <- lookup env var2
        -> val0 + val1 + val2
  | _ -> 0

```
This syntax requires less looking ahead to see whether a guard is a pattern
guard or a boolean guard. Simply: Does it start with `when let`? And while it's
true that expressions can start with `let`, in my experience it's more common
for boolean guards to be short expressions (i.e. not let-expressions), and
regardless you can look a little further ahead for a `<-` vs. a `=` to
distinguish a pattern guard from a let-expression.

The main downside with this syntax is that it's not
obvious how to allow multiple cases in a pattern-guarded RHS.

Similarly, I discarded these options:
  * `'when' 'let' pattern '=' expression '->' expression`. (Also, `=` looks like a total match, but we explicitly want to allow partial matches here.
  * `'when' pattern '<-' expression '->' expression`. (Also, `<pattern> <- <expression>` is ambiguous with instance variable update. While we could get around this in the parser, it's still potentially conceptually ambiguous for the user.)

### Adding a new keyword, like `matches`

`match` is already a keyword. This has a benefit: it's a choice that's
backward-compatible with existing code. But this repurposing of an existing
keyword might be confusing.

Grammatically, `matches` would read better than `match`:

```ocaml
match x with
| Some y when e1 matches P1 -> e2
```

But, I would guess that the standard for adding a new keyword is rather
high.

### Ideas from [gasche's blog post][gasche]

Many ideas were proposed in the blog post. One of the more popular was:

```ocaml
let f y =
  match x with
  | Some x' and lookup env y with Some true -> x'
  | Some x' and lookup env z with (
      | Some true -> x' + 1
      | Some false -> x' + 2
    )
  | _ -> 0
```

Effectively, it's the same as this RFC's proposal, except using `and` instead of
`when` and using `with` instead of `match`.

There are a few other proposals in the blog post for pattern guards that are the
same as this RFC, modulo keywords. To be exhaustive, here's all of the proposals
for what can be filled in as `<case>` in `match e with <case>`.

```
p and e with p -> e
p and e as p -> e
p and match e with p -> e
```

In contrast, this RFC proposes `p when e match p -> e`.

One benefit I see of `p when e match p` over all of these other choices is that it uses
`when`, which is a familiar keyword for other kinds of guards with fallthrough.

But I think the RFC proposal compares favorably to each choice individually as well:
  * `p and e with p -> e`. This sequence of keywords reads fairly naturally in the larger context of `match e with p and e with p -> e`. But it doesn't read as nicely in `function` cases, which, unlike `match`, don't already involve the `with` keyword: `function p and e with p -> e`. In contrast, `function p when e match p -> e` is not as surprising, as functions already support `when` guards.
  * `p and e as p -> e`. I found this proposal less compelling, as the existing `as` keyword isn't as strongly associated with multiple cases of pattern matching in the same way that `with` or `match` is. (That is, I find `with` or `match` to more strongly suggest a pattern match.)
  * `p and match e with p -> e`. This is syntactically heavy compared to `p when e match p -> e`: it uses an extra keyword.

## Comparisons with other languages

I'll start by defining some terms by example. This terminology is consistent with
the "Terminology" section.

In `match e with p when e' match p1 -> e1`, we say:
  * `when e' match p2 -> e2` is a guarded RHS, in particular a pattern-guarded RHS
  * `e'` is the guard scrutinee
  * `p1` is the guard pattern
  * `e2` is a case body
  * Because the pattern-guarded RHS has only a single case, if you squint, `when e' match p2` is morally a "pattern guard" that behaves analogously to existing boolean guards `when e'`. (This point is admittedly confusing, but I mention it because some languages, like Haskell, don't support multiple cases in pattern-guarded RHSes, so really do have a well-defined notion of pattern guard.)

I'll argue that it's important for the guard pattern and the case body to appear on the same side (here, the right side) of the guard scrutinee. That's because, without this property, it's hard to imagine what a multi-case pattern-guarded RHS would look like: there are always exactly as many guard patterns as case bodies. We even have a name for a guard pattern together with a case body: a "case".

I'll use program as a running example:

```ocaml
let add_lookup env val0_opt var1 var2 =
  match val0_opt with
  | Some val0
      when lookup env var1 match Some val1
      when lookup env var2 match Some val2 ->
        val0 + val1 + val2
  | _ -> 0
```

**Form of "pattern guard":** `'when' expr 'match' pat`\
**Form of boolean guard:** `'when' expr`\
**Form of single-case-only guarded RHSes:** `guard+ '->' expr`\
**Can pattern-guarded RHSes have multiple cases?** Yes.

### Haskell

There is a directly analogous feature in Haskell. Here's what the above program
would look like:

```haskell
addLookup env val0_opt var1 var2 =
  case val0_opt of
    Just val0
      | Just val1 <- lookup env var1
      , Just val2 <- lookup env var2
      -> val0 + val1 + val2
    _ -> 0
```

**Form of pattern guard:** `pat '<-' expr`\
**Form of boolean guard:** `expr`\
**Form of single-case-only guarded RHS:** `'|' guard+ '->' expr` (where guards are separated by `,`)\
**Can pattern-guarded RHSes have multiple cases?** No.

This syntax has a major flaw. It is difficult to imagine what the syntax for a
multi-case pattern-guarded RHS could be, as the guard pattern in Haskell appears
to the left of the guard scrutinee but the case body appears to the right.
@goldfirere tells me that there has been desire in the Haskell community for a
multi-case pattern-guarded RHS, but that the existing syntax for pattern guards has
stymied efforts.

Haskell's notion of guard can be used in more situations than OCaml guards, e.g.
in function definitions, which means you can write things like:

```haskell
addLookup env var1 var2
  | Just val1 <- lookup env var1
  , Just val2 <- lookup env var2
  = val1 + val2
  | otherwise = 0
```

Guarded function definitions aren't available in OCaml. But, this is a property
of where guards are allowed to appear in OCaml rather than a property of what
syntactic constructs are allowed to appear *in* guards, and so is independent of
this RFC.

### Successor ML

See [this work](https://github.com/JohnReppy/compiling-pattern-guards/blob/master/ml19-paper.pdf).

```sml
fun add_lookup env val0_opt var1 var2 =
  case val0_opt of
    Some val0
      with Some val1 = lookup env var1
      with Some val2 = lookup env var2 =>
        val0 + val1 + val2
  | _ => 0
```

**Form of "pattern-guarded pattern":** `pat 'with' pat '=' expr`\
**Form of "boolean-guarded pattern":** `pat 'if' expr`\
**Can pattern-guarded RHSes have multiple cases?** No.

The syntax extends the *pattern* grammar with `p with p' = e` (and `p if e`),
rather than extending the case grammar as is done in this RFC. On its own,
this approach has the same downside as the Haskell syntax, where the guard
scrutinee may only correspond to a single case body and so encourages
duplicating the guard scrutinee in multiple cases.

It is worth considering the idea of attaching `when` to patterns instead of
cases, and [indeed some discussion in gasche's blog post was centered on that][gasche].
As noted in that thread, implementing this idea in OCaml would require an
intensive rework of the pattern match compiler. I claim that this is
future work that is compatible with the ideas in this RFC, but that isn't
an essential part of this RFC, so I move to leave this idea aside for
purposes of this RFC.

### MLScript (UCS)

I also considered the syntax explored in ["The Ultimate Conditional
Syntax"][UCS] (UCS). Here's an example program in the ML-Script programming
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

The UCS syntax is whitespace sensitive (it uses the "off-side" rule) and
more aggressively unifies if-condition syntax and match syntax.

I see the benefit of the proposed `when ...  match ...` syntax
over the UCS syntax to be that it's a addition to an existing construct
rather than an entirely new construct. In particular, `when ... match ...`
guards can be added incrementally to existing `match` expressions without
needing to change the text of unaffected cases.

### Dependently typed languages: Agda and Idris

Agda and Idris are dependently typed languages that feature a multi-case
pattern-guarded RHS. My sense is that the syntax ideas from these languages
don't directly transfer to OCaml, largely because their constructs are strongly
tied to multi-clause, multi-parameter function definitions, which OCaml lacks.

The two languages' approaches are similar. Both are inspired by [views in Epigram](https://doi.org/10.1017/S0956796803004829).

Disclaimer: I have not really worked with these languages. If people disagree
with my conclusions, I warmly welcome corrections and opposing ideas.

#### [Idris views](https://docs.idris-lang.org/en/latest/tutorial/views.html#footnote-reference-1)

Idris gives you a pattern guard form that is similar syntactically to adding
another argument to the function clauses that are under the guard:

```idris
natToBin : Nat -> List Bool
natToBin Z = Nil
natToBin k with (parity k)
   natToBin (j + j)     | Even = False :: natToBin j
   natToBin (S (j + j)) | Odd  = True  :: natToBin j
```

To an OCaml programmer, this may look odd as the `natToBin` function name
appears multiple times in the definition, including under the `with (parity k)`
guard. But this is standard in Idris: it's typical to define a function
with multiple clauses that, taken all together, exhaustively pattern match
the function's inputs.

The inner function clauses allow for more pattern matching on the parameter named
`k` in the earlier case: those are the `(j + j)` and `S (j + j)` patterns. These function
clauses are known to be exhaustive because of the relationship between those
patterns and the `Even` and `Odd` guard patterns.
This is useful in Idris because:

> The value of `parity k` affects the form of `k`, because the result of `parity k` depends on `k`.

The idea of compact syntax for re-pattern-matching on outer scrutinees from an inner
pattern match is absent from this RFC. I don't think it's as useful in OCaml
for several reasons:
  * In Idris and Agda, this construct is only available in multi-clause
    definitions of multi-argument functions, and OCaml doesn't have this notion.
  * This idea features more prominently in Idris and Agda because they're
    dependently typed.
  * It's true in OCaml that a pattern match on a GADT constructor refines
    the type information available in the scope of the pattern. But, regardless,
    you can write another inner pattern match to take advantage of the new type
    information you learned from the match on the GADT constructor:

```ocaml
type t = Foo | Bar
type u = Baz | Qux

type _ tag =
  | T : t tag
  | U : u tag

let go (type a) (opt : a option) (tag : a tag) =
  match opt with
  | Some x when tag match (
      | T (* here we know that [a = t] *) when x match Foo -> 3
    )
  | _ -> 4
```

That is, the available syntax is already compact enough. I would argue that
omitting this particular feature from OCaml does not decrease expressiveness.

#### [Agda `with` abstraction](https://agda.readthedocs.io/en/v2.5.2/language/with-abstraction.html)

The construct seems rather similar to Idris's construct.

Like this RFC, you can nest guarded RHSes:

```agda
compare : Nat → Nat → Comparison
compare x y with x < y
compare x y    | false with y < x
compare x y               | false = equal
compare x y               | true  = greater
compare x y    | true = less
```

and you can inspect multiple expressions at once (which you would accomplish
in this RFC by scrutinizing a tuple):

```agda
compare : Nat → Nat → Comparison
compare x y with x < y | y < x
compare x y    | true  | _     = less
compare x y    | _     | true  = greater
compare x y    | false | false = equal
```

Agda is whitespace-sensitive, so many of the syntax ideas don't transfer
directly to OCaml. The syntax is very much tied to multi-clause function
definitions, like Idris's.

## Invitation to weigh in

Comments are warmly welcomed on the corresponding RFC PR.
