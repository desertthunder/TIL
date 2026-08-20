---
title: GitHub Linguist
description: How GitHub detects repository languages and accepts overrides
tags:
  - github
  - git
  - syntax-highlighting
---

GitHub Linguist detects the languages in a repository. GitHub uses its results
for the language bar, syntax highlighting, code navigation, search behavior, and
whether certain files appear in diffs by default.[^1]

Linguist first removes binary, vendored, generated, and documentation files from
the language calculation. It also excludes languages classified as `data` (like json)
or `prose` (like markdown) only `programming` and `markup` languages count by default.
It then identifies each remaining file using an ordered set of strategies:[^2]

1. Vim or Emacs modelines
2. well-known filenames
3. shell shebangs
4. file extensions
5. XML headers and man-page sections
6. heuristics
7. a naïve Bayes classifier

Each stage either settles the language or narrows the candidates for the next
stage. The repository percentages are based on bytes of included code, not file
counts or lines of code.[^2]

On GitHub, a low-priority background job analyzes the repository's default
branch after a push. GitHub caches the result, so the language bar may take time
to reflect a newly committed override.[^2]

## Overriding Linguist

Linguist reads ordinary path patterns from `.gitattributes`. These attributes
change its classification without changing the file itself:[^3]

| Attribute                | Effect                                              |
| ------------------------ | --------------------------------------------------- |
| `linguist-language=Rust` | Classify and highlight the path as Rust             |
| `linguist-generated`     | Exclude it from statistics and hide it in diffs[^4] |
| `linguist-vendored`      | Exclude third-party code from statistics            |
| `linguist-documentation` | Exclude documentation from statistics               |
| `linguist-detectable`    | Include a known `data` or `prose` language          |

Prefix a Boolean attribute with `-` to reverse Linguist's built-in decision. For
example, `-linguist-generated` makes a file that Linguist normally detects as
generated appear normally, while `-linguist-vendored` counts an otherwise
vendored file.[^3]

```gitattributes
# Generated output: omit from language stats and collapse GitHub diffs.
public/search-index.json linguist-generated

# Third-party sources should not describe this project's language mix.
vendor/** linguist-vendored

# This uncommon extension contains Rust.
*.rust-template linguist-language=Rust

# Count SQL even though Linguist classifies it as data.
schema/**.sql linguist-detectable
```

Paths are relative to the `.gitattributes` file containing them.

A pattern like `vendor/*` matches files directly inside that directory. `vendor/**` also
matches its descendants. Language names are case-insensitive and may use an
alias from Linguist's `languages.yml`. Replace spaces in a language name with
hyphens, as in `OpenStep-Property-List`.[^3]

`linguist-detectable` only changes whether a language already known to Linguist
is counted. It cannot introduce a new language definition. Vim and Emacs
modelines can override syntax highlighting for an individual file, but the
`.gitattributes` attributes also control classification and statistics.[^3]

The local `github-linguist` command helps explain surprising results:

```sh
github-linguist --breakdown --strategies
github-linguist --strategies path/to/file
```

`--breakdown` lists the files assigned to each language. `--strategies` reports
which detector won and whether `.gitattributes` confirmed or overrode it. Local
testing reads attributes from the committed repository state, so commit the
`.gitattributes` change before relying on the result.[^1]

[^1]: [GitHub Linguist repository and command-line usage](https://github.com/github-linguist/linguist).

[^2]: [How Linguist works](https://github.com/github-linguist/linguist/blob/main/docs/how-linguist-works.md).

[^3]: [Linguist overrides](https://github.com/github-linguist/linguist/blob/main/docs/overrides.md).

[^4]: [GitHub Docs: customizing how changed files appear](https://docs.github.com/en/repositories/working-with-files/managing-files/customizing-how-changed-files-appear-on-github).
