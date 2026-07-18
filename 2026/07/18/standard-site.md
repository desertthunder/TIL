---
title: standard.site
description: open-social publishing
tags:
  - atproto
  - publishing
  - markdown
---

Standard.site describes long-form publishing with two AT Protocol records:

1. a `site.standard.publication` for the website
2. `site.standard.document` for each article or note.[^1]

The records live on the author's PDS while the website remains the canonical copy.

Verification points in both directions. The site serves its publication AT-URI from
`/.well-known/site.standard.publication`, and each document page includes a
`<link rel="site.standard.document">` with that document's AT-URI.

AT Protocol permits several record-key formats, but both Standard.site collections
require a 13-character timestamp identifier (TID).[^2]

A sync can derive a stable TID for each source path, then use `putRecord` to update the
same record instead of creating a duplicate on every deployment.

The document lexicon leaves `content` as an open union.

Raw Markdown fits there as an `at.markpub.markdown` object, while `textContent` must be
an unformatted plain-text version.[^3]

A small deployment script can authenticate with a revocable app password, which
AT Protocol permits for command-line tools and other single-purpose automation.[^4]

[^1]: Standard.site. “Quick Start.” <https://standard.site/docs/quick-start/>

[^2]: AT Protocol. “Timestamp Identifiers (TIDs).” <https://atproto.com/specs/tid>

[^3]: Markpub.at. “Markdown Lexicon.” <https://markpub.at/>

[^4]: AT Protocol. “SDK Authentication.” <https://atproto.com/guides/sdk-auth>
