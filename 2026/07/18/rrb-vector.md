---
title: RRB Vector
description: Relaxed Radix Balanced Vector
tags:
  - vector
  - functional-programming
  - trees
  - data-structures
---

An RRB vector is a persistent, immutable sequence implemented using a Relaxed Radix
Balanced tree[^1].

It's essentially a persistent vector redesigned to make concatenation and slicing fast,
while retaining efficient indexed access[^2]

```text
            root
     ┌───────┼────────┐
     ▼       ▼        ▼
  subtree  subtree  subtree
   64 el.   41 el.   28 el.
     │        │         │
[leaves containing blocks of values]
```

## RRB

- **Radix**: an index is divided into bit groups used to navigate the tree.
  Implementations commonly use a branching factor of 32.
- **Balanced**: the leaves remain at approximately the same tree depth. i.e. you do not
  want one branch to become arbitrarily deep like a linked list. Keeping the tree
  shallow preserves logarithmic lookup and update behavior.
- **Relaxed**: subtrees do not have to be completely full or uniformly sized.
  Nodes can store cumulative subtree sizes to route index lookups through these uneven
  branches.

## Performance

| Operation           |                  Approximate complexity |
| ------------------- | --------------------------------------: |
| Indexed lookup      |                            `O(log₃₂ n)` |
| Immutable update    |                            `O(log₃₂ n)` |
| Append              | `O(log₃₂ n)`, effectively near constant |
| Concatenate vectors |                              `O(log n)` |
| Materialized slice  |                              `O(log n)` |
| Iteration           |                                  `O(n)` |

### Use Cases

- Unlike an array, it supports efficient immutable changes.
- Unlike a linked list, it provides fast indexed access.
- Unlike a basic persistent vector, it supports efficient concatenation and real,
  non-view slicing.
- Unlike a rope, it remains suitable as a general-purpose indexed sequence.

## See Also

[Gleam impl](https://tangled.org/becca.monster/iv)

[core.rrb-vector](https://github.com/clojure/core.rrb-vector)

[^1]: A Relaxed Radix Balanced tree is a wide, shallow tree for implementing immutable indexed sequences.

[^2]: Bagwell, P., Rompf, T. (2011). RRB-Trees: efficient immutable vectors. <http://infoscience.epfl.ch/record/169879/files/RMTrees.pdf>
