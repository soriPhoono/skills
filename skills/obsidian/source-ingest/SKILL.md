---
name: source-ingest
description: Ingest a single URL into the LLM Wiki. Fetch the content, create a raw source bookmark, write a source summary, cross-link to existing concept pages, and update the index and log.
---

# Source Ingest

Process a single URL or source into the LLM Wiki knowledge base. Fetches the content, creates a durable raw source bookmark, writes a structured source summary, cross-references it with existing concept and entity pages, and appends to the operation log.

For full multi-source research on a topic, see the `quick-research` skill.

## When to Use

- **Single article:** "add this URL to the wiki", "ingest this source"
- **Paper or documentation:** "save this paper", "bookmark this doc"
- **GitHub repo:** "add this repo as a source"
- **After finding something interesting:** "save this for later"

## Input

| Input | Description | Example |
|---|---|---|
| `{{url}}` | The URL to ingest (required) | `https://example.com/article` |
| `{{topic}}` | Optional — topic for categorization | `kubernetes` |
| `{{category}}` | Optional — broad category | `containerd`, `gitops` |

If `{{topic}}` is not provided, infer it from the page content and URL. If `{{category}}` is not provided, ask the user or infer from the topic.

## Workflow

### Step 1: Fetch and Analyze the Source

1. **Fetch the URL** using `exa_web_fetch_exa` or `fetch_fetch` with max_length 12000
2. **Extract metadata:**
   - `title` — from page title or first heading
   - `author` — from byline or site name
   - `date_published` — from article date or metadata
   - `site` — domain name (for file naming)
3. **Classify the source type:**
   - Blog post / documentation → `article`
   - GitHub / GitLab repo → `repo`
   - Academic paper / PDF → `paper`
   - Dataset → `data`
   - Image / diagram → `image`

### Step 2: Determine Topic and Category

1. Check if the user provided a topic or category
2. If not, search existing wiki content with `obsidian_search_notes` using keywords from the article
3. Look at existing `raw/<category>/` directories with `obsidian_list_directory` on `LLM-Wiki/raw/`
4. If still ambiguous, ask: "Which category does this source belong to?"

**Topic assignment rules:**
- A source about a specific tool → the tool's topic (e.g., `k3s` → `kubernetes/`)
- A source about a general concept → the concept's topic (e.g., `GitOps` → `gitops/`)
- A source spanning multiple topics → pick the dominant one, cross-reference later
- A source about something new → create a new topic (ask the user)

### Step 3: Check for Duplicates

1. Use `obsidian_search_notes` to search for the URL in existing raw/ bookmarks
2. If the URL already exists, inform the user: "This source is already in the wiki at `raw/<path>`."
3. Offer to update the existing source summary instead of creating a duplicate

### Step 4: Create Raw Source Bookmark

Create a raw bookmark via `obsidian_write_note`:

**Naming:** `{{domain}}-{{article-slug}}.md`
- Domain: extract from URL (e.g., `kubernetes-docs`)
- Slug: from the path (e.g., `architecture-overview`)
- Example: `kubernetes-docs-architecture-overview.md`

**Path:** `LLM-Wiki/raw/{{category}}/{{type}}/{{filename}}.md`

**Frontmatter:**
```yaml
---
type: {{type}}        # article | repo | paper | data | image
url: "{{url}}"
title: "{{title}}"
author: "{{author}}"
date_published: {{date}}
date_accessed: <today>
tags: [{{category}}, {{topic}}]
---
```

**Content:**
```markdown
> **Curator's note:** <1-2 sentences — why this source is relevant, what it covers>

## Key Excerpts

> <Notable quote or data point from the source>
> <Another notable excerpt>
```

### Step 5: Identify Related Wiki Pages

1. Use `obsidian_search_notes` to find concept pages related to this source by searching keywords from the article
2. Read the index (`LLM-Wiki/wiki/index.md`) to identify relevant topics
3. Build lists of:
   - **Related concepts** — existing concept pages this source informs
   - **Related entities** — existing entity pages this source mentions
   - **Related comparisons** — existing comparisons this source could add to

