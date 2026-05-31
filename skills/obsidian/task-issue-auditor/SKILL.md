---
name: task-issue-auditor
description: Audit the vault to ensure every completed or in-progress task in the daily note has a corresponding issue file in its project's Issues/ directory. Validates issue file freshness (updated date, status match, daily_note backlink) against the daily note's worklog. Designed to run at session close-out to maintain the task-to-issue traceability chain.
---

# Task-Issue Auditor

Audit the task-to-issue traceability chain across the vault. Reads the daily note, scans every project section for completed and in-progress tasks, cross-references each against the project's `Issues/` directory, and validates that existing issue files are up-to-date with the daily note's worklog content.

This skill enforces the traceability contract defined in `Projects/AGENTS.md` and the `session-logger` skill: every kanban task in the daily note should have a corresponding issue write-up, and the issue file should accurately reflect the worklog state.

## When to Use

- **Session close-out:** before ending a work session ("audit my tasks", "check for missing issues", "close out the day")
- **Periodic maintenance:** "run the task auditor", "check issue file coverage"
- **After batch completion:** "I finished several tasks, make sure they all have issues"
- **Before git commit:** "audit before I commit the daily note"
- **On-demand check:** "does task X have an issue file?", "is the issue for task Y up to date?"

This skill is **designed to run at session end** — it should be invoked automatically when the user says "that's all for today", "close out", "end session", or similar session-termination phrases.

## Audit Passes

| Pass | What It Checks | Severity |
|---|---|---|
| **Pass 1: Missing Issue Files** | Every completed (`[x]`) task has a corresponding issue file in `Projects/<project>/Issues/` | Error |
| **Pass 2: In-Progress Coverage** | Every in-progress (`[ ]`) task with narrative details (`#### Task:` subsection) has an issue file | Warning |
| **Pass 3: Stale Issue Files** | Issue file `updated` date is older than the daily note's latest worklog entry for that task | Warning |
| **Pass 4: Status Mismatch** | Issue file `status` matches the daily note task state (completed → `resolved`, in-progress → `in-progress`) | Warning |
| **Pass 5: Missing daily_note Backlink** | Issue file frontmatter contains a `daily_note` field linking to a valid daily note | Error |
| **Pass 6: Worklog Drift** | Issue file content is missing a worklog entry for the current daily note date (the issue file is stale relative to the daily note) | Info |

## Workflow

### Step 1: Determine Audit Scope

**Inputs:**

| Input | Description | Default |
|---|---|---|
| `{{date}}` | The daily note date to audit | Today's date (`YYYY-MM-DD`) |
| `{{project}}` | Optional. Audit only this project's tasks | All projects |
| `{{auto_fix}}` | Whether to auto-create missing issue files | `false` — always ask first |

If `{{date}}` is not provided, use today's date. If `{{project}}` is not provided, scan all projects found in the daily note.

### Step 2: Read the Daily Note

1. **Read the daily note:** `obsidian_read_note("Daily/{{date}}.md")`
   - If the note doesn't exist, report: "No daily note for {{date}}. Nothing to audit." and stop.
2. **Extract all project sections:** Parse the content for `### Project: <name>` headings
3. **For each project section, extract:**
   - The project name
   - The task checklist items (`#### Tasks` section)
   - The narrative subsections (`#### Task: <name>`)
   - The completion status of each item

**Parsing pattern:**

```markdown
### Project: <ProjectName>

#### Tasks

- [x] Completed task ✅ YYYY-MM-DD
- [ ] In-progress task 📅 YYYY-MM-DD

#### Task: <TaskName>

Narrative details...

#### Files Changed

...
```

For each task, build a record:

```
{
  project: "<ProjectName>",
  task_name: "<TaskName>",
  status: "completed" | "in-progress",
  completed_date: "YYYY-MM-DD" (if completed),
  has_narrative: true/false (whether a #### Task: subsection exists),
  details: "<extracted narrative text>",
  files: [{path, summary}]
}
```

### Step 3: Inventory Existing Issue Files

For each unique project found in the daily note:

1. Check if the project directory exists: `obsidian_list_directory("Projects/{{project}}/Issues/")`
   - If the directory doesn't exist, note: "Project `{{project}}` has no Issues/ directory. All tasks are missing issue files."
   - If the directory exists but is empty, note: "Issues/ directory exists but is empty."

2. If the Issues/ directory exists, list all files and read their frontmatter:

```bash
obsidian_list_directory("Projects/{{project}}/Issues/")
```

For each `.md` file found, read its frontmatter with `obsidian_get_frontmatter`:

