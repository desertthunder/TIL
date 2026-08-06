---
title: Embeddings
description: Learned vector representations of meaning and similarity
tags:
  - artificial-intelligence
  - machine-learning
  - embeddings
---

An **embedding** is a learned mapping from an object to a fixed-length
[vector](/notes/2026/08/06/vectors/). Text, images, audio, users, and products can all
be embedded. The goal is to arrange the vectors so that relationships useful to a task
become geometric: similar items tend to lie closer together, while dissimilar items lie
farther apart.[^1]

Embeddings replace awkward high-dimensional representations with dense, lower-dimensional
ones. A one-hot word vector has one dimension for every word in a vocabulary and almost
all its values are zero. A learned word embedding might use a few hundred dense values
instead, with relationships distributed across them.[^1] Those dimensions are useful to
the model but usually cannot be labeled cleanly by a person.

## How an embedding learns meaning

An encoder converts an input into a vector. During training, an objective changes the
encoder so that useful examples move together and unsuitable examples move apart. What
counts as “useful” depends on the training data and objective: two passages may be close
because they discuss the same topic, answer the same question, describe substitutable
products, or show the same object.

This means there is no context-free, universal meaning of similarity. An embedding model
trained for sentence retrieval may be poor for source-code search, another language, or
a specialized medical vocabulary. Evaluation has to use the queries and relevance
judgments of the intended application.

**Static embeddings** assign one vector to each word. They cannot distinguish the river
“bank” from a financial “bank.” **Contextual embeddings** calculate a representation
from the surrounding text, so the same token can receive different vectors in different
sentences.[^2] Modern retrieval systems commonly embed a sentence, paragraph, or document
chunk rather than looking up one fixed vector per word.

## Bi-encoders and rerankers

A **bi-encoder** embeds the query and candidate separately. Candidate vectors can be
computed ahead of time, indexed, and compared cheaply with a new query. Sentence-BERT
demonstrated this architecture for semantic textual similarity and retrieval.[^3]

A **cross-encoder** reads the query and a candidate together and produces a relevance
score. Joint attention can capture finer interactions, but every query-candidate pair
requires a model call, so it is too expensive to score an entire large collection.
Search systems often use a bi-encoder for broad retrieval and a cross-encoder to rerank
the small candidate set.

## Similarity is part of the model contract

Embedding vectors are compared with cosine similarity, dot product, or a distance such
as Euclidean distance. These functions are related but not interchangeable in every
model. Vector magnitude affects dot product, for example, while cosine similarity ignores
it.[^4] Model documentation or evaluation should determine the metric and whether vectors
must be normalized.

Queries and indexed items must also be embedded with compatible models. Some retrieval
models use the same encoder for both. Others intentionally use different query and
document encoders trained into one shared space. Replacing either side without rebuilding
the index makes the coordinates incomparable. Production records should therefore retain
the model name, version, dimensions, preprocessing, and creation time.

## Building an embedding pipeline

A practical text pipeline usually does the following:

1. Extract and normalize text without discarding meaningful structure.
2. Split long documents into coherent chunks that fit the model's input limit.
3. Keep the original text and attach source, section, date, language, and permission
   metadata.
4. Embed the chunks in batches and store their vectors in an index.
5. Apply the same preprocessing and compatible encoder to incoming queries.
6. Retrieve nearest candidates, optionally combine them with keyword results, filter by
   metadata, and rerank.

Chunking changes what the vector represents. Tiny chunks may lack enough context to be
understood. Large chunks may blend unrelated ideas and make a relevant sentence harder
to retrieve. Structure-aware boundaries such as headings and paragraphs are a useful
starting point, but chunk size and overlap should be treated as evaluated parameters,
not universal constants.[^5]

## What embeddings can and cannot do

Embeddings support semantic search, recommendation, clustering, duplicate detection,
classification features, and retrieval for
[retrieval-augmented generation](/notes/2026/08/06/rag/). They are compact indexes of
patterns, not databases of verified facts.

Several limitations follow:

- **Information is compressed.** A single vector cannot preserve every detail of its
  input, and two semantically related passages need not answer the same question.
- **Exact strings can disappear.** Product codes, proper names, and uncommon terms may
  be better served by keyword or hybrid search.
- **The training data shapes the space.** Biases, language coverage, domain gaps, and
  stale concepts can all affect which items are placed near one another.
- **Scores are relative.** A nearest result is merely the best candidate available but
  may still be irrelevant. A fixed similarity threshold does not transfer reliably
  between models or datasets.
- **Model changes are migrations.** New dimensions or learned coordinates generally
  require re-embedding the corpus, rebuilding the index, and repeating retrieval tests.

Good evaluation separates the embedding model from the rest of the search pipeline.
Measure whether known-relevant items appear in the first `k` results, inspect failures by
query type, compare against a keyword baseline, and include cases with no relevant item.
Only then can latency, storage, and index recall be traded against actual relevance.

[^1]: Google for Developers. “Embeddings.” <https://developers.google.com/machine-learning/crash-course/embeddings>.

[^2]: Google for Developers. “Obtaining embeddings.” <https://developers.google.com/machine-learning/crash-course/embeddings/obtaining-embeddings>.

[^3]: Nils Reimers and Iryna Gurevych. “Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks.” 2019. <https://arxiv.org/abs/1908.10084>.

[^4]: Google for Developers. “Measuring similarity from embeddings.” <https://developers.google.com/machine-learning/clustering/dnn-clustering/supervised-similarity>.

[^5]: Microsoft. “Develop a RAG solution: Chunking phase.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-chunking-phase>.
