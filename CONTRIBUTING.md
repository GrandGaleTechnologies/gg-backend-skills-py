# Contributing

Thanks for improving the GrandGale backend skill pack. This repo is consumed as a git submodule by every Python backend project, so changes here ship widely — keep edits focused and well-scoped.

## Repository layout

```
gg-backend-skills-py/
├── README.md           # human-facing index
├── AGENTS.md           # entry point for AI agents
├── skills.json         # machine-readable index (regenerate after edits)
├── LICENSE
├── CONTRIBUTING.md
└── skills/
    └── <skill-name>/
        ├── SKILL.md
        └── references/
            └── *.md
```

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: <skill-name>
   description: "<one-paragraph summary>. Use when: <comma-separated trigger phrases>."
   ---
   ```
   The `description` is what AI agents match against to decide whether to load the skill — write the trigger phrases concretely.
2. Keep `SKILL.md` short. Push depth (long examples, edge cases, framework deep-dives) into `references/<topic>.md`.
3. Add a row to the table in [`README.md`](README.md).
4. Regenerate [`skills.json`](skills.json):
   ```bash
   python3 scripts/build_index.py   # if/when added; otherwise edit by hand
   ```
5. Update [`AGENTS.md`](AGENTS.md) activation rules if the skill needs an explicit trigger.

## Editing an existing skill

- Preserve the existing frontmatter shape — downstream agents rely on it.
- Avoid project-specific names (e.g. "MarketMachine", "ShipNLogic"). Use **"GrandGale backend projects"** or describe the pattern generically.
- If a code example references a project-specific service name, leave it as a clearly contextual example.

## Style

- Markdown, GitHub-flavored.
- Tables for rule reference; fenced code blocks with language tags for examples.
- Prefer **"Yes / No"** example pairs over prose when illustrating conventions.
- Keep line length reasonable (~120 chars) but don't hard-wrap prose.

## Commit messages

Follow [`skills/commit-messages/SKILL.md`](skills/commit-messages/SKILL.md):

```
<type>(<scope>): <subject>
```

Examples:
- `feat(skills): add redis-caching skill`
- `docs(grandpython): clarify mapped_column nullability rule`
- `chore(index): regenerate skills.json`

## Releasing

This repo uses commit SHAs as the unit of release — downstream submodules pin to a specific commit. There are no version tags. After merging changes to `main`, downstream projects (which mount this repo at `.claude/skills/`) pull via:

```bash
git submodule update --remote --merge .claude/skills
```
