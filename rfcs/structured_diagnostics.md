# Structured diagnostics

A limitation of the current compiler diagnostics (error messages, warnings, time
profile information and other compiler developer debugging output) is that it
cannot be parsed easily and reliably by developer tools.

This means that not only those tools have the choice to either parse the
diagnostics produced by the compiler, which is brittle; or lose meaningful
information.

Consider for instance, a moderately large OCaml error message:
```ocaml
File "functor_example.ml", line 7, characters 11-15:
7 | module M = F(Y)
               ^^^^
Error: This application of the functor F is ill-typed.
       These arguments:
         Y
       do not match these parameters:
         functor (X : x) (Y : y) -> ...
       1. An argument appears to be missing with module type x
       2. Module Y matches the expected module type y
```
The internal compiler report data types for this error messages contains
one explicit location, one main message
```ocaml
Error: This application of the functor F is ill-typed.
       These arguments:
         Y
       do not match these parameters:
         functor (X : x) (Y : y) -> ...
```

and two submessages
```ocaml
       1. An argument appears to be missing with module type x
```

```ocaml
       2. Module Y matches the expected module type y
```
decorated by three distinct semantic tags.

All this information is lost when printed as a plain text. Even worse for tools,
the compiler is using `Format` to shape the output to adjust it to a 78-column
width and semantic tags to control the styling of the text through ANSI terminal
escape sequences. All this formatting is counterproductive for developer tools
that would rather render the error message in another text shape, with another
styling context.

The aim of this RFCs is to add structured diagnostics to the compiler as a way
to render the internal compiler ADT in any structured file formats and write
down which backward compatibility guarantee we will offer in the future.
For instance, if rendered to s-expressions, the following error message should look
like

```lisp
((metadata ((version (1 0)) (valid Full)))
((kind Report_error)
(main
  ((msg
     ((Open_box (HV 0)) (Text "This application of the functor ")
     (Open_tag Inline_code) (Text "F") Close_tag (Text " is ill-typed.")
     (Simple_break (1 0)) (Text "These arguments:") (Simple_break (1 2))
     (Open_box (B 0)) (Open_tag Preservation) (Text "Y") Close_tag Close_box
     (Simple_break (1 0)) (Text "do not match these parameters:")
     (Simple_break (1 2)) (Open_box (B 0)) (Text "functor")
     (Simple_break (1 0)) (Open_tag Insertion) (Text "(") (Text "X")
     (Text " : ") (Open_box (B 2)) (Text "x") Close_box (Text ")") Close_tag
     (Simple_break (1 0)) (Open_tag Preservation) (Text "(") (Text "Y")
     (Text " : ") (Open_box (B 2)) (Text "y") Close_box (Text ")") Close_tag
     (Simple_break (1 0)) (Text "-> ...") Close_box Close_box))
  ((file "functor_example.ml") (start_line 7) (stop_line 7)
  (characters (11 15)))))
(sub
  (((msg
      ((Tab_break (0 0)) Open_tbox (Open_tag Insertion) (Text "1")
      (Text ". ") Close_tag Set_tab (Open_box (HV 2))
      (Text "An argument appears to be missing with module type")
      (Simple_break (1 2)) (Open_box (B 0)) (Open_box (B 2)) (Text "x")
      Close_box Close_box Close_box Close_tbox)))
  ((msg
     ((Tab_break (0 0)) Open_tbox (Open_tag Preservation) (Text "2")
     (Text ". ") Close_tag Set_tab (Open_box (HV 2)) (Text "Module ")
     (Text "Y") (Text " matches the expected module type") (Text " ")
     (Open_box (B 2)) (Text "y") Close_box Close_box Close_tbox)))))
(quotable_locs
  (((file "functor_example.ml") (start_line 7) (stop_line 7)
   (characters (11 15)))))))
```

Moreover, this kind of output is useful beyond error messages. Currently, the
compiler already has debugging flags for outputting a textual representation of
the parsetree, typedtree, lambda IR, and more. All these flags could benefits
being cleanly separated rather than being mixed on stdout. Similarly, the output
of the REPL could be split by kind of contents, or the `-config` flag could
benefit being available in structured file formats.


