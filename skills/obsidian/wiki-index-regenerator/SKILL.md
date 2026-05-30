---
name: wiki-index-regenerator
description: Rebuild LLM-Wiki/wiki/index.md from the actual filesystem — re-detect topic folders, recount pages per section, refresh statistics, and update the timestamp. Ensures the master catalog reflects the true state of the wiki.
---

# Wiki Index Regenerator

Rebuilds `LLM-Wiki/wiki/index.md` by walking the actual wiki directory structure, reading frontmatter from every page, and composing accurate topic tables, page counts, and statistics.

## When to Use

- **After batch ingest:** "update the index", "regenerate index"
- **After reorganization:** "I moved some pages, rebuild the index"
- **Stale index suspicion:** "is the index up to date?", "rebuild index"
- **Periodic maintenance:** "refresh the wiki index"
- **Pre-commit check:** "update index before I commit"

## Workflow

### Step 1: Inventory the Filesystem

Walk the four wiki page-type directories using `obsidian_list_directory`:

| Directory | Page Type | Expected Structure |
|---|---|---|
| `LLM-Wiki/wiki/concepts/` | concept | `<topic>/<page-name>/<PascalCase.md>`, `graphics/`, `sources/` |
| `LLM-Wiki/wiki/entities/` | entity | `<topic>/<page-name>.md` |
| `LLM-Wiki/wiki/sources/` | source-summary | `<topic>/<source-name>.md` |
| `LLM-Wiki/wiki/comparisons/` | comparison | `<topic>/<comparison-name>/<PascalCase.md>`, `graphics/`, `sources/` |

For each directory:

1. List top-level entries — these are **topic folders** (e.g., `containerd/`, `kubernetes/`, `gitops/`)
2. For each topic folder, list its contents — these are **page folders** (for concepts/comparisons) or **page files** (for entities/sources)
3. For concept and comparison page folders, read the PascalCase `.md` file inside (the actual writeup)

### Step 2: Read Frontmatter Per Page

For each discovered page, use `obsidian_get_frontmatter` to read:

- `title` — the page title
- `type` — confirm it matches the directory (concept/entity/source-summary/comparison)
- `created` — creation date
- `updated` — last modified date
- `confidence` — high/medium/low
- `status` — draft/reviewed/stale (if present)

Track these statistics:
- Total pages per type (concept, entity, source-summary, comparison)
- Total pages per topic
- Average confidence per topic
- Number of stale/draft/reviewed pages
- Topics with < 3 pages (emergent / low-coverage)

### Step 3: Detect Topics

From the directory walk, compile a list of all discovered topics (directory names under each `<type>/`).

A topic is considered **active** if it has ≥ 1 page in any of the four sections (concepts, entities, sources, comparisons).

Detect **topics with pages in some sections but missing others** — e.g., topic has sources but no concept page. Flag these for the report but still include them in the index.

### Step 4: Build the Index Content

Compose the new `index.md` with the following structure:

```markdown
---
title: "Wiki Index"
type: index
updated: YYYY-MM-DD
total_pages: N
topics: N
---

# Wiki Index

Last updated: YYYY-MM-DD

## Statistics

| Metric | Count |
|---|---|
| Total pages | N |
| Total topics | N |
| Concepts | N |
| Entities | N |
| Sources | N |
| Comparisons | N |
| Stale pages | N |
| Draft pages | N |
| Reviewed pages | N |

## Topics

| Topic | Concepts | Entities | Sources | Comparisons | Total |
|---|---|---|---|---|---|
| `containerd/` | 3 | 1 | 3 | 1 | 8 |
| `kubernetes/` | 6 | 1 | 3 | 0 | 10 |
| ... | ... | ... | ... | ... | ... |
| **Total** | **N** | **N** | **N** | **N** | **N** |

## Concepts

### containerd/

| Page | Created | Updated | Confidence | Status |
|---|---|---|---|---|
| [[concepts/containerd/containerd/Containerd\|Containerd]] | 2026-05-29 | 2026-05-29 | high | reviewed |
| ... | ... | ... | ... | ... |

### kubernetes/

...

## Entities

### containerd/

| Page | Created | Updated | Confidence |
|---|---|---|---|
| [[entities/containerd/containerd-project\|containerd-project]] | ... | ... | high |

...

## Source Summaries

### containerd/

| Page | Author | Published | Ingested |
|---|---|---|---|
| [[sources/containerd/containerd-github-readme\|containerd-github-readme]] | containerd | 2026-05-29 | 2026-05-29 |

...

## Comparisons

### containerd/

| Page | Created | Updated | Confidence |
|---|---|---|---|
| [[comparisons/containerd/container-runtimes/Container-Runtimes-Containerd-Vs-CRI-O-Vs-Docker\|Container Runtimes]] | ... | ... | medium |

...
```

**Wikilink format:** Use full topic-prefixed paths matching the vault's `[[wikilink]]` convention:
- Concepts: `[[concepts/<topic>/<page-folder>/<PascalCase-file>]]`
- Entities: `[[entities/<topic>/<entity-file>]]`
- Sources: `[[sources/<topic>/<source-file>]]`
- Comparisons: `[[comparisons/<topic>/<comparison-folder>/<PascalCase-file>]]`

### Step 5: Write the Index

Use `obsidian_write_note` to overwrite `LLM-Wiki/wiki/index.md` with the rebuilt content.

**Safety:** First write to a temporary path (e.g., `LLM-Wiki/wiki/index-rebuild-preview.md`) and show the user a diff summary before overwriting the real index.

### Step 6: Present Changes

Show a concise report of what changed:

```markdown
Index Regenerated — YYYY-MM-DD

Previously: 52 pages, 6 topics
Now:        55 pages, 7 topics (+3 pages, +1 topic)

Changes detected:
- New topic: `gitops/` (3 pages)
- New concept: `wiki/concepts/gitops/gitops.md`
- New concept: `wiki/concepts/gitops/argocd.md`
- New comparison: `wiki/comparisons/gitops/argocd-vs-fluxcd.md`
- No topics removed

Page counts by topic:
- containerd: 8 (same)
- kubernetes: 10 (same)
- gitops: 3 (NEW)
- ...
```

## Edge Cases

- **Empty wiki** (no pages at all) → create a minimal index with `Total pages: 0`
- **Page without frontmatter** → include in count but mark as "⚠ missing frontmatter" in the table
- **Page with invalid frontmatter** → include but flag as "⚠ unparseable frontmatter"
- **Orphan topic folders** (empty directory with no pages) → skip, don't create a topic row
- **Topic naming collision** → two topics with same name but different case → treat as one, warn
- **Obsidian canvas files** (`.canvas`) → skip (they're not markdown pages)
- **Template files or AGENTS.md** found inside wiki/ → skip, they're not content pages
