# Site setup

Themeless Jekyll site. One layout, one stylesheet, no theme gem, no build step.
GitHub Pages builds the markdown; light/dark follows the reader's OS setting.

## Files

```
_config.yml            site title, url, baseurl
_layouts/default.html  the only layout — every page uses it
assets/css/style.css   the entire theme; colors live in the two :root blocks
index.md               home page + auto-generated table of contents
resume.md              publishes to /resume/
_notes/                one markdown file per page
Gemfile                local preview only; not used by GitHub Pages
```

## Before the first push

In `_config.yml` set:

- `url` — your Pages URL, no trailing slash
- `baseurl` — `""` for a user site (`USERNAME.github.io`), or `"/reponame"` for
  a project site

Wrong `baseurl` renders the site unstyled: `relative_url` emits the wrong
stylesheet path and it 404s. That symptom is nearly always this line.

## Deploy

1. Push to `main`.
2. Repo -> Settings -> Pages -> Build and deployment -> Source:
   **Deploy from a branch**, branch `main`, folder `/ (root)`. Save.
3. Actions tab shows "pages build and deployment", 30-90 seconds.

## Adding a page

Drop a markdown file into `_notes/`:

```markdown
---
title: OSPF route leakage from a WANSIM routing instance
description: One line, optional.
updated: 2026-08-11
---

Content here.
```

It appears in the home-page table of contents automatically, sorted by title.
Only `title` is required. To control order instead of alphabetizing, add
`order: 1` to each file and change `sort: "title"` to `sort: "order"` in
`index.md`.

Files must be UTF-8 **without** a byte-order mark. Jekyll detects front matter
by testing whether the file begins literally with `---`; a BOM puts three bytes
in front of it and the page publishes as raw text with visible `---`. VS Code
and Notepad default to no-BOM. Generating files from PowerShell is where this
bites: use `-Encoding utf8NoBOM` (PowerShell 7+) or
`[System.IO.File]::WriteAllText()` (Windows PowerShell 5.1, where
`-Encoding utf8` writes a BOM). Verify with `Format-Hex file.md` — first bytes
should be `2D 2D 2D`, not `EF BB BF`.

## Changing the theme

Everything visual is in `assets/css/style.css`. The two `:root` blocks at the
top hold every color; edit those and the whole site follows. Light/dark
switches with the OS setting — there is no toggle to maintain, no JS, no flash
of the wrong theme on load.

Foreground/background pairs meet WCAG AA. If you change colors, re-check
contrast rather than eyeballing it.

To use a dyslexia-friendly typeface, download the Atkinson Hyperlegible woff2
files into `assets/fonts/`, add an `@font-face` rule, and put the family first
in `--font-body`.

## Local preview (optional)

Not required — GitHub builds server-side, so pushing to a branch works as a
substitute.

```
bundle install
bundle exec jekyll serve --livereload
```

The `github-pages` gem pins Jekyll 3.9.x, which fails on Ruby 3.4+ because
`csv`, `base64`, and `logger` left the standard library. If you see
`cannot load such file -- csv`: install Ruby 3.2 or 3.3, or add `gem "csv"`,
`gem "base64"`, `gem "logger"` to the Gemfile, or skip local preview.

On Windows, install Ruby+Devkit from rubyinstaller.org and let `ridk install`
run option 3 (MSYS2 + MINGW toolchain) or native gems won't compile. If the
console garbles extended characters, run `chcp 65001` first.

For current Jekyll and unrestricted plugins, switch Settings -> Pages -> Source
to **GitHub Actions** and take the Jekyll starter workflow. That drops the gem
pin and runs the build on Ubuntu, sidestepping the Ruby version problem
entirely.
