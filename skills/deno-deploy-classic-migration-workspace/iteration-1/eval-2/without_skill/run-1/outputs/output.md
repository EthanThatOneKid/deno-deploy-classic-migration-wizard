# Agent Runbook: Deploy Classic to New Deno Deploy

Source checklist: `deno-deploy-classic-migration-wizard.sh`.

Official stage map: Deno Deploy Classic migration guide at `https://docs.deno.com/deploy/migration_guide/`.

Workspace audited: `C:\Users\ethan\Documents\GitHub\deno-deploy-classic-migration-wizard`.

Important scope note: this workspace contains the migration wizard and eval metadata, not a deployable Deno app source tree. The runbook below is therefore adapted to the source actually present here and marks app-specific checks that must be rerun from the real application repository root before cutover.

## Current Repository Evidence

Commands run from the workspace root:

```powershell
rg -n 'std(@[0-9.]+)?/http/server\.ts|@std/http/server|serve\(' . --glob '!*.lock' --glob '!.git'
rg -n "Deno\.Kv\.(enqueue|listenQueue)|\.enqueue\(|listenQueue\(" . --glob '!*.lock' --glob '!.git'
rg -n "Deno\.openKv|Deno\.Kv|openKv\(" . --glob '!*.lock' --glob '!.git'
rg -n "deployctl|deno deploy|Deno\.cron|cron\(" . --glob '!*.lock' --glob '!.git'
bash -n "deno-deploy-classic-migration-wizard.sh"
```

Observed evidence:

| Area | Evidence | Migration impact |
| --- | --- | --- |
| Wizard syntax | `bash -n deno-deploy-classic-migration-wizard.sh` returned no output | Wizard shell syntax passed. |
| HTTP server API | Matches are in wizard guidance, skill/eval text, and another eval output. No deployable app handler was found in this workspace. | No source edit is available in this repo. Rerun scan in the real app root and replace legacy imported `serve()` with `Deno.serve()`. |
| Queues | Matches are prompt/eval text only. | No app queue code found here. Rerun scan in the real app root; `Deno.Kv.enqueue` and `listenQueue` must be replaced before cutover. |
| KV | Matches are wizard-generated one-time export/import script text and docs. | This repo does not expose app KV prefixes. The real app needs prefix discovery and export/import verification. |
| Deploy method | Matches are wizard/docs guidance around `deno deploy` and `deployctl` sunset. | No GitHub Action or deployment config for an app was found here. In the real app, remove `deployctl` workflows or replace with new Deploy GitHub integration / `deno deploy`. |
| Cron | Only wizard guidance states `Deno.cron()` works on new Deploy. | Rerun scan in the real app root and verify schedules/logs after first deployment. |

## Agent-Operable Runbook

Use this as a stateful checklist. Record evidence in a migration log for the app, with timestamps, command output snippets, dashboard screenshots/URLs, and IDs copied from the dashboards.

### 1. Identify the Classic Project

Wizard stage: `Identify the project`.

Dashboard-only work:

1. Open `https://dash.deno.com/projects`.
2. Select the Deploy Classic project being migrated.
3. Record `DENO_CLASSIC_PROJECT_NAME` and `DENO_CLASSIC_PROJECT_URL`.

Agent-editable work:

1. From the real app repo root, create a local migration notes file only if the user allows repo edits; otherwise keep notes outside source control.
2. Do not change production traffic yet.

Verification evidence to capture:

```text
DENO_CLASSIC_PROJECT_NAME=<classic-project-name>
DENO_CLASSIC_PROJECT_URL=<classic-project-url>
Classic dashboard opened: yes/no
```

### 2. Create New Organization and App Shell

Wizard stage: `Create the new organization and app shell`.

Dashboard-only work:

1. Open `https://console.deno.com`.
2. Create or select the target organization.
3. Create a new app for the Classic project.
4. Record `DENO_ORG_NAME`, `DENO_APP_NAME`, and visible app/project ID.

