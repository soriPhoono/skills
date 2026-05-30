---
name: frontmatter-linter
description: Lint the entire Obsidian vault for frontmatter correctness, broken [[wikilinks]], orphan pages, stale content, and cross-system inconsistencies. Agnostic across daily notes, wiki pages, raw sources, and kanban boards.
---

# Frontmatter Linter

Comprehensive linting of the Obsidian vault, covering YAML frontmatter validation, graph integrity, and cross-system consistency. Detects which note system a file belongs to and applies the correct schema rules.

## When to Use

- **Routine vault maintenance:** "lint the vault", "check frontmatter", "run health check"
- **Before git commits:** "check for issues before I commit"
- **After batch ingest:** "audit what we just created"
- **Stale content review:** "find stale pages", "what needs updating"

## How It Works

The linter scans every `.md` note in the vault, determines its **note system** from path and `type:` field, then applies the correct validation rules. Results are grouped by severity and written to `LLM-Wiki/outputs/lint-YYYY-MM-DD.md`.

## Note System Detection

| Detection Method | System | Schema |
|---|---|---|
| `type: daily` in frontmatter OR path starts with `Daily/` | **Daily note** | Daily note schema |
| `type: concept\|entity\|source-summary\|comparison` OR path starts with `LLM-Wiki/wiki/` | **Wiki page** | Wiki page schema (subtype check) |
| Path starts with `LLM-Wiki/raw/` | **Raw source** | Raw source bookmark schema |
| Path matches `Projects/*.kanban.md` | **Kanban** | Minimal — valid YAML only |
| Anything else (`AGENTS.md`, templates, root files) | **Other** | Valid YAML if frontmatter present |

## Check Categories

### 1. Frontmatter Validation (per Schema)

For each note, extract YAML frontmatter and validate against the detected schema:

#### Daily Note Schema (`type: daily` or `Daily/` path)

| Field | Required | Validation |
|---|---|---|
| `title` | Yes | Must match `YYYY-MM-DD` date format |
| `type` | Yes | Must equal `daily` |
| `tags` | Yes | Must be a YAML array |

#### Wiki Page Schema (`type: concept | entity | source-summary | comparison` or `LLM-Wiki/wiki/` path)

**All wiki subtypes share these base fields:**

| Field | Required | Validation |
|---|---|---|
| `title` | Yes | Non-empty string |
| `type` | Yes | Must be one of: `concept`, `entity`, `source-summary`, `comparison` |
| `created` | Yes | Must be a valid date `YYYY-MM-DD` |
| `updated` | Yes | Must be a valid date `YYYY-MM-DD` |
| `confidence` | Yes | Must be one of: `high`, `medium`, `low` |
| `status` | Yes | Must be one of: `draft`, `reviewed`, `stale` |

**Subtype-specific fields:**

| `type` | Required Fields | Notes |
|---|---|---|
| `concept` | `sources` (array), `related` (array with `[[wikilinks]]`) | — |
| `entity` | `sources` (array), `related` (array with `[[wikilinks]]`) | — |
| `source-summary` | `source_url` (URL), `source_file` (path), `author`, `date_published`, `date_ingested`, `concepts` (array), `entities` (array) | `source_file` must point to an existing `raw/` path |
| `comparison` | `sources` (array), `related` (array with `[[wikilinks]]`) | — |

**Cross-field checks for wiki pages:**
- `created` must be ≤ `updated`
- `source_file` must be a path under `LLM-Wiki/raw/` that actually exists on disk
- `related:` entries must use `[[wikilink]]` syntax (`[[path/to/page]]`), not bare text
- `concepts:` and `entities:` entries (in source-summaries) must use `[[wikilink]]` syntax

#### Raw Source Schema (`LLM-Wiki/raw/` path)

| Field | Required | Validation |
|---|---|---|
| `type` | Yes | Must be one of: `article`, `paper`, `repo`, `data`, `image` |
| `url` | Yes | Must be a valid URL |
| `title` | Yes | Non-empty string |
| `author` | Yes | Non-empty string |
| `date_published` | Yes | Valid date `YYYY-MM-DD` |
| `date_accessed` | Yes | Valid date `YYYY-MM-DD` |
| `tags` | Yes | Must be a YAML array |

**Cross-field checks for raw sources:**
- `date_accessed` must be today or in the past (not future-dated)
- File must be in correct type subdirectory (`articles/`, `repos/`, `papers/`, `data/`, `images/`) matching its `type:` field

#### Kanban Schema (`Projects/*.kanban.md`)

Minimal check — only validates that YAML is parseable. No required fields.

#### Other Notes (AGENTS.md, templates, etc.)

If frontmatter exists, validate it's parseable YAML. No schema enforcement.

### 2. Graph Integrity

After frontmatter passes, scan note content for `[[wikilinks]]` and verify:

| Check | Method | Issue Severity |
|---|---|---|
| **Broken links** | Extract all `[[target]]` from content; verify a note with that path exists | Error |
| **Target format mismatch** | Wikilinks use `[[section]]` (internal heading) vs `[[note]]` (file) — detect wrong format | Warning |
| **Orphan pages** | Page with zero incoming `[[wikilinks]]` from other pages | Info |
| **Dead-end pages** | Page with zero outgoing `[[wikilinks]]` (excluding tags and sections) | Info |

