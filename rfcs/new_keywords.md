# New keywords for OCaml

New language features require new syntax, and often the best new
syntax involves one or more new keywords. However, adding keywords to
a language brings backwards-compatibility concerns, as programs that
use the keyword as an identifier stop working.

The aim of this RFC is to make it easier to add new keywords to
OCaml. Specifically, two small changes are proposed to the lexical
syntax:

  - a new class of "optional keywords", initially empty, which can be
    disabled by a command-line option or lexer directive. (If
    disabled, the keywords remain usable as identifier)

  - a new syntax of "raw identifiers", which allow words to be used as
    identifiers regardless of whether or not they are keywords.

With these changes, new keywords can be added as optional keywords,
disabled by default.  Old code works as normal, new code can opt in to
the keyword, and identifiers colliding with the keyword can still be
used as raw identifiers. Eventually, the new keyword can be enabled by
default, but old code continues to work with an explicit compiler flag.

## Goals and proposal

The main goal is to allow OCaml to be extended with new keywords in a
backwards-compatible way. Specifically:

  1. Old code should still be usable, even if it uses identifiers that
     now collide with keywords.

  2. New and old code should mix, even if the new code needs to refer
     to an old identifier which is now a keyword.

The proposal is in two parts:

  1. Add compiler options `-use-keyword foo`, `-no-keyword foo` and lexer
     directives `#use_keyword foo`, `#no_keyword foo` to enable and disable an
     optional keyword `foo`.

     If `foo` is not recognised as an optional keyword, then `-use-keyword foo`
     and `#use_keyword foo` are errors, while `-no-keyword foo` and `#no_keyword
     foo` are silently ignored. (Silently ignoring these allows compatibility to
     be maintained before and after an optional keyword is introduced).

  2. Add a new syntax `\#foo` called a **raw identifier**. This syntax is
     equivalent to a plain identifier `foo`, except that `\#foo` is always an
     identifier even if `foo` is a keyword.

     This syntax can be used anywhere that `foo` can be used: `~\#foo` is a
     labelled argument, `` `\#Foo`` is a polymorphic variant, `'\#foo` is a type
     variable, `\#Foo` is a module or constructor name, etc.

     (This feature is present in C# (`@foo`), Swift (`` `foo` ``) and Rust
     (`r#foo`). None of these syntaxes can be reused directly without conflicts,
     and the proposal here is closest to Rust's)

Part 1 of the proposal ensures that old code continues to work, although once a
new keyword is enabled by default old code will need to use the `-no-keyword
foo` flag to compile unmodified. Part 2 ensures that new code can refer to
identifiers exported by old code, even if they are now keywords.

## Alternative approaches

There are many ways to extend a language. Here are some possibilties, which seem
less preferable to the proposal above:

  - **Just add keywords**

    The traditional approach is to add keywords and accept some amount of
    breakage.  The last time this occurred (adding `nonrec` in 4.02) it
    generated [a long discussion](https://github.com/ocaml/ocaml/issues/6016) on
    whether it is acceptable to add a new keyword, even one that will break no
    known code. For keyword proposals that are known to break code
    (e.g. `effect`, `macro`, `implicit`, `unboxed`), this seems unworkable.

  - **Symbols instead of keywords**

    Instead of adding keywords, it is possible to introduce new syntax using
    entirely non-alphabetic characters. However, it's hard to read and look up
    unfamiliar symbolic syntax, and the space of remaining options is
    small as the OCaml lexical syntax is quite crowded. (For an example of this,
    see [RFC #10](https://github.com/ocaml/RFCs/pull/10), which further overloads
    the `#` symbol in types, trying to keep this separate from the various current
    meanings of `#`).

  - **Attributes instead of keywords**

    The `[@attribute]` syntax can be used to add arbitrary annotations to the
    parse tree, and has already been used for several new language features,
    including immediate types, unboxed types and explicit tail calls.

    It has a couple of disadvantages: first, the syntax is noiser than and
    inconsistent with existing keyword-based syntax. For example, a single-field
    record declaration may be annotated as any of mutable, private or
    unboxed. Two of these are a keyword, while one is spelled `[@@unboxed]`,
    where the distinction is based mostly on the date that the feature was
    introduced.

    Secondly, since attributes are valid anywhere, subtle bugs are possible if
    they end up on the wrong parsetree node. For instance, `type t
    [@@@immediate]` is silently accepted and declares a non-immediate type: the
    extra `@` in `@@@immediate` makes it a standalone annotation disconnected
    from `type t`, so that it gets parsed and ignored.

  - **Contextual keywords**

    Some languages (notably, C#) allow words to be used as identifiers yet be
    recognised as keywords in certain contexts, which provides a high degree of
    backwards compatibility at the expense of more complex parsing.

    However, there are two reasons why this approach is less effective in OCaml:
    first, OCaml distinguishes fewer contexts. In particular, the C# trick of
    making a word be a keyword in statement but not in expression context is not
    useful in a language that does not distinguish statements and expressions.
    Second, OCaml accepts a sequence of arbitrary space-separated identifiers as a
    function application, so it is harder to find a construction that does not
    already mean something.

  - **Overloaded keywords**

    It is tempting to reuse an existing keyword, by giving it a new meaning in a
    context which it cannot currently be used. While it does preserve
    compatibility, this is mostly a bad idea: for instance, see the various
    confusing meanings of `static` in C. In particular, this sort of keyword
    reuse removes the ability to easily look up the meaning of some syntax,
    which is one of the main reasons to use keywords in the first place.

  - **Multiple parsers**

    Finally, we could ship multiple versions of the parser that accepted
    different editions of OCaml's syntax. This does have certain advantages, but
    has an unusually high maintenance cost, and seems undesirable on that basis
    alone.