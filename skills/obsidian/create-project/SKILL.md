---
name: create-project
description: Create a new project directory in the Obsidian vault under Projects/. Sets up the standard project structure: About.md (project write-up), Project.md (kanban board), and Issues/ (issue write-up directory with README index). Populates initial Backlog items from user-provided goals.
---

# Create Project

Create a new project directory in the vault's `Projects/` system. Every project follows a standard layout with three components: an about page describing the project, a kanban board for goal tracking, and an issues directory for write-ups.

For the full project system specification, read `Projects/AGENTS.md` in the vault.

## When to Use

- **New project:** "create a project for X", "start tracking work on Y"
- **New initiative:** "I want to start a project for the website redesign"
- **Adding structure:** "set up a project directory for the monitoring stack"
- **Onboarding new work:** before running `session-logger` for a task that belongs to a project that doesn't exist yet

## Input

| Input | Description | Example |
|---|---|---|---|
| `{{project_name}}` | The project's display name (PascalCase) | `Website`, `Monitoring`, `Data-Fortress` |
| `{{description}}` | Optional. One-paragraph project purpose | "Monorepo for the team website and blog" |
| `{{github_url}}` | Optional. Link to the GitHub repository | `https://github.com/soriPhoono/website` |
| `{{architecture_notes}}` | Optional. Architecture overview, key files, tech stack notes (goes in About.md ## Architecture section) | "Flux CD with SSH bootstrap, k0s runtime, NetBird VPN" |
| `{{initial_goals}}` | Optional. List of Backlog items to seed the kanban | `["Design landing page", "Set up CI/CD", "Write docs"]` |

If `{{project_name}}` is provided in a different format (kebab-case, space-separated), convert it:
- `my-project` → `My-Project`
- `my project` → `My-Project`
- `My Project` → `My-Project`

## Workflow

### Step 1: Validate the Project Doesn't Already Exist

Check the vault before creating anything:

1. **List existing projects:** `obsidian_list_directory("Projects/")` — check the dirs array for your project name
2. **If it exists:** Inform the user: "A project named `{{project_name}}` already exists at `Projects/{{project_name}}/`. Opening existing project instead."
   - Read `About.md` and `Project.md` to summarize current state
   - Do NOT create a duplicate
3. **If it doesn't exist:** Proceed with creation.

### Step 2: Gather Missing Input

If any of the optional inputs are missing, ask the user:

- **`{{description}}`**: "What is the purpose of this project? (1-2 sentences)"
- **`{{github_url}}`**: "Is there a GitHub repository for this project? (optional)"
- **`{{architecture_notes}}`**: "Any architecture notes or key files to document? (optional — can be added later)"
- **`{{initial_goals}}`**: "Any initial goals or tasks to seed the Backlog? (optional, comma-separated list)"

If the user doesn't provide optional fields, leave sensible defaults:
- No description → use "{{project_name}} project" as a minimal placeholder
- No GitHub URL → omit the Repositories section
- No initial goals → leave Backlog empty
- No architecture notes → omit the Architecture section from About.md

### Step 3: Create the Project Directory

Create the directory structure. `obsidian_write_note` auto-creates parent paths, so writing the first file will create the directory.

**Create `Projects/{{project_name}}/About.md`:**

Per `Projects/AGENTS.md`, the About.md should contain "project description, architecture notes, and GitHub link." Create the file with as many of these as the user has provided:

```markdown
---
title: "About {{project_name}}"
type: project-about
project: "{{project_name}}"
updated: {{today}}
---

# {{project_name}} — {{project_name}} Project

## Purpose

{{description}}

{% if github_url %}
## Repositories

- **GitHub:** [{{github_url}}]({{github_url}})
{% endif %}

{% if architecture_notes %}
## Architecture

{{architecture_notes}}
{% endif %}
```

If the user provided architecture notes, include them. If not, omit the Architecture section — it can be added later as the project evolves (per `Projects/AGENTS.md`, About.md is "updated throughout sessions like the daily note").

**Create `Projects/{{project_name}}/Project.md`:**

The kanban board in Obsidian Kanban plugin format:

```markdown
---
kanban-plugin: board
---

## Backlog

{% for goal in initial_goals %}
- [ ] {{goal}}
{% endfor %}

## In Progress


## Done



%% kanban:settings
```
{"kanban-plugin":"board","list-collapse":[false,false,false]}
```
%%
```

**Create `Projects/{{project_name}}/Issues/README.md`:**

```markdown
---
title: "{{project_name}} Issues"
type: issue-index
project: "{{project_name}}"
---

# {{project_name}} — Issue Write-Ups

This directory contains write-ups from issue solving sessions on the kanban board. Each file documents an issue from discovery through resolution.

## Active Issues

<!-- List active issue documents here as they are created -->

## Resolved Issues

<!-- List resolved issue documents here -->
```

### Step 4: Verify Project Structure

Confirm all files were created by reading them back:

```
obsidian_read_note("Projects/{{project_name}}/About.md")
obsidian_read_note("Projects/{{project_name}}/Project.md")
obsidian_read_note("Projects/{{project_name}}/Issues/README.md")
```

Verify:
- `About.md` has correct frontmatter (`type: project-about`, `project: "{{project_name}}"`)
- `Project.md` has the `kanban-plugin: board` frontmatter
- `Issues/README.md` has correct frontmatter (`type: issue-index`, `project: "{{project_name}}"`)

### Step 5: Present Results

```markdown
## Project Created: {{project_name}}

### Directory Structure

```
Projects/{{project_name}}/
├── About.md          Project write-up
├── Project.md        Kanban board
└── Issues/
    └── README.md     Issue index
```

### {{project_name}} Kanban

**Backlog ({{N}} items):**
- {% for goal in initial_goals %}{{goal}}
- {% endfor %}

### Next Steps

- Add tasks to the kanban board's Backlog or In Progress columns
- Reference the project in today's daily note: `📋 [[Projects/{{project_name}}/Project|{{display_name}} Kanban]]`
  - Use a descriptive display name in the kanban link (e.g., "Guenivir Cluster Kanban", not just "Guenivir Kanban")
- Tag daily note entries with the lowercase kebab-case version of the project name (e.g., `guenivir`, `llm-wiki`)
- Run `session-logger` to log work against this project
- If this project needs a GitHub repository synced, configure it separately
```

## Templates Reference

### Project Directory Structure

```
Projects/<Project-Name>/
├── About.md               Project write-up with architecture notes and GitHub links
├── Project.md             Kanban board (Obsidian Kanban plugin format)
└── Issues/                Issue write-ups from kanban problem solving
    └── README.md          Issue index
```

### About.md Frontmatter

| Field | Required | Validation |
|---|---|---|
| `title` | Yes | `"About <Project Name>"` format |
| `type` | Yes | Must be `project-about` |
| `project` | Yes | PascalCase project name matching the directory |
| `updated` | Yes | Valid date `YYYY-MM-DD` |

### Project.md Frontmatter

| Field | Required | Validation |
|---|---|---|
| `kanban-plugin` | Yes | Must be `board` |

### Issues/README.md Frontmatter

| Field | Required | Validation |
|---|---|---|
| `title` | Yes | `"<Project Name> Issues"` format |
| `type` | Yes | Must be `issue-index` |
| `project` | Yes | PascalCase project name matching the directory |

### Naming Conventions (per Projects/AGENTS.md)

| Convention | Rule | Example |
|---|---|---|
| **Directory names** | PascalCase matching the project name | `Guenivir`, `LLM-Wiki`, `My-Project` |
| **About filenames** | Always `About.md` | `Projects/<Name>/About.md` |
| **Kanban filenames** | Always `Project.md` | `Projects/<Name>/Project.md` |
| **Display names** | Use full descriptive name in kanban links | "Guenivir Cluster Kanban", not "Guenivir Kanban" |
| **Daily note tags** | Lowercase kebab-case of project name | `guenivir`, `llm-wiki`, `my-project` |

### Standard Kanban Columns

| Column | Purpose |
|---|---|
| **Backlog** | Future goals, not yet started |
| **In Progress** | Active work items |
| **Done** | Completed items |

An optional **Blocked** column can be added for items waiting on external dependencies.

### Kanban-Task Relationship

Each kanban item maps to a `#### Task:` subsection in the daily worklog:

```
Kanban board (Projects/X/Project.md)     Daily worklog (Daily/YYYY-MM-DD.md)
─────────────────────────────            ──────────────────────────────────────
Done                                         ### Project: X
  task name  ── maps to ──>                  #### Task: task name
                                              Narrative of what was done
```

### Project Lifecycle (per Projects/AGENTS.md)

Once created, a project follows this lifecycle:

| Phase | Actions |
|---|---|
| **Starting a task** | Move kanban item to In Progress → create `#### Task:` in daily note → add narrative + files |
| **Completing a task** | Move to Done → check off in daily note with `✅` → update Files Changed table |
| **Blocked task** | Move to Blocked column → note blocker in daily note → link external references |
| **Issue write-up** | For deep investigations, create `Issues/<title>.md` documenting problem → research → resolution |
| **Session end** | Move completed items to Done → reschedule Backlog → commit with `feat(project):` |

## Edge Cases

- **Project already exists** → open existing project, don't overwrite
- **Project name with spaces or hyphens** → convert to PascalCase directory: `My Project` → `My-Project`, `my-project` → `My-Project`
- **Project name starts with lowercase** → PascalCase: `homelab` → `Homelab`
- **Project name contains special characters** → strip non-alphanumeric (except hyphens) for directory name; keep original in display name
- **Hyphenated PascalCase** (e.g., `Data-Fortress`) → keep hyphens; directory name is `Data-Fortress`, tag is `data-fortress`
- **User provides a repo URL that's not GitHub** → still include it in About.md under Repositories section, label with the correct provider
- **User says "just set up the directory, I'll fill in details later"** → create with minimal About.md (title + purpose heading only) and empty kanban
- **Accidental creation** → no undo; files must be manually deleted or moved
- **User provides initial goals but they're too vague** → ask for clarification rather than adding ambiguous Backlog items
- **User provides architecture notes that are too brief** → still include them; they can be expanded later (About.md is live-updated)