Agent-editable work:

1. None until deployment settings are known.

Verification evidence to capture:

```text
DENO_ORG_NAME=<new-org>
DENO_APP_NAME=<new-app>
DENO_APP_ID=<new-app-id-if-visible>
New app shell URL=<console-url>
```

### 3. Audit and Update HTTP Server API

Wizard stage: `Audit HTTP server API`.

Why: new Deploy does not support the legacy standard-library `serve()` API; app entrypoints should use `Deno.serve()`.

Agent-editable work in the real app repo:

1. Run:

```powershell
rg -n 'std(@[0-9.]+)?/http/server\.ts|@std/http/server|serve\(' . --glob '!*.lock' --glob '!.git'
```

2. For true legacy app matches, replace this pattern:

```ts
import { serve } from "https://deno.land/std/http/server.ts";

serve(handler);
```

with:

```ts
Deno.serve(handler);
```

3. Preserve explicit ports/options if they exist:

```ts
Deno.serve({ port: 8000 }, handler);
```

Dashboard-only work:

1. None.

Verification evidence to capture:

```powershell
deno check <entrypoint.ts>
deno test --allow-all
rg -n 'std(@[0-9.]+)?/http/server\.ts|@std/http/server|serve\(' . --glob '!*.lock' --glob '!.git'
```

Expected result: no remaining legacy imported `serve()` in app runtime files.

### 4. Audit Queues, Cron, and KV

Wizard stage: `Audit queues and KV`.

Agent-editable work in the real app repo:

1. Run queue scan:

```powershell
rg -n "Deno\.Kv\.(enqueue|listenQueue)|\.enqueue\(|listenQueue\(" . --glob '!*.lock' --glob '!.git'
```

2. If any `Deno.Kv.enqueue` / `listenQueue` usage exists, do not cut over until replaced with an external queue, database-backed job table, or another explicitly chosen job runner.
3. Run KV scan:

```powershell
rg -n "Deno\.openKv|Deno\.Kv|openKv\(" . --glob '!*.lock' --glob '!.git'
```

4. Extract exact KV key prefixes used by the app. Record them as JSON arrays, for example `['users']`, `['sessions']`, or `['tenant', '<id>', 'settings']`.
5. Run cron/deploy scan:

```powershell
rg -n "deployctl|deno deploy|Deno\.cron|cron\(" . --glob '!*.lock' --glob '!.git'
```

6. Keep `Deno.cron()` code if present, but verify schedules and logs after deploying to new Deploy.

Dashboard-only work:

1. If the app uses KV, provision a new KV database in `console.deno.com` and assign it to the app/timeline.
2. Generate or collect source and target KV connect details/tokens.

Verification evidence to capture:

```text
Queue scan result=<none|matches with files>
Cron scan result=<none|matches with files>
KV scan result=<none|matches with files>
Selected KV prefixes=<json-list>
Target KV database ID=<database-id>
```

### 5. Migrate KV Data When Needed

Wizard stage: `Audit queues and KV`; helper: `show_kv_migration_scripts`.

Use the wizard's generated export/import scripts as the starting point. The important behavior is preview first, dry-run import second, then real import only after counts look correct. Existing target keys should be skipped unless an explicit overwrite run is approved.

Agent-editable work:

1. Save the wizard's export code as `kv-export-once.ts` outside normal app runtime, or in a temporary migration folder if approved.
2. Save the wizard's import code as `kv-import-once.ts` outside normal app runtime, or in a temporary migration folder if approved.
3. For every app prefix, preview the source:

```powershell
$env:DENO_KV_ACCESS_TOKEN="<source-token>"
$env:KV_URL="<source-kv-connect-url>"
$env:KV_PREFIX='["<prefix>"]'
deno run --unstable-kv --allow-env --allow-net kv-export-once.ts --preview
```

4. Export after preview is accepted:

