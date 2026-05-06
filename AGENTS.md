# Agent Instructions

This repository is a **skill pack** consumed by AI coding agents working in GrandGale Technologies' Python backend projects.

## How to use this repo

1. Treat each folder under [`skills/`](skills/) as an independent skill.
2. Read the skill's `SKILL.md` **first** — its YAML frontmatter `description` tells you when to activate it.
3. Only open a file from `references/` if its filename clearly matches the current sub-task. Don't pre-load all references.
4. When a skill's instructions conflict with this file, the **skill wins** — skills are the authoritative source.

## Activation rules

Match the user's request against each skill's `description: "Use when: ..."` clause:

- Writing/altering an Alembic migration → load `skills/alembic-migrations/SKILL.md` and `skills/migration-naming/SKILL.md`.
- Writing any Python file in `backend/app/` → load `skills/grandpython/SKILL.md`.
- Building or modifying a FastAPI route, dependency, or response model → load `skills/fastapi/SKILL.md`.
- Building, editing, or testing a Pydantic AI agent → load `skills/building-pydantic-ai-agents/SKILL.md`.
- Adding observability, tracing, or `logfire.*` calls → load `skills/logfire-instrumentation/SKILL.md`.
- Writing code that imports `pywa` or `pywa_async` → load `skills/pywa-usage/SKILL.md`.
- PostHog + Next.js App Router setup → load `skills/integration-nextjs-app-router/SKILL.md`.
- Composing a git commit message → load `skills/commit-messages/SKILL.md`.

When in doubt, scan [`skills.json`](skills.json) — it lists every skill with its description and reference files.

## Editing this repo

If you (the agent) are asked to add or modify a skill **in this repo itself**:

- Keep `SKILL.md` concise; push depth into `references/<topic>.md`.
- Maintain YAML frontmatter with at minimum `name` and `description`.
- After any change, update [`README.md`](README.md) and [`skills.json`](skills.json) so the index stays in sync.
- Use the conventions in [`skills/commit-messages/SKILL.md`](skills/commit-messages/SKILL.md) for the commit.
