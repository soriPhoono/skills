---
name: deep-research
description: Full two-level research workflow for the LLM Wiki. Given a category and topic, create infrastructure, research both levels, build all page types (concepts, entities, sources, comparisons), write cross-references, and update the index and log. MCP-portable version of categorical-research.
---

# Deep Research

Full two-level research for the LLM Wiki: given a broad `{{category}}` and a specific `{{topic}}`, researches both, creates infrastructure if needed, builds all page types (concept, entity, source summaries, comparisons), writes cross-references across the knowledge graph, and updates the master index and operation log.

For lighter single-topic research, see `quick-research`. For ingesting a single URL, see `source-ingest`.

## When to Use

- **New domain research:** "research Kubernetes, specifically k3s"
- **Deep enrichment:** "research containerd covering runc"
- **Category + topic pairs:** any request specifying both a broad category and a specific subject within it

## Inputs

| Input | Description | Example |
|---|---|---|
| `{{category}}` | Broad domain or topic area | `kubernetes`, `containerd`, `gitops`, `docker` |
| `{{topic}}` | Specific subject within the category | `k3s`, `runc`, `argocd`, `docker-compose` |

If the user provides only one input, ask for the missing one.

## Workflow

### Step 1: Determine State

Check what already exists in the wiki:

1. **Check category infrastructure:** Use `obsidian_list_directory` on `LLM-Wiki/raw/`, `LLM-Wiki/wiki/concepts/`, `LLM-Wiki/wiki/sources/` to see if `{{category}}` subdirectories exist
2. **Read the index:** `obsidian_read_note` on `LLM-Wiki/wiki/index.md` to check if `{{category}}` appears in topic tables
3. **Search for topic pages:** `obsidian_search_notes` with `{{topic}}` to find any existing notes

Record the state:
- **New category, new topic** — neither exists anywhere
- **Existing category, new topic** — category has pages, topic does not
- **Existing category, existing topic** — both exist (enrichment mode)

### Step 2: Create Category Infrastructure (if new)

If `{{category}}` does not exist as a topic:

1. **Create directory structure** — use `obsidian_write_note` to create placeholder notes establishing the topic directories:
   - `LLM-Wiki/raw/{{category}}/articles/.gitkeep`
   - `LLM-Wiki/raw/{{category}}/repos/.gitkeep`
   - `LLM-Wiki/wiki/concepts/{{category}}/`
   - `LLM-Wiki/wiki/entities/{{category}}/`
   - `LLM-Wiki/wiki/sources/{{category}}/`
   - `LLM-Wiki/wiki/comparisons/{{category}}/`

   (Obsidian MCP creates parent paths implicitly when writing notes.)

2. **Research the category broadly** — use `exa_web_search_exa`:
   - Query: `{{category}} architecture how it works internals overview`
   - numResults: 5
   - Fetch top 2-3 results with `exa_web_fetch_exa` (max_length: 12000)

### Step 3: Read Existing Content (Enrichment Mode)

If category already has wiki pages:

1. **Read all concept pages** under `LLM-Wiki/wiki/concepts/{{category}}/` — use `obsidian_list_directory` recursively, then `obsidian_read_note` on each
2. **Read all source summaries** under `LLM-Wiki/wiki/sources/{{category}}/`
3. **Read the category concept page** to understand what's already covered
4. **Note gaps** — what aspects of the category are not yet documented

### Step 4: Research the Category

1. Search with Exa: `{{category}} architecture how it works internals key concepts` (numResults: 5)
2. Fetch top 2-3 results in full
3. Extract:
   - Core definition and purpose
   - Architecture and components
   - Key features and capabilities
   - Relationship to other existing wiki topics

### Step 5: Create Category Source Bookmarks

For each fetched source, create a raw source bookmark via `obsidian_write_note`:

**Path:** `LLM-Wiki/raw/{{category}}/{{type}}/{{domain}}-{{slug}}.md`

**Frontmatter:**
```yaml
---
type: article | repo
url: <URL>
title: "<Title>"
author: "<Author>"
date_published: YYYY-MM-DD
date_accessed: <today>
tags: [{{category}}]
---
```

**Content:**
```markdown
> Curator's note: <1-2 sentences on relevance>
```

### Step 6: Create Category Source Summaries

For each source bookmark, create a source summary:

**Path:** `LLM-Wiki/wiki/sources/{{category}}/{{domain}}-{{slug}}.md`

**Frontmatter:**
```yaml
---
title: "<Title>"
type: source-summary
source_url: <URL>
source_file: LLM-Wiki/raw/{{category}}/{{type}}/{{filename}}.md
author: "<Author>"
date_published: YYYY-MM-DD
date_ingested: <today>
concepts:
  - "[[concepts/{{category}}/{{category}}/{{PascalCategory}}]]"
entities:
  - "[[entities/{{category}}/{{entity}}]]"
created: <today>
updated: <today>
confidence: high
---
```

