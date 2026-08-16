---
title: HTTP Entity Tags
description: Cache validation and conditional updates with ETag, If-None-Match, and If-Match
tags:
  - http
  - caching
  - concurrency
  - web-development
---

An entity tag, or ETag, is an opaque identifier for one representation of an HTTP
resource. An origin server sends it in a response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
ETag: "7b3a"

{"name":"Ada"}
```

The value is opaque to clients and may be a revision number, a content hash, file
metadata, or another value chosen by the server. Clients only store and return it. The
quotes are part of the field syntax, so `ETag: 7b3a` is invalid.[^1]

An ETag identifies a **representation**, not a resource in the abstract. One URL can
produce different representations through content negotiation. Gzip-compressed and
uncompressed content, for example, need distinct strong ETags because content coding is
part of the representation data.[^1]

## Cache validation

When a stored response needs validation, the client sends its tag in `If-None-Match`:

```http
GET /people/ada HTTP/1.1
Host: example.com
If-None-Match: "7b3a"
```

If the current representation still matches, the server returns `304 Not Modified`
without content. The client reuses its stored response and updates its metadata from the
304 response. If the representation has changed, the server returns the new content and
its new ETag in a normal `200 OK` response.[^2]

```http
HTTP/1.1 304 Not Modified
Date: Sun, 16 Aug 2026 18:42:00 GMT
ETag: "7b3a"
Cache-Control: max-age=60
```

An ETag does not determine how long a response is fresh. `Cache-Control` or `Expires`
does that:

- A fresh response can be reused without contacting the origin server.
- Once it is stale, its ETag lets the cache ask whether the content changed instead of
  downloading it again.
- `Cache-Control: no-cache` still permits storage, but requires successful validation
  before each reuse.[^2]

`If-None-Match` takes precedence over the date-based `If-Modified-Since` condition when
both appear. Entity tags avoid the one-second resolution and clock-consistency limits of
HTTP modification dates.[^1]

## Strong and Weak tags

A tag is strong unless it begins with the case-sensitive `W/` prefix:

```http
ETag: "7b3a"
ETag: W/"7b3a"
```

A strong ETag must change whenever the representation data observable in a `200 OK`
response changes. A weak ETag may remain the same across representations that the server
considers equivalent for a particular purpose, even if their bytes differ.[^1]

HTTP uses two comparison rules:

| Tags                      | Strong comparison | Weak comparison |
| ------------------------- | ----------------- | --------------- |
| `"7b3a"` and `"7b3a"`     | match             | match           |
| `W/"7b3a"` and `"7b3a"`   | no match          | match           |
| `W/"7b3a"` and `W/"7b3a"` | no match          | match           |

Cache validation with `If-None-Match` uses weak comparison because it only asks whether
the stored response is still an acceptable substitute. `If-Match` uses strong comparison
because a state-changing request must detect every observable change. Weak tags therefore
work for revalidation but not for guarding an update.[^1]

## Preventing lost updates

ETags also support optimistic concurrency. A client first retrieves a representation and
its tag, edits the data locally, then makes the update conditional on that exact version:

```http
PUT /people/ada HTTP/1.1
Host: example.com
Content-Type: application/json
If-Match: "7b3a"

{"name":"Ada Lovelace"}
```

The server applies the update only if `"7b3a"` still matches the current representation.
If another client changed it in the meantime, the condition fails and the server can
return `412 Precondition Failed`. This prevents a later write based on stale data from
silently overwriting an earlier one.[^1]

The wildcard forms express existence conditions:

- `If-Match: *` runs the method only if the resource has a current representation.
- `If-None-Match: *` runs the method only if no current representation exists, which can
  make creation conditional and prevent an accidental overwrite.

These checks have two important requirements:

- The origin server must evaluate the condition before performing the method.
- The comparison and update must share an atomic boundary. A read-then-write check in
  application code is insufficient if another write can occur between those operations.

## Generating ETags

The server owns the generation scheme. A database revision can be cheaper than hashing a
large response, while a collision-resistant hash naturally tracks known bytes. Whatever
the scheme, it must satisfy the advertised strength and distinguish every representation
that might be selected for the resource. Clients must not infer meaning from the value or
compare values obtained for different resources.[^1]

Changing an ETag unnecessarily causes cache misses and failed update preconditions.
Reusing a strong ETag after observable representation data changes is worse: it can join
incompatible partial responses, validate stale content, or allow an update based on the
wrong version. If the server cannot guarantee strong-validator behavior, it must mark the
tag as weak.[^1]

[^1]: IETF, [RFC 9110: HTTP Semantics, Sections 8.8 and 13](https://www.rfc-editor.org/rfc/rfc9110.html#section-8.8).

[^2]: IETF, [RFC 9111: HTTP Caching, Section 4.3](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.3).
