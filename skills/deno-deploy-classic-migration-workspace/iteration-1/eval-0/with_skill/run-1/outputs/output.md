## Migration Status
- Overall: partially ready
- Official guide: https://docs.deno.com/deploy/migration_guide/
- KV reference: https://docs.deno.com/deploy/reference/deno_kv/
- Wizard: `deno-deploy-classic-migration-wizard.sh`

## Completed
- Repository inspection: searched for legacy HTTP server usage, queues, KV, cron, `deployctl`, env vars, deployment config, and domain/DNS references.
- HTTP server audit: no application `std/http/server.ts`, `@std/http/server`, or legacy `serve()` usage was present in this checkout. Matches were only in the migration wizard/skill docs.
- Queue audit: no `Deno.Kv.enqueue`, `listenQueue`, or `.enqueue()` app usage was present in this checkout. Matches were only eval prompts/documentation.
- Deploy tooling audit: no GitHub Action or deployment config using `deployctl` was present. The wizard correctly recommends new Deploy GitHub integration or `deno deploy`.
- KV/custom-domain evidence: this checkout contains the migration wizard and eval metadata, not the actual app source. The prompt says the deployed app uses `Deno.openKv` and a custom domain, but no app files are present here to infer concrete KV key prefixes or domain names.

## Code Changes
- None. The task prohibited source changes except writing this output file.
- No safe app code edits could be made because the repository does not include the app source tree that calls `Deno.openKv`.

## What Needs To Change For New Deploy
- Create the new organization/app in `https://console.deno.com` and deploy via the new GitHub integration or `deno deploy`; do not build new automation around `deployctl`.
- Provision a new Deno KV database in the new Deploy console and assign it to the app/timeline before relying on `Deno.openKv()` in production.
- In deployed app code, prefer `await Deno.openKv()` with no hardcoded Classic database URL. One-time migration scripts should use `await Deno.openKv("https://api.deno.com/v2/databases/<Database ID>/connect")` plus `DENO_KV_ACCESS_TOKEN`.
- Recreate environment variables in the new Deploy dashboard across the correct contexts: production, development, and build.
- Add the custom domain to the new app, create the dashboard-provided `_acme-challenge` CNAME for TLS, then update the domain's CNAME/ANAME target to the new Deploy target.
- Keep the Classic project and KV source intact until new app smoke tests, KV count checks, representative key checks, and custom-domain traffic verification all pass.

## Manual Dashboard Steps
- `https://console.deno.com`: create or select the new organization and app.
- `https://console.deno.com`: connect GitHub or prepare CLI deploy with `deno deploy`.
- `https://console.deno.com`: create and assign the target KV database to the app/timeline.
- `https://dash.deno.com/projects`: inspect the Classic project settings for current env vars, custom domain, and source KV/database details.
- `https://console.deno.com`: recreate env vars in production/development/build contexts.
- `https://console.deno.com`: add the custom domain to the new app.
- DNS provider: add the `_acme-challenge` CNAME shown by new Deploy for TLS provisioning.
- DNS provider: update the domain CNAME/ANAME to the new Deploy target only after the new app is deployed and smoke-tested.

## KV Plan
- Source connection: use the Classic KV connect URL/token if available for the source database. Do not delete or detach the source database.
- Target connection: use `https://api.deno.com/v2/databases/<Database ID>/connect` from the new Deploy KV database with `DENO_KV_ACCESS_TOKEN`.
- Prefix selection: inspect the real app source for all `kv.get`, `kv.set`, `kv.list`, `kv.delete`, and `Deno.openKv` call sites, then migrate only known prefixes. This checkout does not contain those call sites, so prefixes are currently unknown.
- Export strategy: run a preview first with a small limit or selected prefix, then export each confirmed prefix to JSONL.
- Import strategy: run a dry-run against target KV first. The wizard's import script skips existing keys by default using `atomic().check({ versionstamp: null })`; only use overwrite with an explicit destructive flag.
- Verification: compare exported/imported counts per prefix, sample representative keys in target KV, run the app against the target database, and verify one user-visible workflow that reads/writes KV.
- TTL caveat: simple `kv.list()` export/import does not preserve key expiration metadata. If the app relies on `expireIn`, reapply TTLs in an app-specific migration script or regenerate ephemeral keys instead of copying them blindly.
- Support fallback: contact `support@deno.com` only if the Classic source database cannot be remotely accessed, the dataset is too large for a self-serve JSONL migration, the keyspace cannot be identified confidently, or verification cannot establish parity.

Example command sequence after saving the wizard's generated scripts as `kv-export-once.ts` and `kv-import-once.ts`:

```sh
DENO_KV_ACCESS_TOKEN="<source-token>" \
KV_URL="<source-kv-connect-url>" \
KV_PREFIX='["<confirmed-prefix>"]' \
deno run --unstable-kv --allow-env --allow-net kv-export-once.ts --preview

DENO_KV_ACCESS_TOKEN="<source-token>" \
KV_URL="<source-kv-connect-url>" \
KV_PREFIX='["<confirmed-prefix>"]' \
KV_OUT="kv-export.jsonl" \
deno run --unstable-kv --allow-env --allow-net --allow-write kv-export-once.ts

DENO_KV_ACCESS_TOKEN="<new-deploy-token>" \
KV_URL="https://api.deno.com/v2/databases/<Database ID>/connect" \
KV_IN="kv-export.jsonl" \
deno run --unstable-kv --allow-env --allow-net --allow-read kv-import-once.ts --dry-run

DENO_KV_ACCESS_TOKEN="<new-deploy-token>" \
KV_URL="https://api.deno.com/v2/databases/<Database ID>/connect" \
KV_IN="kv-export.jsonl" \
deno run --unstable-kv --allow-env --allow-net --allow-read kv-import-once.ts
```

## Risks
- Actual app source is absent from this repository, so KV key prefixes, domain name, env vars, and any hardcoded Classic assumptions could not be verified.
- If the app uses queues in code not present here, `Deno.Kv.enqueue`/`listenQueue` must be replaced before cutover because queues are not supported on new Deploy.
- If the app imports legacy `serve` in code not present here, replace that with `Deno.serve()` before deploying.
- DNS propagation can take up to 48 hours; keep the Classic custom domain/project available until traffic is confirmed on the new app.

## Verification
- Source search for HTTP API: completed; only wizard/skill documentation matches found.
- Source search for queue API: completed; only eval/documentation matches found.
- Source search for KV API: completed; only wizard/skill/eval metadata matches found.
- Source search for deploy tooling/cron: completed; only wizard/skill/eval metadata matches found.
- Source search for env/domain/DNS references: completed; only wizard/skill/eval metadata matches found.
- No tests or deploy commands were run because the task requested an inspection/report and this repository does not contain an app implementation to execute.
