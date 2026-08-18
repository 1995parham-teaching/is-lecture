# is-lecture

Slide decks for the **Introduction to Information Security** course at Amirkabir
University of Technology. Hugo builds the site; each deck is a
[reveal.js](https://revealjs.com) presentation written as raw HTML.

Per-deck notes live in `content/lectures/<slug>/CLAUDE.md`.

This repository is a sibling of
[`ie-lecture`](https://github.com/1995parham-teaching/ie-lecture) and shares its
machinery — the layouts, the SCSS, the TOC builder and both CI gates were
carried over unchanged. **A fix to any of those belongs in both repositories.**

## Build and check

```bash
npm install          # or: just install
hugo server          # or: just dev
```

Requires **Hugo 0.165.0 extended** (pinned in `.github/workflows/github-pages.yml`)
and Node. The macOS extended build ships only as a `.pkg` — there is no tarball,
so `xar`/`cpio` it open if you need a throwaway copy rather than a system install.

Two CI gates, both on every push and PR:

| Workflow           | Command                              |
| ------------------ | ------------------------------------ |
| `format.yml`       | `npx prettier --check .`             |
| `github-pages.yml` | `hugo --minify` then deploy to Pages |

**Always run `npx prettier --write <file>` after editing a deck.** Prettier
rewraps HTML, which means any exact string you matched on before the run will
have moved afterwards — re-read the file before making a second edit.

## Layout

```
config.toml                         site params: semester, university, instructor
content/lectures/<slug>/index.html  one deck
layouts/lectures/single.html        deck shell + the generated title slide
layouts/home.html                   the index page (Bootstrap, dark-mode toggle)
assets/scss/lecture.scss            all deck styling
assets/js/index.js                  reveal.js bootstrap
assets/js/toc.ts                    TOC builder — a Hugo template, see below
data/references.yml                 the References section on the home page
```

Decks are raw HTML, which Hugo refuses by default since v0.16x; `[security]
allowContent` in `config.toml` is what permits it. Do not remove it.

## Front matter

```yaml
---
index: 3 # numbered course lecture; drives home-page order and the "03" on the card
title: Cryptography
topics: # feeds both the title card and the TOC slides
  - Foundations
  - Symmetric Encryption
lecture: false # guest/extra deck: no number, listed in the second home-page table
---
```

A deck has **either** `index` (a numbered lecture) **or** `lecture: false` (an
extra deck). The home page selects on exactly that.

## Deck conventions

**Never hand-write a title slide.** `layouts/lectures/single.html` generates one
from front matter for every deck, so a hand-written one is a duplicate.

**Slides** are `<section>`. A `<section>` containing `<section>`s is a vertical
stack (down-arrow navigation).

**TOC dividers** — the deck opens with the full list, then a highlighted divider
before each topic:

```html
<section><div class="toc"></div></section>
<section><div class="toc" data-selected="0"></div></section>
...
<section><div class="toc" data-selected="1"></div></section>
```

`data-selected` is a **0-based index into `topics`**. `assets/js/toc.ts` is run
through `resources.ExecuteAsTemplate` with the page's params, so it contains Go
template actions and cannot be parsed by Prettier — it is in `.prettierignore`.

This splits one fact across two files: the **indices live in the HTML** and the
**titles array lives in the generated `toc.js`**. Every asset in
`layouts/lectures/single.html` is therefore piped through `| fingerprint`. GitHub
Pages serves with `cache-control: max-age=600` from stable paths, so without a
content hash a browser can pair fresh HTML with a ten-minute-old `toc.js` — and a
`data-selected` past the end of the stale titles array highlights nothing, with
no error. **Do not remove the fingerprints.**

**Highlight colours** come from the `$highlights` map in `assets/scss/lecture.scss`:
`hl-orange`, `hl-green`, `hl-cyan`, `hl-red`, `hl-material`, `hl-violet`,
`hl-yellow`, `hl-pink`. Any other `hl-*` class renders **silently unstyled**.
Check the map before inventing one.

**Hands-on slides** carry `class="hands-on"` on the `<section>` itself, which
paints the marker via `section.hands-on:not(:has(> section))::after`. Putting it
on a child `<ul>` breaks any slide that ends with a code block.

**Code blocks** are `<pre><code class="lang-go">`. Escape `<`, `>` and `&` inside
them — the HTML, JavaScript and bitwise-operator samples are where this bites.

## Things that fail silently

- **Overflow.** The slide is 960×700 logical px at `--r-main-font-size: 24px`.
  Content past that is simply cut off; nothing warns you. Several tables in this
  repository carry an explicit `font-size` between `0.78em` and `0.85em` for
  exactly this reason — the per-deck notes say which.
- **Code blocks.** `.reveal pre code` is `overflow: hidden !important`
  (reveal.css) with `max-height: 400px` (league). A long line or a tall block is
  clipped with no scrollbar. Roughly 25 lines is the ceiling.

## Content rules

This material is taught from, so correctness beats speed:

- **Run the code.** Go samples must `go build` and `go vet` clean; C samples must
  compile; Python must execute. Do not put a snippet on a slide you have not run.
- **Captured output must be real.** Every transcript on a hands-on slide is an
  actual captured run. If you change the command, re-capture the output — do not
  edit the transcript to match. Where a measurement could not be made honestly,
  the deck cites a published source instead; `lecture-7/CLAUDE.md` records one
  such case.
- **Do not publish the instructor's own network.** Captured `arp`, `ifconfig` or
  route output contains real MAC and private addresses. This repository is
  public; that material was deliberately left out of lecture 7.
- **Check external links resolve.** Requests to `.ir` hosts may need the proxy
  stripped (`--noproxy '*'`).

## The teaching arc

The decks are ordered as one argument and cross-reference each other by number.
Breaking a link breaks the sequence:

1. **1 → 2** — lecture 1 ends on _name the property, name the threat, then pick
   the mechanism_; lecture 2 turns that into X.800 services and policy.
2. **2 → 3** — nearly every X.800 mechanism is cryptographic.
3. **3 → 4** — the toolbox, then who is allowed to use what.
4. **5 → 6** — the buffer overflow and SQL injection are the same bug, one layer
   apart. Lecture 6's Golden Rule slide names lecture 5 explicitly.
5. **5 → 7** — the base-rate fallacy is why egress filtering is a good signal.
6. **7** — the closing slide closes the course, repeating lecture 1's sentence
   with _and know what it does not do_ added.

If you add or drop a lecture, fix the syllabus slide in lecture 1, the README,
and any deck that refers to it by number.
