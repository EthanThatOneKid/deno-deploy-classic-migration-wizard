## Migration Status

- Overall: partially ready
- Official guide: https://docs.deno.com/deploy/migration_guide/
- Wizard checklist: `deno-deploy-classic-migration-wizard.sh`
- Target: move one `dash.deno.com` Deploy Classic project to a new `console.deno.com` app.
- Source evidence: this repository contains the migration wizard, skill, README files, and eval metadata. It does not contain an app source tree such as `deno.json`, `main.ts`, `src/`, `.github/workflows/`, or deployment config. No app code was edited.

## Agent-Operable Runbook

### 1. Identify The Classic Project

- Classification: manual dashboard step
- Wizard stage: `Identify the project`
- Open: https://dash.deno.com/projects
- Record in the migration log:
- `DENO_CLASSIC_PROJECT_NAME=<classic project name>`
- `DENO_CLASSIC_PROJECT_URL=<classic project URL>`
- `CLASSIC_PRODUCTION_URL=<current production URL>`
- Evidence to capture: screenshot or copied project settings showing the Classic project name and current deployment URL.

### 2. Read The Official Migration Guide

- Classification: ready
- Wizard stage: `Read the migration guide`
- Open: https://docs.deno.com/deploy/migration_guide/
- Agent note: treat the guide as platform truth and this repository's wizard as the ordered checklist.
- Evidence to capture: link the guide version/date in the migration log.

### 3. Create The New Organization And App Shell

- Classification: manual dashboard step
- Wizard stage: `Create the new organization and app shell`
- Open: https://console.deno.com
- Create or select the target organization.
- Create a new app for the Classic project.
- Record:
- `DENO_ORG_NAME=<new organization name>`
- `DENO_APP_NAME=<new app name>`
- `DENO_APP_ID=<new app/project ID if visible>`
- Evidence to capture: console URL for the new app and screenshot or copied app ID/name.

### 4. Inspect App Source Before Editing

- Classification: blocked in this repository until actual app source is supplied
- Wizard stages covered: `Audit HTTP server API`, `Audit queues and KV`, `Deploy the app`
- Run these from the real app repository root, not from this wizard-only repository:

```bash
rg -n 'std(@[0-9.]+)?/http/server\.ts|from ["'"'"']@std/http/server|serve\(' . --glob '!*.lock' --glob '!.git'
rg -n 'Deno\.Kv\.(enqueue|listenQueue)|\.enqueue\(|listenQueue\(' . --glob '!*.lock' --glob '!.git'
rg -n 'Deno\.openKv|Deno\.Kv|openKv\(' . --glob '!*.lock' --glob '!.git'
rg -n 'deployctl|deno deploy|Deno\.cron|cron\(' . --glob '!*.lock' --glob '!.git'
```

- Also inspect `deno.json`, `deno.jsonc`, `.github/workflows/*`, README deployment docs, custom domain references, and environment variable reads such as `Deno.env.get(...)`.
- If legacy `serve()` is imported from `std/http/server.ts` or `@std/http/server`, edit it to use `Deno.serve(handler)` and rerun type/tests.
- If `deployctl` appears in CI or docs, replace with the new Deploy GitHub integration or `deno deploy`.
- If `Deno.cron()` appears, no code migration is usually required, but verify schedules and logs after deploying on new Deploy.
- If `Deno.Kv.enqueue`, `listenQueue`, `.enqueue(`, or `listenQueue(` appear, block cutover until the app has an external queue or database-backed jobs replacement. New Deploy does not support Deno queues.
- Evidence to capture: command output with zero or fixed findings, git diff for any safe edits, and test/typecheck output.

### 5. Migrate Environment Variables

- Classification: manual dashboard step with agent checklist support
- Wizard stage: `Migrate environment variables`
- Open Classic env vars: https://dash.deno.com/projects
- Open new app env vars: https://console.deno.com
- Copy every Classic variable into the correct new Deploy context: production, development, and build.
- Do not paste secrets into the runbook. Store only key names and context mapping.
- Evidence table to fill:

| Key | Secret? | Classic context | New Deploy context | Verified? |
| --- | --- | --- | --- | --- |
| `<KEY>` | yes/no | `<classic>` | production/development/build | yes/no |

- Verification: after deploy, confirm the app behavior that depends on each variable. If the app has a health endpoint, use it as the first check.

### 6. KV Migration Plan