```
file: "Projects/{{project}}/Issues/<filename>.md"
  title: <from frontmatter>
  status: <from frontmatter>
  daily_note: <from frontmatter>
  updated: <from frontmatter>
  wiki_pages: <from frontmatter>
```

Build a lookup map keyed by task slug:

```
issue_map[<project>][<slug>] = { title, status, daily_note, updated, wiki_pages }
```

**Slug matching:** Convert task names to kebab-case slugs for cross-referencing:

| Task Name | Slug |
|---|---|
| `Deploy control plane to production` | `deploy-control-plane-to-production` |
| `Update deployment config` | `update-deployment-config` |
| `github_token.age cleanup` | `github-token-age-cleanup` |

The slug is the filename without the `.md` extension. Match by converting each task name to its slug and checking if it exists in the issue map.

### Step 4: Run Pass 1 — Missing Issue Files

For each **completed** task (`[x]`), check if a matching issue file exists:

1. Convert the task name to kebab-case slug
2. Look up `issue_map[project][slug]`
3. If not found, this is a **missing issue file**

**Exception:** Tasks marked as done with no narrative details (`#### Task:` subsection) and no files changed are considered "trivial completions" — they don't require issue files. Example: a simple checkbox that was ticked with no write-up.

**Detection output:**
```
ERROR — Missing issue file for completed task:
  Project: {{project}}
  Task:    {{task_name}}
  Action:  Create Projects/{{project}}/Issues/{{task-slug}}.md from daily note data
```

**Auto-fix:** If `{{auto_fix}}` is true (or user confirms), create the missing issue file using the daily note's task data:

1. Read the `#### Task: {{task_name}}` subsection for narrative details and files
2. Create `Projects/{{project}}/Issues/{{task-slug}}.md` with frontmatter:
   ```yaml
   ---
   title: "{{task_name}}"
   type: issue
   project: "{{project}}"
   status: resolved
   daily_note: "[[Daily/{{date}}|{{date}}]]"
   created: {{completed_date}}
   updated: {{completed_date}}
   ---
   ```
3. And content derived from the daily note's worklog entry for that task

### Step 5: Run Pass 2 — In-Progress Coverage

For each **in-progress** task (`[ ]`) that has a narrative `#### Task:` subsection, check if a matching issue file exists.

**Rationale:** If work is significant enough to warrant a narrative write-up in the daily note, it should have an issue file for auditability.

**Detection output:**
```
WARNING — In-progress task with narrative details but no issue file:
  Project: {{project}}
  Task:    {{task_name}}
  Suggestion: Create an issue file to track this work
```

**Do NOT auto-create** issue files for in-progress tasks without confirmation — the user may not want formal issue tracking for every in-progress item.

### Step 6: Run Pass 3 — Stale Issue Files

For each task that HAS a matching issue file, compare the issue file's `updated` date against the daily note:

