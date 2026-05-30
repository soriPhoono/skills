---
name: deep-research
description: Self-triggering deep research for the LLM Wiki. Activates when the agent finds insufficient information in the Obsidian vault. Given a category and topic, creates infrastructure, researches both levels, builds all page types (concepts, entities, sources, comparisons), writes cross-references, and updates the index and log. 
---

# Deep Research

Full two-level research for the LLM Wiki: given a broad `{{category}}` and a specific `{{topic}}`, researches both, creates infrastructure if needed, builds all page types (concept, entity, source summaries, comparisons), writes cross-references across the knowledge graph, and updates the master index and operation log. This skill is **self-triggering** — it activates automatically when the agent detects a knowledge gap in the Obsidian vault during task execution, not only when explicitly called.

For ingesting a single URL, see `source-ingest`.

## Trigger — Knowledge Gap Detection

This skill must be used when, during any task, the agent searches the LLM Wiki in the Obsidian vault and finds insufficient information about a concept, tool, technology, or domain. It is the primary research workflow for filling gaps between what the vault knows and what the agent needs.

### Detection Mechanism

When the agent requires information to complete a task:

1. **Consult the LLM Wiki first** — Search the index (`LLM-Wiki/wiki/index.md`) and relevant topic areas using `obsidian_search_notes` with the target topic as query; also check `obsidian_list_directory` on `LLM-Wiki/wiki/concepts/` for relevant topic directories
2. **Evaluate coverage** — Assess whether existing wiki pages adequately cover what you need: do they answer the question at hand? Are they detailed enough? Are sources recent?
3. **Trigger conditions** — If any of the following are true, proceed with the deep research workflow:

   | Condition | Description | Action |
   |---|---|---|
   | **No coverage** | The topic does not exist anywhere in the wiki | Full research: create category infrastructure + topic pages |
   | **Insufficient depth** | The topic exists but lacks architecture details, key concepts, or specific information needed | Enrichment mode: add sources, expand concept pages, create entity/comparison pages |
   | **Stale content** | Existing pages are marked with low confidence, draft status, or lack recent sources | Refresh: re-research, add up-to-date sources, update confidence |
   | **Connected topic** | You find related pages that hint at a concept but do not document it directly | Bridging research: create the missing topic and cross-link |

### Self-Triggering Flow

```
Agent encounters knowledge gap during task execution
    ↓
Agent searches LLM Wiki (obsidian_search_notes, obsidian_read_note on index.md)
    ↓
┌─ Coverage exists and is sufficient? ──→ Continue task (no research needed)
└─ Coverage insufficient (one of the 4 conditions above)?
         ↓
    Extract {{category}} and {{topic}} from the search context
         ↓
    Invoke this deep-research workflow from Step 1
         ↓
    Return results; agent resumes original task with enriched knowledge
```

### When NOT to Trigger

- The wiki already has comprehensive, up-to-date coverage of the topic
- The information is simple enough to answer from the agent's general knowledge (e.g., common programming language syntax)
- The user explicitly instructs you not to research the topic
- A single URL would provide the answer — use `source-ingest` instead
- The question is about the user's own codebase or project (not a general concept)

### Example Trigger Scenarios

| Agent Task | Wiki Search Result | Trigger? | Research Action |
|---|---|---|---|
| "Explain how k3s differs from k8s" | No results for k3s | ✅ No coverage | Create `kubernetes/k3s` topic |
| "What is CRI-O and how does it relate to containerd?" | Concept page for containerd exists but no CRI-O page | ✅ Connected topic | Create `kubernetes/cri-o` with cross-links to containerd |
| "Write a Terraform module that uses the Kubernetes provider" | Kubernetes concept page is thorough, Terraform page exists | ❌ Sufficient | Continue task |
| "How does runc work internally?" | containerd topic exists but runc is only mentioned in passing | ✅ Insufficient depth | Create `containerd/runc` enrichment |
| "What's new in Docker Compose v2?" | Docker Compose page dates from 2023 with confidence:low | ✅ Stale content | Refresh docker-compose topic |

## Inputs

These are **not provided explicitly by the user.** They are derived automatically from the context of the knowledge gap detected during the agent's trigger flow.

| Input | Description | How Derived |
|---|---|---|
| `{{category}}` | Broad domain or topic area (e.g., `kubernetes`, `containerd`, `gitops`) | Extracted from the search query that returned empty/insufficient results. If the gap concerns a sub-topic, the parent concept determines the category |
| `{{topic}}` | Specific subject within the category (e.g., `k3s`, `runc`, `argocd`) | Extracted from the specific entity, tool, or concept the agent was looking up |

If only one can be confidently inferred, derive the other:
- Known category + specific gap → the gap is the topic
- Specific tool/concept with clear parent → the parent is the category
- If ambiguous, make a reasonable best-guess and note the assumption in the research report

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
