# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **static publish target**, not an application. It is the GitHub Pages site for 영닥터 (a
family-medicine physician and mission-organization board member), serving book notes, lecture
handouts, event reports and official letters at `wiki.youngdoctor.kr`.

There is **no build system, no test suite, no linter, no package manifest, and no CI workflow** in
this repo — deliberately. Every `.html` file here is generated output that was committed from
elsewhere. `.nojekyll` disables Jekyll so GitHub Pages serves the files verbatim; `CNAME` holds the
custom domain. Pages deploys from `main`.

Do not invent build/test commands. If a task seems to need one, the tooling is not here.

## The generator lives outside this repo

Pages are produced by an external tool ("marktl") that runs against an Obsidian vault on the
author's machine. `index.json` records this in each item's `sourcePath`, e.g.
`wiki/books/결혼건축가_래리크랩.md` (some older entries leaked an absolute
`/Users/isanghyeob/.../04_Obsidian_Vault/...` path instead — both forms appear and both are fine).

Consequences that matter:

- The markdown sources are **not in this repository**. You cannot regenerate a page from source here.
- Asked to change a page's content, you must edit the generated HTML directly — and say so, because
  the next regeneration from the vault will overwrite it unless the author also edits the source.
- Each note page is **fully self-contained**: all CSS is inlined in a `<style>` block, the only
  external dependency is a Google Fonts link. There is no shared stylesheet to edit.

## The 6-file publish unit — the central invariant

Publishing one note is an atomic change to **six** files. A publish that updates fewer leaves the
site inconsistent. For slug `S` and short id `ID`:

| File | Role |
|---|---|
| `marktl/<S>/index.html` | the note page itself (self-contained) |
| `marktl/s/<ID>/index.html` | short-link: meta-refresh redirect + OG tags |
| `marktl/index.json` | catalog ledger entry |
| `marktl/index.html` | main catalog — **data is baked in**, not fetched |
| `marktl/topics/index.html` | topics dashboard — also baked in |
| `marktl/topics/b/<S>/index.html` | the book/topic page for this note |

`index.json` is a **manifest, not a runtime data source**. Neither `index.html` nor
`topics/index.html` fetches it; both embed their data at generation time (`const blocks=[…]`).
Editing `index.json` alone changes nothing a visitor sees — the dashboards must be regenerated too.

## URLs and the reversed canonical convention

This trips people up, so read carefully — `index.json`'s field names are the **opposite way round**
from the HTML tags:

- `index.json` `url` → the **short-link** (`…/marktl/s/<ID>/`)
- `index.json` `canonicalUrl` → the **slug page**, percent-encoded (`…/marktl/%EA%B2%B0…/`)
- In `marktl/<S>/index.html`, `<link rel="canonical">` and `og:url` point at the **short-link**
- In `marktl/s/<ID>/index.html`, canonical/`og:url` point back at the **slug page**, plus
  `<meta http-equiv="refresh">` to it

So the note page names the short link as canonical, and the short link redirects to the note page.
This is the established pattern across all pages — match it rather than "fixing" it.

## Ledgers and vocabulary

Two ledgers, with **no overlap** in slug or shortId:

- `marktl/index.json` — listed items; shape `{version, updatedAt, items[]}`
- `marktl/_unlisted.json` — items deliberately kept out of the catalog and dashboards

Item fields: `slug`, `shortId`, `url`, `canonicalUrl`, `sourcePath`, `artifactType`, `title`,
`excerpt`, `tags`, `updatedAt`, and **optionally** `layout` and `palette` — roughly an eighth of
entries omit those two, so always read them with a default.

- `artifactType` — overwhelmingly `research-report`; also `leaflet`, `faithful-note`, `dashboard`,
  `collection`
- `layout` — `review` (most), `publish`, `read`, `dashboard`, `custom`
- `palette` — a single name or a `a+b` pair drawn from `navy, teal, ochre, copper, crimson, indigo,
  forest, sage, slate, plum, rose, terracotta`. The palette is not a class name: it is baked into
  each page as CSS custom properties on `:root` (`--accent`, `--hero-start/mid/end`, `--pq`, …).
  To restyle a page, edit those variables in its own `<style>` block.
- `tags` — namespaced, `domain/…`, `discipline/…`, `topic/…`

### Other namespaces

- `marktl/topics/t/<tag>/` — one page per tag. Far more of these than there are notes.
- `marktl/topics/b/<book>/` — book pages. These are `noindex` and exist for books *referenced* in
  notes, not only for published ones, so **most have no `index.json` entry**. That is expected, not drift.
- `marktl/x/<name>/` — collections. These use an **empty `shortId`** and carry the `x/` prefix inside
  `slug` (e.g. `x/영아부`). Skip empty shortIds in any short-link check.

## Commit conventions

Match the existing subject lines exactly, including the trailing colon on `publish:`:

```
publish: <slug> [<shortId>] (<layout>/<palette>):
publish (unlisted, custom): <slug>
update: topics/books dashboard + note pages rebuild
chore: retrigger pages deploy (<slug>)
```

Bodies are normally empty. Re-publishing the same slug repeatedly is routine and expected.

## Verifying a change

With no test suite, the meaningful check is ledger↔disk consistency. Run from the repo root:

```bash
python3 - <<'PY'
import json, os
os.chdir('marktl')
ledger = json.load(open('index.json'))['items'] + json.load(open('_unlisted.json'))['items']
bad = []
for it in ledger:
    sid, slug = it['shortId'], it['slug']
    if sid and not os.path.isfile(f's/{sid}/index.html'):
        bad.append(f'missing short-link  s/{sid}/  ({slug})')
    if not os.path.isfile(f'{slug}/index.html'):
        bad.append(f'missing note page   {slug}/  (shortId {sid or "-"})')
seen = {}
for it in ledger:
    if it['shortId']:
        seen.setdefault(it['shortId'], []).append(it['slug'])
bad += [f'duplicate shortId   {s} -> {v}' for s, v in seen.items() if len(v) > 1]
bad += [f'orphan short-link   s/{o}/' for o in sorted(set(os.listdir('s')) - {i["shortId"] for i in ledger})]
print('\n'.join(bad) if bad else 'all consistent')
print(f'--- {len(bad)} issue(s); {len(ledger)} ledger entries')
PY
```

Preview locally with `python3 -m http.server 8000` from the repo root, then open
`http://localhost:8000/marktl/`. Note that paths in the pages are absolute
`…github.io/allpeaceclinic/…` URLs, so navigation will jump to the live site.

### Known drift (pre-existing, as of the last audit)

The check above reports three stale `index.json` entries whose directory was later renamed, so their
`canonicalUrl` 404s while the short link still resolves to the dead URL:

| stale `slug` in `index.json` | actual directory on disk |
|---|---|
| `5무교회가온다-황인권` | `5무교회가온다` |
| `의료현장에서제자도를어떻게실천하는가` | `의료현장제자도실천` |
| `정체성-identity` | `정체성` |

A durable fix belongs in the external generator, not in hand-edited JSON. Treat these three as
expected output of the check; anything **beyond** them is new breakage introduced by the change at hand.

## Writing conventions

Content is Korean throughout — titles, slugs, directory names, tags and `excerpt` text. Keep
`word-break:keep-all` on Korean text blocks (it prevents mid-word wrapping) and keep `lang="ko"` on
`<html>`. Slugs and directory names are unencoded Korean on disk; only the URLs in JSON are
percent-encoded.