- Classification: app-specific, blocked until actual KV usage and source/target database IDs are known
- Wizard stage: `Audit queues and KV`
- New Deploy reference: https://docs.deno.com/deploy/reference/deno_kv/
- Platform fact: KV data is not automatically migrated from Deploy Classic.
- App source inspection required: identify exact key prefixes and value shapes before export. Do not perform a generic whole-database import unless the app owner accepts that scope.
- Source connection approach: use the Classic KV connect URL and source `DENO_KV_ACCESS_TOKEN` for a one-time export script.
- Target connection approach: create/assign a new Deploy KV database, then use `https://api.deno.com/v2/databases/<Database ID>/connect` with a new Deploy `DENO_KV_ACCESS_TOKEN`.
- Export/import behavior: preview first, export JSONL, dry-run target import, then import with skip-existing writes by default.
- TTL warning: simple `kv.list()` export/import does not preserve key expiration metadata; reapply TTLs in app-specific migration code if the app relies on expiration.
- Evidence to capture:
- Source prefixes selected: `<prefix list>`
- Preview counts by prefix: `<count>`
- Export file path and checksum: `<path>`, `<checksum>`
- Dry-run result: processed/imported/skipped counts
- Real import result: processed/imported/skipped counts
- Verification samples: at least 3 representative keys read from target KV or app-level workflows proving the data is present
- Escape hatch: use `support@deno.com` only if source remote access fails, dataset size is too large for self-serve export/import, keyspace is unknown, or verification confidence is low.

### 7. Deploy The App

- Classification: manual dashboard step or agent command, depending on chosen deployment path
- Wizard stage: `Deploy the app`
- Preferred path: connect the GitHub repository in the new Deploy dashboard and configure branch/build settings.
- CLI path: from the real app repository root, run `deno deploy` after Deno is installed/upgraded.
- Avoid: `deployctl`, because it is being sunset for this migration path.
- Record:
- `DENO_APP_URL=<new app URL>`
- `DEPLOY_METHOD=github-integration|deno-deploy-cli`
- `DEPLOYMENT_ID=<if visible>`
- Evidence to capture: successful deployment URL, deployment logs, commit SHA, and build/runtime settings.

### 8. Migrate Custom Domains

- Classification: manual dashboard/DNS step
- Wizard stage: `Migrate custom domains`
- Custom domain tutorial: https://docs.deno.com/examples/migrate_custom_domain_tutorial/
- In the new app, add each custom domain.
- Create the `_acme-challenge` CNAME shown by the dashboard for TLS provisioning.
- Update DNS CNAME/ANAME records to the new Deploy target only after the new app is healthy on its default URL.
- Allow up to 48 hours for DNS propagation before removing the domain from Deploy Classic.
- Evidence to capture:
- Domain: `<domain>`
- TLS challenge record: `<record>`
- App target record: `<record>`
- DNS lookup before/after: `dig` or `nslookup` output
- HTTPS verification: `curl -I https://<domain>` result

### 9. Smoke Test The New App

- Classification: agent-operable after app URL is known
- Wizard stage: `Smoke test the new app`
- Test the homepage and one important authenticated or write-path workflow.
- Check logs, traces, metrics, cron executions, and errors in the new Deploy dashboard.
- Suggested commands:

```bash
curl -I "$DENO_APP_URL"
curl -fsS "$DENO_APP_URL/health"
```

- Replace `/health` with the app's actual health route if different.
- Evidence to capture: HTTP status, response headers, representative screenshots, log screenshots, and any workflow-specific output.

### 10. Decommission Deploy Classic

- Classification: manual dashboard step, do not automate without explicit approval
- Wizard stage: `Decommission Deploy Classic`
- Preconditions:
- New app has passed smoke tests.
- Env vars are recreated and verified.
- KV export/import/verification is complete if applicable.
- Queue replacement is complete if applicable.
- Custom domain DNS has propagated and HTTPS works on new Deploy.
- Cron schedules have run or have been manually verified.
- Only then remove custom domains from Classic and delete or disable the Classic project.
- Evidence to capture: final approval, final DNS check, final new-app smoke test, and Classic decommission screenshot.

## Completed

