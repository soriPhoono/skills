---
name: daily-note-manager
description: Create today's daily note from template, review yesterday's for incomplete tasks, and validate section ordering against the Daily/AGENTS.md specification.
---

# Daily Note Manager

Manages the lifecycle of daily notes: creation from template, morning review of yesterday's unfinished business, and structural validation against the format spec in `Daily/AGENTS.md`.

## When to Use

- **Morning start:** "create today's note", "start my daily note", "open daily note"
- **Daily review:** "review yesterday", "check what's still open from yesterday"
- **Format check:** "validate my daily note", "check format compliance"
- **Any session start** (should be the first skill called at the beginning of a work session)

## Workflow

### Step 1: Check for Today's Note

1. Use `obsidian_search_notes` with today's date (formatted `YYYY-MM-DD`) to find if `Daily/YYYY-MM-DD.md` exists
2. If found, read it with `obsidian_read_note` to verify it's complete
3. If not found, proceed to Step 2 (creation)

### Step 2: Create Today's Note (if missing)

1. **Read the template** at `Daily/_templates/daily-note.md` via `obsidian_read_note`
2. **Process template variables:**
   - Replace all `{{title}}` with today's date (e.g., `2026-05-30`)
   - Replace all `{{date}}` with today's date
3. **Ask the user** for:
   - Initial tags (suggest common ones from `Daily/AGENTS.md`: `guenivir`, `llm-wiki`, `terraform`, `flux`, `k0s`, `netbird`, `authentik`)
   - What they plan to work on today (to pre-populate the Summary)
4. **Write the note** via `obsidian_write_note`:
   - Path: `Daily/YYYY-MM-DD.md`
   - Frontmatter: `title: "YYYY-MM-DD"`, `type: daily`, `tags: [tag1, tag2]`
   - Content: the processed template
5. **Confirm**: "Created Daily/YYYY-MM-DD.md with tags: [tag1, tag2]"

### Step 3: Review Yesterday's Note (optional)

If today is a new day and yesterday's note exists:

1. Read `Daily/YYYY-MM-DD.md` where date = today - 1 day
2. Scan for incomplete tasks (`- [ ]` items without `✅`)
3. Present them to the user:

```
Yesterday's incomplete tasks:

- [ ] Task description (from Project: guenivir)
- [ ] Another task (from General todo)

Reschedule, close, or carry them forward?
```

Options for each task:
- **Carry forward:** Copy the task to today's `## General todo` section with today's date
- **Mark done:** If the user confirms completion, update yesterday's note
- **Cancel:** Mark as cancelled (remove or note as wontfix)
- **Defer:** Update the `📅` date on the task

### Step 4: Validate Section Order

If today's note already exists, validate its structure against `Daily/AGENTS.md`:

**Required sections in order:**
1. YAML frontmatter
2. `# YYYY-MM-DD` heading
3. `## Summary`
4. `## TOC`
5. `## General todo`
6. `## Worklog`

**Checks:**
- All sections present? → warn if any are missing
- Sections in correct order? → warn if out of order
- Frontmatter has `title`, `type`, `tags`? → flag missing fields
- `## Summary` is not empty? → info if blank
- `## TOC` has valid `[[#Section]]` links? → warn if broken

Report findings clearly. Offer to auto-fix ordering issues (reorder sections) and add missing empty sections.

### Step 5: Present Status

Summarize the state of today's and yesterday's notes:

```
Daily Note Status — 2026-05-30

📝 Today: Created with tags [guenivir, obsidian]
📅 Yesterday: 2 incomplete tasks found (1 in General, 1 in llm-wiki)
✓ Format: All sections present and in correct order
```

## Edge Cases

- **Template missing** (`Daily/_templates/daily-note.md` doesn't exist) → create a minimal daily note with the required sections from memory: frontmatter, `# {{date}}`, `## Summary`, `## TOC`, `## General todo`, `## Worklog`. Warn the user the template is missing.
- **Yesterday's note doesn't exist** → skip review, no action needed
- **Weekend gap** (Monday reviewing Friday) → review the last note in the `Daily/` directory that has a weekday date
- **Note already exists and is complete** → report "All good, nothing to do"
- **Template has custom variables** → prompt user for each unknown `{{variable}}`
