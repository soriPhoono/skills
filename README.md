# sphoono/skills

Custom agent skills for maintaining an Obsidian vault with **daily notes** and **LLM Wiki** systems.

Built for use with the [Agent Skills](https://agentskills.io) ecosystem — installable via `npx skills add`.

## Skills

Organized in a catalog layout: `skills/<category>/<name>/SKILL.md`

| Category | Skill | Description |
|---|---|---|
| **daily** | [`session-logger`](skills/daily/session-logger/SKILL.md) | Append structured worklog entries to today's daily note during a work session — project, tasks, files changed, status. |
| **daily** | [`daily-note-manager`](skills/daily/daily-note-manager/SKILL.md) | Create today's daily note from template, review yesterday's for completeness, validate section ordering. |
| **health** | [`frontmatter-linter`](skills/health/frontmatter-linter/SKILL.md) | Validate frontmatter across all vault note types (daily notes, wiki pages, raw sources, kanbans). Detects broken `[[wikilinks]]`, orphan pages, stale content, and cross-system inconsistencies. |
| **health** | [`tag-sanitizer`](skills/health/tag-sanitizer/SKILL.md) | Audit all vault tags: find near-duplicates, singletons, non-kebab-case tags. Propose and execute merges. |
| **wiki** | [`wiki-index-regenerator`](skills/wiki/wiki-index-regenerator/SKILL.md) | Rebuild `LLM-Wiki/wiki/index.md` from the actual filesystem — refresh topic tables, page counts, and statistics. |
| **infra** | [`vault-git-sync`](skills/infra/vault-git-sync/SKILL.md) | Stage grouped changes by vault area (daily/, wiki/, raw/, config/) and commit with conventional messages. |

## Dependencies

All skills interact with the vault exclusively through the **Obsidian MCP Server** (`obsidian_*` tools), making them portable across any vault that exposes the same MCP interface.

| Dependency | Required By | Purpose |
|---|---|---|
| [Obsidian MCP Server](https://github.com/nickolay/obsidian-mcp) | All skills | All vault read/write operations — no hardcoded paths. |
| [Obsidian Git Plugin](https://github.com/denolehov/obsidian-git) | `vault-git-sync` | Auto-commit, auto-push, and manual sync operations. The skill stages and commits via bash `git` but relies on this plugin for the push/sync pipeline. |

### Design Principle: MCP-First

Skills never hardcode vault paths. Every note access, search, write, or metadata query goes through the Obsidian MCP tools:

- `obsidian_search_notes` — find notes by content or frontmatter
- `obsidian_read_note` / `obsidian_write_note` / `obsidian_patch_note` — read and modify content
- `obsidian_get_frontmatter` / `obsidian_update_frontmatter` — frontmatter operations
- `obsidian_manage_tags` — add/remove/list tags
- `obsidian_list_directory` / `obsidian_get_vault_stats` — vault structure discovery
- `obsidian_list_all_tags` — tag inventory

The sole exception is `vault-git-sync`, which uses bash `git` commands for staging and committing, since the MCP protocol currently has no git interface.

## Installation

```bash
npx skills add /path/to/skills
```

Or once published on GitHub:

```bash
npx skills add sphoono/skills
```

## License

MIT