**How to check link existence:**
1. Extract `[[link]]` patterns from note content with regex `\[\[([^\]]+)\]\]`
2. Strip anchors (`page#section` → `page`), display text (`page|text` → `page`)
3. For each target, use `obsidian_search_notes` with the target as query to verify at least one result exists
4. For path-style wikilinks (`concepts/containerd/containerd/Containerd`), check the exact path with `obsidian_read_note` (tolerating 404)

### 3. Staleness Detection

| Check | Method | Issue Severity |
|---|---|---|
| **Stale daily notes** | `Daily/` notes with last frontmatter `updated` > 7 days ago | Info |
| **Stale wiki pages** | `updated` > 30 days ago AND `status: draft` | Warning |
| **Stale wiki pages (reviewed)** | `updated` > 90 days ago AND `status: reviewed` | Info |
| **Daily note incomplete** | Daily note missing one or more required sections (`## Summary`, `## TOC`, `## General todo`, `## Worklog`) | Warning |

### 4. Cross-System Consistency

| Check | Method | Issue Severity |
|---|---|---|
| **Topic mismatch: raw/ exists but no wiki/** | `raw/<topic>/` directory exists but no `wiki/concepts/<topic>/` | Warning |
| **Topic mismatch: wiki/ exists but no raw/** | `wiki/concepts/<topic>/` exists but no `raw/<topic>/` | Info |
| **Empty topic** | Topic folder exists but contains < 3 total pages across all wiki sections | Info |
| **Source_file dead** | Wiki page `source_file:` references a file that doesn't exist in `raw/` | Error |
| **raw/ type misalignment** | Source filed under `articles/` but `type: repo` in frontmatter (or vice versa) | Error |

## Workflow

### Step 1: Inventory the Vault

1. Use `obsidian_get_vault_stats` to get total note count and modified dates
2. Use `obsidian_search_notes` with empty/ broad query to paginate all notes (iterate through results)
3. Alternatively, walk directories with `obsidian_list_directory` on `/` recursively

Build an in-memory map of:
- All note paths
- Each note's frontmatter (use `obsidian_get_frontmatter` per note)
- Each note's content (use `obsidian_read_note`)

### Step 2: Classify Each Note

For each note, determine its system:
1. Read frontmatter with `obsidian_get_frontmatter`
2. Check `type:` field:
   - `daily` → daily note schema
   - `concept`, `entity`, `source-summary`, `comparison` → wiki page schema
3. If no `type:` field, infer from path:
   - Starts with `Daily/` → daily note schema
   - Starts with `LLM-Wiki/raw/` → raw source schema
   - Starts with `LLM-Wiki/wiki/` → wiki page schema (warn: missing `type`)
   - Matches `Projects/*.kanban.md` → kanban schema
   - Otherwise → other

Record classification for each note.

### Step 3: Run Frontmatter Checks

For each note, validate fields against its schema using the tables above. Collect findings:

```
path:field — ERROR: description
path:field — WARNING: description
path — INFO: description
```

### Step 4: Run Graph Integrity Checks

Scan all note content for `[[wikilinks]]`:

1. **Collect all outgoing links** per page
2. **Verify each link target exists** via `obsidian_search_notes`
3. **Calculate incoming link counts** per page
4. **Identify orphans** (zero incoming links, excluding AGENTS.md, templates, and index/log pages)

### Step 5: Run Staleness Checks

Compare `updated` dates against today's date.

### Step 6: Run Consistency Checks

Compare `raw/` topic directories against `wiki/` topic directories.

### Step 7: Write Report

Compose the full lint report and write it to `LLM-Wiki/outputs/lint-YYYY-MM-DD.md` using `obsidian_write_note`:

```markdown
---
title: "Lint Report YYYY-MM-DD"
type: lint-report
created: YYYY-MM-DD
---

# Lint Report: YYYY-MM-DD

## Summary

| Category | Errors | Warnings | Info |
|---|---|---|---|
| Frontmatter | N | N | N |
| Graph Integrity | N | N | N |
| Staleness | — | N | N |
| Consistency | N | N | N |
| **Total** | **N** | **N** | **N** |

## By System

| System | Notes Scanned | Issues |
|---|---|---|
| Daily notes | N | N |
| Wiki pages | N | N |
| Raw sources | N | N |
| Kanbans | N | N |
| Other | N | N |

## Errors

### path:field — Description
...

## Warnings

...

## Info

...
```

### Step 8: Present Results

Show a concise summary to the user with the totals and the top 5 most severe findings. Offer to fix auto-fixable issues:

- **Auto-fixable:** Missing `created`/`updated` dates (set to today), non-kebab-case tags (rename), malformed `[[wikilinks]]` syntax (fix bracket pairs)
- **Requires judgment:** Orphan pages (keep or delete?), stale content (refresh or mark stale?), broken links (redirect or create missing page?)

Ask: "Shall I auto-fix the N auto-fixable issues?"

## Edge Cases

- **No frontmatter at all** → flag as info (common for scratch notes, not an error)
- **Binary/canvas files** → skip (`.excalidraw`, `.canvas`, PDFs are not markdown)
- **Templates** (`Daily/_templates/`) → skip frontmatter validation, they use template variables
- **AGENTS.md files** → classify as "other", only check parseable YAML
- **Empty `sources:` array** → warning for concept/entity pages (should have at least one source)
- **`related:` with bare text instead of `[[wikilinks]]`** → error for wiki pages
- **Future-dated `date_accessed`** → error for raw sources
