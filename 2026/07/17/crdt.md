---
title: CRDT
tags:
  - database
---

A conflict-free replicated data type (CRDT[^1]) is a data structure that can be replicated across
multiple computers in a network.

1. The application can update any replica independently, concurrently and without coordinating
   with other replicas.
2. An algorithm automatically resolves any inconsistencies that might occur. Although replicas
   may have different state at any particular point in time, they're guaranteed to eventually converge.

[^1]: Local-First Software: You Own Your Data, in Spite of the Cloud. 1 Apr. 2019, https://www.inkandswitch.com/essay/local-first/.
