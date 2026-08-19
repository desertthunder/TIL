---
title: Trigram Indexes
description: Using overlapping three-character sequences for substring and fuzzy text search
tags:
  - databases
  - indexes
  - postgresql
  - text-search
---

A trigram is three consecutive characters. A trigram index stores the
sequences found in each value, so a search can look up pieces of a string rather than
scan every complete value.

At the storage level, a PostgreSQL GIN trigram index is an inverted index. Each distinct
trigram is a key with a posting list of row IDs:

```text
"app" -> {7, 19}
"ppl" -> {7}
"ple" -> {7, 42}
```

A row appears in many lists because its value contains many trigrams. For a pattern such
as `%apple%`, `pg_trgm` extracts the usable trigrams, combines their posting lists to
find candidate rows, then the database evaluates the original predicate. The index
stores trigram membership, not the full text, so it can produce candidates that need a
recheck.

A B-tree can efficiently handle `name LIKE 'ann%'`, but a leading wildcard prevents it
from narrowing the search range. `pg_trgm` can index `LIKE`, `ILIKE`, regular expressions,
and similarity operators instead:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX users_name_trgm_idx
  ON users USING GIN (name gin_trgm_ops);
```

GiST stores a lossy bitmap signature of each row's trigrams. Its default signature is
12 bytes; increasing `siglen` reduces false positives at the cost of a larger index.
GiST also supports nearest-match queries ordered by trigram distance. Patterns with no
extractable trigram, such as very short searches, may still scan many index entries.[^1][^2]

[^1]: PostgreSQL, [`pg_trgm` documentation](https://www.postgresql.org/docs/current/pgtrgm.html).
[^2]: PostgreSQL, [GIN indexes](https://www.postgresql.org/docs/current/gin.html).