### Step 6: Create Source Summary

Create a source summary via `obsidian_write_note`:

**Path:** `LLM-Wiki/wiki/sources/{{category}}/{{filename}}.md`

**Frontmatter:**
```yaml
---
title: "{{title}}"
type: source-summary
source_url: "{{url}}"
source_file: LLM-Wiki/raw/{{category}}/{{type}}/{{filename}}.md
author: "{{author}}"
date_published: {{date}}
date_ingested: <today>
concepts:
  - "[[concepts/{{category}}/{{concept1}}/{{Concept1}}]]"
  - "[[concepts/other-category/{{concept2}}/{{Concept2}}]]"
entities:
  - "[[entities/{{category}}/{{entity1}}]]"
created: <today>
updated: <today>
confidence: high
---
```

**Content:**
```markdown
## TL;DR

<1-3 sentence summary of the source>

## Key Takeaways

1. <Takeaway with supporting detail>
2. <Takeaway with supporting detail>

## Relevance to Existing Knowledge

- Links to [[concepts/{{category}}/{{concept1}}/{{Concept1}}]] — <how this source adds to that concept>
- Complements [[concepts/other-category/{{concept2}}/{{Concept2}}]] — <new perspective>

## New Concepts Discovered

- <Concept name> — <brief description if this source reveals something new>
```

### Step 7: Enrich Existing Pages (Optional)

For each related concept or entity page, consider updating it to include insights from the new source:

1. Read the existing concept page with `obsidian_read_note`
2. Check if the source adds new information not already covered
3. If yes, use `obsidian_patch_note` to add:
   - A new bullet point in `## Key Ideas`
   - A new entry in `## Sources`
   - A mention in `## Relationship to Other Concepts`
4. Update the frontmatter `updated` date and add to `sources:` array

**Do NOT modify pages without user confirmation.** Instead, present a proposal:

```
This source adds new information to these existing pages:
- concepts/kubernetes/k3s — add section on etcd vs SQLite storage
- entities/kubernetes/kubernetes — add mention of new release feature

Shall I update them?
```

### Step 8: Update Index and Log

**Update `LLM-Wiki/wiki/index.md`:**
- Add the new source summary to the Sources table under the correct topic
- Update the total page count
- Write with `obsidian_write_note`

**Update `LLM-Wiki/wiki/log.md`:**
- Append a log entry:
  ```markdown
  ---

  ## <today> — Ingest: {{title}}

  **Type:** Source ingest

  **Source:**
  - `raw/{{category}}/{{type}}/{{filename}}.md` — "{{title}}" by {{author}}

  **Pages created:**
  - `wiki/sources/{{category}}/{{filename}}.md`

  **Pages enriched:**
  - `wiki/concepts/{{category}}/{{concept}}` — added source reference

  **Key relevance:** <1-2 sentences>
  ```
- Write with `obsidian_write_note` in append mode

### Step 9: Report

Present a summary:

```
## Ingest Complete: {{title}}

### Raw Source
raw/{{category}}/{{type}}/{{filename}}.md

### Source Summary
wiki/sources/{{category}}/{{filename}}.md

### Cross-References
→ concepts/{{category}}/{{concept1}}
→ concepts/other-category/{{concept2}}

### Enriched Pages
- concepts/kubernetes/k3s — added source reference

Shall I run frontmatter-linter to validate the new pages?
```

## Templates Reference

See `quick-research` skill for the full template reference — the same raw bookmark and source summary formats apply here.

## Edge Cases

- **URL unreachable** (fetch fails) → inform the user, ask for an alternative or suggest a manual summary
- **Duplicate URL** → report existing path, offer to update rather than duplicate
- **No clear topic** → ask the user: "I couldn't determine the topic. Which category should this go under?"
- **PDF or binary URL** → still create the bookmark with available metadata, note that content couldn't be fetched
- **GitHub repo URL** → use `type: repo`, fetch the README, extract description and metadata from the GitHub API or page
- **YouTube / video URL** → create bookmark with `type: article`, note that it's a video; extract metadata from page title and description
