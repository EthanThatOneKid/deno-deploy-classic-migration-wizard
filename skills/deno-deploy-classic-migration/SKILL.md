---
name: deno-deploy-classic-migration
description: Guides and executes migrations from Deno Deploy Classic to the new Deno Deploy platform. Use this skill whenever the user mentions Deploy Classic migration, dash.deno.com shutdown, moving an app to console.deno.com, deployctl sunset, Deno KV migration, Deno queues migration, custom domain migration, Deno.serve updates, or asks an agent to handle the migration instead of only walking a human through a wizard.
---

# Deno Deploy Classic Migration

Use this skill to turn the repository's shell wizard into an agent-operable migration workflow. The shell wizard is still the codified flow for a human operator, but the agent should actively inspect the app, make safe code changes when asked, and produce concrete migration artifacts.

Primary references:

- Official migration guide: https://docs.deno.com/deploy/migration_guide/
- Deno KV on new Deploy: https://docs.deno.com/deploy/reference/deno_kv/
- Local wizard in this repo: `deno-deploy-classic-migration-wizard.sh`

## Operating Model

Start from the official migration guide, then use the shell wizard as the local checklist. The guide defines the platform facts; the wizard defines the step order and prompts; this skill defines what an agent should do beyond prompting a human.

The migration guide covers these stages:

- Create a new organization and app in `console.deno.com`.
- Deploy via GitHub integration or `deno deploy`; `deployctl` is being sunset.
- Recreate environment variables across production, development, and build timelines.
- Migrate custom domains with new TLS challenge records and DNS cutover.
- Replace legacy standard-library `serve()` with built-in `Deno.serve()`.
- Keep `Deno.cron()` usage, but review scheduling and logs after deploy.
- Replace unsupported Deno queues with an external or database-backed queue.
- Treat KV data as not automatically migrated.
- Plan around changed region availability.

## Agent Workflow

1. Inspect the repository before advising.
2. Run source searches for the migration guide's risk areas.
3. Classify each finding as `ready`, `needs edit`, `manual dashboard step`, or `blocked`.
4. Make small code changes directly when the user asked the agent to handle the migration.
5. Use the shell wizard as the evidence checklist, not as a substitute for source inspection.
6. Keep dashboard-only tasks explicit and grouped for the human.
7. Verify syntax/tests with the repo's existing tools, or state exactly why verification could not run.

## Source Searches

Run searches equivalent to these before making recommendations:

```bash
rg -n 'std(@[0-9.]+)?/http/server\.ts|from ["'\'' ]?@std/http/server|serve\(' . --glob '!*.lock' --glob '!.git'
rg -n 'Deno\.Kv\.(enqueue|listenQueue)|\.enqueue\(|listenQueue\(' . --glob '!*.lock' --glob '!.git'
rg -n 'Deno\.openKv|Deno\.Kv|openKv\(' . --glob '!*.lock' --glob '!.git'
rg -n 'deployctl|deno deploy|Deno\.cron|cron\(' . --glob '!*.lock' --glob '!.git'
```

Also inspect deployment config, GitHub Actions, README deployment docs, environment variable usage, and DNS/custom domain references if present.

## How To Use The Shell Wizard

Use `deno-deploy-classic-migration-wizard.sh` in one of three ways:

- Run it when the user wants a human-guided interactive checklist.
- Read it when the agent needs the canonical local flow without interacting.
- Keep its stage names in the final report so a human can map agent work back to the wizard.

Do not hide behind the wizard. If the repo has code that can be migrated safely, edit the code and verify it.

## KV Migration

The migration guide says existing KV data is not automatically migrated. The new Deploy KV reference says apps normally call `Deno.openKv()` after a database is assigned, while one-time local scripts can connect with:

```ts
await Deno.openKv("https://api.deno.com/v2/databases/<Database ID>/connect");
```

Those local scripts authenticate with `DENO_KV_ACCESS_TOKEN`.

When the app needs KV data migrated:

- Inspect the app's actual key prefixes and key shapes.
- Prefer app-specific one-time scripts over generic whole-database copy advice.
- Default to preview/export, dry-run import, and skip-existing writes.
- Never delete source data automatically.
- Warn that key expiration metadata is not preserved by simple `kv.list()` export/import scripts; reapply TTLs in app-specific code if the app relies on expiration.
- Keep `support@deno.com` as an escape hatch for remote access failures, very large datasets, unknown keyspace, or low confidence verification.

Useful generated artifacts:

- `scripts/kv-export-once.ts` for source previews and JSONL export.
- `scripts/kv-import-once.ts` for target dry-run/import.
- A short migration runbook containing source/target database IDs, selected prefixes, dry-run counts, import counts, and verification samples.

## Code Change Guidance

For HTTP server migration:

- Remove imports from `deno.land/std/.../http/server.ts` or `@std/http/server` when they provide the old `serve()` API.
- Replace `serve(handler)` with `Deno.serve(handler)`.
- Preserve existing handler behavior and response types.

For queues:

- Do not pretend queues still work on new Deploy.
- If the user asks for implementation, introduce the smallest viable external queue or database-backed job model for the app.
- If queue replacement needs product choices, document the blocker rather than inventing infrastructure.

For deployment:

- Prefer `deno deploy` or new Deploy GitHub integration.
- Remove or update stale `deployctl` instructions when the user asks for repo migration.

## Final Report

Return this structure:

```markdown
## Migration Status
- Overall: ready | partially ready | blocked
- Official guide: https://docs.deno.com/deploy/migration_guide/
- Wizard: deno-deploy-classic-migration-wizard.sh

## Completed
- [stage]: evidence

## Code Changes
- file: what changed and why

## Manual Dashboard Steps
- step and exact console/doc URL

## KV Plan
- source/target connection approach, prefixes, dry-run/import/verification status

## Risks
- risk and mitigation

## Verification
- command: result
```

If no code changes were made, say so directly and explain whether the remaining work is dashboard-only, blocked by credentials, or waiting for user confirmation.