```powershell
$env:DENO_KV_ACCESS_TOKEN="<source-token>"
$env:KV_URL="<source-kv-connect-url>"
$env:KV_PREFIX='["<prefix>"]'
$env:KV_OUT="kv-export-<prefix>.jsonl"
deno run --unstable-kv --allow-env --allow-net --allow-write kv-export-once.ts
```

5. Dry-run import into the target database:

```powershell
$env:DENO_KV_ACCESS_TOKEN="<new-deploy-token>"
$env:KV_URL="https://api.deno.com/v2/databases/<Database ID>/connect"
$env:KV_IN="kv-export-<prefix>.jsonl"
deno run --unstable-kv --allow-env --allow-net --allow-read kv-import-once.ts --dry-run
```

6. Import only when dry-run counts are expected:

```powershell
$env:DENO_KV_ACCESS_TOKEN="<new-deploy-token>"
$env:KV_URL="https://api.deno.com/v2/databases/<Database ID>/connect"
$env:KV_IN="kv-export-<prefix>.jsonl"
deno run --unstable-kv --allow-env --allow-net --allow-read kv-import-once.ts
```

Dashboard-only work:

1. Keep the Classic KV source intact until production traffic and representative reads are verified on new Deploy.
2. If remote source access or validation fails, use `support@deno.com` as the fallback escape hatch, not the first plan.

Verification evidence to capture:

```text
Prefix=<prefix>
Preview count=<count>
Export file=<path>
Export count=<count>
Dry-run processed=<count>
Dry-run imported=<count>
Dry-run skipped=<count>
Real import processed=<count>
Real import imported=<count>
Real import skipped=<count>
Representative key checks=<key/result samples>
```

### 6. Migrate Environment Variables

Wizard stage: `Migrate environment variables`.

Dashboard-only work:

1. In `dash.deno.com`, open Classic project settings and copy every environment variable name.
2. In `console.deno.com`, add each variable to the correct new Deploy context: production, development, and/or build.
3. Do not paste secrets into repo files or logs.

Agent-editable work in the real app repo:

1. Search for env usage to make sure every required variable is accounted for:

```powershell
rg -n "Deno\.env\.get|Deno\.env\.has|process\.env|env\(" . --glob '!*.lock' --glob '!.git'
```

2. Update docs or example env files only if requested and safe.

Verification evidence to capture:

```text
Env var inventory=<names only, no values>
Production vars configured=yes/no
Development vars configured=yes/no
Build vars configured=yes/no
Missing vars=<none|names>
```

### 7. Deploy the App

Wizard stage: `Deploy the app`.

Dashboard-only work:

1. Preferred path: connect the GitHub repository through the new Deploy dashboard and configure branch/build settings.
2. Alternative path: use the Deno CLI from the app root with `deno deploy`.

Agent-editable work in the real app repo:

1. Remove or replace stale `deployctl` GitHub Actions if present.
2. Verify the app locally before first deploy:

```powershell
deno check <entrypoint.ts>
deno test --allow-all
```

3. If using CLI deployment:

```powershell
deno deploy
```

Verification evidence to capture:

```text
Deployment method=<GitHub integration|deno deploy>
Deployment ID=<id>
New app URL=<url>
Build logs reviewed=yes/no
Runtime logs reviewed=yes/no
```

### 8. Migrate Custom Domains

Wizard stage: `Migrate custom domains`.

Dashboard-only work:

1. If the app has a custom domain, open `https://docs.deno.com/examples/migrate_custom_domain_tutorial/`.
2. Add the custom domain to the new app in `console.deno.com`.
3. Create the `_acme-challenge` CNAME shown by the dashboard.
4. Update the DNS CNAME or ANAME record to point at the new Deploy target.
5. Allow propagation before removing the domain from Deploy Classic.

Agent-editable work:

1. None unless DNS is managed in infrastructure-as-code; if so, update only the DNS record for the migrated domain and include a diff in the evidence.

Verification evidence to capture:

