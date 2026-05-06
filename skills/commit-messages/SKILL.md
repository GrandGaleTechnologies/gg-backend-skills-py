---
name: commit-messages
description: "Generate commit messages for GrandGale backend projects. Use when: writing commit messages, generating commits, committing code, staging changes, git commit, summarizing changes for a commit."
argument-hint: "Describe the changes or leave blank to auto-generate from staged diff"
---

# Commit Message Convention

## Format

```
<type>(<scope>): <subject>
```

- **All lowercase** — no capital letters anywhere in the message.
- **No period** at the end of the subject.
- **Subject** is imperative mood, present tense (e.g. "add", not "added" or "adds").
- **Max 72 characters** for the full line.

## Types

| Type | When to use |
|------|------------|
| `feat` | A new feature or user-facing capability |
| `fix` | A bug fix |
| `chore` | Maintenance, config, dependencies, tooling |
| `refactor` | Code restructuring with no behavior change |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `style` | Formatting, linting, whitespace (no logic change) |
| `perf` | Performance improvement |
| `ci` | CI/CD pipeline changes |
| `build` | Build system or external dependency changes |

## Scopes

Use the scope that best matches the area of the codebase affected.

### Backend scopes

| Scope | Maps to |
|-------|---------|
| `auth` | `backend/app/auth/` — authentication & authorization |
| `billing` | `backend/app/billing/` — payments, Paystack |
| `businesses` | `backend/app/businesses/` — business CRUD |
| `campaigns` | `backend/app/campaigns/` — campaign management |
| `contacts` | `backend/app/contacts/` — contact management |
| `dashboard` | `backend/app/dashboard/` — dashboard analytics |
| `agents` | `backend/app/agents/` — AI agents (chatbot, campaign generator, etc.) |
| `whatsapp` | `backend/app/whatsapp/` — WhatsApp integration |
| `workers` | `backend/app/workers/` — background jobs, cron, Arq |
| `db` | `backend/app/db/`, `backend/alembic/` — database, migrations |
| `api` | General backend API changes spanning multiple modules |

### Frontend scopes

| Scope | Maps to |
|-------|---------|
| `ui` | Shared components, design system |
| `onboarding` | `frontend/src/app/onboarding/` |
| `join` | `frontend/src/app/join/` — team invite flow |
| `dashboard-ui` | `frontend/src/app/dashboard/` |
| `hooks` | `frontend/src/hooks/` |
| `lib` | `frontend/src/lib/` — utilities, API client |

### Cross-cutting scopes

| Scope | Maps to |
|-------|---------|
| `infra` | `infra/`, `docker-compose.yml`, Dockerfiles, Railway |
| `deps` | Dependency updates (`pyproject.toml`, `package.json`) |
| `config` | Configuration files (env, linting, TS config) |
| `docs` | `docs/`, READMEs |

## Procedure

1. Look at the staged diff or described changes.
2. Pick the **type** that best fits the primary intent of the change.
3. Pick the **scope** from the tables above. If a change touches multiple scopes equally, use the most significant one or omit the scope.
4. Write a concise imperative subject describing *what* changed.
5. Output the commit message.

## Examples

```
feat(campaigns): add bulk campaign scheduling endpoint
fix(whatsapp): handle expired session token on message send
chore(deps): bump pydantic to 2.10
refactor(agents): extract shared prompt builder
test(auth): add login rate-limit edge cases
docs: update whatsapp setup guide
ci(infra): add staging deploy workflow
feat(onboarding): add business profile step
fix(dashboard-ui): correct chart date range filter
style(config): fix ruff linting warnings
perf(db): add index on contacts.phone_number
build(infra): upgrade python base image to 3.12
feat(billing): integrate paystack subscription webhooks
chore: update makefile targets
```

## Rules

- One commit = one logical change. Don't mix unrelated changes.
- If scope is obvious from context or the change is project-wide, scope may be omitted: `chore: update makefile targets`.
- Never use a scope not listed above unless the codebase gains a new module — then add it to this file.
