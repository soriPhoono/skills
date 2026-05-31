---
name: session-closeout
description: Composite end-of-day skill that runs the full vault maintenance pipeline: task-issue-auditor, frontmatter-linter, tag-sanitizer, organize-raw-sources, and wiki-index-regenerator. Invoke this at session close-out instead of running each skill individually.
---

# Session Close-Out

Composite skill that orchestrates the full end-of-day vault maintenance pipeline. Runs five skills in sequence — each one depends on the previous step having a clean state — then suggests `vault-git-sync` to commit all changes.

For the final worklog entry before running this, use `session-logger`.

## When to Use

- **End of day:** "close out", "end session", "wrap up", "that's all for today"
- **Pre-commit validation:** "audit everything before I commit"
- **Batch maintenance:** "run the full vault health check"
- **After a heavy session:** "clean up after all that work"

## Sequence

```
session-logger (final entry) ← run this first if you haven't already
  ↓
session-closeout (this skill)
  │
  ├── 1. task-issue-auditor    — audit task → issue traceability
  ├── 2. frontmatter-linter    — lint frontmatter, graph, stale content
  ├── 3. tag-sanitizer         — clean up tags
  ├── 4. organize-raw-sources  — audit raw/ bookmark placement
  └── 5. wiki-index-regenerator — rebuild master index
  │
  ↓
vault-git-sync (commit all changes)
```

The order is intentional:
- **task-issue-auditor** first because it may create/update issue files, which affects the frontmatter landscape
- **frontmatter-linter** second to validate all frontmatter (including any issue files just created)
- **tag-sanitizer** third — tags are cross-cutting, run after content changes settle
- **organize-raw-sources** fourth — source bookmarks are standalone, fine to run last among content checks
- **wiki-index-regenerator** last — rebuilds the table of contents after everything else is finalized

## Workflow

### Step 1: Invoke task-issue-auditor

Run `task-issue-auditor` on today's daily note to verify every completed task has a corresponding issue file.

**Load the skill:**
```
Use the `skill` tool to load `task-issue-auditor`
```

**What it does:**
- Scans today's daily note for all `### Project:` sections
- Cross-references completed/in-progress tasks against `Issues/` directories
- Reports missing issue files, stale files, status mismatches, and missing `daily_note` backlinks

**Expected output:** A structured audit report. Offer auto-fixes for missing issue files and stale entries if any are found.

### Step 2: Invoke frontmatter-linter

Run `frontmatter-linter` to validate YAML frontmatter, graph integrity, orphan pages, and stale content across the entire vault.

**Load the skill:**
```
Use the `skill` tool to load `frontmatter-linter`
```

**What it does:**
- Scans every `.md` note in the vault
- Classifies each note by system (daily, wiki, raw, kanban, other)
- Validates frontmatter fields per schema
- Checks `[[wikilinks]]` graph integrity (broken links, orphans, dead-ends)
- Detects stale content by comparing `updated` dates
- Checks cross-system consistency (raw/ vs wiki/ topic alignment)

**Expected output:** A lint report written to `LLM-Wiki/outputs/lint-YYYY-MM-DD.md`. Present the summary (errors/warnings/info) and offer auto-fixes for any fixable issues (missing dates, malformed wikilinks).

### Step 3: Invoke tag-sanitizer

Run `tag-sanitizer` to audit vault tags for near-duplicates, singletons, non-kebab-case violations, and orphan tags.

**Load the skill:**
```
Use the `skill` tool to load `tag-sanitizer`
```

**What it does:**
- Inventories all tags across the vault
- Detects near-duplicate tags (e.g., `kubernetes` vs `k8s`)
- Finds single-use tags that may be typos
- Validates kebab-case formatting
- Detects orphan tags (referenced in frontmatter but no pages use them)

**Expected output:** A tag audit with proposed merges and cleanup. Offer to execute the cleanup.

### Step 4: Invoke organize-raw-sources