```powershell
Resolve-DnsName _acme-challenge.<domain> -Type CNAME
Resolve-DnsName <domain>
curl.exe -I https://<domain>
```

Expected result: TLS is provisioned on new Deploy and `curl -I` returns the expected status from the new app.

### 9. Smoke Test New Deploy Before Cutover

Wizard stage: `Smoke test the new app`.

Agent-operable tests:

1. Hit the homepage:

```powershell
curl.exe -I <new-app-url>
```

2. Hit at least one important route or workflow endpoint:

```powershell
curl.exe -i <new-app-url>/<important-route>
```

3. If the app writes KV, perform a safe write/read/delete test against non-production test keys only.
4. If the app uses cron, inspect new Deploy logs after the next scheduled run.
5. If the app uses authentication, test login/logout in a browser and confirm callback URLs are updated for the new app URL/domain.

Dashboard-only work:

1. Review logs, traces, metrics, deployment status, KV assignment, and domain status in `console.deno.com`.

Verification evidence to capture:

```text
Homepage status=<status>
Important route status=<status>
Auth flow checked=<yes/no/not applicable>
KV read/write checked=<yes/no/not applicable>
Cron checked=<yes/no/not applicable>
Errors in logs=<none|summary>
```

### 10. Decommission Deploy Classic

Wizard stage: `Decommission Deploy Classic`.

Dashboard-only work:

1. Confirm all previous evidence is complete.
2. Confirm DNS points to the new app and has propagated.
3. Confirm production traffic works on the new app URL/custom domain.
4. Remove the custom domain from Classic only after the new app serves it correctly.
5. Disable or delete the Classic project.

Agent-editable work:

1. Remove temporary migration scripts and exported KV data from the local machine if they contain sensitive values.
2. Update deployment docs to reference `console.deno.com` and the new deployment path.

Final verification evidence:

```text
Classic project disabled/deleted=<yes/no>
Classic custom domain removed=<yes/no/not applicable>
New app serving production traffic=<yes/no>
Rollback window/plan=<description>
```

## Dashboard-Only Work Summary

The agent cannot complete these without authenticated browser/dashboard access:

| Task | Dashboard |
| --- | --- |
| Identify Classic project details | `dash.deno.com` |
| Create new organization/app | `console.deno.com` |
| Provision and assign target KV database | `console.deno.com` |
| Copy env vars from Classic and create scoped env vars in new Deploy | `dash.deno.com`, `console.deno.com` |
| Configure GitHub integration if used | `console.deno.com` |
| Add custom domain and TLS challenge | `console.deno.com`, DNS provider |
| Review logs, traces, metrics | `console.deno.com` |
| Disable/delete Classic project | `dash.deno.com` |

## Agent-Editable Repository Work Summary

Run these in the actual app source tree, not this wizard-only workspace:

1. Replace legacy `std/http` imported `serve()` with `Deno.serve()`.
2. Replace unsupported `Deno.Kv.enqueue` / `listenQueue` usage before cutover.
3. Keep `Deno.cron()` if used, but verify schedules and logs after deploy.
4. Inventory `Deno.openKv()` usage and derive exact key prefixes for migration.
5. Replace `deployctl` workflows/docs with new Deploy GitHub integration or `deno deploy`.
6. Run `deno check`, `deno test`, deployment, route smoke tests, and KV verification.

## Go/No-Go Gate

Cut over only when all are true:

1. New app exists in `console.deno.com` and has a successful deployment.
2. All required env vars are present in the correct contexts.
3. No legacy `serve()` runtime usage remains.
4. No unsupported queue usage remains, or a replacement is deployed and tested.
5. KV data is either not used or exported, dry-run imported, imported, and sampled successfully.
6. Custom domain DNS and TLS are verified on new Deploy, if applicable.
7. Homepage and critical workflows pass smoke tests on the new URL/domain.
8. New Deploy logs show no blocking runtime errors.
9. Rollback plan is documented before Classic is disabled.
