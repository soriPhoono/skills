---
name: tag-sanitizer
description: Audit all vault tags for near-duplicates, singletons, non-kebab-case violations, and orphan tags. Propose merges and execute cleanup via the Obsidian MCP server.
---

# Tag Sanitizer

Inventory, analyze, and clean up tags across the entire vault. Detects inconsistencies, proposes merges, and executes tag rename operations safely.

## When to Use

- **Periodic maintenance:** "clean up my tags", "sanitize tags", "audit tags"
- **After batch ingest:** "check tags on new wiki pages"
- **Format enforcement:** "make all tags kebab-case"
- **Pre-commit cleanup:** "tidy tags before I commit"

## Workflow

### Step 1: Inventory All Tags

1. Use `obsidian_list_all_tags` to get every tag in the vault with its occurrence count

This returns data like:

```json
[
  {"tag": "kubernetes", "count": 19},
  {"tag": "k8s", "count": 1},
  {"tag": "architecture", "count": 10},
  {"tag": "container-runtime", "count": 3},
  {"tag": "Container Runtime", "count": 2},
  ...
]
```

2. Read `Daily/AGENTS.md` to get the list of canonical/common tags for reference

### Step 2: Detect Issues

Run four detection passes:

#### Pass 1: Near-Duplicates

Find pairs of tags that likely refer to the same concept:

| Pattern | Examples | Merge Direction |
|---|---|---|
| Abbreviation vs full name | `k8s` / `kubernetes` | `k8s` → `kubernetes` |
| Hyphenated vs unhyphenated | `argocd` / `argo-cd` | Prefer project's canonical spelling |
| Singular vs plural | `container` / `containers` | Prefer singular |
| Acronym variations | `mcp-server` / `mcpserver` | Prefer hyphenated |
| Substring overlap | `n8n` / `n8n-mcp-server` | Keep both if distinct concepts |
| Spelling variants | `helm` / `Helm` (case) | Prefer lowercase kebab-case |

For each candidate pair:
- If they are **identical concepts**, propose merging the lower-count tag into the higher-count one
- If they are **distinct concepts** (e.g., `mcp` vs `mcp-server`), keep both
- If uncertain, ask the user

#### Pass 2: Singletons (count = 1)

Tags used exactly once. These may be:
- **Valid** (narrowly scoped tag on a single page) → keep
- **Mistakes** (typo or one-off usage that should use an existing tag) → flag for review

Flag all singleton tags for user review.

#### Pass 3: Non-Kebab-Case

Tags that violate the vault convention (lowercase kebab-case):

| Violation | Example | Correction |
|---|---|---|
| Uppercase letters | `ContainerRuntime` | `container-runtime` |
| Spaces | `container runtime` | `container-runtime` |
| Underscores | `container_runtime` | `container-runtime` |
| Mixed case | `K8s-Cluster` | `k8s-cluster` |

For each violation, propose the kebab-case equivalent.

#### Pass 4: Orphan Tags (against conventions)

Tags not listed in the `Daily/AGENTS.md` conventions section that have count > 1 — these may be emergent valid tags or drift.

For each, suggest:
- Adding to conventions if it's a legitimate recurring topic
- Merging if it overlaps with a conventional tag

### Step 3: Propose Cleanup Plan

Compose a structured proposal:

```markdown
## Tag Sanitization Proposal — YYYY-MM-DD

### Near-Duplicates (N pairs)
| Merge | Into | Count affected |
|---|---|---|
| `k8s` (1) | `kubernetes` (19) | 1 note |
| `microk8s` (1) | `micro-k8s` | 0 |

### Singletons for Review (N tags)
| Tag | Note | Action |
|---|---|---|
| `kube-proxy` | concepts/kubernetes/kube-proxy | Keep — valid narrow tag |
| `Container Runtime` | .../some-page.md | Fix case to `container-runtime` |

### Non-Kebab-Case (N tags)
| Current | Fixed | Count |
|---|---|---|
| `Container Runtime` | `container-runtime` | 2 |

### Summary
- N tags to merge (auto-fix)
- N tags to rename (auto-fix)
- N singletons requiring judgment
```

**Do NOT execute without confirmation.** Present the plan and ask:

> "Shall I apply the auto-fixable changes (N merges, N renames)? I'll leave the N singletons for you to review."

### Step 4: Execute Cleanup

For each approved change:

1. **Find all notes** with the old tag using `obsidian_search_notes` with the tag as query
2. For each note found:
   - Use `obsidian_manage_tags` with operation `remove` and the old tag
   - Use `obsidian_manage_tags` with operation `add` and the new tag
3. If merging tags, `remove` the old, `add` the new

**Safety:** Process tags one at a time. If a note already has the new tag, skip the `add` step (no duplicate tag creation — Obsidian tags are sets).

### Step 5: Report

After execution, report what was done:

```markdown
## Tag Cleanup Complete — YYYY-MM-DD

### Applied
- Merged `k8s` → `kubernetes` (1 note updated)
- Renamed `Container Runtime` → `container-runtime` (2 notes updated)
- Renamed `K8s-Cluster` → `k8s-cluster` (1 note updated)

### Pending Review (N singletons)
- `kube-proxy` — appears valid, keeping
- `weird-typo-tag` — user needs to decide

### Unchanged (N tags)
- N tags were already clean and correct
```

## Edge Cases

- **Tag with leading `#`:** `obsidian_list_all_tags` may return with or without `#` prefix. Normalize to without `#`.
- **Frontmatter tags vs inline tags:** The MCP tools handle both. `obsidian_manage_tags` works on both. Prefer frontmatter tags for wiki pages, inline tags for daily notes.
- **Tag in AGENTS.md conventions:** If a tag appears in the conventions list but has zero occurrences, note it as "unused but defined" — may warrant removal from conventions.
- **User declines proposal:** Report "Cleanup skipped. No changes made."
- **Multiple merge candidates for same tag:** Merge into the one with highest count (most established).
