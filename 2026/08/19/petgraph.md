---
title: Graphs in Rust with petgraph
description: Choosing graph storage and algorithms in Rust
tags:
  - rust
  - graphs
  - algorithms
  - crates
---

`petgraph` 0.8.3[^2] is a Rust library for graph data structures and algorithms, and
Graphviz import/export.[^1] Its nodes and edges can hold arbitrary associated data,
which the API calls _weights_. Edges can be directed or undirected.

The usual starting point is `Graph`, an adjacency-list collection with `O(|V| + |E|)`
space. It identifies nodes and edges with compact indices, so removing an item can move
the last item into its slot and invalidate an index. `StableGraph` keeps indices stable
across removals, trading some memory and API coverage for that property.[^1]

The other graph types suit different identities or layouts:

- `GraphMap` uses copyable, hashable node values as keys, has constant-time edge
  existence checks, and does not allow parallel edges.
- `MatrixGraph` stores an adjacency matrix and fits dense graphs.
- `Csr` stores a sparse adjacency matrix and is useful when the graph is built once
  and queried many times.

The `algo` module includes Dijkstra, A\*, Bellman–Ford, topological sorting, strongly
connected components, minimum spanning trees, maximum flow, graph isomorphism, and
more. `visit` provides reusable depth-first and breadth-first traversals.[^1]

```rust
use petgraph::algo::dijkstra;
use petgraph::graph::UnGraph;

let graph = UnGraph::<(), ()>::from_edges(&[(0, 1), (1, 2), (2, 3)]);
let distances = dijkstra(&graph, 0.into(), Some(3.into()), |_| 1);
assert_eq!(distances[&3.into()], 3);
```

[^1]: [petgraph API documentation](https://docs.rs/petgraph/latest/petgraph/).

[^2]: [petgraph 0.8.3 crate documentation](https://docs.rs/crate/petgraph/0.8.3).
