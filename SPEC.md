# Phase 1 — Spec

Status: **draft for review**. No code until this is approved.

## 1. Vault schema

### 1.1 Directory layout

```
00-meta/
  README.md            # human-readable overview of the vault
  schema.md            # canonical frontmatter spec (this section, exported)
  tags.md              # tag + topic taxonomy
  indexes/             # generated indexes (Phase 2+); empty for now
10-sources/
  medium/              # one note per saved Medium article
  oreilly/             # reserved for Phase 2; created empty with .gitkeep
20-mocs/               # maps of content (manual)
30-notes/              # my own syntheses (manual)
```

The skeleton ships these directories and the three `00-meta/` files. Everything else is created by the extractor or by hand.

### 1.2 Frontmatter

Every note in `10-sources/**` has this frontmatter. YAML, in the order below for diff stability.

| Field            | Type            | Required | Managed by  | Notes                                                                 |
|------------------|-----------------|----------|-------------|-----------------------------------------------------------------------|
| `title`          | string          | yes      | extractor   | Source title; quotes only when needed.                                |
| `authors`        | list[string]    | yes      | extractor   | Empty list `[]` if unknown — never omit the field.                    |
| `source`         | enum            | yes      | extractor   | `medium \| oreilly \| manual \| web`.                                  |
| `url`            | string          | yes      | extractor   | Canonical URL (after Medium tracking-param strip). Identity key.      |
| `type`           | enum            | yes      | extractor   | `article \| book \| paper \| video \| podcast \| thread \| other`.    |
| `date_published` | date \| null    | yes      | extractor   | ISO `YYYY-MM-DD`; `null` when unknown.                                |
| `date_added`     | date            | yes      | extractor   | When the item entered the vault (creation only; never overwritten).   |
| `date_read`      | date \| null    | yes      | user        | Filled by hand.                                                       |
| `status`         | enum            | yes      | mixed       | `queued \| reading \| read \| archived \| dropped`. Default `queued`. |
| `topics`         | list[string]    | yes      | user        | Controlled vocabulary (see `tags.md`). May be `[]`.                   |
| `tags`           | list[string]    | yes      | mixed       | Obsidian tags. Auto-tags are additive; user tags preserved.           |
| `rating`         | int 1–5 \| null | yes      | user        | `null` when unrated.                                                  |
| `summary`        | string \| null  | yes      | extractor*  | One-paragraph summary. `null` until populated. \*See §1.4.            |

Identity key: `url`. The extractor must locate an existing note by `url` before deciding to create vs. update.

### 1.3 Body template

```markdown
## Summary

<one paragraph; from frontmatter `summary` or empty>

## Highlights

- <user-owned>

## Notes

<user-owned>

> [!quote]- Full text
> <full extracted markdown, blockquoted line by line>
```

The `> [!quote]-` callout is collapsed by default in Obsidian; it remains plain markdown for LLM/MCP consumers.

### 1.4 Managed vs. user-owned fields

Re-runs are idempotent (success criterion). To make that safe:

- **Managed (always rewritten on re-run):** `title`, `authors`, `source`, `url`, `type`, `date_published`, full-text callout body.
- **Create-only (written once, never touched again):** `date_added`.
- **User-owned (preserved verbatim if present):** `date_read`, `status` (unless still `queued` and a new extraction succeeds — see below), `topics`, `rating`, the `## Highlights` and `## Notes` sections.
- **Mixed:**
  - `tags`: extractor maintains a set of auto-tags (e.g. `source/medium`, `type/article`); user-added tags are preserved. On re-run we recompute auto-tags and merge with the existing tag list minus any auto-tags that no longer apply.
  - `summary`: written by extractor on first create. Re-run only overwrites if the user hasn't edited it. We detect user edits by storing the auto-generated summary's hash in a hidden frontmatter field `_summary_hash`; if the live summary's hash matches, we may regenerate, otherwise leave it.
  - `status`: if currently `queued` and the re-run successfully extracts a body (was a stub before), bump to `queued` still — i.e. no automatic transition. Status changes are user-driven; the extractor only sets the initial value.

### 1.5 Slug + filename

Format: `YYYY-MM-DD-short-title.md`.

- Date component: `date_added` (stable across re-runs; survives republished sources).
- Title component: lowercased, ASCII-folded, non-alphanumerics → `-`, collapsed runs, trimmed; truncated to 60 chars on a word boundary.
- Collisions (same date + slug): append `-2`, `-3`, … deterministically.
- Renaming: filenames are fixed at creation. If `title` changes upstream, the filename does not move (avoids breaking `[[wikilinks]]`). `title` in frontmatter still updates.

### 1.6 Tag + topic taxonomy

`topics` is a small controlled vocabulary, free to grow but documented in `00-meta/tags.md`. Initial seed:

