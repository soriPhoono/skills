---
name: vault-git-sync
description: Stage, commit, and push vault changes organized by category (daily/, wiki/, raw/, config/) using conventional commit messages. Relies on the Obsidian Git plugin for the push/sync pipeline.
---

# Vault Git Sync

Stage grouped changes in the vault git repository, create a structured conventional commit, and trigger sync. Manages the git workflow for the Obsidian vault while respecting the Obsidian Git plugin for push operations.

## When to Use

- **Session end:** "sync the vault", "commit my changes", "save everything"
- **After ingest:** "commit the new wiki pages"
- **Before shutting down:** "save and push"
- **Periodic checkpoint:** "checkpoint the vault"

## Dependencies

This skill uses **bash `git` commands** for staging and committing (the MCP protocol has no git interface), but relies on the **Obsidian Git Plugin** being installed and configured in the vault for the push/sync pipeline. The Obsidian Git plugin handles auto-push on commit or manual sync.

## Workflow

### Step 1: Check Repository State

1. Determine the vault path — observe the vault root from context (e.g., `~/Nextcloud/Notes/`)
2. Run `git -C <vault-path> status` to see current state
3. Note: modified, added, deleted, untracked files

### Step 2: Categorize Changes

Group all changed files into categories based on their path:

| Category | Path Prefix | Example |
|---|---|---|
| **Daily notes** | `Daily/` | `Daily/2026-05-30.md` |
| **Wiki pages** | `LLM-Wiki/wiki/` | `LLM-Wiki/wiki/concepts/kubernetes/k3s/...` |
| **Raw sources** | `LLM-Wiki/raw/` | `LLM-Wiki/raw/containerd/articles/...` |
| **Config/Infra** | `.obsidian/`, `*.kanban.md`, `AGENTS.md`, `.gitignore`, root files | `.obsidian/plugins/...`, `Projects/guenivir.kanban.md` |
| **Outputs** | `LLM-Wiki/outputs/` | Gitignored — skip unless explicitly requested |
| **Other** | Anything else | Flag for user review |

### Step 3: Build the Commit Message

Use conventional commit format with a summary line and categorized bullet details:

```
type(scope): Brief summary

Category:
- file — what changed
- file — what changed

Category:
- file — what changed
```

**Type selection:**
- `feat` — new notes, new wiki pages, new sources
- `fix` — corrections, lint fixes, broken link repairs
- `refactor` — reorganization, renaming, restructuring
- `docs` — AGENTS.md changes, documentation-only
- `chore` — config changes, gitignore, .obsidian/ changes

**Scope selection:**
- `daily` — daily notes only
- `wiki` — LLM-Wiki wiki/ changes
- `raw` — LLM-Wiki raw/ changes
- `vault` — mixed changes across categories
- `config` — configuration only

**Example:**
```
feat(vault): add k0s research and update daily log

Daily:
  - 2026-05-30.md — created for k0s migration session

Wiki:
  - concepts/kubernetes/k0s/k0s-architecture.md — new concept page
  - sources/kubernetes/k0s-official-docs.md — new source summary
  - index.md — updated topic tables and counts

Raw:
  - kubernetes/articles/k0s-architecture-deep-dive.md — new source bookmark

Config:
  - Projects/guenivir.kanban.md — added k0s migration task
```

### Step 4: Stage and Commit

1. **Stage by category** using `git add` with path prefixes:
   ```bash
   git add Daily/ LLM-Wiki/wiki/ LLM-Wiki/raw/ Projects/ AGENTS.md
   ```
   This stages all changed files in those directories.

2. **Check for untracked files** in each category and ensure they're included

3. **Commit** with the constructed message:
   ```bash
   git commit -m "feat(vault): summary"
   ```

### Step 5: Push (via Obsidian Git Plugin)

After committing, inform the user:

```
Committed to vault git repo.

To push: The Obsidian Git plugin will auto-sync based on its
configured interval. To push immediately, open Obsidian and
run "Obsidian Git: Push" from the command palette.
```

Do NOT run `git push` from the command line — the Obsidian Git plugin manages credentials and remotes. Respect the plugin's sync cycle.

### Step 6: Report

Present a summary:

```
Vault Sync Complete

Commit: a1b2c3d — feat(vault): add k0s research and update daily log

Daily:    2 files changed (1 modified, 1 new)
Wiki:     3 files changed (2 new, 1 modified)
Raw:      1 file new
Config:   1 file modified

Push: Pending Obsidian Git plugin sync
```

## Edge Cases

- **No changes detected** → report "Nothing to commit. Vault is clean."
- **Uncommitted changes in outputs/** → skip (gitignored by convention)
- **Git not available** → warn that git must be installed. The vault path should have git available since it's already version-controlled.
- **Merge conflicts** → flag immediately. Do NOT attempt to resolve automatically. Tell the user: "Merge conflict detected in <file>. Please resolve in Obsidian and run sync again."
- **Large commit** (> 20 files) → suggest splitting into multiple commits by category:
  "This is a large commit (30 files). Shall I split into separate commits by category?"
- **Detached HEAD** → warn and abort. The vault should be on a branch (usually `main`).
