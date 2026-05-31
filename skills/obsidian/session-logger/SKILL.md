---
name: session-logger
description: Append structured worklog entries to today's daily note during an active work session. Logs project tasks, narrative descriptions, files changed, and task status in the format defined by Daily/AGENTS.md. For in-depth task work, creates issue summary files in Projects/<project>/Issues/ with traceability back to daily note entries and LLM Wiki research pages.
---

# Session Logger

Appends structured entries to today's daily note `## Worklog` section as work progresses. Handles creation of missing daily notes from template, per-project worklog subsections, task checklists, narrative write-ups, Files Changed tables, **and cross-project todo notifications (pings)**.

## When to Use

- **Starting work:** "log that I'm working on X"
- **During a session:** "add a task", "I finished Y", "mark Z as done"
- **After making changes:** "I changed these files", "update the worklog"
- **Cross-project notification:** "add a todo for Project B about updating the deployment config"
- **End of session:** "log what I did today", "close out my session" — triggers the session close-out sequence (final log entry → `task-issue-auditor` → `vault-git-sync`)
- **Creating an issue:** "create an issue for this task", "track this work as an issue", "write up this investigation"

This skill is designed to be called **repeatedly** during a work session — each call appends to the growing worklog. When a task involves significant investigation (especially cross-project work or research-backed tasks), it also creates a durable issue file in the project's `Issues/` directory for auditability.

---

## Cross-Project Todo Notifications (Pings)

Projects often depend on each other. Work in one project can create work for another. For example, a change in Project A's configuration may require an update in Project B's deployment pipeline.

**Mechanism:** Because all daily notes live in the shared Obsidian vault, any agent can place a todo item under *any* project's section in today's daily note. This acts as a passive notification — the target project's agent will discover it at session start when it scans the daily note.

**The notification lifecycle:**

```
Agent working on Project A               Agent working on Project B
─────────────────────────                 ──────────────────────────
1. Realises Project B needs a change
2. Calls session-logger:
   "add a todo for Project B:
    update the deployment config"
                                ──→   Daily/2026-05-30.md:
                                       ### Project: Project B
                                       #### Tasks
                                       - [ ] 📩 [from:
                                             Project A] update
                                             deployment config 📅 2026-05-30
                                       #### Task: [Ping from Project A]
                                       ...
                                                      
3. Next session start, Project B agent
   searches daily note for its project
   section, finds the 📩 ping        ←── reads and acts on it
```

### Repo-to-Project Mapping

By default, when an agent is running inside a repository directory, its "current project" is the name of that repository directory's corresponding vault project. The mapping between repository paths and vault project names is defined by the user's environment — typically in the workspace root `AGENTS.md`, the user's opencode config, or a `Repo-to-Project` convention document in the vault.

If no mapping is configured, the agent should ask the user for the project name explicitly.

---

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
|---|---|---|---|
| `{{project}}` | Current project — the one you're actively working on | `project-a`, `project-b`, `my-project` |
| `{{target_project}}` | **Optional.** Target project for cross-project notifications. If set and **different** from `{{project}}`, the entry goes under the target's section, not the current project's. Defaults to `{{project}}` (same-project entry). | `project-b`, `project-a` |
| `{{task}}` | Description of what was done or what needs to be done | `Deploy control plane to production` |
| `{{status}}` | Task state | `in-progress`, `done`, `blocked` |
| `{{files}}` | Optional list of changed files with summaries | `repo/path/to/config.yml — added service definition` |
| `{{details}}` | Optional free-form narrative | `Troubleshoot cluster join timeout...` |
| `{{create_issue}}` | **Optional.** Whether to create an issue write-up file in `Projects/<project>/Issues/`. Auto-triggered when the task originates from a cross-project notification. Can also be explicitly requested via "create an issue for this" or "track this as an issue". | `true`, `false` |
| `{{wiki_pages}}` | **Optional.** List of LLM Wiki pages created during research for this task. Links the issue to the auditable research cache. | `["wiki/concepts/kubernetes/k0s/K0s-Control-Plane", "wiki/sources/kubernetes/k0sproject-architecture"]` |

If the user provides a rich description, extract these fields from it. If any are missing, ask.

**Determining `{{project}}` from context:**
- If running in a known repo directory (see Repo-to-Project Mapping), use the matching project
- If the user says "working on X" or "in the X repo", use X as the project
- If ambiguous, ask the user

