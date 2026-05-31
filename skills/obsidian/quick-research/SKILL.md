---
name: quick-research
description: Lightweight research workflow for the LLM Wiki. Given a topic, search the web, create raw source bookmarks, write a source summary and concept page, and update the index and log. Good for "what is X?" queries.
---

# Quick Research

Lightweight, single-pass research for the LLM Wiki. Given a topic, the skill searches the web, creates source bookmarks in `raw/`, writes a structured source summary and concept page in `wiki/`, and updates the master index and operation log.

**Before activating this skill**, search the wiki thoroughly (see [Step 1](#step-1-determine-state)). If the wiki already covers the topic, no research is needed.

## When to Use — vs `deep-research`

| Use Case | Tool |
|---|---|
| **Simple "what is X?"** — one tool, one concept, no category context needed | ✅ `quick-research` |
| **Bounded topic** — e.g., "How does k0s control plane work?" as a single concept | ✅ `quick-research` |
| **Scouting** — exploring a topic before deciding if full research is warranted | ✅ `quick-research` |
| **Category + sub-topic** — e.g., researching `kubernetes/k3s` with category context and comparisons | ❌ Use `deep-research` |
| **Connected topic** — new topic that cross-references heavily with existing wiki topics | ❌ Use `deep-research` |
| **Entity pages or comparisons needed** — the topic is a project/organization or needs comparison with peers | ❌ Use `deep-research` |
| **Single URL to ingest** — you have a specific URL, not a topic | ❌ Use `source-ingest` |

**Quick rule of thumb:** If you only need a concept page + source summaries (no entities, no comparisons, no category-level infrastructure), use `quick-research`. If you need the full two-level treatment with cross-references, use `deep-research`.

## Input

| Input | Description | Example |
|---|---|---|
| `{{topic}}` | The subject to research | `k3s`, `containerd-shim`, `argo-cd`, `netbird` |

If the topic has multiple words, convert to kebab-case for file paths: `Container Runtime Interface` → `container-runtime-interface`.

## Workflow

### Step 1: Determine State — Search the Wiki First

**Before doing any external research, exhaust the wiki.** The information may already exist. Run ALL of these checks:

1. **Read the index:** `obsidian_read_note("LLM-Wiki/wiki/index.md")` — check if `{{topic}}` appears in any topic table
2. **Search vault-wide with multiple keyword variations:**
   - `obsidian_search_notes("{{topic}}")` — exact topic name
   - `obsidian_search_notes("{{topic}}", searchContent: true)` — broader content match
   - Try synonyms or related terms if initial searches return nothing
3. **Walk topic directories:**
   - `obsidian_list_directory("LLM-Wiki/raw/")` — check which categories exist
   - `obsidian_list_directory("LLM-Wiki/wiki/concepts/")` — look for a matching concept folder
   - `obsidian_list_directory("LLM-Wiki/wiki/sources/")` — check for existing source summaries
4. **If you find candidate pages, read them:** `obsidian_read_note` on each to assess whether they already cover what you need

**If the wiki already has sufficient coverage**, skip research entirely and report what exists — no need to re-research.

**If the topic does not exist** but is a simple, well-bounded concept, proceed with this `quick-research` workflow.

**If the topic is actually a broad category** (e.g., "research Kubernetes networking" — spans many sub-topics, needs comparisons and entities), stop and use `deep-research` instead. See the decision table in [When to Use](#when-to-use---vs-deep-research).

If the top-level category is ambiguous after searching, ask the user: "Which category does `{{topic}}` belong to?" (e.g., `kubernetes`, `containerd`, `gitops`, `docker`).

### Step 2: Research the Topic

1. **Search with Exa** using `exa_web_search_exa`:
   - Query: `{{topic}} architecture how it works overview`
   - numResults: 5
   - Prioritize official documentation, well-regarded technical blogs, and project pages

2. **Fetch full content** from the best 2-3 results using `exa_web_fetch_exa` or `fetch_fetch`:
   - max_length: 8000 per source

3. **Extract key information:**
   - Core definition and purpose
   - Architecture and components
   - Key features and capabilities
   - Related concepts and technologies

### Step 3: Determine Topic Placement

Decide which `raw/<category>/` directory the topic belongs in:

- If the user specified a category, use that
- If the topic maps clearly to an existing category (e.g., `containerd-shim` → `containerd/`), use it
- If the topic spans categories or is new, create a new category (ask the user first if uncertain)

### Step 4: Create Source Bookmarks

For each fetched source, create a raw source bookmark via `obsidian_write_note`:

**Path:** `LLM-Wiki/raw/{{category}}/articles/{{source-domain}}-{{article-slug}}.md`

**Frontmatter:**
```yaml
---
type: article | repo | paper | data | image
url: <source URL>
title: "<Source Title>"
author: <Author or Organization>
date_published: YYYY-MM-DD
date_accessed: <today's date>
tags: [{{category}}, {{topic}}]
---
```

**Content:** Raw bookmark with a brief curator's note:
```markdown
> Curator's note: <1-2 sentences summarizing the source's relevance and content>
```

**Type directory placement:** Place the bookmark in the type subdirectory matching its `type:` field:
- `type: article` → `raw/<category>/articles/`
- `type: repo` → `raw/<category>/repos/`
- `type: paper` → `raw/<category>/papers/`
- `type: data` → `raw/<category>/data/`
- `type: image` → `raw/<category>/images/`

### Step 5: Create Source Summary

Create a source summary in `wiki/` via `obsidian_write_note`:

**Path:** `LLM-Wiki/wiki/sources/{{category}}/{{topic}}-{{source-slug}}.md`

**Frontmatter:**
```yaml
---
title: "<Source Title>"
type: source-summary
source_url: <URL>
source_file: LLM-Wiki/raw/{{category}}/<type>/<filename>.md
author: <Author>
date_published: YYYY-MM-DD
date_ingested: <today>
concepts:
  - "[[concepts/{{category}}/{{topic}}/{{PascalCaseTopic}}]]"
entities: []
created: <today>
updated: <today>
confidence: high
status: draft
---
```

**Content:**
```markdown
## TL;DR
<1-2 sentence summary>

## Key Takeaways
1. <Takeaway with supporting detail>
2. <Takeaway with supporting detail>

## New Concepts Discovered
- [[concepts/{{category}}/{{topic}}/{{PascalCaseTopic}}]] — <brief description>
```

### Step 6: Create Concept Page

Create a concept page in `wiki/` via `obsidian_write_note`:

**Path:** `LLM-Wiki/wiki/concepts/{{category}}/{{topic}}/{{PascalCaseTopic}}.md`

Also create the `graphics/` and `sources/` subdirectories (note: MCP doesn't support empty dirs well — create a `.gitkeep` or just note them).

**Frontmatter:**
```yaml
---
title: "{{Topic Title}}"
type: concept
sources:
  - LLM-Wiki/raw/{{category}}/articles/<filename>.md
related:
  - "[[related-concept-1]]"
  - "[[related-concept-2]]"
created: <today>
updated: <today>
confidence: medium
status: draft
---
```

**Content structure:**
```markdown
## Definition

<Clear, concise definition of the concept>

## Key Ideas

- <Core insight 1>
- <Core insight 2>
- <Core insight 3>

## Architecture

<If applicable — structured explanation of how it works>

## Relationship to Other Concepts

- [[concepts/{{category}}/related-concept/RelatedConcept]] — <how they relate>
- [[concepts/other-category/other-concept/OtherConcept]] — <cross-topic relationship>

## Sources

- [Source Title](source URL) — raw/{{category}}/articles/<filename>.md
```

### Step 7: Check and Create Category Infrastructure

If `{{category}}` does not yet exist as a topic in the wiki, the source summary and concept page creation above will create the topic subfolders implicitly (because `obsidian_write_note` creates parent paths).

However, also create the standard category directories:
- `LLM-Wiki/raw/{{category}}/articles/`
- `LLM-Wiki/raw/{{category}}/repos/`

Use `obsidian_write_note` to create a `.gitkeep` placeholder if needed (or just skip — topic dirs will exist once pages are written).

### Step 8: Update Index and Log

**Update `LLM-Wiki/wiki/index.md`:**
- Read the current index with `obsidian_read_note`
- Add the new concept entry to the Concepts table (under the correct topic)
- Add the new source summary entry to the Sources table
- Update statistics (total pages, topic counts)
- Write with `obsidian_write_note` in overwrite mode

**Update `LLM-Wiki/wiki/log.md`:**
- Read the current log with `obsidian_read_note`
- Append a new entry:
  ```markdown
  ---

  ## <today> — Quick Research: {{topic}}

  **Type:** Quick research

  **Sources ingested:**
  - `raw/{{category}}/articles/<filename>.md` — "<Title>" by <Author>

  **Pages created:**
  - `wiki/sources/{{category}}/<filename>.md`
  - `wiki/concepts/{{category}}/{{topic}}/{{PascalCaseTopic}}.md`

  **Noteworthy:** <key findings or insights>

  **Affected pages:** N created
  ```
- Write with `obsidian_write_note` in append mode

### Step 9: Present Results

Show a clean summary:

```markdown
## Quick Research Complete: {{topic}}

### Pages Created
- `raw/{{category}}/articles/<source>.md` — "<Title>"
- `wiki/sources/{{category}}/<summary>.md` — Source summary
- `wiki/concepts/{{category}}/{{topic}}/{{PascalCaseTopic}}.md` — Concept page

### Key Takeaways
1. <brief highlight>
2. <brief highlight>

### Next Steps
- Run `frontmatter-linter` to validate the new pages
- Run `organize-raw-sources` to audit source placement
- Run `wiki-index-regenerator` to rebuild the full index
- If the topic turns out to be broader than expected, run `deep-research` to expand with full category infrastructure, entities, and comparisons
```

## Templates Reference

Embedded here for portability (these match the LLM Wiki standard conventions):

### Raw Source Bookmark
```yaml
---
type: article | repo | paper | data | image
url: <URL>
title: "<Title>"
author: <Author>
date_published: YYYY-MM-DD
date_accessed: YYYY-MM-DD
tags: [tag1, tag2]
---
```

### Source Summary
```yaml
---
title: "<Title>"
type: source-summary
source_url: <URL>
source_file: raw/<category>/<type>/<file>.md
author: <Author>
date_published: YYYY-MM-DD
date_ingested: YYYY-MM-DD
concepts:
  - "[[concepts/<category>/<page>/<Page>]]"
entities: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: high | medium | low
status: draft | reviewed | stale
---
```

### Concept Page
```yaml
---
title: "<Concept Name>"
type: concept
sources:
  - raw/<category>/<type>/<file>.md
related:
  - "[[concepts/<category>/<page>/<Page>]]"
created: YYYY-MM-DD
updated: YYYY-MM-DD
confidence: high | medium | low
status: draft | reviewed | stale
---
```

## Edge Cases

- **Topic already well-covered in wiki** → skip research, show what exists
- **No Exa results** → fall back to `fetch_fetch` with Wikipedia and known good URLs
- **Ambiguous topic** (e.g., "CRI" could be Container Runtime Interface or a file format) → ask the user for clarification
- **Category doesn't exist** → create it automatically (raw/ topic dirs + wiki topic dirs)
- **Topic with spaces** → kebab-case for paths, Title Case for page titles: `container runtime interface` → paths: `container-runtime-interface`, title: "Container Runtime Interface"
- **User provides a URL instead of a topic** → delegate to `source-ingest` skill
- **Wiki already covers the topic thoroughly** → do not research; report what exists and continue the task
- **Wiki has partial coverage** → run `quick-research` to fill specific gaps rather than re-researching the whole topic; check which sub-topics are missing and target only those
- **Topic discovered mid-research to be broader than expected** (e.g., "CRI" turns out to span Kubernetes and containerd) → stop `quick-research` and hand off to `deep-research` for the full two-level treatment
