---
name: organize-raw-sources
description: Audit and reorganize LLM-Wiki raw/ source bookmarks to ensure correct topic and type placement. Detects type/directory mismatches, broken source_file references, orphan bookmarks and summaries, missing frontmatter fields (status, confidence), and cross-category misplacements. Moves files to correct raw/<topic>/<type>/ paths and updates all source_file frontmatter references across wiki/sources/ pages.
---

# Organize Raw Sources

Audit and reorganize `raw/` source bookmarks and their corresponding `wiki/sources/` summaries. Ensures every source is in the correct category directory and type subdirectory (`articles/`, `repos/`, `papers/`, `data/`, `images/`), all frontmatter references are valid, and the bidirectional link between raw bookmarks and wiki summaries is intact.

For schema validation (frontmatter completeness, dates, wikilinks), see the `frontmatter-linter` skill. This skill focuses on **structural organization** — file placement, directory consistency, and cross-referencing integrity.

## When to Use

- **Periodic maintenance:** "organize raw sources", "audit raw directory", "check source placement"
- **After batch ingest:** "check that everything landed in the right place"
- **When the wiki topic tree evolves:** "I added a new category, see if any sources should move"
- **When the linter flags source_file errors:** "fix the broken source references"
- **Before pruning:** "find orphan sources that can be cleaned up"
- **During onboarding:** "check the state of the whole wiki source graph"

## How It Works

The organizer walks every file in `raw/<category>/<type>/` and `wiki/sources/<category>/`, builds a bidirectional inventory, and runs five validation passes:

| Pass | What It Checks | Severity |
|---|---|---|
| **Pass 1: Frontmatter Type vs Directory** | Every raw bookmark's `type:` field must match its enclosing `articles/`, `repos/`, `papers/`, `data/`, or `images/` directory | Error |
| **Pass 2: source_file Integrity** | Every wiki source summary's `source_file` must point to an existing raw bookmark | Error |
| **Pass 3: Orphan Detection** | raw bookmarks without a wiki summary (orphan bookmarks) and wiki summaries without a raw bookmark (orphan summaries) | Warning |
| **Pass 4: Topic Placement** | Sources that belong semantically in a different category than where they're filed | Warning |
| **Pass 5: Missing Standard Fields** | Wiki source summaries missing `status`, `confidence`, or bare-text `concepts`/`entities` that should be `[[wikilinks]]` | Warning |

Results are presented as a structured report with auto-fix proposals for Passes 1-3. Passes 4-5 require human judgment.

## Workflow

### Step 1: Inventory All Sources

Build a complete map of both the raw source bookmarks and wiki source summaries.

#### 1a. Walk raw/ Directory

List all category directories under `LLM-Wiki/raw/` using `obsidian_list_directory`. For each category, enumerate files in each type subdirectory:

| Type Subdirectory | Valid `type:` Values | Example |
|---|---|---|
| `articles/` | `article` | Blog posts, documentation, news |
| `repos/` | `repo` | GitHub/GitLab repositories |
| `papers/` | `paper` | Academic papers, PDF links |
| `data/` | `data` | Datasets, statistics, raw data |
| `images/` | `image` | Diagrams, screenshots, visual assets |

If a category is missing a type subdirectory, note it as an infrastructure gap (info-level).

For each file found, record:

```
raw/<category>/<type>/<filename>.md
  - category: <category>
  - actual_type_dir: <type>
  - frontmatter_type: <parsed from file>
  - source_url: <from frontmatter>
```

**Command pattern for each file:**
```
obsidian_get_frontmatter("LLM-Wiki/raw/<category>/<type>/<filename>.md")
```

#### 1b. Walk wiki/sources/ Directory

List all files under `LLM-Wiki/wiki/sources/<category>/` using `obsidian_list_directory` for each category.

For each file found, record:

```
wiki/sources/<category>/<filename>.md
  - category: <category>
  - source_file: <from frontmatter>
  - title: <from frontmatter>
  - status: <from frontmatter>
  - confidence: <from frontmatter>
  - concepts: <from frontmatter>
  - entities: <from frontmatter>
```

**Command pattern:**
```
obsidian_get_frontmatter("LLM-Wiki/wiki/sources/<category>/<filename>.md")
```

Build two maps from this inventory:

