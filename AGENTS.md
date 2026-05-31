# AGENTS.md

Custom agent skills for AI-assisted development workflows in the Obsidian vault and NixOS environment. This repository belongs to the **Agentics** vault project (`Projects/Agentics/` in the Obsidian vault).

> [!TIP]
> Skills are registered into the Nix skills system via the Homelab repo's `nix/homes/sphoono/configs/agentics/agents/skills.nix` registry. After adding or updating a skill here, update the registry and lock file there.

## Repository Structure

```
skills/
├── AGENTS.md                    This file
├── skills/                      Skill definitions organized by category
│   ├── obsidian/                Vault integration skills (file I/O, search, tags)
│   │   ├── session-logger/
│   │   ├── daily-note-manager/
│   │   ├── frontmatter-linter/
│   │   ├── tag-sanitizer/
│   │   ├── wiki-index-regenerator/
│   │   ├── vault-git-sync/
│   │   ├── deep-research/
│   │   ├── quick-research/
│   │   ├── source-ingest/
│   │   ├── organize-raw-sources/
│   │   ├── create-project/
│   │   ├── task-issue-auditor/
│   │   └── session-closeout/
│   └── ... (other skill categories)
├── nix/                         (optional) Nix packaging if skills need dependencies
└── flake.nix                    (optional) Flake for Nix-consumable skill set
```

## Important Commands & Workflows

- **No build step.** Skills are Markdown files — edit directly and commit.
- **Validation:** Before committing, verify skill frontmatter is valid YAML and the `name:` field matches the directory name.
- **Commit convention:** Use conventional commits:
  - `feat(skills):` — new skill
  - `docs(skills):` — documentation or SKILL.md updates
  - `fix(skills):` — bug fixes in skill instructions
- **Skills registry:** After adding a new skill here, it must be registered in `github:soriPhoono/homelab` at `nix/homes/sphoono/configs/agentics/agents/skills.nix`, then deployed via `nh os switch .`.

## Skill Format (SKILL.md)

Every skill is a directory containing a `SKILL.md` file with:

```yaml
---
name: skill-name
description: One-line description of what the skill does and when it activates.
---
```

The SKILL.md contains:
- **Frontmatter** — `name` and `description` fields
- **Overview** — what the skill does
- **When to Use** — trigger phrases and use cases
- **Inputs** — parameters the skill accepts
- **Workflow** — step-by-step instructions
- **Templates Reference** — file formats, frontmatter schemas
- **Edge Cases** — error handling and odd scenarios

## Vault Project Mapping

| Repo | Vault Project | Vault Path |
|---|---|---|
| `github:soriPhoono/skills` | **Agentics** (Skills) | `Projects/Agentics/` |
| `github:soriPhoono/homelab` | Homelab | `Projects/Homelab/` |
| `github:soriPhoono/guenivir` | Guenivir | `Projects/Guenivir/` |

Skills are developed in this repo, registered in the Homelab repo's Nix config, and consumed by agents working in the Obsidian vault.

## Key Dependencies

- **Obsidian vault** (`~/Nextcloud/Notes/`) — the target environment for Obsidian category skills
- **Homelab repo** (`github:soriPhoono/homelab`) — skills registry that consumes these definitions
- **Nix flake input** — the skills repo is added as a flake input to the Homelab repo

## Gotchas

- **Skill name must match directory name.** The `name:` frontmatter field in SKILL.md and the parent directory name must be identical. This is how the loading mechanism discovers skills.
- **Description is the trigger hint.** The `description:` field is what agents see when deciding which skill to load. Make it descriptive of the trigger scenario.
- **No file renaming without registry update.** If you rename a skill directory or subpath, you must also update the registry in `skills.nix` and the `skills-lock.json` in the Homelab repo.
- **Cross-skill references.** Skills can reference each other in their SKILL.md (e.g., "For deeper research, see `deep-research`"). Use backtick code spans for skill names so agents can discover them.
- **Quick-research was missed.** When initially set up, `quick-research` existed in the repo but was not registered in the Nix config. Always verify registration after adding new skills.
