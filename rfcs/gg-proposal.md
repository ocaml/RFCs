Proposal for the GC
===================

I have proposal which is twofold (the two are relatively independant, but I thought about the first one while reasonning about correctness of the implementation of the first one):

- split the minor heap in slices
- replace the grayval list by a reference counter

This ideas are quite natural and I am probably not the first one to think about this.

Split the minor into slices
---------------------------

The idea is that the minor heap is splitted in sevaral block, like 8x64k
blocks.  There is a /current/ block.  lets us call $n_m$ the number of minor
heap blocks and $c_m$ the index of the current block. Index are modulo $n_m$,
but we compare block relatice to the current block: we say the a block index
$k$ is older that a block index $l$ if $k < l$ when $k$ and $l$ are
represented by the smallest integer strictly greater than $c_m$.

- Allocation is performed in the current minor heap block

- At the end of a minor GC, only the block $c_m + 1$ is promoted before
  incrementing $c_m$.  Older blocks are promoted first.

- there are several ways to handle scanning/greyval:

  1. consider block in the block $k$ pointed from a older block as a greyval
  and add them to the list. This gives a more complex test when mutating a
  value.

  2. scan the whole minor heap, with the argument that old blocks should be
  sparse and the complexity of marking is likely smaller that the size of the
  minor heap. But the worst case is still proportionnal to the full size of
  the minor heap.

  3. consider greyval only when the age difference of the block in greater
  than some constant $C$. This is actually not more complex than 1., checking
  the address of the block to know if we are in the current block is not much
  more instructions than checking the age difference using modular arithmetic.

I think 3 is the best idea with C=2 or 3.

Benefits:

- less latency of allocation if the block are small and if we use solution
  1. or 3. above.

- no more early propmotion. All recently allocated block in the minor heap are
  likely wrongly promoted in the current approach while here ony blocks that
  have resisted $n_m - 1$ complete minor GC are promoted.

- cost of allocation is the minor heap is more or less the same as now

- All values: n_m, size of blocks and C can be parametized by the user,
  allowing to tune the GC for the need of your application.


Cost:

- wasted memory (but we can afford much larger minor heap ?)

- slightly more costly test in mutation for proposal 1. and 3.

- the maximum size that we can allocate in the minor-heap is bounded by the
  size of one block.

- more ?

Reference counter for greyval
-----------------------------

The title is almost self sufficient: why not add in each block in the minor
heap a reference counter that count the number of references from the major
heap or from a block that is too old in the minor heap ?

Benefit:

- This replace adding and removing from the greyval list by incrementing and
  decrementing a counter in the minor heap. This seems much less costly with
  domains ?

- The problem wich cyclic data structure and reference counter does not occur
  here as pointer are oriented by the age of the block.

- We can check the counter in the code compacting the heap for debugging, as
  ref counter are more complex to debug (easier to check if a value is well
  marker as a greyval than verify the value of a counter).

Cost:
- Nothing ?

Questions
---------

I don't know haw both proposal affect ephemeron ?