**Content:**
```markdown
## TL;DR

<1-2 sentence summary>

## Key Takeaways

1. <Takeaway>
2. <Takeaway>
```

### Step 7: Create Category Concept Page

Create the broad category concept page:

**Path:** `LLM-Wiki/wiki/concepts/{{category}}/{{category}}/{{PascalCategory}}.md`

**Frontmatter:**
```yaml
---
title: "{{Category Title}}"
type: concept
sources:
  - LLM-Wiki/raw/{{category}}/articles/<source1>.md
  - LLM-Wiki/raw/{{category}}/articles/<source2>.md
related:
  - "[[concepts/related-category/related-topic/RelatedTopic]]"
created: <today>
updated: <today>
confidence: medium
status: draft
---
```

**Content:**
```markdown
## Definition

<Clear definition of {{category}}>

## Architecture

<Structured explanation of core architecture>

## Key Ideas

- <Core insight 1>
- <Core insight 2>

## Relationship to Other Concepts

- [[concepts/other-category/topic/Topic]] — <relationship>

## Sources

- [Source Title](URL) — raw/{{category}}/articles/<file>.md
```

### Step 8: Research the Specific Topic

Now drill into the specific `{{topic}}`:

1. **Search with Exa:** `{{category}} {{topic}} how it works architecture` (numResults: 5)
2. **Fetch full content** from best 2-3 results (max_length: 12000)
3. **Cross-reference against existing wiki content:**
   - Does this topic connect to other categories? (e.g., `k3s` → `containerd`, `kubernetes`, `docker-swarm`)
   - Does it relate to entities already in the wiki?

### Step 9: Create Topic Source Bookmarks

Same as Step 5, but tagged with both `{{category}}` and `{{topic}}`:

**Path:** `LLM-Wiki/raw/{{category}}/{{type}}/{{topic}}-{{slug}}.md`

Tag with: `[{{category}}, {{topic}}]`

### Step 10: Create Topic Source Summaries

Same pattern as Step 6, with `concepts:` pointing to both the category page and the new topic page.

### Step 11: Create Topic Concept Page

**Path:** `LLM-Wiki/wiki/concepts/{{category}}/{{topic}}/{{PascalTopic}}.md`

Include:
- `graphics/` subdirectory (for canvas diagrams — note its existence in the report)
- `sources/` subdirectory (for source reference stubs)

Frontmatter and content follow the same templates as Step 7.

### Step 12: Create Entity Page (if warranted)

If the topic is a project, organization, or person (not just an abstract concept):

**Path:** `LLM-Wiki/wiki/entities/{{category}}/{{entity}}.md`

```yaml
---
title: "<Entity Name>"
type: entity
sources:
  - LLM-Wiki/raw/{{category}}/articles/<source>.md
related:
  - "[[concepts/{{category}}/{{topic}}/{{PascalTopic}}]]"
created: <today>
updated: <today>
confidence: medium
status: draft
---
```

**Content:**
```markdown
## Overview

<Who/what this entity is>

## Key Contributions / Products

- <Notable work or output>

## Related Entities

- [[entities/other-category/other-entity]] — <relationship>

## Sources

- [Title](URL) — raw/{{category}}/articles/<file>.md
```

### Step 13: Create Comparison Page (if warranted)

If the topic is commonly compared with another tool or concept already in the wiki:

**Path:** `LLM-Wiki/wiki/comparisons/{{category}}/{{topic}}-vs-{{other}}/{{PascalComparison}}.md`

```yaml
---
title: "{{Topic}} vs {{Other}}"
type: comparison
sources:
  - LLM-Wiki/raw/{{category}}/articles/<source>.md
related:
  - "[[concepts/{{category}}/{{topic}}/{{PascalTopic}}]]"
  - "[[concepts/{{category}}/{{other}}/{{PascalOther}}]]"
created: <today>
updated: <today>
confidence: medium
status: draft
---
```

**Content:**
```markdown
| Dimension | {{Topic}} | {{Other}} |
|---|---|---|
| <Dimension 1> | ... | ... |

## When to Use {{Topic}}
<Scenarios>

## When to Use {{Other}}
<Scenarios>

## Verdict
<Synthesis>
```

Only create a comparison if there's a natural counterpart. Don't force it.

### Step 14: Write Cross-References

Ensure all new pages connect to the existing knowledge graph:

1. **Concept pages:** `related:` frontmatter must include at least 2-3 related concepts from the same or other topics
2. **Entity pages:** Link to concept pages they relate to
3. **Source summaries:** List concepts and entities the source covers
4. **Comparisons:** Reference both concepts being compared