Run `organize-raw-sources` to audit and reorganize LLM-Wiki `raw/` source bookmarks.

**Load the skill:**
```
Use the `skill` tool to load `organize-raw-sources`
```

**What it does:**
- Walks every file in `raw/<category>/<type>/`
- Validates `type:` frontmatter matches the directory (articles/ → article, repos/ → repo, etc.)
- Cross-references `source_file` paths in `wiki/sources/` against actual raw/ files
- Detects orphan bookmarks and summaries
- Flags topic misplacements

**Expected output:** An organization report with type/directory mismatches, broken references, and orphans. Offer to auto-fix moves and reference updates.

### Step 5: Invoke wiki-index-regenerator

Run `wiki-index-regenerator` to rebuild `LLM-Wiki/wiki/index.md` from the actual filesystem.

**Load the skill:**
```
Use the `skill` tool to load `wiki-index-regenerator`
```

**What it does:**
- Walks `wiki/concepts/`, `wiki/entities/`, `wiki/sources/`, `wiki/comparisons/`
- Reads frontmatter from every page
- Rebuilds topic tables, page counts, and statistics
- Writes the updated index

**Expected output:** A diff showing what changed (new topics, new pages, updated counts). Write the updated index.

### Step 6: Summary & Git Sync

Present a final summary of everything that was checked and fixed:

```markdown
## Session Close-Out Complete: YYYY-MM-DD

### Checks Run

| Skill | Status | Findings |
|---|---|---|
| task-issue-auditor | ✅ / ⚠️ | N issues created, N updated |
| frontmatter-linter | ✅ / ⚠️ | N errors, N warnings, N info |
| tag-sanitizer | ✅ / ⚠️ | N merges proposed, N tags cleaned |
| organize-raw-sources | ✅ / ⚠️ | N files moved, N references updated |
| wiki-index-regenerator | ✅ / ⚠️ | N topics, N pages indexed |

### Next Steps

- Run `vault-git-sync` to commit all changes
- Or review individual findings above before committing
```

Then suggest running `vault-git-sync` to commit everything, using conventional commit messages appropriate to what changed:

| What Changed | Commit Type |
|---|---|
| Issue files created/updated | `feat(project):` |
| Frontmatter fixes | `fix(vault):` |
| Tag cleanup | `chore(vault):` |
| Source reorganization | `feat(wiki):` |
| Index rebuilt | `docs(wiki):` |

If multiple categories changed, use a single commit with a summary message or commit per category depending on user preference.

## Integration with session-logger

This skill replaces the individual skill references in the `session-logger` close-out sequence. Instead of listing five separate skills, the close-out edge case should say:

```
User says "that's all for today" →
  1. session-logger → final wrap-up entry
  2. session-closeout → full vault health pipeline (runs all 5 checks)
  3. vault-git-sync → commit all changes
```

## Edge Cases

- **One or more sub-skills are unavailable** (not loaded/registered) → skip that step and report: "⚠ `frontmatter-linter` not available, skipping." Continue with the remaining pipeline.
- **task-issue-auditor finds issues but user declines auto-fix** → still proceed with the rest of the pipeline. Flag in the final summary that issues remain open.
- **frontmatter-linter takes a long time** (large vaults) → it scans every note; this is expected. Wait for completion.
- **User interrupts mid-pipeline** → report partial results for completed steps, note what was skipped.
- **No daily note for today** → skip task-issue-auditor (nothing to audit), run the rest. Report: "No daily note found — skipping task-issue-auditor."
- **tag-sanitizer proposes major tag changes** → present clearly and get confirmation before executing. Tag renames are irreversible.
- **organize-raw-sources finds many mismatches** → batch the moves and reference updates, then re-run the skill to verify the fixes resolved everything before proceeding.
- **vault-git-sync fails** → report the error clearly. Offer to retry or commit manually with `git` CLI commands.
- **Session close-out run mid-day (not at end)** → still valid; runs all checks. The wiki-index-regenerator in particular is useful at any point.