```
ml, llm, infra, distsys, security, leadership, product, career, tools, math, writing, work
```

`tags` are Obsidian-native (no spaces, `/` for hierarchy). Reserved namespaces:

- `source/<name>`        — set by extractor (`source/medium`).
- `type/<name>`          — set by extractor (`type/article`).
- `domain/work`          — manual marker for confidentiality-relevant notes.
- `status/queued|read|…` — mirrors `status` for Obsidian search; **not** maintained automatically in Phase 1 to avoid drift. User toggles by hand if wanted.

Free user tags are anything not in the reserved namespaces.

## 2. Medium extractor

### 2.1 CLI

```
uv run medium-import \
  --zip <path-to-medium-export.zip> \
  --vault <path-to-vault-root> \
  [--html-cache <dir>] \
  [--dry-run] \
  [--limit N] \
  [-v]
```

- `--vault` points at the directory containing `10-sources/`.
- `--html-cache` (optional) is a directory of pre-saved article HTML files. If present, the extractor will use those for full-text extraction in addition to anything in the ZIP. This is the only way to get full text in Phase 1; no live fetching.
- `--dry-run` prints planned actions, writes nothing.
- `--limit N` processes only the first N bookmarks (useful for sample runs).
- Exit non-zero on any unhandled error; partial successes are committed.

### 2.2 ZIP layout assumed

Based on a current Medium "Download your information" export. The extractor must validate these files exist and fail fast with a clear message if not (so we can adjust):

- `bookmarks/bookmarks.html` — list of saved articles. Source of truth for the set.
- `lists/*.html` — custom lists (treated as additional bookmarks; deduped by URL).
- `claps/claps.html` — optional; if present, used to seed `rating` heuristic (see §2.5). Off by default behind a flag, but documented.
- `profile/profile.html` — used only to set the export's "exported on" date for logs.

**Open question for review:** does your ZIP actually include any cached article bodies? My working assumption is **no** — the export gives URLs + titles + dates only, so every bookmark becomes a stub unless `--html-cache` supplies the HTML. Confirm before implementation.

### 2.3 Pipeline

Per bookmark:

1. **Parse bookmark entry** → `{url, title?, date_added?}`. Strip Medium tracking params (`?source=…`, `?sk=…`, `gi=…`) to get a canonical URL.
2. **Locate existing note** by canonical `url` via the URL→path index (built once at start by reading frontmatter from `10-sources/medium/*.md`).
3. **Try to extract full text:**
   - If `--html-cache/<sha1(url)>.html` exists → load it.
   - Else if the ZIP contains an article body for that URL (only if §2.2 open question resolves "yes") → load it.
   - Else → no body available, this is a stub.
4. **HTML → markdown:**
   - Primary: `trafilatura.extract(html, output_format="markdown", include_links=True, include_images=False)`.
   - Fallback: if trafilatura returns empty/None, run `markdownify` on the parsed `<article>` (or `<main>`, or `<body>`).
   - Stub if both yield nothing.
5. **Derive metadata:**
   - `authors`: from extractor output > meta tag `author` > Medium-specific `<a rel="author">` > `[]`.
   - `date_published`: from extractor output > `<time datetime>` > `null`.
   - `summary`: first 280-char prose paragraph, sentence-trimmed; `null` for stubs.
   - `type`: hard-coded `article` for Medium in Phase 1.
   - `tags`: `[source/medium, type/article]`.
6. **Build the note** (create or update — see §2.4) and write atomically (write to `*.md.tmp`, fsync, rename).

### 2.4 Idempotent upsert

- **Create path:** new file, full template, `date_added = today`, `status = queued`, `_summary_hash` set if summary present.
- **Update path:**
  - Recompute managed fields from the source.
  - Read existing file, parse frontmatter + body sections.
  - For user-owned fields/sections (§1.4): copy through unchanged.
  - For `tags`: `(existing_tags - old_auto_tags) ∪ new_auto_tags`. Old auto-tags are recognized by reserved namespaces (`source/`, `type/`).
  - For `summary`: rewrite only if `_summary_hash` matches the existing summary's hash.
  - For the full-text callout: always rewrite.
- **Stub → populated transition:** treated as a normal update. `status` stays `queued`. A log line is emitted: `upgraded stub: <url>`.
- **Populated → stub:** never. If a re-run can't extract but a previous run did, keep the old body. Log a warning.

The "second run produces zero new files, zero duplicate-content warnings" success criterion is enforced by:
- Index lookup before create.
- Byte-identical output for unchanged inputs (deterministic YAML key order, stable list ordering, LF line endings, no trailing whitespace).

### 2.5 Stubs

A stub note is a valid note with:

- `status: queued`
- `summary: null`
- `## Notes` section contains: `> TODO: full text not extractable; paywalled or not in cache.`
- No `> [!quote]-` callout (omitted entirely; re-runs may add it later).

