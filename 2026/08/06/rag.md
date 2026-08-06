---
title: Retrieval-Augmented Generation
description: Grounding language-model answers in retrieved source material
tags:
  - artificial-intelligence
  - large-language-models
  - retrieval-augmented-generation
---

**Retrieval-augmented generation** (RAG) gives a language model selected source material
at answer time. Instead of asking the model to answer only from patterns stored in its
parameters, a retriever finds material relevant to the question and adds it to the
prompt. The model then uses that context to compose an answer.

The 2020 paper that named RAG combined a sequence-to-sequence generator with a dense
vector index of Wikipedia. It explored retrieving one set of passages for an entire
answer and allowing different passages for different generated tokens.[^1] In current
practice, “RAG” describes a broader family of systems involving keyword search, vector
search, databases, knowledge graphs, tools, or several of these together.

RAG does not automatically make a model truthful. It gives the model better evidence and
makes sources easier to update and cite. A bad retriever can omit the answer. A good
retriever can return misleading text and a generator can ignore or misstate correct
context. Each stage must be evaluated separately.

## Two Pipelines

Most document RAG systems have an offline indexing pipeline and an online query pipeline.

The **indexing pipeline** prepares the knowledge base:

1. Ingest documents and preserve their source, version, and permissions.
2. Parse text and structure, including headings, tables, lists, and page boundaries.
3. Split the content into chunks that each carry a coherent idea.
4. Enrich chunks with metadata such as title, section, date, language, and entities.
5. Create [embeddings](/notes/2026/08/06/embeddings/) for semantic search.
6. Store the original text, vectors, searchable fields, and metadata in an index.

The **query pipeline** answers a request:

1. Check the user's identity and determine which content they may access.
2. Rewrite or decompose the query when necessary.
3. Retrieve a broad candidate set from authorized content with semantic, keyword,
   structured, or hybrid search.
4. Apply other metadata filters.
5. Rerank and deduplicate candidates, then select context that fits the prompt budget.
6. Ask the model to answer from that context and cite the underlying sources.
7. Record retrieval and generation traces needed for evaluation and debugging.

The user query and document chunks must use the same compatible embedding space for
vector retrieval.[^2] The text, not merely its vector, is passed to the generator.

## Retrieval strategies

**Sparse retrieval** matches terms in an inverted index. It excels at exact names,
identifiers, quotations, and uncommon words. **Dense retrieval** compares query and
document [vectors](/notes/2026/08/06/vectors/) and can connect different wording with
similar meaning. Dense Passage Retrieval showed how separately encoded questions and
passages could support open-domain question answering.[^3]

**Hybrid retrieval** runs both and combines their ranked lists. This is often a stronger
default than choosing one representation, because their failure modes differ. A later
reranker can jointly inspect the question and each candidate to improve precision. The
initial retriever should emphasize recall and the reranker narrows those candidates to
the small number worth placing in the prompt.[^4]

Metadata filters are part of retrieval, not an afterthought. They constrain results by
tenant, access level, language, date, product, jurisdiction, or document type. Permissions
must be enforced before protected content reaches the model, since asking the model not
to reveal unauthorized text is not an access-control boundary.[^5]

Complex questions may need **query decomposition**. “Compare the last two annual reports”
requires finding each report, retrieving comparable sections, and joining the evidence.
An agentic RAG system lets an orchestrator choose searches and perform several retrieval
steps, while a standard RAG pipeline follows a fixed sequence.[^6] Agentic retrieval can
handle multi-hop questions but adds latency, cost, and more behavior to test.

## Chunking and context assembly

Chunking determines the unit of retrieval. Fixed token windows are simple, but they can
separate a heading from its paragraph or split a table. Structure-aware chunks preserve
meaning better when the source format exposes reliable structure. Overlap can rescue
facts near a boundary, at the cost of duplicate results, storage, and prompt space.

A useful chunk contains enough local context to stand alone, but not so many topics that
its embedding becomes diffuse. Store parent-document and section identifiers so adjacent
or parent content can be recovered after retrieval. Tables, images, code, and scanned
documents may need format-specific parsing rather than flattening everything into plain
text.[^6]

More retrieved text is not necessarily better. Irrelevant chunks dilute the prompt and
consume attention and token budget. Experiments on long-context models found that they
often used relevant information less reliably when it appeared in the middle than at the
beginning or end.[^7] Context selection, ordering, and deduplication therefore matter even
when the model can technically accept a much longer prompt.

The generation prompt should distinguish instructions from retrieved evidence, require
claims to be supported by that evidence, preserve source identifiers for citations, and
allow an “insufficient evidence” answer. Retrieved documents are untrusted input: they
can contain accidental or malicious instructions, so prompt injection and document
provenance belong in the threat model.[^8]

## Evaluating RAG

End-to-end answer quality alone does not reveal which stage failed. A useful evaluation
set contains realistic questions, expected supporting passages, answer criteria, and
unanswerable or adversarial examples. It should represent different document types,
languages, freshness requirements, and permission levels.

Evaluate retrieval first:

- **recall@k**: the fraction of questions whose relevant evidence appears in the first
  `k` results
- **precision@k**: the fraction of those results that are relevant
- **mean reciprocal rank**: rewards placing the first relevant result near the top
- **nDCG**: rewards good ordering when relevance has several grades
- latency, index freshness, filter correctness, and empty-result behavior

Then evaluate generation:

- **groundedness**: whether claims are supported by the supplied context
- **completeness**: whether the answer covers all parts of the question
- answer correctness and relevance
- citation correctness: whether each citation entails the nearby claim and resolves to
  the intended source
- refusal quality when the evidence is absent or conflicting[^8]

Metrics need human-reviewed examples. Model-based graders can scale evaluation, but they
are themselves fallible and should be calibrated against people. Because generation can
vary between runs, comparisons should use a stable test set, record configuration and
model versions, and repeat samples where variance matters.

## RAG, long context, and fine-tuning

These techniques solve different problems:

- **RAG** is useful when knowledge changes, must remain outside model weights, needs
  source attribution, or differs by user permissions.
- **Long-context prompting** is simpler when the complete source set is small enough to
  supply directly. It removes a retrieval failure point but does not guarantee the model
  will use every part of the context well.[^7]
- **Fine-tuning** changes behavior, style, format, or task performance. It is usually a
  poor mechanism for keeping a changing factual corpus current or exposing citations.

They can be combined: a fine-tuned model can follow a domain-specific answer format while
RAG supplies current evidence. The simplest adequate system is still preferable. A small,
well-curated corpus may need only full-text search and a prompt. Adding embeddings,
rerankers, or agents is justified when evaluation shows which failure they address.

[^1]: Patrick Lewis et al. “Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks.” 2020. <https://arxiv.org/abs/2005.11401>.

[^2]: Microsoft. “Develop a RAG solution: Generate embeddings phase.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-generate-embeddings>.

[^3]: Vladimir Karpukhin et al. “Dense Passage Retrieval for Open-Domain Question Answering.” 2020. <https://arxiv.org/abs/2004.04906>.

[^4]: Microsoft. “Develop a RAG solution: Information-retrieval phase.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-information-retrieval>.

[^5]: Microsoft. “Design a secure multitenant RAG inferencing solution.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/secure-multitenant-rag>.

[^6]: Microsoft. “Design and develop a RAG solution.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide>.

[^7]: Nelson F. Liu et al. “Lost in the Middle: How Language Models Use Long Contexts.” 2023. <https://arxiv.org/abs/2307.03172>.

[^8]: Microsoft. “Develop a RAG solution: Large language model end-to-end evaluation phase.” <https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-llm-evaluation-phase>.
