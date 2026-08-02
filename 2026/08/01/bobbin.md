---
title: Bobbin
description: Tangled's diskless, API-only AppView
tags:
  - atproto
  - cache
  - distributed-systems
---

Bobbin is [Tangled](https://tangled.org)'s read-only [AppView](https://atproto.com/guides/glossary#app-view).

It exposes `sh.tangled.*` records through XRPC, giving clients one API for repositories,
issues, pulls, comments, stars, and follows instead of making them query every user's
Personal Data Server (PDS).[^1]

Bobbin has no durable database. Its state is disposable rather than nonexistent such
that every process starts with empty in-memory indexes and rebuilds them from upstream
data.

## How it works

1. [Hydrant](https://tangled.org/microcosm.blue/hydrant) collects AT Protocol records
   from PDSes.
   Bobbin connects to its WebSocket stream at cursor `0`, replays the `sh.tangled.*`
   dataset, and then follows live events.
2. The replay builds in-memory edge, issue-state, pull-state, and search indexes.
   Bobbin reports itself ready after reaching recent events.
   If the stream is already quiet, it uses an idle-window fallback.[^2]
3. Record bodies are kept in a size-bounded, Zstandard-compressed LRU cache.
   A cache miss is fetched from Slingshot, an AT Protocol record and identity cache,
   so point lookups can work while Bobbin is still warming.[^3]
4. Git data does not have a Hydrant equivalent. Bobbin forwards those reads to the
   repository's Knot server, or to a configured mirror first.[^4]

Rebuilding from the source dataset makes reconciliation the normal startup path.
A missed event or bad deployment can be repaired by replacing the process instead of
migrating or manually fixing a derived database.

The trade-off is startup time and dependence on upstream services.

Tangled reported a typical full replay of 30–90 seconds, but as long as 20 minutes when
Hydrant also had to refetch records from PDSes.

The same post reported roughly 100 MB for the dataset after compression.

Bobbin's current configuration also sizes caches, search memory, ingest parallelism,
and request concurrency against the container's memory budget.[^1][^5]

Tangled runs its public instance at [`api.tangled.org`](https://api.tangled.org) in a
long-lived Cloudflare Container. Keeping the container awake avoids paying the replay
cost after an idle shutdown.[^6]

[^1]: Lewis. “Introducing Bobbin.” Tangled, 13 July 2026. <https://blog.tangled.org/bobbin/>

[^2]: Tangled. “Bobbin ingest pipeline.” <https://tangled.org/tangled.org/core/tree/master/bobbin/crates/ingest/src/lib.rs>

[^3]: Tangled. “Bobbin record LRU.” <https://tangled.org/tangled.org/core/tree/master/bobbin/crates/record-lru/src/lib.rs>

[^4]: Tangled. “Bobbin service entry point.” <https://tangled.org/tangled.org/core/tree/master/bobbin/crates/bobbin/src/main.rs>

[^5]: Tangled. “Bobbin example configuration.” <https://tangled.org/tangled.org/core/tree/master/bobbin/example.toml>

[^6]: Tangled. “Bobbin Cloudflare Worker.” <https://tangled.org/tangled.org/core/tree/master/bobbin/worker/src/index.ts>
