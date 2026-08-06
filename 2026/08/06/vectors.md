---
title: Vectors
description: How vectors represent data and make similarity search possible
tags:
  - artificial-intelligence
  - mathematics
  - vector-search
---

A **vector** is simply an ordered list of numbers. In geometry, a two-dimensional vector
such as `[3, 4]` can describe a direction and a length. Machine-learning systems use the
same idea with hundreds or thousands of coordinates. Each coordinate is a **dimension**,
and the whole list identifies a point in a high-dimensional space.

A vector is only a container. Its meaning comes from how its coordinates were produced.

A hand-designed vector might record a house's floor area, age, and number of bedrooms.
An [embedding](/notes/2026/08/06/embeddings/) is a vector learned from data, whose
individual dimensions usually do not have simple human-readable meanings.[^1]

## Dense and sparse vectors

A **dense vector** stores a value in every dimension. Embedding models usually produce
dense vectors because meaning is distributed across many coordinates. A
**sparse vector** has mostly zero values. A bag-of-words representation, with one
dimension per term in a vocabulary, is sparse because a document contains only a small
fraction of all possible terms.

This distinction matters in search:

- sparse search is good at exact words, identifiers, names, and rare phrases
- dense search can match related language even when the query and document share few
  words
- hybrid search combines both signals, then merges or reranks their results[^2]

Neither representation is universally better. A query for `ERR_CONN_RESET` benefits
from exact matching. A query for “the browser suddenly lost its connection” may benefit
from semantic matching.

## Comparing vectors

Similarity search embeds a query and each candidate into the same coordinate space,
computes a score, and returns the nearest candidates. Three common measures are:

- **Euclidean distance**: the straight-line distance
  $d(a,b)=\sqrt{\sum_i(a_i-b_i)^2}$. Smaller is nearer.
- **Cosine similarity**: the angle between vectors
  $\cos(a,b)=\frac{a\cdot b}{\lVert a\rVert\lVert b\rVert}$. Larger is more aligned.
- **Dot product**: $a\cdot b=\sum_i a_i b_i$. Larger means more similar, but the score
  is also affected by vector magnitude.[^3]

**Normalization** rescales a vector to length one. For unit vectors, dot product and
cosine similarity are identical, while squared Euclidean distance is
$2-2\cos(a,b)$. They therefore produce the same ranking even though their reported
scores differ.[^3] The right measure is the one expected by the model and index and
should not be chosen independently after vectors have been generated.

Vector closeness is not a probability, proof, or factual judgment. It means only that
the representation and comparison function placed two items near one another.

## Exact and approximate search

An exact **k-nearest-neighbor** search compares the query with every stored vector and
returns the nearest `k`. It has perfect recall relative to that distance measure, but
its work grows with the collection. Large systems often use **approximate nearest
neighbor** (ANN) indexes. They avoid examining every vector, trading some recall for
lower latency.[^4]

Two common ANN index families illustrate the tradeoff:

- **HNSW** builds a multilayer proximity graph. A search starts in a sparse upper layer
  and descends toward increasingly local neighbors. It usually offers a strong
  speed-recall tradeoff, but the graph takes time and memory to build.[^5]
- **IVF** divides the space into regions and searches only the most promising ones. It
  is cheaper to build and store, but its clusters must be trained and its quality is
  sensitive to how many regions the query probes.[^4]

ANN tuning is application-specific. Searching more graph connections or more IVF
regions usually improves recall at the cost of latency. Metadata filters create another
failure mode: some indexes find approximate neighbors first and apply filters afterward,
so a highly selective filter can return fewer than `k` results unless the system scans
farther.[^4]

## What a vector index stores

A useful search record usually contains more than a vector:

- a stable identifier
- the original text or object reference
- the embedding vector and model version
- source, date, language, tenant, and access-control metadata
- fields used for keyword search or filtering

The vector finds candidates and the attached content is what a user or downstream model
actually consumes. Keeping model and version metadata is important because vectors from
different embedding models generally occupy different coordinate spaces. When the model
changes, the collection normally has to be re-embedded and reindexed.

Vector search is therefore a stack of choices: representation, distance measure, index,
filtering, and evaluation. Fast nearest-neighbor lookup is useful only if “nearest”
corresponds to relevance for real queries.

[^1]: Google for Developers. “Embedding space and static embeddings.” <https://developers.google.com/machine-learning/crash-course/embeddings/embedding-space>.

[^2]: Microsoft. “Develop a RAG solution: Information-retrieval phase.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-information-retrieval>.

[^3]: Google for Developers. “Measuring similarity from embeddings.” <https://developers.google.com/machine-learning/clustering/dnn-clustering/supervised-similarity>.

[^4]: pgvector. “Exact and approximate nearest neighbor search.” <https://github.com/pgvector/pgvector>.

[^5]: Yu. A. Malkov and D. A. Yashunin. “Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs.” _IEEE Transactions on Pattern Analysis and Machine Intelligence_, 2018. <https://arxiv.org/abs/1603.09320>.