- **raw_index:** `{ "<filename>": { category, actual_type_dir, frontmatter_type, ... } }` — keyed by exact filename
- **wiki_index:** `{ "<filename>": { category, source_file_path, title, status, ... } }` — keyed by exact filename
- **source_file_map:** `{ "<source_file_path>": true }` — all source_file values from wiki summaries

### Step 2: Run Pass 1 — Frontmatter Type vs Directory

For each raw bookmark, compare `frontmatter_type` against `actual_type_dir`:

| Expected | Mismatch Example | Action |
|---|---|---|
| `type: article` in `articles/` | `type: repo` in `articles/` | Move file to `repos/` and update any `source_file` references |
| `type: repo` in `repos/` | `type: article` in `repos/` | Move file to `articles/` |
| `type: paper` in `papers/` | `type: article` in `papers/` | Move file to `articles/` |
| `type: data` in `data/` | `type: image` in `data/` | Move file to `images/` |

**Cross-reference check:** Also verify any raw file that has `type: repo` but is filed under `articles/` (or vice versa).

**Detection output:**
```
ERROR — raw/<category>/articles/example.md: type is 'repo' but filed under articles/
  → Move to raw/<category>/repos/example.md and update source_file references
```

**Auto-fix for Pass 1:**
1. Record the move: rename the file to the correct type subdirectory via `obsidian_move_file`
2. Search all wiki source summaries for `source_file` pointing to the old path using `obsidian_search_notes`
3. For each match, use `obsidian_patch_note` to update `source_file` to the new path
4. Note: `obsidian_move_file` requires `confirmOldPath` and `confirmNewPath` — use exact strings from the inventory

### Step 3: Run Pass 2 — source_file Integrity

For each wiki source summary, check that its `source_file` frontmatter field points to a file that actually exists in `raw/`.

The `source_file` value follows the format: `LLM-Wiki/raw/<category>/<type>/<filename>.md`