**Determining `{{target_project}}`:**
- If the user says "add a todo for \<project\>" or "notify \<project\> about X" or "scope this to \<project\>", set `target_project` to that project
- If `project` is explicitly provided but no target_project is mentioned, `target_project` defaults to `project`
- If the user says "add a task for the \<project\> repo" or "the \<project\> project needs X", treat that as a cross-project notification

**Status mapping:**
- `done`, `completed`, `finished`, `✅` → completed task (`[x]` with `✅ YYYY-MM-DD`)
- `blocked`, `stuck`, `waiting` → pending task with note (`[ ]` with `🔒`)
- `in-progress`, `wip`, `working`, `started` → pending task (`[ ]` with `📅 YYYY-MM-DD`)
- `cancelled`, `wontfix` → skip (don't add to log)
- For **cross-project notifications**, the status should always default to `in-progress` (pending) unless explicitly stated otherwise — the notification represents a new pending item for the target project.

**`{{create_issue}}` auto-trigger logic:**
- If the task is a cross-project notification (`{{target_project}}` ≠ `{{project}}`), set `create_issue: true` automatically — the notification represents a hand-off that should be tracked as an issue.
- If the user says "create an issue", "track this", "write up an issue", or "file this as an issue" → set `create_issue: true`.
- If the user says "quick note" or "just log it" → set `create_issue: false` (skip issue creation).
- If `status` is `done` and this is a completion log (not a new task), default `create_issue: false` — the work is already complete.
- For new tasks (`status: in-progress`, `started`, `wip`) with significant `{{details}}`, default `create_issue: true` — if there's enough narrative to warrant a write-up, create the issue file.

### Step 4: Locate or Create the Project Subsection

**Important:** Use `{{target_project}}` (not `{{project}}`) to locate the subsection. For same-project entries these are identical. For cross-project notifications, the entry gets placed under the **target** project's section.

Read the current note and find the `## Worklog` section. Inside it:

1. **Search for `### Project: {{target_project}}`** — case-insensitive, trimmed
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

#### Scenario D: Cross-project todo notification

When `{{target_project}}` differs from `{{project}}` (i.e., you're scoping a todo for another project):

The entry goes under `### Project: {{target_project}}`. The format signals that this is an incoming notification, not a self-assigned task:

```markdown
#### Task: [Ping from {{project}}] {{task}}

**Requested by:** `{{project}}` agent · 📅 {{today's date}}
**Context:** {{details}}

> This item was created by the {{project}} work session. It appears here
> as a cross-project notification because it requires action from
> {{target_project}}.
```

**Checklist item format** (under `#### Tasks` in the target project section):

```markdown
- [ ] 📩 [from: {{project}}] {{task}} 📅 {{today's date}}
```

The `📩` icon is the convention for "this is an incoming notification from another project." It distinguishes cross-project pings from self-assigned tasks.

**No Files Changed table** is added for cross-project notifications (no files were changed in the target project during this session).

**Kanban link:** Use the **target** project's kanban:
```markdown
📋 [[Projects/{{target_project-kebab}}|{{target_project}} Kanban]]
```

### Step 6: Insert the Entry

Use `obsidian_patch_note` to surgically insert content:

**Case: Project subsection exists**
1. Find the last `####` heading under the target project subsection
2. Insert after it using `oldString` = last heading line → `newString` = last heading + new content

**Case: Project subsection does not exist**
1. Find the end of `## Worklog` (the next `##` heading or EOF)
2. Insert at that point using the full section template:

**For same-project entries:**
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

   📋 [[Projects/{{project-kebab}}|{{project}} Kanban]]
   ```

**For cross-project notifications:**
   ```markdown
   ### Project: {{target_project}}

   #### Tasks

   - [ ] 📩 [from: {{project}}] {{task}} 📅 {{today's date}}

   #### Task: [Ping from {{project}}] {{task}}

   **Requested by:** `{{project}}` agent · 📅 {{today's date}}
   **Context:** {{details}}

   > This item was created by the {{project}} work session. It appears here
   > as a cross-project notification because it requires action from
   > {{target_project}}.

   📋 [[Projects/{{target_project-kebab}}|{{target_project}} Kanban]]
    ```

### Step 7: Create Issue Summary File (if applicable)

If `{{create_issue}}` is `true` AND the target project has an `Issues/` directory, create an issue write-up file to provide an auditable trail from task to implementation. This is the durable record of the work — distinct from the daily note entry (which is the temporal log).

**Issue file naming:** Slugify the task name to kebab-case:

| Task | Issue Filename |
|---|---|
| `Deploy control plane to production` | `deploy-control-plane-to-production.md` |
| `Research k0s control plane architecture` | `research-k0s-control-plane-architecture.md` |
| `[Ping from Project A] Update deployment config` | `update-deployment-config.md` (strip the ping prefix) |

**Path:** `Projects/{{target_project}}/Issues/{{task-slug}}.md`

**Step 7a: Check if issue already exists**

Before creating, check if an issue file for this task already exists:

1. Use `obsidian_search_notes` with the task slug as query, scoped to `Projects/{{target_project}}/Issues/`
2. If a matching file exists, read it with `obsidian_read_note` and append new worklog content instead of overwriting
3. If no file exists, proceed with creation

**Step 7b: Determine issue status**

| Worklog Status | Issue Status |
|---|---|
| `in-progress`, `started`, `wip` | `in-progress` |
| `done`, `completed` | `resolved` |
| `blocked` | `blocked` |

**Step 7c: Build the issue frontmatter**

```yaml
---
title: "{{task}}"
type: issue
project: "{{target_project}}"
status: {{issue_status}}
daily_note: "[[Daily/{{today}}|{{today}}]]"
wiki_pages:
{% for page in wiki_pages %}
  - "[[{{page}}]]"
{% endfor %}
created: {{today}}
updated: {{today}}
---
```

**Frontmatter rules:**
- `daily_note` — always set to today's daily note `[[wikilink]]`. This is the traceability anchor: every issue file links back to the daily note where work was logged, and the daily note's task links forward to the issue.
- `wiki_pages` — optional. If `{{wiki_pages}}` was provided (research was done), list each page as a `[[wikilink]]`. This connects the issue to the auditable research cache in the LLM Wiki.
- If the issue already existed and you're appending, update `updated: {{today}}` and merge any new `wiki_pages` entries without duplicating.

**Step 7d: Build the issue content**

```markdown
## Summary

{{task}}

## Context

{{details}}

{% if files %}
## Files Changed

| File | Summary |
|---|---|
{% for file in files %}| `{{file.path}}` | {{file.summary}} |
{% endfor %}{% endif %}

## Daily Note Reference

Tracked in [[Daily/{{today}}|{{today}}]] under `### Project: {{target_project}}`.

{% if wiki_pages %}
## LLM Wiki References

Research conducted for this issue:
{% for page in wiki_pages %}
- [[{{page}}]]
{% endfor %}{% endif %}
```

**For cross-project notifications specifically** (where `{{target_project}}` ≠ `{{project}}`), add an additional section noting the origin:

```markdown
## Origin

This issue was created from a cross-project notification sent by the **{{project}}** project. See the notification in [[Daily/{{today}}|today's daily note]] under `### Project: {{target_project}}`.
```

**Step 7e: Write or append the issue file**

- **New issue:** Use `obsidian_write_note` with the full frontmatter and content
- **Existing issue (append mode):** Use `obsidian_read_note` to get current content, then use `obsidian_write_note` in `append` mode with a new worklog subsection:

  ```markdown
  ---

  ### Worklog: {{today}}

  {{details}}

  {% if files %}
  #### Files Changed

  | File | Summary |
  |---|---|
  {% for file in files %}| `{{file.path}}` | {{file.summary}} |
  {% endfor %}{% endif %}

  → Daily note: [[Daily/{{today}}|{{today}}]]
  ```

Then also `obsidian_update_frontmatter` to update the `updated` date.

**Step 7f: Verify the issue file**

Read back the issue file with `obsidian_read_note` to confirm:
- Frontmatter contains `daily_note: "[[Daily/{{today}}|...]]"` — traceability anchor is present
- `project:` matches `{{target_project}}`
- If `wiki_pages` was provided, they appear in the frontmatter

### Step 8: Update Summary

If this is the first worklog entry of the day, also update the `## Summary` section at the top of the note.

**For same-project entries:**
Patch the existing summary line to include this project:

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

**For cross-project notifications:**
Update the summary to note both the current project and the cross-project ping:

**After:**
```
## Summary

Working on {{project}}: {{task}} · Pending notification for {{target_project}}: {{task}}
```

If a summary already exists, append to it using the same pattern.

### Step 9: Confirm

Present a summary of what was logged:

**Same-project entry (with issue):**
```
Logged to Daily/2026-05-30.md:

### Project: Project A
- [x] Deploy control plane to production ✅ 2026-05-30

## Summary updated: "Working on Project A: deploy control plane to production"

📄 Issue created: Projects/Project A/Issues/deploy-control-plane-to-production.md
```

**Same-project entry (no issue):**
```
Logged to Daily/2026-05-30.md:

### Project: Project A
- [x] Deploy control plane to production ✅ 2026-05-30

## Summary updated: "Working on Project A: deploy control plane to production"
```

**Cross-project notification (with issue):**
```
Logged cross-project notification to Daily/2026-05-30.md:

### Project: Project B  ← target project section
- [ ] 📩 [from: Project A] Update deployment config 📅 2026-05-30

## Summary updated: "Working on Project A: ... · Pending notification for Project B: ..."

➡️  Project B's agent will discover this item when it scans the daily note.
📄 Issue created: Projects/Project B/Issues/update-deployment-config.md
```

**Cross-project notification (no issue):**
```
Logged cross-project notification to Daily/2026-05-30.md:

### Project: Project B  ← target project section
- [ ] 📩 [from: Project A] Update deployment config 📅 2026-05-30

## Summary updated: "Working on Project A: ... · Pending notification for Project B: ..."

➡️  Project B's agent will discover this item when it scans the daily note.
```

## Discovering Cross-Project Notifications (Agent Instructions)

This section is for **any agent reading the daily note at session start** to discover work items that were scoped for their project by other projects' agents.

### Protocol: Session Start Scan

When beginning work on a project, always scan today's daily note for items directed at you:

1. **Open today's note** — `obsidian_read_note` at `Daily/YYYY-MM-DD.md`
2. **Find your project's section** — look for `### Project: <YourProject>`
3. **Check for `📩` items** — under `#### Tasks`, any checklist item starting with `📩 [from: ...]` is a cross-project notification
4. **Read the task body** — the `#### Task: [Ping from X] ...` subsection has full context

### Example Scan

If an agent working on Project B scans the daily note and finds:

```markdown
### Project: Project B

#### Tasks

- [ ] 📩 [from: Project A] Update deployment config 📅 2026-05-30

#### Task: [Ping from Project A] Update deployment config

**Requested by:** `Project A` agent · 📅 2026-05-30
**Context:** The base infrastructure now requires an updated
deployment pipeline to match the new version.
```

The agent should:
1. Note the pending item as actionable work
2. Present it to the user as part of their session start summary
3. Optionally move it to the In Progress column on the kanban board
4. Track it in the worklog when work begins

### Session Close-Out

At the end of every work session, the agent should run the full close-out sequence:

1. **Final worklog entry** via `session-logger` — wrap-up summary of what was accomplished
2. **Task-issue audit** via `task-issue-auditor` — verifies every completed task has an issue file and existing files are up-to-date
3. **Vault git sync** via `vault-git-sync` — commit all changes with conventional commit messages

This ensures the traceability chain is maintained: daily note tracks when work happened, issue files track what work was done and why.

### Acknowledging a Cross-Project Notification

When a cross-project ping has been picked up and acted upon, mark it in the daily note:

1. Change task status: `[ ]` → `[x]` with `✅ YYYY-MM-DD`
2. Optionally add a follow-up note:
   ```markdown
   **Acknowledged by:** `Project B` agent · ✅ 2026-05-30
   ```
3. When the originating agent scans the daily note in a future session, they'll see the completion and know the notification was handled.
4. Run `task-issue-auditor` at session close-out to ensure the issue file status reflects the resolution.

---

## Format Reference

All output follows the format defined in `Daily/AGENTS.md` and `Projects/AGENTS.md`. Key rules:

### Daily Note Conventions

- `#### Tasks` section uses Tasks plugin format: `- [ ] task 📅 YYYY-MM-DD` or `- [x] task ✅ YYYY-MM-DD`
- `#### Task: <name>` subsections contain narrative write-ups
- `#### Files Changed` tables have `| File | Summary |` header with paths in backticks
- `📋 [[Projects/<name>|<name> Kanban]]` goes at the END of each project section
- Project names must match kanban board filenames in `Projects/`
- Cross-project notifications use the `📩` prefix in the `#### Tasks` checklist
- Cross-project notification `#### Task:` headings use the format `[Ping from <source>] <task>`

### Issue File Conventions (Projects/<Project>/Issues/<task-slug>.md)

**Frontmatter fields:**

| Field | Required | Description |
|---|---|---|
| `title` | Yes | The task description (same as the daily note task) |
| `type` | Yes | Must be `issue` |
| `project` | Yes | The project this issue belongs to (PascalCase) |
| `status` | Yes | One of: `in-progress`, `resolved`, `blocked`, `cancelled` |
| `daily_note` | Yes | `[[wikilink]]` to the daily note where work is tracked — this is the traceability anchor |
| `wiki_pages` | No | Array of `[[wikilinks]]` to LLM Wiki pages created during research |
| `created` | Yes | Date the issue was first created |
| `updated` | Yes | Date of last update |

**Traceability chain:**

```
Daily note task (Daily/YYYY-MM-DD.md)
  │
  ├──→ Issue file (Projects/Project/Issues/task-slug.md)
  │       └── daily_note: [[Daily/YYYY-MM-DD]]  ← backlink to daily note
  │
  └──→ LLM Wiki pages (wiki/concepts/...)
          └── referenced in issue's wiki_pages frontmatter
```

Every issue file **must** have a `daily_note` frontmatter field linking back to the daily note entry. This ensures the two-way traceability: the daily note records when work happened, the issue file records what work was done and why.

## Edge Cases

- **No today's note exists** → create from template first (Step 2)
- **Project section exists but has no `#### Tasks`** → add it before the first `#### Task:`
- **Multiple entries for same project in same session** → append chronologically under the existing project section
- **No files changed** → skip the Files Changed table entirely
- **User wants to log without a specific project** → use project name `General` (no kanban link)
- **Daily note has unexpected format** → append gracefully, don't break existing content
- **Cross-project notification to a project with no section yet** → create the section under `## Worklog` (same as creating a new project section, but with the notification format)
- **Cross-project notification to a project that already has a section** → append under the existing section, after the last `####` heading
- **Multiple cross-project notifications in one session** → each goes under the appropriate target section; if the same target receives multiple pings, all are grouped under its existing section
- **User says "that's all for today"** → trigger the session close-out sequence:
  1. Run `session-logger` for the final wrap-up entry (completion summary, overall progress)
  2. Run `task-issue-auditor` on today's daily note to audit task-to-issue traceability
  3. Present the audit report and offer auto-fixes for missing/stale issue files
  4. After fixes, suggest running `vault-git-sync` to commit all changes
- **Cross-project notification for a project that doesn't have a kanban board yet** → skip the `📋` kanban link and add a note: "Create kanban board at `Projects/<Project>.md`"
- **Acknowledging a cross-project notification from the target side** → update the checklist item from `[ ]` to `[x]` with `✅ YYYY-MM-DD`; add `**Acknowledged by:**` line to the task body; if an issue file exists, update its status to `resolved` and add a resolution note
- **Ambiguous project name** → ask the user to clarify which vault project they mean; check the workspace AGENTS.md or vault conventions for the project-to-repo mapping
- **Issue file already exists for this task** → append new worklog content to the existing file (don't overwrite); update the `updated` date in frontmatter
- **Project has no Issues/ directory** → check if `Projects/<project>/Issues/` exists; if not, skip issue creation and log a note: "No Issues/ directory found for `{{project}}`. Create one with the `create-project` skill or manually."
- **`{{create_issue}}` is true but the task is marked `done` with no details** → skip issue creation (nothing to write up); a one-line completion doesn't warrant a full issue file
- **`{{create_issue}}` is true but the project doesn't exist in Projects/** → skip issue creation; log a note: "Project `{{project}}` doesn't exist yet. Create it with the `create-project` skill first."
- **Appending to an existing issue with new wiki_pages** → merge new `wiki_pages` entries into the frontmatter array without duplicating existing ones; use `obsidian_update_frontmatter` with the merged array
- **User says "close the issue" or "resolve the issue"** → update the issue file's `status` to `resolved`, update `updated` date, and mark the corresponding daily note task as done
- **Multiple daily note entries for the same task** → each additional entry appends a new worklog subsection to the existing issue file (see Step 7e append mode)
