# Frozen heap

Some OCaml users report a similar problem. They have a large amount of
immutable data that they plan to keep around forever, and they
complain that the major GC keeps traversing this data again and
again. (Think: the OCaml compiler loads possibly-large .cmi files,
Coq loads possibly-large .vo files, etc.)

The Ancient library was a implemented by Richard Jones in 2006 to
solve this problem. (Ancient was focusing on the use case of sharing
this large amount of data across processes, and storing it in swap
memory, maybe more than on saving GC time.) See
http://git.annexia.org/?p=ocaml-ancient.git;a=blob;f=README.txt;hb=HEAD
for more details.

The present document is inspired by a potential feature request for
Coq that I heard from @gadmm and @ppedrot. It describes
a reasonably-simple implementation of this idea elaborated during
a coffee-break conversation with @damiendoligez.

## Proposed API

    Gc.freeze : 'a -> unit
    (* or *)
    Obj.freeze : Obj.t -> unit
  
Calling [freeze v] freezes [v], that is, it migrates [v] and all
memory reachable from it into the "frozen heap".

The frozen heap has the following specification:
- it may not contain pointers into the young or major heaps
- it is illegal to write into a block stored on the frozen heap
  (trying to do this segfaults)


## Proposed implementation

We propose to allocate the frozen heap in a dynamically-growing area
of memory distinct from the major heap. All objects are marked as
alive-and-already-traversed -- the GC stops when it encouters a frozen
object -- following the standard approach for naked pointers to OCaml
values outside the GCed heap.

If available on the operating system, we mark the adress range of the
frozen heap as read-only most of the time (growing the heap requires
marking it read-write temporarily). We say that the frozen heap is
"locked" when it is setup as read-only and "unlocked' when it is
read-write.

The implementation of `freeze` performs the following steps:

- stop the world to ensure that no mutators that may access the value are running concurrently
- unlock the frozen heap
- move the values (recursively) to the frozen heap, replacing each value with
  a pointer to its new location
- lock the frozen heap again
- traverse the whole heap, rewriting pointers to moved values to point
  to their new (frozen) location

Traversing the heap to rewrite pointers in this way is part of the
compactor logic, so we can reuse the code there. (Compaction relocates
the values present in a given set of major pools, into other major
pools. We relocate values reachable from a given point into the frozen
heap.)

## Extensions / advanced concerns

### Deserialization

    value caml_input_frozen_value_from_block(const char *data, intnat len)
    val input_frozen_value : input_channel -> 'a

The functions unmarshal directly into the frozen heap. The
implementation is much more efficient than `freeze (input_value ic)`
as we know that the values are not referenced in the heap, so we can
skip the traverse-the-whole-heap final step.

(Serialized values may reference to atoms that are allocated in
program memory; freezing an atom must be a no-op.)

### Batch freezing

    val Obj.freeze_all : Obj.t array -> unit
    
This amortizes the cost of the heap traversal, freezing many values at once.

### Finalization

If we are trying to freeze a block with an OCaml finalizer attached,
we drop the finalizer -- the value will never become dead during the
program lifetime.

(If we wanted to guarantee that all finalizers eventually get called,
we could instead store a reference to the frozen value and its
finalizer in a new 'finalize_at_exit' list, to be collected at program
exit.)

### Partial freezing

It can be useful to have values that are "partially" frozen, some
subvalues are not frozen -- in particular they may be mutable. We
suggest in this section an invasive, type-directed way to do this: we
implement a `'a not_frozen` type that will not get frozen, and the
programmer must explicitly use it in their datatype definitions.

We can implement `'a not_frozen`, without implementing any extra
feature on top of the RFC. It would be implemented as an OCaml block
with tag Abstract_tag, which:

- contains a `malloc` block containing the `'a`, registered as a root 
  (or a boxroot)

- and has an OCaml finalizer attached to it, to unregister the root
  and free the `malloc` block if the OCaml block becomes dead
  before being frozen