Use full topic-prefixed wikilinks: `[[concepts/kubernetes/k3s/K3s]]`, not `[[k3s]]`.

Search for existing pages in other topics that should link back to the new pages — add `[[wikilinks]]` to their `related:` frontmatter.

### Step 15: Update Index and Log

**Update `LLM-Wiki/wiki/index.md`:**
- Read current index
- Add new entries to Concepts, Entities, Sources, and Comparisons tables (under correct topics)
- Update statistics (total pages, per-topic counts)
- Write

**Update `LLM-Wiki/wiki/log.md`:**
- Append a log entry:
  ```markdown
  ---

  ## <today> — Deep Research: {{category}} / {{topic}}

  **Type:** Deep research

  **Sources ingested:**
  - `raw/{{category}}/articles/<file>.md` — "<Title>" by <Author>
  - ...

  **Pages created:**
  - `wiki/sources/{{category}}/<file>.md`
  - `wiki/concepts/{{category}}/{{topic}}/{{PascalTopic}}.md`
  - `wiki/entities/{{category}}/<entity>.md`  (if created)
  - `wiki/comparisons/{{category}}/<comparison>.md`  (if created)

  **Pages enriched:**
  - `wiki/concepts/{{category}}/{{category}}/{{PascalCategory}}.md` — added related concept links

  **Cross-references added:**
  - [[concepts/{{category}}/{{topic}}/{{PascalTopic}}]] ← linked from [[concepts/other-category/topic/Topic]]

  **Noteworthy:** <key architectural insights, connections discovered>
  ```
- Write in append mode

### Step 16: Present Results

**DO NOT auto-commit.** Present a structured summary:

```
## Deep Research Complete: {{category}} / {{topic}}

### Infrastructure
- Created new topic: `{{category}}/` (raw/ + wiki/ directories)

### Sources Ingested (N)
- raw/{{category}}/articles/<file1>.md — "<Title>"
- raw/{{category}}/articles/<file2>.md — "<Title>"

### Wiki Pages Created (N)
- wiki/sources/{{category}}/<file1>.md
- wiki/concepts/{{category}}/{{topic}}/{{PascalTopic}}.md
- wiki/entities/{{category}}/<entity>.md
- wiki/comparisons/{{category}}/<comparison>.md

### Pages Enriched
- wiki/concepts/{{category}}/{{category}}/{{PascalCategory}}.md — added sections: Relationship to Other Concepts

### Cross-References Added
- [[concepts/{{category}}/{{topic}}/{{PascalTopic}}]] ← [[concepts/containerd/containerd/Containerd]]
- [[concepts/{{category}}/{{topic}}/{{PascalTopic}}]] ← [[concepts/kubernetes/k3s/K3s]]

### Key Findings
1. <Architectural insight>
2. <Connection discovered>

Shall I run frontmatter-linter on the new pages?
```

Wait for user confirmation before taking further action.

## Templates Reference

All templates follow the LLM Wiki standard conventions. Key formats used throughout this skill:

### Page Types & Required Frontmatter

| Type | Required Fields |
|---|---|
| `concept` | `title`, `type: concept`, `sources` (array), `related` (array), `created`, `updated`, `confidence`, `status` |
| `entity` | `title`, `type: entity`, `sources` (array), `related` (array), `created`, `updated`, `confidence` |
| `source-summary` | `title`, `type: source-summary`, `source_url`, `source_file`, `author`, `date_published`, `date_ingested`, `concepts` (array), `entities` (array), `created`, `updated`, `confidence` |
| `comparison` | `title`, `type: comparison`, `sources` (array), `related` (array), `created`, `updated`, `confidence` |

### Raw Source Types

| `type` | Directory |
|---|---|
| `article` | `raw/<category>/articles/` |
| `repo` | `raw/<category>/repos/` |
| `paper` | `raw/<category>/papers/` |
| `data` | `raw/<category>/data/` |
| `image` | `raw/<category>/images/` |

## Edge Cases

- **Category + topic already well-documented** → skip creation, focus on enrichment: add new sources, update cross-references, improve existing pages
- **Bridge topic** (spans two categories, like CRI between kubernetes and containerd) → place in the category that defines the contract, cross-reference heavily to the implementing category
- **Low-quality search results** → try alternative queries, ask user for specific URLs if still poor
- **Topic with spaces** → kebab-case for file paths: `docker swarm` → `docker-swarm`
- **Duplicate source detected** → don't recreate; add to existing source summary's Key Takeaways
- **Category exists but has no concept page** → create the category concept page and link existing entity/source pages to it
- **User declines enrichment proposals** → skip, note in report
