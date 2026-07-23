---
name: vault-structure
description: Guidelines and rules for interacting with the obsidian vault and managing files within. Use any time you are interacting with the obsidian vault for context as to it's layout and structure, or to understand where different types of files should go.
---

# Vault Structure & Interaction Rules

This skill provides full context on the layout, naming conventions, and constraints of the user's personal Obsidian vault. Use this guideline whenever reading, writing, or organizing notes, daily logs, or project documents.

## Vault Location

The Obsidian vault is located on the filesystem at:
- **Path**: `~/Nextcloud/Vault` (evaluates to `/home/sphoono/Nextcloud/Vault` on the local machine).

---

## Directory Layout

The vault is structured with a numerical folder hierarchy:

### `00 Dashboard`
Contains core landing pages, hubs, and interactive indexes for the vault.
*   **Examples**: `Welcome.md`, `Question ideas for Q&A.md`.

### `01 Daily`
Contains daily logs. Notes are structured under directories by Year and Month.
*   **Path structure**: `/01 Daily/<Year>/<MonthNumber>-<MonthName>/`
    *   *Example*: `/01 Daily/2026/07-July/`
*   **Naming convention**: `YYYY-MM-DD-dddd.md`
    *   *Example*: `2026-07-05-Sunday.md`

### `02 Weekly`
Contains weekly notes tracking objectives and summaries.
*   **Path structure**: `/02 Weekly/<Year>/`
*   **Naming convention**: `YYYY-W[WW].md`
    *   *Example*: `2026-W23.md`

### `03 Monthly`
Contains monthly logs.
*   **Path structure**: `/03 Monthly/<Year>/`
*   **Naming convention**: `YYYY-MM-<MonthName>.md`
    *   *Example*: `2026-06-June.md`

### `04 Quarterly`
Contains quarterly checkpoints.
*   **Path structure**: `/04 Quarterly/<Year>/`
*   **Naming convention**: `YYYY-Q[Q].md`
    *   *Example*: `2026-Q2.md`

### `05 Yearly`
Contains high-level yearly reviews.
*   **Path structure**: `/05 Yearly/`
*   **Naming convention**: `YYYY.md`
    *   *Example*: `2026.md`

### `06 Templates`
System configuration skeletons, modular components, and templating scripts.
*   **`Parents/`**: Skeletal outlines for periodic notes (e.g., `Daily.md`, `Weekly.md`, `Monthly.md`).
*   **`Components/`**: Modular layout sections embedded via `tp.file.include` and Obsidian Meta Bind (e.g., charts, progress wheels, navigation bars).
*   **`Images/`**: Banners, GIFs, icons, audio, and visual assets used in note styling.
*   **`Scripts/`**: Support routines categorized under `api/`, `templater/`, `services/`, etc.

### `07 Notes`
Academic, study, or general reference notes not tied to a specific project.
*   **Subdirectories**: `Images/`, `Extras/` (e.g., homework, post-lab questions).

### `08 Projects`
The primary **agentic workspace** within the vault. This folder houses notes, plans, designs, and research for all personal development projects (e.g., `Homelab`, `Guenivir`).
*   **Structure per project**:
    *   `08 Projects/<ProjectName>/Index.md`: Roadmap, task lists, and index of topic notes.
    *   `08 Projects/<ProjectName>/Topics/`: Folder containing specific topic-based design documents, research, and planning notes.

---

## Frontmatter & Metadata Standards

Notes inside the vault utilize frontmatter YAML block metadata. When interacting with notes:

1.  **Preserve Existing Fields**: Never discard existing YAML properties (such as `date`, `cssclasses`, `tags`, or project relations). Update or append fields rather than recreating the block.
2.  **Daily Note Fields**:
    ```yaml
    cssclasses:
      - image-borders
      - image-small
      - timegarden-daily
      - <weekday-lowercase>
      - daily
    date: YYYY-MM-DD
    alias: <optional-alias>
    dayRating: 1
    tags: "#type/daily-note"
    journal: daily
    journal-date: YYYY-MM-DD
    journal-start-date: YYYY-MM-DD
    journal-end-date: YYYY-MM-DD
    aiAnswer: ""
    ```
3.  **Project Topic Fields**:
    ```yaml
    date: YYYY-MM-DD
    project: <ProjectName>
    tags:
      - topic-tags
    ```

---

## Rules for Editing and Interaction

### 1. Templater Syntax Protection
*   Periodic notes and templates contain active Templater blocks (`<% ... %>` or `<%* ... %>`).
*   **DO NOT** modify, corrupt, or strip Templater execution logic unless specifically instructed to edit a template itself.

### 2. Meta Bind & Interactive UI Elements
*   The vault heavily integrates the **Obsidian Meta Bind** and **Dataview** plugins.
*   Retain Meta Bind button tags (e.g. `BUTTON[prev-day, current-week, next-day]`), code blocks (`meta-bind-button`, `meta-bind-slider`), and any embedded Javascript or charts.

### 3. Wiki Links (`[[Link]]`)
*   Prefer double-bracket wiki links `[[Note Name]]` for referencing internal vault notes.
*   When referencing files in subfolders, use absolute paths relative to the vault root, such as `[[/08 Projects/Homelab/Index]]` or `[[/01 Daily/2026/07-July/2026-07-05-Sunday]]`.

### 4. Project Updates
*   When writing or updating project plans, ensure items are linked back to the project's main `Index.md` (roadmap/TODO list).
*   Structure task lists using markdown checkboxes (`- [ ]`) and maintain consistent styling (e.g. adding date checks like `✅ YYYY-MM-DD` on completed tasks).

