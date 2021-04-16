## `with` syntax

### Proposal

Some languages offer a flexible `with` construct that binds the result of an expression within a scope and performs some actions when the scope is exited.  For example, here is a Python excerpt that opens a file, processes the results, and automatically closes the file when execution leaves the scope:

```python
with open "data.csv" as fd:
    process(fd)
```

This RFC proposes a similar construct for OCaml:


```ocaml
with opened "data.csv" as fd do
    process fd
done
```

#### Desugaring

The proposed syntax has a simple desugaring into a function application:

```ocaml
with e as x do e' done
↝
e (fun x -> e')
```

### Related proposals:

[ocaml/ocaml#9887](https://github.com/ocaml/ocaml/pull/9887) proposed reusing `let@` syntax for a similar purpose.  However, there were objections about unclear scoping:

  @lpw25:
  >  it suffers from the same obvious drawback: it takes a construct
  >  whose entire purpose is to delineate a region of code and makes
  >  the end of that region implicit and unclear. There will be bugs
  >  caused by using this style.
  
  @xavierleroy:
  > However, like @lpw25 I would not use either notation for
  > higher-order functions such as with_file_output, because their
  > function argument is not a continuation and it matters very much
  > where the function ends.

This PR resolves those objections.

@gadmm gave an [example](https://github.com/ocaml/ocaml/pull/9887#issuecomment-692117523) (of possible dangerous code!) in [ocaml/ocaml#9887](https://github.com/ocaml/ocaml/pull/9887), comparing the proposed syntax with sugar-free code.  Here's a threefold comparison of his example written without syntactic sugar, with the `let@` syntax, and with the syntax proposed here:

```ocaml
let my_monadic_function args f =
  with_file name (fun input ->
    fresh_variable () @@ fun var ->
    f input var)

let my_monadic_function args f =
  let& input = with_file name in
  fresh_variable () @@ fun var ->
  f input var

let my_monadic_function args f =
  with file name as input do
   with fresh_variable as var do
     f input var
   done
  done
```

### More/fewer arguments:

The syntax extends naturally to multiple arguments:


```ocaml
with e as x and y and z do e' done
↝
e (fun x y z -> e')
```

We might also add a succinct syntax for the no argument (i.e. single `unit` argument) case :


```ocaml
with e do e' done
↝
e (fun () -> e')
```

### Other use cases

Since `with` is such a flexible word (with 40 main entries in the <abbr title="Oxford English Dictionary">OED</abbr>), the proposed construct can serve many purposes, e.g.:

1. Iteration
  ```ocaml
      with each [1;2;3] as x do
         print x
      done
  ```

2. Indexed iteration
   ```ocaml
   with enumeration (each_line fd) as line and i do
          ...
   done
   ```

3. Guarded debugging (i.e. executing certain statements only at a certain debugging level):
   ```ocaml
     with debugging_level Info do
       Printf.printf "%s\n" ...
     done
   ```


5. Fancy loops, with labeled `break` and `continue`:
   ```ocaml
   with loop 1 5 as i outer do
     with loop 1 5 as j inner do
       if j > i + 1 then outer#continue
       else if j > 3 then inner#break
       else Printf.printf "%s%d %d\n" (spaces i) i j
     done
   done
   ```

6. Managing evaluation order (cf. [ocaml/ocaml#10177](https://github.com/ocaml/ocaml/pull/10177)):
   ```ocaml
   let rec of_tree t =
     with delay do
       match t with
       | Leaf v -> Seq.return v
       | Node(l, r) -> Seq.append (of_tree l) (of_tree r)
     done
   ```

7. folds

   ```ocaml
    let sum l =
       with fold l 0 as elem and acc do
         elem + acc
      done
   ```

### Questions (and some answers)


**Question**: In some of the examples the body has a non-`unit` type.  However, constructs with `do ... done` usually have `unit` type in OCaml.  Should we enforce a `unit` type here, too?

**Question**: are there any grammar conflicts?  
**Answer**: Perhaps!
There's an apparent clash with `try` expressions with a trailing semicolon in the body: `try e1; e2 with p -> e3` is fine, but `try e1; e2; with p -> e3` may conflict with the new syntax.

**Question**: Is there a significant advantage to this over just writing `with e (fun x -> e')`?  
**Answer**: Several keywords in OCaml could be replaced in this way.
For example, we could have a function `while` rather than a keyword and write `while (fun () -> e) (fun () -> body)`, and similarly for `match`, `if`, `for`, etc.
Syntactic sugar is useful, _especially_ when it has a straightforward translation into core constructs!

**Question**: Could we extend it to branching?  
**Answer**: Yes.  We could write:
   ```ocaml
      with discriminate collection as Left x do
          handle_left x
       done as Right y do
          handle_right y
       done
   ```
   But that's probably a step too far.
