---
title: Git Attributes
description: Setting per-path behavior for line endings, diffs, merges, and archives
tags:
  - git
  - configuration
  - version-control
---

A `.gitattributes` file assigns attributes to paths in a Git repository. While
`.gitignore` decides which untracked files Git should ignore, `.gitattributes`
changes how Git and tools built on Git handle files that are tracked.[^1]

Each non-comment line contains a pattern followed by one or more attributes:

```gitattributes
* text=auto
*.sh text eol=lf
*.bat text eol=crlf
*.png binary
docs/** export-ignore
```

An attribute has four possible states:

- `text` sets it to true.
- `-text` sets it to false.
- `eol=lf` assigns a string value.
- `!text` returns it to the unspecified state.

When several patterns match a path, later rules override earlier ones one
attribute at a time. Patterns mostly follow `.gitignore` syntax, with two
important differences: negative patterns such as `!file` are forbidden, and a
directory pattern does not recurse. Use `docs/**`, rather than `docs/`, to match
everything below a directory.[^1]

## Where attributes come from

A committed `.gitattributes` distributes project rules to every clone. Files in
subdirectories can add rules relative to their own location. Rules closer to a
path take precedence over rules higher in the tree.[^1]

Git also supports attributes that are not committed:

- `$GIT_DIR/info/attributes` contains rules for one clone and has the highest
  precedence.
- `core.attributesFile` names a per-user file for all repositories.
- the system attributes file applies to every user and has the lowest
  precedence.

Use `git check-attr` to see the resolved value and its source:

```sh
git check-attr -a -- path/to/file
git check-attr --source text eol -- script.sh
```

## Common uses

`text` controls end-of-line normalization. Setting `* text=auto` lets Git detect
text files and store their line endings as LF. `eol=lf` or `eol=crlf` chooses the
line endings written to the working tree. When introducing normalization to an
existing repository, `git add --renormalize .` reapplies the new rules; review
the resulting changes before committing them.[^1]

The `diff` attribute can force text or binary treatment or select a named diff
driver. Named drivers can provide language-aware hunk headings, choose a diff
algorithm, or turn a binary format into text for human-readable diffs. The
`merge` attribute similarly selects the normal text merge, binary handling, the
`union` driver, or a custom merge driver. Driver definitions live in Git config,
so collaborators need the matching configuration.[^1]

Clean and smudge `filter` drivers transform content between the working tree and
the repository. Git LFS is a familiar example: the repository stores a pointer,
and the working tree receives the external object. A required filter should be
configured as such; otherwise Git treats a missing or failed filter as a
pass-through.[^1]

For release archives, `export-ignore` omits matching paths and `export-subst`
expands commit placeholders. Other attributes can control whitespace checking,
working-tree character encoding, and conflict-marker length.[^1]

GitHub and other consumers may define their own attributes. GitHub Linguist uses
`linguist-generated`, `linguist-vendored`, and related attributes to control
language statistics, syntax highlighting, and diff presentation.[^2]

[^1]: [Git documentation: `gitattributes`](https://git-scm.com/docs/gitattributes).

[^2]: [GitHub Linguist overrides](https://github.com/github-linguist/linguist/blob/main/docs/overrides.md).
