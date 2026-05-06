# gg-backend-skills-py

Shared **agent skills** for GrandGale Technologies' Python backend projects.

This repository is a curated, versioned collection of reusable skill packs that guide AI coding agents (GitHub Copilot, Claude Code, Cursor, and any agent that understands [agents.md](https://agents.md) / [Anthropic-style skills](https://docs.claude.com/en/docs/claude-code/skills)) when working in our codebases.

It is designed to be consumed as a **git submodule** so every project pulls from one source of truth.

## What's inside

| Skill | Purpose |
|-------|---------|
| [alembic-migrations](skills/alembic-migrations/SKILL.md) | Generate Alembic migration files with our inline-constraint style. |
| [building-pydantic-ai-agents](skills/building-pydantic-ai-agents/SKILL.md) | Build production agents with Pydantic AI — tools, streaming, testing. |
| [commit-messages](skills/commit-messages/SKILL.md) | Conventional, lowercase, imperative commit messages. |
| [fastapi](skills/fastapi/SKILL.md) | FastAPI conventions, dependencies, streaming, current best practices. |
| [grandpython](skills/grandpython/SKILL.md) | The GrandGale Python backend styleguide (services, routers, models, schemas, imports). |
| [integration-nextjs-app-router](skills/integration-nextjs-app-router/SKILL.md) | PostHog integration for Next.js App Router (cross-stack helper). |
| [logfire-instrumentation](skills/logfire-instrumentation/SKILL.md) | Add Pydantic Logfire spans, logging, and tracing. |
| [migration-naming](skills/migration-naming/SKILL.md) | Naming convention for Alembic migration files. |
| [pywa-usage](skills/pywa-usage/SKILL.md) | Build WhatsApp Cloud API integrations with `pywa` / `pywa_async`. |

A machine-readable index is also available at [skills.json](skills.json).

## Skill format

Every skill is a folder under [`skills/`](skills/) containing:

```
skills/<skill-name>/
├── SKILL.md          # entry point — YAML frontmatter + body
└── references/       # optional deep-dive reference docs
    └── *.md
```

The `SKILL.md` frontmatter follows the [Anthropic Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) shape:

```yaml
---
name: <skill-name>
description: "<what it does>. Use when: <trigger conditions>."
---
```

Agents should load `SKILL.md` first and only pull a file from `references/` if its title indicates the user's task needs it.

## Consuming this repo as a submodule

Downstream projects mount this repo at **`.claude/skills/`** so [Claude Code](https://docs.claude.com/en/docs/claude-code/skills) auto-discovers every skill alongside any project-local skills.

```bash
mkdir -p .claude
git submodule add https://github.com/GrandGaleTech/gg-backend-skills-py.git .claude/skills
git submodule update --init --recursive
```

This lays out:

```
<project>/
└── .claude/
    └── skills/                    # ← this repo, as a submodule
        ├── AGENTS.md
        ├── skills.json
        └── skills/
            ├── grandpython/SKILL.md
            ├── fastapi/SKILL.md
            └── ...
```

Add to your project's `AGENTS.md` (or `CLAUDE.md`, `.github/copilot-instructions.md`, etc.):

```markdown
## Skills

This project consumes shared skills from `.claude/skills/` (see
`.claude/skills/AGENTS.md`). Load any skill from
`.claude/skills/skills/<skill-name>/SKILL.md` when its trigger conditions
match the task.
```

### Pulling updates

When this repo is updated upstream, downstream projects refresh with:

```bash
git submodule update --remote --merge .claude/skills
git add .claude/skills && git commit -m "chore(skills): bump gg-backend-skills-py"
```

Pin to a specific commit by checking out that SHA inside the submodule before committing.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The short version:

1. Add or edit a skill under `skills/<skill-name>/`.
2. Keep `SKILL.md` as the single entry point; push depth into `references/`.
3. Update [README.md](README.md), [skills.json](skills.json), and [AGENTS.md](AGENTS.md) so the index stays accurate.
4. Use the `commit-messages` skill in this repo for the commit itself.

## License

[MIT](LICENSE) © GrandGale Technologies
