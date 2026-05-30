---
name: session-logger
description: Append structured worklog entries to today's daily note during an active work session. Logs project tasks, narrative descriptions, files changed, and task status in the format defined by Daily/AGENTS.md.
---

# Session Logger

Appends structured entries to today's daily note `## Worklog` section as work progresses. Handles creation of missing daily notes from template, per-project worklog subsections, task checklists, narrative write-ups, and Files Changed tables.

## When to Use

- **Starting work:** "log that I'm working on X"
- **During a session:** "add a task", "I finished Y", "mark Z as done"
- **After making changes:** "I changed these files", "update the worklog"
- **End of session:** "log what I did today", "close out my session"

This skill is designed to be called **repeatedly** during a work session — each call appends to the growing worklog.

## Workflow

### Step 1: Determine Today's Date

Use today's actual date. Format as `YYYY-MM-DD` for the daily note filename.

### Step 2: Find or Create Today's Daily Note

1. **Search** for today's note: `obsidian_search_notes` with `2026-05-30` (or whatever today is)
2. **If found:** Read it with `obsidian_read_note` to understand current state of the worklog
3. **If not found:** Create from template:
   - Read template: `obsidian_read_note` at `Daily/_templates/daily-note.md`
   - Replace `{{title}}` with today's date, `{{date}}` with today's date
   - Write to `Daily/YYYY-MM-DD.md` with `obsidian_write_note`
   - Add frontmatter: `title: YYYY-MM-DD`, `type: daily`, `tags: [tag1, tag2]` (ask user for tags if unclear)

### Step 3: Extract Input from User

Parse the user's request to extract:

| Input | Description | Example |
|---|---|---|
| `{{project}}` | Project name — should match a kanban file in `Projects/` | `guenivir`, `llm-wiki`, `obsidian-vault-tooling` |
| `{{task}}` | Description of what was done | `Deploy k0s control plane to bare metal` |
| `{{status}}` | Task state | `in-progress`, `done`, `blocked` |
| `{{files}}` | Optional list of changed files with summaries | `nixos/hosts/guenivir/configuration.nix — added k0s service` |
| `{{details}}` | Optional free-form narrative | `Troubleshoot etcd cluster join...` |

If the user provides a rich description, extract these fields from it. If any are missing, ask.

**Status mapping:**
- `done`, `completed`, `finished`, `✅` → completed task (`[x]` with `✅ YYYY-MM-DD`)
- `blocked`, `stuck`, `waiting` → pending task with note (`[ ]` with `🔒`)
- `in-progress`, `wip`, `working`, `started` → pending task (`[ ]` with `📅 YYYY-MM-DD`)
- `cancelled`, `wontfix` → skip (don't add to log)

### Step 4: Locate or Create the Project Subsection

Read the current note and find the `## Worklog` section. Inside it:

1. **Search for `### Project: {{project}}`** — case-insensitive, trimmed
2. **If found:** Note its position in the document for appending
3. **If not found:** We'll add it at the end of the `## Worklog` section (before any trailing content)

### Step 5: Build the Worklog Entry

Construct the entry fragment based on what you're logging:

#### Scenario A: Adding a new task

```markdown
#### Task: {{task}}

{{details}}

#### Files Changed

| File | Summary |
|---|---|
| `path/to/file` | Description of changes |
```

#### Scenario B: Updating task status

Find the existing task checklist item and patch it:
- `[ ]` → `[x]` and append `✅ YYYY-MM-DD`
- Add completion date

#### Scenario C: Adding a file change to existing task

Append a row to the existing `#### Files Changed` table in the current task's subsection.

### Step 6: Insert the Entry

Use `obsidian_patch_note` to surgically insert content:

**Case: Project subsection exists**
1. Find the last `####` heading under the project subsection
2. Insert after it using `oldString` = last heading line → `newString` = last heading + new content

**Case: Project subsection does not exist**
1. Find the end of `## Worklog` (the next `##` heading or EOF)
2. Insert at that point:
   ```markdown
   ### Project: {{project}}

   #### Tasks

   - {{status_icon}} {{task}} {{status_suffix}}

   #### Task: {{task}}

   {{details}}

   #### Files Changed

   | File | Summary |
   |---|---|
   | `path/to/file` | Description of changes |

   📋 [[Projects/{{project-kebab}}.kanban|{{project}} Kanban]]
   ```

### Step 7: Update Summary

If this is the first worklog entry of the day, also update the `## Summary` section at the top of the note. Patch the existing summary line to include this project:

**Before:**
```
## Summary

<!-- Brief overview of the day's work -->
```

**After:**
```
## Summary

Working on {{project}}: {{task}}
```

If a summary already exists, append the new project/task to it.

### Step 8: Confirm

Present a summary of what was logged:

```
Logged to Daily/2026-05-30.md:

### Project: guenivir
- [x] Deploy k0s control plane ✅ 2026-05-30

## Summary updated: "Working on guenivir: deploy k0s control plane"
```

## Format Reference

All output follows the format defined in `Daily/AGENTS.md`. Key rules:

- `#### Tasks` section uses Tasks plugin format: `- [ ] task 📅 YYYY-MM-DD` or `- [x] task ✅ YYYY-MM-DD`
- `#### Task: <name>` subsections contain narrative write-ups
- `#### Files Changed` tables have `| File | Summary |` header with paths in backticks
- `📋 [[Projects/<name>.kanban|<name> Kanban]]` goes at the END of each project section
- Project names must match kanban board filenames in `Projects/`

## Edge Cases

- **No today's note exists** → create from template first (Step 2)
- **Project section exists but has no `#### Tasks`** → add it before the first `#### Task:`
- **Multiple entries for same project in same session** → append chronologically under the existing project section
- **No files changed** → skip the Files Changed table entirely
- **User wants to log without a specific project** → use project name `General` (no kanban link)
- **Daily note has unexpected format** → append gracefully, don't break existing content
- **User says "that's all for today"** → optionally suggest running `vault-git-sync`