1. **Check issue `updated` date:**
   - The issue file frontmatter has `updated: YYYY-MM-DD`
   - The daily note was written on `{{date}}`
   - If `updated < {{date}}` (the issue hasn't been touched since before today), the issue is stale
   - If the issue has a worklog subsection for `{{date}}` (e.g., `### Worklog: {{date}}`), it's up-to-date
   - If the issue has NO worklog subsection for `{{date}}` but the daily note has new details for that task, the issue is stale

2. **Check worklog date alignment:**
   - Search the issue file content for `### Worklog: {{date}}` or `---` separators
   - If the latest worklog entry predates the daily note's latest entry, the issue is behind

**Detection output:**
```
WARNING — Stale issue file:
  File:     Projects/{{project}}/Issues/{{task-slug}}.md
  Updated:  {{issue_updated_date}}
  Task last active in daily note: {{date}}
  Action:   Append new worklog from daily note to issue file
```

**Auto-fix:** If confirmed, append the daily note's narrative details to the issue file:

1. Read the issue file: `obsidian_read_note`
2. Append a new worklog subsection:
   ```markdown
   ---

   ### Worklog: {{date}}

   {{details_from_daily_note}}

   #### Files Changed

   | File | Summary |
   |---|---|
   | `path/to/file` | Description |
   ```
3. Update frontmatter: `updated: {{today}}`
4. Write with `obsidian_write_note`

### Step 7: Run Pass 4 — Status Mismatch

For each task with a matching issue file, compare:

| Daily Note Task State | Expected Issue `status` |
|---|---|
| `[x]` (completed) with `✅` | `resolved` |
| `[ ]` with `📅` (in-progress) | `in-progress` |
| `🔒` (blocked) | `blocked` |
| Cancelled / won't fix | `cancelled` |

**Detection output:**
```
WARNING — Status mismatch:
  File:     Projects/{{project}}/Issues/{{task-slug}}.md
  Issue status:  in-progress
  Task state:    completed (✅ {{date}})
  Action:   Update issue status to 'resolved'
```

**Auto-fix:** If confirmed, use `obsidian_update_frontmatter` to set `status: resolved` (or the correct status) and `updated: {{today}}`.

### Step 8: Run Pass 5 — Missing daily_note Backlink

For each issue file, verify:

1. Frontmatter contains a `daily_note` field
2. The value is a `[[wikilink]]` in the format `[[Daily/YYYY-MM-DD]]` or `[[Daily/YYYY-MM-DD|display]]`
3. The linked daily note actually exists (use `obsidian_search_notes` or `obsidian_read_note` on the path)

**Detection output:**
```
ERROR — Missing or broken daily_note backlink:
  File:     Projects/{{project}}/Issues/{{task-slug}}.md
  daily_note: (missing)  or  daily_note: [[Daily/2025-01-01]] (note does not exist)
  Action:   Add/update daily_note to [[Daily/{{date}}|{{date}}]]
```

**Auto-fix:** If confirmed, use `obsidian_update_frontmatter` to set or update the `daily_note` field.

### Step 9: Run Pass 6 — Worklog Drift

For each task that HAS an issue file AND has a narrative `#### Task:` subsection in the daily note, check if the issue file already contains a worklog entry for the current date.

**How to check:**
1. Read the issue file content with `obsidian_read_note`
2. Search for `### Worklog: {{date}}` in the content
3. If NOT found, the issue file hasn't been updated with today's work

**Detection output:**
```
INFO — Worklog drift detected:
  File:     Projects/{{project}}/Issues/{{task-slug}}.md
  Missing:  Worklog entry for {{date}}
  The daily note has a narrative for this task but the issue file hasn't been updated.
```

**Do NOT auto-fix** — worklog drift may be intentional (e.g., the task was touched briefly but no meaningful update to the issue is needed). Present as info for the user to decide.

### Step 10: Generate Report

Compile all findings into a structured report:

```markdown
## Task-Issue Audit: {{date}}

### Summary

| Check | Total | Errors | Warnings | OK |
|---|---|---|---|---|
| Tasks scanned | N | — | — | — |
| Missing issue files | N | E | — | N |
| In-progress gaps | N | — | W | N |
| Stale issue files | N | — | W | N |
| Status mismatches | N | — | W | N |
| Missing backlinks | N | E | — | N |
| Worklog drift | N | — | — | I |

### By Project

| Project | Tasks | Issues | Missing | Stale | Status |
|---|---|---|---|---|---|
| `Project A` | 3 | 2 | 1 | 0 | ⚠️ |
| `Project B` | 2 | 2 | 0 | 1 | ⚠️ |
| `Project C` | 1 | 1 | 0 | 0 | ✅ |

### Errors (must fix)

1. **Missing issue file** — Project: `Project A`, Task: "Deploy control plane"
   → Create at `Projects/Project A/Issues/deploy-control-plane.md`

2. **Missing daily_note backlink** — `Projects/Project B/Issues/some-task.md`
   → Update frontmatter with `daily_note: "[[Daily/{{date}}|{{date}}]]"`

### Warnings (should fix)

1. **Stale issue file** — `Projects/Project B/Issues/update-config.md`
   → Append worklog from {{date}} daily note

2. **Status mismatch** — `Projects/Project A/Issues/fix-networking.md`
   → Update status from 'in-progress' to 'resolved'

### Info

1. **Worklog drift** — `Projects/Project C/Issues/research-k0s.md`
   → Missing worklog entry for {{date}}

### Auto-Fix Proposal

The following can be fixed automatically:
1. Create N missing issue files from daily note data
2. Update N stale issue files with new worklog entries
3. Fix N status mismatches
4. Add N missing daily_note backlinks

Shall I proceed with the auto-fixes?
```

### Step 11: Execute Auto-Fixes (if confirmed)

If the user approves auto-fixes, execute in order:

1. **Create missing issue files** (Pass 1): For each completed task without an issue, create the issue file using `obsidian_write_note` with the template from Step 4.
2. **Append worklog entries** (Pass 3): For each stale issue, append the daily note's content using `obsidian_patch_note` or `obsidian_write_note` in append mode, then update frontmatter.
3. **Fix status mismatches** (Pass 4): Use `obsidian_update_frontmatter` to correct the `status` field and bump `updated`.
4. **Fix missing backlinks** (Pass 5): Use `obsidian_update_frontmatter` to add `daily_note: "[[Daily/{{date}}|{{date}}]]"`.
5. **After all fixes, re-run Passes 1-6** on the affected tasks to verify everything resolved.

### Step 12: Present Results

```markdown
## Audit Complete: {{date}}

### Actions Taken
- ✅ Created N issue files
- ✅ Updated N stale issue files with new worklogs
- ✅ Fixed N status mismatches
- ✅ Added N daily_note backlinks

### Remaining Issues
- N items require manual review (see warnings above)

### Next Steps
- Run `wiki-index-regenerator` if any LLM Wiki pages were referenced
- Run `vault-git-sync` to commit the changes
```

## Integration with session-logger

This skill is designed to run automatically at session close-out. When the user signals end of session (e.g., "that's all for today", "close out", "end session"), the agent should:

1. Run `session-logger` for the final worklog entry
2. Run this `task-issue-auditor` to audit the day's tasks
3. Present the audit results and offer auto-fixes
4. Run `vault-git-sync` to commit

The integration flow:

```
User: "that's all for today"
  ↓
session-logger: final entry (completion summary)
  ↓
task-issue-auditor: audit today's daily note ({{date}})
  ↓
Present audit report + auto-fix proposal
  ↓
Execute auto-fixes (if confirmed)
  ↓
vault-git-sync: commit all changes
```

## Templates Reference

### Task Record (from Daily Note)

```
{
  project: "<ProjectName>",
  task_name: "<TaskName>",
  status: "completed" | "in-progress" | "blocked" | "cancelled",
  completed_date: "YYYY-MM-DD" | null,
  has_narrative: bool,
  details: "<extracted text>",
  files: [{path: "...", summary: "..."}]
}
```

### Issue File Frontmatter

| Field | Required | Expected Value |
|---|---|---|
| `title` | Yes | Task name (from daily note) |
| `type` | Yes | `issue` |
| `project` | Yes | PascalCase project name |
| `status` | Yes | `in-progress` / `resolved` / `blocked` / `cancelled` |
| `daily_note` | Yes | `[[Daily/YYYY-MM-DD|display]]` wikilink |
| `wiki_pages` | No | Array of `[[wikilink]]` to LLM Wiki pages |
| `created` | Yes | Date first created |
| `updated` | Yes | Date of last update |

### Task Status to Issue Status Mapping

| Daily Note State | Issue `status` |
|---|---|
| `[x]` with `✅` | `resolved` |
| `[ ]` with `📅` | `in-progress` |
| `🔒` | `blocked` |
| Cancelled / won't fix | `cancelled` |

## Edge Cases

- **No daily note for the audit date** → stop and report: "No daily note found for {{date}}. Nothing to audit."
- **Daily note has no `### Project:` sections** → report as info: "No project tasks found in the daily note. Nothing to audit." This is valid — the user may have had a non-project day.
- **Project has no `Issues/` directory** → report as error for each task: "Project `{{project}}` has no Issues/ directory." Offer to create it with the `create-project` skill or manually.
- **Task name is too long for a filename** → truncate to 80 chars for the slug, or use a shortened version that's still unique within the project
- **Task name contains special characters** → strip non-alphanumeric characters for slug: `Fix: github_token.age cleanup!` → `fix-github-token-age-cleanup`
- **Multiple tasks with the same name in the same project** → append a numeric suffix: `deploy-control-plane.md`, `deploy-control-plane-2.md`
- **Issue file exists but has no frontmatter** → flag as error: "Issue file `{{path}}` exists but has no parseable frontmatter. Cannot validate."
- **Issue file references a different project** → flag as error: "Issue file `{{path}}` has `project: OtherProject` but is filed under `Projects/{{project}}/Issues/`. Mismatch between file location and frontmatter."
- **Task completed without a `✅` date marker** → treat as completed but note the missing date as info
- **Cross-project notification (`📩`) task in daily note** → audit it under the **target** project (not the source project)
- **Audit run mid-session (not at close-out)** → only report missing issue files (Pass 1) and backlinks (Pass 5); skip staleness checks since work is ongoing
- **Daily note references a project that doesn't exist in Projects/** → report as warning: "Project `{{project}}` referenced in daily note but not found in `Projects/` directory." Offer to create it with the `create-project` skill.
- **Issue file updated date is in the future** → treat as today (may be from timezone differences); update to today's date as part of auto-fix