- `Identify the project`: runbook specifies exact Classic fields to capture; actual dashboard values are pending.
- `Read the migration guide`: official guide linked and used as platform reference.
- `Create the new organization and app shell`: manual console steps and evidence fields specified.
- `Audit HTTP server API`: source search pattern captured; this repository only matched the wizard/skill text, not app code.
- `Audit queues and KV`: source search patterns captured; this repository only matched the wizard/skill/eval text, not app code.
- `Migrate environment variables`: context-aware table provided; actual variables pending dashboard access.
- `Deploy the app`: GitHub integration and `deno deploy` paths specified; `deployctl` marked as sunset.
- `Migrate custom domains`: TLS challenge and DNS evidence requirements specified.
- `Smoke test the new app`: curl/log/screenshot evidence requirements specified.
- `Decommission Deploy Classic`: gated as final manual step only after verification.

## Code Changes

- None. The task explicitly prohibited repository source edits except writing this output artifact.
- No app source was present in the supplied repository, so no `Deno.serve`, queue, KV, deployment workflow, or docs migration edits could be safely applied.

## Manual Dashboard Steps

- Open Classic projects: https://dash.deno.com/projects
- Open migration guide: https://docs.deno.com/deploy/migration_guide/
- Open new Deploy console: https://console.deno.com
- Open KV reference if KV is used: https://docs.deno.com/deploy/reference/deno_kv/
- Open custom domain tutorial if a custom domain is used: https://docs.deno.com/examples/migrate_custom_domain_tutorial/
- Create/select organization and new app.
- Recreate environment variables in production, development, and build contexts.
- Provision and assign target KV database if applicable.
- Connect GitHub integration or prepare `deno deploy` CLI deployment.
- Add custom domains and create TLS/DNS records.
- Review logs, traces, metrics, cron runs, and final health before decommissioning Classic.

## KV Plan

- Status: blocked until the real app source and Classic/New Deploy KV dashboard values are available.
- Source: Classic KV connect URL plus source `DENO_KV_ACCESS_TOKEN`.
- Target: new Deploy database URL in the form `https://api.deno.com/v2/databases/<Database ID>/connect` plus new Deploy `DENO_KV_ACCESS_TOKEN`.
- Prefix selection: inspect actual `Deno.openKv()` callers and all `kv.get`, `kv.set`, `kv.list`, `kv.atomic`, and key construction sites before export.
- Run order: preview source prefix, export JSONL, dry-run import, real import, verify representative keys through app behavior and direct reads.
- Safety: never delete Classic KV data during migration; default imports should skip existing target keys.

## Risks

- Missing app source: this runbook cannot certify code readiness until the actual application repository is scanned.
- Environment drift: Classic env vars must be manually copied into the correct new Deploy contexts.
- KV data loss or partial copy: mitigated by prefix-specific export, dry-run import, skip-existing writes, counts, checksums, and representative key verification.
- Queue incompatibility: any `Deno.Kv.enqueue` or `listenQueue` usage blocks cutover until replaced.
- Domain downtime: mitigated by provisioning TLS challenge records before traffic cutover and allowing DNS propagation before Classic removal.
- Cron behavior drift: `Deno.cron()` remains supported, but schedules and logs must be verified on the new app.

## Verification

- `deno --version`: passed. Observed `deno 2.9.3 (stable, release, x86_64-pc-windows-msvc)`.
- `Glob **/*`: passed. Observed repository files are the wizard, README files, skill files, eval metadata, and this output workspace; no app source tree was present.
- `rg -n 'std(@[0-9.]+)?/http/server\.ts|from ["'"'"']@std/http/server|serve\(' . --glob '!*.lock' --glob '!.git'`: ran via search tooling. Matches were only in `deno-deploy-classic-migration-wizard.sh`, `SKILL.md`, and eval metadata, not app source.
- `rg -n 'Deno\.Kv\.(enqueue|listenQueue)|\.enqueue\(|listenQueue\(' . --glob '!*.lock' --glob '!.git'`: ran via search tooling. Matches were only in skill/eval metadata, not app source.
- `rg -n 'Deno\.openKv|Deno\.Kv|openKv\(' . --glob '!*.lock' --glob '!.git'`: ran via search tooling. Matches were only in the wizard, skill, and eval metadata, not app source.
- `rg -n 'deployctl|deno deploy|Deno\.cron|cron\(' . --glob '!*.lock' --glob '!.git'`: ran via search tooling. Matches were only in the wizard, skill, and eval metadata, not app source.
- Shell verification note: a combined PowerShell command containing nested quoted regex failed due to quote parsing. The searches above were completed with the repository search tool instead.
