---
name: migration-naming
description: "Name Alembic migration files for GrandGale backend projects. Use when: creating migrations, generating alembic revisions, naming database migration files."
argument-hint: "Describe the migration operation (e.g. create businesses table, add column to users)"
---

# Migration Naming Convention

## Format
{sequence}_{operation}_{table}

- **sequence** — zero-padded 3-digit integer that determines run order: `000`, `001`, `002`, …
- **operation** — 2-letter code (see table below)
- **table** — snake_case table name

## Operation Codes

| Code | Meaning | Example |
|------|---------|---------|
| `ct` | create table | `000_ct_businesses` |
| `at` | alter table | `005_at_users` |
| `ac` | add column | `006_ac_contacts_opted_in` |
| `rc` | remove column | `007_rc_campaigns_legacy_field` |
| `rt` | rename table | `008_rt_tenants_to_businesses` |
| `rn` | rename column | `009_rn_users_tenant_id_to_business_id` |
| `dt` | drop table | `010_dt_legacy_table` |
| `ci` | create index | `011_ci_contacts_phone` |
| `di` | drop index | `012_di_contacts_old_idx` |
| `ck` | add constraint | `013_ck_users_phone_unique` |
| `dk` | drop constraint | `014_dk_users_old_constraint` |
| `sd` | seed data | `015_sd_billing_plans` |

## Rules

- Always use the **next available sequence number** — check existing migrations in `backend/alembic/versions/` first.
- When naming with Alembic CLI: `alembic revision --autogenerate -m "000_ct_businesses"`
- The Alembic-generated filename will be `000_ct_businesses_{rev}.py` — the message becomes the human-readable prefix with the revision hash appended.
- For multiple related operations in one migration, pick the dominant operation code.
- If a migration spans multiple tables, use the primary/driving table in the name.

## Examples
000_ct_businesses — initial businesses table
001_ct_users — initial users table
002_ct_contacts — contacts table
003_ct_campaigns — campaigns table
004_ac_businesses_logo_url — add logo_url column to businesses
005_sd_billing_plans — seed starter/growth/pro plans
006_ci_contacts_phone — add index on contacts(business_id, phone)
