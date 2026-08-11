---
title: Example note
description: Drop a markdown file in _notes/ with these three lines of front matter and it shows up on the home page.
updated: 2026-08-10
---

Everything below the front matter is normal markdown. This file publishes to
`/notes/example-note/` — the URL comes from the filename.

The only required field is `title`. `description` adds a line under the link in
the table of contents; `updated` prints a date under the heading. Both are
optional.

## Code blocks work

```
set security zones security-zone trust interfaces reth0.0
set security zones security-zone untrust interfaces reth1.0
```

## So do tables

| Node | Role | Priority |
| --- | --- | --- |
| node0 | primary | 200 |
| node1 | secondary | 100 |