**Validation logic:**
1. Parse the path from `source_file` — it should be `LLM-Wiki/raw/<category>/<type>/<filename>.md`
2. Check if that exact file exists in the `raw_index` map
3. Also check the path exists by reading the file with `obsidian_read_note` (tolerating the error if it doesn't exist)
4. If the raw file exists but under a different category, flag as both a broken reference AND a possible topic misplacement

**Broken source_file causes:**
- Raw file was deleted without updating references
- Raw file was moved (rename not propagated)
- Typo in the source_file path
- Raw bookmark never created (source summary written without bookmark)

**Detection output:**
```
ERROR — wiki/sources/<category>/<file>.md: source_file 'LLM-Wiki/raw/<category>/<type>/<nonexistent>.md' not found
```

**Auto-fix candidates:**
- **File exists under a different type dir:** Propose updating source_file to the correct path
- **Typo in filename:** Search raw/ for files with similar names and propose the correct one
- **Missing raw bookmark entirely:** Flag as orphan summary — requires human judgment to create or remove

### Step 4: Run Pass 3 — Orphan Detection

#### 4a. Orphan Raw Bookmarks (raw/ exists but no wiki/sources/ summary)

For each file in `raw_index`, check if a corresponding file exists in `wiki_index` by matching filename.

**Matching logic:**
- A raw bookmark at `raw/<category>/articles/example.md` should have a corresponding wiki source summary at `wiki/sources/<category>/example.md`
- The filename must match exactly (both are derived from the same source during the ingest workflow)

**Detection output:**
```
INFO — raw/<category>/<type>/<file>.md: No corresponding wiki source summary found
```

**Action:** Present as info. The raw bookmark exists and is valid — the source summary may simply not have been written yet. Offer to:
- Create a source summary from the raw bookmark (skim the content or URL, write a brief summary)
- Or mark as "intentionally raw-only" (some bookmarks are placeholders)

#### 4b. Orphan Wiki Summaries (wiki/sources/ exists but no raw/ bookmark)

For each file in `wiki_index`, check that its `source_file` field points to an existing raw bookmark (reuses Pass 2 results). Additionally, check if ANY raw bookmark could correspond to the wiki summary — try matching by source URL.

**Detection output:**
```
WARNING — wiki/sources/<category>/<file>.md: source_file points to a non-existent raw bookmark; no raw bookmark found with matching source URL
```

**Action:** Present as warning. Three options:
1. **Create the missing raw bookmark** — fetch the URL from `source_url`, create the raw bookmark
2. **Remove the wiki summary** — if the source is no longer relevant
3. **Fix the path** — if the raw bookmark exists but was renamed

### Step 5: Run Pass 4 — Topic Placement (Semantic)

Check whether raw source bookmarks are filed under the correct category based on their content, tags, and cross-references.

**Heuristics for misplacement:**

| Signal | Example | Likely Correct Category |
|---|---|---|
| Raw tags include a different category | `tags: [podman, comparison]` filed under `raw/docker/` | Could be `docker/` or `podman/` — check wiki cross-references |
| Wiki concepts reference a different category | `concepts: [[concepts/podman/podman-architecture/...]]` but summary is under `docker/` | Source may bridge both — keep in dominant, cross-reference the other |
| Source URL domain belongs to a different ecosystem | GitHub repo `k3s-io/k3s` filed under `raw/docker/` instead of `raw/kubernetes/` | Move to `raw/kubernetes/` |
| Filename matches a different category's naming pattern | `docker-vs-podman-*.md` exists in both `docker/` and `podman/` | Check if duplicate; if not, file under the category that the source covers most |

**Bridge sources** (sources that legitimately span two categories, like a Docker vs Podman comparison):
- Keep the raw bookmark in the primary category
- Ensure the wiki source summary cross-references concepts in both categories
- File under the category of the PRIMARY subject (e.g., a comparison of vs Podman with Docker as the baseline → `docker/`)

**Detection output:**
```
WARNING — raw/<category>/<type>/<file>.md: tags include '<other-category>' but filed under '<category>'
  → Consider moving to raw/<other-category>/<type>/<file>.md
  → Or keep here and ensure wiki concepts cross-reference both categories
```

**Action:** Present for human judgment. Do NOT auto-fix topic misplacements — they require understanding the source's primary subject.

### Step 6: Run Pass 5 — Missing Standard Fields

For each wiki source summary, check for missing fields that the LLM Wiki schema requires:

| Field | Importance | What to Check |
|---|---|---|
| `status` | Required | Must be `draft`, `reviewed`, or `stale` |
| `confidence` | Required | Must be `high`, `medium`, or `low` |
| `concepts` | Recommended | Array of `[[wikilinks]]` — if present, entries must use `[[wikilink]]` syntax |
| `entities` | Optional | If present, entries must use `[[wikilink]]` syntax |

**Detection output:**
```
WARNING — wiki/sources/<category>/<file>.md: missing required field 'status'
INFO — wiki/sources/<category>/<file>.md: missing 'concepts' array (recommended for graph connectivity)
WARNING — wiki/sources/<category>/<file>.md: 'concepts' entry uses bare text 'Containerd' instead of [[wikilink]] syntax
```

**Auto-fix candidates:**
- Missing `status` → add `status: draft` (safe default for any wiki page)
- Missing `confidence` → add `confidence: medium` (conservative default)
- Bare text in `concepts`/`entities` → convert to `[[wikilink]]` syntax (only if you can confidently resolve the target)

**Do NOT auto-fix missing `concepts` or `entities` arrays** — these require understanding the source's subject matter.

### Step 7: Generate Report

Compose the full organization report and present it to the user:

```markdown
## Source Organization Report: YYYY-MM-DD

### Inventory Summary

| Category | Raw Bookmarks | Wiki Summaries | Matched | Orphans (raw) | Orphans (wiki) |
|---|---|---|---|---|---|
| containerd | 4 | 4 | 4 | 0 | 0 |
| kubernetes | 25 | 22 | 22 | 3 | 0 |
| cloud-storage | 18 | 7 | 7 | 11 | 0 |
| ... | ... | ... | ... | ... | ... |
| **Total** | **N** | **N** | **N** | **N** | **N** |

### Pass 1: Type/Directory Mismatches (N errors)

| File | Issue | Fix |
|---|---|---|
| raw/docker/articles/docker-vs-podman-techplained.md | type: 'article' → correct, filed in articles/ ✅ | — |

### Pass 2: Broken source_file References (N errors)

| Wiki Summary | source_file | Status |
|---|---|---|
| wiki/sources/.../file.md | raw/.../file.md | ✅ OK / ❌ Not found |

### Pass 3: Orphans

**Raw bookmarks without wiki summaries (N):**
- raw/.../file.md — could create summary or mark as intentional

**Wiki summaries without raw bookmarks (N):**
- wiki/sources/.../file.md — source_file missing, URL available, could recreate bookmark

### Pass 4: Topic Placement Flags (N warnings)

| Source | Current Category | Suggested | Reason |
|---|---|---|---|
| raw/docker/articles/docker-vs-podman-techplained.md | docker | bridge (docker + podman) | Tags mention both, concepts cross both categories |

### Pass 5: Missing Standard Fields (N warnings)

| Wiki Summary | Missing Fields |
|---|---|
| wiki/sources/containerd/youngju-containerd-architecture.md | status |

### Auto-Fix Proposal

The following can be fixed automatically:
1. Add `status: draft` to N source summaries
2. Move N raw files to correct type subdirectories
3. Update N source_file references after moves

Shall I proceed with the auto-fixes?
```

### Step 8: Execute Auto-Fixes (if confirmed)

If the user approves auto-fixes, execute them in order:

1. **Fix missing `status` fields:** For each wiki source summary missing `status`, use `obsidian_patch_note` to add it just before the closing `---`:
   - Add a line `status: draft` in the frontmatter block
   - **Pattern:** Insert after the last frontmatter field, before `---`

2. **Fix missing `confidence` fields:** Same pattern, add `confidence: medium`.

3. **Move raw files to correct type directories:**
   - Use `obsidian_move_file` to rename (old path → new path)
   - Requires `confirmOldPath` and `confirmNewPath` — use exact strings

4. **Update source_file references after moves:**
   - Search `wiki/sources/` for the old path with `obsidian_search_notes`
   - For each match, use `obsidian_patch_note` to replace the old path with the new path

**Important:** After each move + ref update batch, re-run Pass 2 to verify no references broke.

### Step 9: Update Index (if content changed)

If any raw files were moved or any wiki source summaries were modified, offer to run the `wiki-index-regenerator` skill to update the master index with accurate page counts.

## Templates Reference

### Raw Bookmark Structure

```
raw/<category>/<type>/<filename>.md
  type: article | repo | paper | data | image
  url: <URL>
  title: "<Title>"
  author: "<Author>"
  date_published: YYYY-MM-DD
  date_accessed: YYYY-MM-DD
  tags: [<category>, <topic>, ...]
```

### Wiki Source Summary Structure

```
wiki/sources/<category>/<filename>.md
  title: "<Title>"
  type: source-summary
  source_url: <URL>
  source_file: LLM-Wiki/raw/<category>/<type>/<filename>.md
  author: "<Author>"
  date_published: YYYY-MM-DD
  date_ingested: YYYY-MM-DD
  concepts:
    - "[[concepts/<category>/<page>/<Concept>]]"
  entities:
    - "[[entities/<category>/<entity>]]"
  created: YYYY-MM-DD
  updated: YYYY-MM-DD
  confidence: high | medium | low
  status: draft | reviewed | stale
```

### Directory-to-Type Mapping

| Directory | Valid `type:` Values | Typical Contents |
|---|---|---|
| `raw/<cat>/articles/` | `article` | Blog posts, news, docs, tutorials |
| `raw/<cat>/repos/` | `repo` | GitHub/GitLab READMEs, repo bookmarks |
| `raw/<cat>/papers/` | `paper` | arXiv links, PDFs, research papers |
| `raw/<cat>/data/` | `data` | Datasets, CSV stats, raw data dumps |
| `raw/<cat>/images/` | `image` | Diagrams, screenshots, architecture visuals |

## Edge Cases

- **Bridge sources** (span two categories, like Docker vs Podman) → keep raw in the primary category; cross-reference the secondary category in wiki concepts
- **Source migrated between categories** (e.g., k3s was reclassified from `docker/` to `kubernetes/`) → move the raw file AND all wiki references in one batch
- **Deep wiki source directories** (some categories may have sub-topics as subdirectories) → flatten to `wiki/sources/<category>/` — sub-topics go in category-level concept pages, not nested source dirs
- **No raw bookmark exists but URL is still valid** → offer to create the raw bookmark (use `source-ingest` skill)
- **No URL, only raw metadata** (rare — manual bookmark without URL) → validate the rest, skip URL checks
- **Category not in raw/ but appears in wiki/sources/** (emergent topic) → offer to create the raw/ category infrastructure (directories)
- **Empty type subdirectories** (e.g., `papers/` exists but empty) → report as info: "infrastructure present, no files"
- **Obsidian canvas/Excalidraw files in raw/** → skip (non-markdown files are not source bookmarks)
- **File with no frontmatter** → flag as error: raw bookmarks must have frontmatter per the LLM Wiki schema