### 2.6 Logging + reporting

End-of-run summary to stdout:

```
medium-import: 412 bookmarks, 387 notes (38 created, 349 updated), 25 stubs, 0 errors
```

`-v` adds per-bookmark lines. Errors per bookmark do not abort the run; they're collected and reprinted at the end with the offending URL.

### 2.7 Dependencies

Pinned in `pyproject.toml` (managed by `uv`):

- `trafilatura`
- `markdownify`
- `beautifulsoup4` (already a trafilatura dep, but used directly for bookmark parsing)
- `python-frontmatter` for YAML frontmatter round-tripping (preserves order with `ruamel.yaml` under the hood; verify before commit).
- `click` for the CLI.
- Test-only: `pytest`, `pytest-snapshot` (or inline string asserts — TBD), `responses` not needed since no HTTP.

Python 3.11+. No optional system deps.

### 2.8 Project layout

```
pyproject.toml
src/
  medium_import/
    __init__.py
    cli.py
    bookmarks.py        # parse Medium bookmark/list HTML
    extract.py          # HTML → markdown via trafilatura/markdownify
    note.py             # frontmatter + body model, render, parse
    upsert.py           # index, create-or-update, atomic write
    slug.py
    tags.py
tests/
  fixtures/
    sample_bookmark.html
    sample_article.html
    sample_paywalled.html
    existing_note.md
  test_slug.py
  test_note.py
  test_upsert.py
  test_extract.py
  test_bookmarks.py
```

## 3. Test list

Unit tests, all hermetic (no network, tmp_path for FS):

1. **`test_slug.py`**
   - ASCII-only title slugs correctly.
   - Unicode title (e.g. `"Café résumé — naïveté"`) folds to ASCII.
   - Long title (>60 chars) truncates on a word boundary.
   - Punctuation-only / empty title falls back to `untitled`.
   - Same date + slug collision yields `-2`, `-3` deterministically.

2. **`test_note.py`**
   - Frontmatter round-trip: parse → serialize is byte-identical for a canonical sample.
   - Field order is fixed (matches §1.2 table).
   - `null` fields render as `null`, not omitted.
   - `authors: []` renders as `[]`, not omitted.
   - Full template render matches a golden string for a known input.
   - Body section parse: extracts `## Summary`, `## Highlights`, `## Notes`, full-text callout independently.

3. **`test_upsert.py`**
   - Create: new URL writes a new file with expected name and content.
   - Update — managed fields rewrite (changed `title` upstream → file updates).
   - Update — user fields preserved (`rating`, `date_read`, `## Notes` content, user-added tag `domain/work`).
   - Update — auto-tags refreshed (old auto-tag removed if no longer applicable; new auto-tag added).
   - Update — `summary` preserved when user-edited (`_summary_hash` mismatch).
   - Update — `summary` regenerated when `_summary_hash` matches.
   - Stub upgrade: existing stub gains full-text callout, status stays `queued`.
   - Populated → no-extract: previous body retained, warning emitted.
   - Re-run on identical input is byte-identical (idempotency).
   - Two bookmarks with same canonical URL but different tracking params resolve to one note.

4. **`test_extract.py`**
   - Known sample HTML → expected markdown (golden).
   - Trafilatura empty → markdownify fallback returns non-empty for the same fixture (a deliberately tricky one).
   - Both empty → returns `None` cleanly, no exception.
   - Author / `date_published` extraction from meta tags.
   - Summary truncation: ends on sentence boundary, ≤ 280 chars.

5. **`test_bookmarks.py`**
   - Parses Medium `bookmarks.html` fixture into expected list of `(url, title, date_added)`.
   - Strips tracking params on canonicalization.
   - Deduplicates across `bookmarks.html` + a `lists/*.html` fixture.
   - Malformed entry → skipped with a warning, doesn't abort parsing.

Manual smoke test (post-suite): run `medium-import --vault <real vault> --zip <real ZIP> --dry-run` then without dry-run; eyeball 3 sample notes in Obsidian (frontmatter parses, callout collapses, links intact); re-run and confirm 0 created / 0 modified.

## 4. Open questions for you

1. **Article bodies in the ZIP** — does your Medium export include any HTML for saved articles, or only URLs + metadata? (Affects whether `--html-cache` is the only extraction path or merely the supplementary one.)
2. **`status/*` mirror tags** — do you want me to maintain them automatically despite the drift risk, or leave them off (current proposal)?
3. **`claps.html` → `rating` heuristic** — drop entirely from Phase 1? It's gimmicky.
4. **Vault root** — should the extractor accept `--vault` pointing at the vault root, or directly at `10-sources/medium/`? Current: vault root.
5. **Filename rename on title change** — current proposal locks filenames. Confirm or override.

Once you've signed off (or marked up edits), I'll implement against this spec.
