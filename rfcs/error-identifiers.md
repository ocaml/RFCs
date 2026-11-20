# Short identifiers for error messages

## Motivation

We propose to assign stable, short identifiers to each error produced
by the OCaml compiler, and include this short name in the error
message. This makes it much easier to refer to those errors message in
documentation, user questions / answers, and to search on the web for
material on a specific error.

This feature was inspired by Rust which does precisely this:

- Example rust error message:

   ```
   error[E0268]: `break` outside of a loop
    --> src/lib.rs:2:5
     |
   2 |     break rust;
     |     ^^^^^^^^^^ cannot `break` outside of a loop
   ```

  The error identifier is `E0268` here.

- Index of all errors:
  https://doc.rust-lang.org/error-index.html


## Imaginary example

Current behavior:

```
# let x = 2 + "foo";;
Error: This expression has type string but an expression was expected of type
         int
```

New behavior:

```
# let x = 2 + "foo";;
Error[tyco007]: This expression has type string but an expression
  was expected of type int
```

In this example the error identifier is `tyco007`.

## Proposed identifier format

(We are of course happy to change this if there is demand.)

We propose messages such as "tyco007" formed of a "group", the prefix
"tyco", and a number, 7.

We propose to keeep a fixed length for messages, 7 characters,
following the Rust approach. (A guess about the motivations: it makes
formatting more consistent and search easier.) Currently the format we
use has one group per compiler module emitting errors (hre "tyco" is
short for "typecore"), but one could move to E123456 later in the
future using a single group "E".

```ocaml
type error_identifier = {
  group: string; (* 4 characters *)
  num: int; (* 0 <= num < 1000 *)
}
```

Note that the format proposed here is very different from the format
of warning identifiers (`missing-record-field-pattern`). Users want to
refer to specific warnings in their build system and code to
enable/disable them, so we want descriptive, human-friendly
identifiers. In constrat we don't know of a reason to refer to
a specific error in our code, so error identifiers need not be
readable and informative; we want them short and to the point.

(Why the group prefix then? Actually it simplifies the implementation
to give per-module identifiers instead of trying to pick global
numbers and avoiding conflicts between two errors trying to use the
same number. It might also help users identify classes of errors.)


## Impact on the compiler implementation

An important question with this proposal is the maintainability impact on the compiler codebase. We sketched an implementation to see how invasive it would be.

Currently our error-printing logic has code looking as follows, in each module reporting new errors:

```ocaml
let report_error ~loc env = function
  | Constructor_arity_mismatch(lid, expected, provided) ->
      Location.errorf ~loc
       "@[The constructor %a@ expects %i argument(s),@ \
        but is applied here to %i argument(s)@]"
       longident lid expected provided
  | [..]
```

We assume a `identifier_of_error : error -> Location.error_identifier` in each such module,
so the change to `report_error` would be minimal:

```ocaml
let report_error ~loc env err =
  let identifier = identifier_of_error err in
  match err with
  | Constructor_arity_mismatch(lid, expected, provided) ->
      Location.errorf ~loc ~identifier
       "@[The constructor %a@ expects %i argument(s),@ \
        but is applied here to %i argument(s)@]"
       longident lid expected provided
  | [..]
```

### Assigning identifiers to error values

This cannot easily be done automatically, because the error number
must be stable, even if the error type changes (a constructor is
renamed, reordered, split in two, several constructors are factored
into one...).

We thus propose to have a manually-written `identifier_of_error` function

```ocaml
let identifier_of_error err =
  let group = "tyco" in
  let open Location in
  match err with
  | Constructor_arity_mismatch _ -> { group; num = 7 }
  | [...]
```

(Note: in general there is no one-to-one mapping between constructors
in the error type and the user-level concept of "errors", two
constructors could be the same error or one constructor could be
different errors depending on its argument.)

The exhaustivity check for pattern-matching on this function
guarantees that we always assign am identifier to all errors, and in
particular don't forget picking an identifier for new errors.


#### Advanced

If we wanted to be able to generate nice documentation for each error,
or at least compute statically the set of error identifiers currently in
use, we could attach an "example" of each error (an OCaml value
belonging to this error case, chosen to be representative for this
error and read nicely when printed by the error report) to each case
of the `identifier_of_error` function using a bespoke attribute
`[error.example]`:

```ocaml
let identifier_of_error err =
  let group = "tyco" in
  let open Location in
  match[@error.table] err with
  | Constructor_arity_mismatch _ ->
    { group; num = 7 }
    [@error.example (Constructor_arity_mismatch (Lident "foo", 1, 2))]
  | [...]
```

Then a custom ppx could easily extract a list of all "errors"
expressions in the `[error.list]` match, and for example print all
errors or build the set of error identifiers. (We already have
a similar ppx running on utils/warnings.ml to check the links to the
manual.)

Having the examples in this function is safer than simply defining
a global list of all errors separately: if we add a new error
constructor, exhaustivity warnings will force us to add a new case to
`identifier_of_error` and thus include an example.

Such a list of all errors can also be used to check that no two errors
are given the same identifier.