## Requirements

Discussing with dune developers, the OCaml maintainers, and examining existing
compilers hook, we have identified five requirement:

1. Unobstrusive and short-circuitable

   We don't want to remove the current direct output that prints directly to the
   terminal. The construction of the structured diagnostic should be quite
   similar to the existing printing mechanism.
   
2. Versioned with a specified update policy

  If we want the compiler diagnostic to be usable without friction by non-internal tools, we
  should write down which kind of stability guarantee we offer, and have a clear update paths
  for future changes. For instance, one possibility to keep in mind would be the
  follow [SARIF](https://sarifweb.azurewebsites.net) specifications.

3. Independent of specific formats

  Some part of the OCaml ecosystem prefer S-expressions to JSON. Both format are
  quite similar when seen as text-based serialization format for the algebraic data
  types that already exist internally in the compiler. A good option here is to
  provide enough information for compiler-library users to implement their own
  serialization format.

4. Redirectable to specific files

  The compiler already provides `-dump-into-files` and `-dump-dir` to output
  debugging output to files. The compiler library makes it possible to redirect
  warnings independently of errors. The structured diagnostic implementation
  should be able to redirect fields.
  
5. Easily derivable specifications

  One way to make it easier for external tools to use compiler diagnostics is to
  ensure that external tools can easily derive specifications for the diagnostics
  by using compiler-libs.

## Versioning and subtyping rules for maintainability

An important requirement for structured compiler output is the ability to guarantee a
sufficient level of backward compatibility to other developer tools.

An easy first step is to add a metadata field to toplevel compiler diagnostics
which describes the version of the logger used to output the diagnostics:

```json
  "metadata" : { "version" : [1, 0] }
```

However, since we are constructing the structured diagnostics incrementally, we
cannot guarantee at build time that all constructed diagnostics will satisfy the
expected schema.

Thus, we propose to add a runtime check, and a metadata field which confirms
that the diagnostic is well typed

```json
  "metadata" : { "version" : [1, 0], "valid" : "Full"},
```

In case of invalid or deprecated diagnostics, we propose to report those invalid
fields. For instance, if the parsetree output was made compulsory by accident,
but ended up being absent in the actual diagnostic:

```json
  "metadata" : { "version" : [1, 0], "valid" : "Full", "invalid_paths" : [["debug", "parsetree"]] },
```


Beyond a version number, we should enforce a clear update policy for schamata
between. The policy that we propose is a data-oriented version of semantic
versioning for algebraic data types.

- minor updates can only refine schemata into a subtype of the previous schema
- major update can delete deprecated fields from record type or add new
  constructors to sum type.
- only one update of diagnostic versions by compiler minor compiler version.

Then a versioned and structured diagnostic can be seen as an algebraic data types annotated
with versioning and evolution annotation. For instance, for an hypothetical
future history of the `error_report` data type, we could have:

```ocaml
type error_report = 
  { kind: error_kind [@published (1,0)];
    main: msg [@published (1,0)];
    sub: msg list
    [@published (1,0)]
    [@deprecated (1,1)]
    [@deleted (2,0)]; 
   }
type error_kind =
  | Error of string [@published (1,0)]
  | Warning of
      { contents:string; number:int; name:string; as_error:bool }
      [@published (1,0)] [@expanded (1,1)]
  | Warning_as_error of string [@published (1,1)] [@deleted (1,1)]
  | Alert of
      { contents:string; as_error:bool }
      [@published (1,0)] [@expanded (1,1)]
  | Alert_as_error of string  [@published (1,0)] [@deleted (1,1)]
  | Fatal of string [@derived Error (1,1)][@published (2,0)]
...
```

With this policy, we can ensure that a logger at version `major.minor` can
output a structured diagnostic for any version `major.prev` where `prev<=minor`.
by coercing the diagnostic at version `major.minor` into its subtype at version
`major.prev`.

## Subtyping rules for minor updates

We propose two group of subtyping rules, one for variant, another for record
to account for their opposite polarity.

### Record subtyping rule

The subtyping rules for records are straightforward because we can
easily elide fields from future versions:

1. add new fields to a record type
2. promote an optional field to a required field
3. deprecate a field

The first rule only require to delete fields when coercing a newer version to an
older one and the last two rules only have effects on the schema of diagnostics
and do not affect coercion.

### Variant constructors subtyping rules

The constructor rules are more complex and can be divided in two parts, the dual
rule for record field addition

1. remove constructors from a sum type

and the derived constructor rules

2. expand the argument type of a variant constructor
3. introduce a new derived variant constructor

The first rule is straightforward, and correspond to an identity coercion. It is
also insufficient by itself. Indeed, this rule only allows us to remove unused
clutter, with no room to evolve sum types without a new major version.

A hackish solution do exist when the sum type appears inside a field of a record
type. In this case, we can

1. duplicate the sum type `s_old` as a separate copy `s_new`
2. update the fresh copy `s_new` with new constructors `c_new_k`
3. deprecate the `f_old` field containing the previous sum type
4. add a new field `f_new` containing the updated copy of the sum type.

Note that this works only if we can fill out the deprecated field `f_old` from
the new content in `f_new` in all circumstances.

In other words, this require us to have a projection between the new
constructors and the old ones. Then the new record field `f_new` essentially
contain a preview of the `s_new` type which will only become the official
version of `s` when a future major update will delete the `f_old` field and
`s_old` type.

Nevertheless, this complex dance has the disadvantage of requiring many fresh
names. To avoid this issue, we can import the core idea behind this scheme
directly in our subtyping rules: we may update a sum type in a backward
compatible way if we can approximate new constructors by older ones.

There are two cases that seem relatively straightforward to handle:

1. A constructor `c` exists in both `v1 < v2` with only a change of its argument type,
   with the argument new type being a subtype of the previous one.

For maximal backward compatibility, we could require that the `v2` argument type
is necessary a record, which contains a `contents` field holding the previous argument.


2. A new constructor `c2` in `v2>v1` can be approximated by an older constructor
`c1` in `v1`

With those possibility, the life cycles of variant constructors is then restricted to

1. derived inception
2. publication
3. expansion
4. deletion

For instance, one of the sum type presents in compiler diagnostics is the error kind,
which is represented as:

```ocaml
type report_kind =
  | Report_error
  | Report_warning of string
  | Report_warning_as_error of string
  | Report_alert of string
  | Report_alert_as_error of string
```

It seems possible in the future that we may want to expand the `Report_warning`
argument to contain the warning number, name and error status while deleting the
`Report_warning_as_error` constructor:


```ocaml
type report_kind =
  | Report_error
  | Report_warning of { contents:string; name:string; number:string; as_error:bool}
  | Report_alert of string
  | Report_alert_as_error of string
```

With the deletion and expansion rule, this change can be done in a minor update.


### Backward-compatibility beyond major versions

Going back to major version, when publish a new major version
of the diagnostics, we may

0. delete unused records and sum types
1. remove a deprecated field from a record type
2. add a new variant constructor to a variant type


With those changes, enforcing compatibility between major versions is not always
possible, in particular for the variant constructor side.
However, we can afford some long term guarantees.

First, to avoid reuse of previous names with different meaning, we must keep a
set of tombstones for deleted nominal types, record fields, and sum
constructors.

Second, in the record case, the specification of deleting only deprecated fields
means that we already keep producing deprecated fields for at least one OCaml
version.

It might be reasonable to keep deleted fields around for at least one OCaml
version, and for at most one major diagnostic version. With this rule, this
means that a freshly created record field would be available for at least one
year and half.


Third, backward-compatibility for variant constructor is more complex. If there
is a completely new constructor in version `major.0`, it is not always possible
to translate this new constructor to the diagnostic in the previous version. We
could replace future constructor by a special constructor `<future>`, but it is
unclear if this would help tools more than having an unknown constructor name.
Thus reliable deserializer that expect to work across major version would need
to parse variant sum as if they were always extensible.


Finally, in order to keep track of all version subtypes, we propose to keep an
append-only history of the diagnostic schemata. However, it seems sensible to
enforce this append-only property of this history at a meta-level by committing
the history of the diagnostic to the OCaml testsuite.
