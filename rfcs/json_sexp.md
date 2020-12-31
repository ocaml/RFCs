## Adding common types (JSON/S-exp/...) to `stdlib` to aid interoperability between third-party libraries

Today if you are writing a library and you want to generate or consume values in
JSON format (or S-exp, or ...) you have no alternative but to depend on a
specific library implementing the format. This has two downsides:

- If all you need is the type definition, you end up depending on a (potentially
  large) third-party library simply to gain access to this definition.

- Since different libraries typically have different type definitions, the
  values that you generate are only compatible with your chosen library.

This is the same problem that we used to have with the type of Unicode code
points, which was solved by introducing a common type to the `stdlib`, namely
`Uchar.t`.

The proposal here is to do the same with JSON and S-exp: add a common type to
the `stdlib` so that third-party libraries can use this common type, aiding
interoperability.  It would also allow libraries which generate or consume
values in a simple way to do so without depending on any third-party libraries.

The proposed types are below.

### JSON

```
type t =
  | Object of (string * t) list
  | Bool of bool
  | Float of float
  | Int of int
  | Array of t list
  | Null
  | String of string
```

### S-exp

```
type t =
  | Atom of string
  | List of t list
```
