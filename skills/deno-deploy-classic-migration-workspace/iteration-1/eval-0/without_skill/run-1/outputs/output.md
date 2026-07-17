# Deno Deploy Classic Migration Assessment

## Repo Findings

- This repository does not contain the actual deployed Deno app source. I found no `deno.json`, `deno.jsonc`, `main.ts`, `server.ts`, `mod.ts`, TypeScript app files, deployment YAML, or Deploy app entrypoint.
- The repo is a migration wizard. The relevant implementation is `deno-deploy-classic-migration-wizard.sh`, which already encodes the right migration concerns: `Deno.serve()`, KV export/import, env vars, `deno deploy`, and custom domains.
- Because the target app source is absent, I cannot confirm the exact `Deno.openKv()` call sites, key prefixes, queue usage, old `std/http serve()` usage, or custom-domain hostname from code. Those must be checked in the actual app repository or Deploy Classic dashboard.

## Code Changes To Check In The Actual App

- HTTP server: replace legacy standard-library `serve()` imports from `deno.land/std/.../http/server.ts` or `@std/http/server` with built-in `Deno.serve()`. The official Deploy Classic migration guide says old `serve()` can fail on new Deploy during warmup.
- Deployment command: stop using `deployctl`; new Deploy uses the dashboard GitHub integration or `deno deploy`.
- KV connection in deployed app: keep app code as `await Deno.openKv()` after assigning a Deno KV database to the app in the new Deploy console. Do not hard-code the `https://api.deno.com/v2/databases/<Database ID>/connect` URL in production app code; that URL is for local/admin scripts outside Deploy.
- Queues: if the app uses `Deno.Kv.enqueue()` or `listenQueue()`, replace that path. Deno queues are not supported on the new Deploy platform.
- Cron: `Deno.cron()` should continue to work, but verify schedules and logs after deployment.
- Env vars: recreate Deploy Classic env vars in new Deploy, choosing the right production/development/build contexts.

## Manual Dashboard And DNS Steps

- Create/sign in to a new Deno Deploy organization at `https://console.deno.com`.
- Create a new app for the Classic project; Classic projects are not automatically transferred.
- Deploy through the GitHub integration or `deno deploy`.
- Provision a new Deno KV database under the organization and assign it to the app. New Deploy isolates databases per timeline, so verify which timeline receives migrated production data.
- Add the custom domain under the new app's Production Domains.
- Add the `_acme-challenge` CNAME shown by the dashboard and provision TLS.
- Replace existing domain CNAME/A/AAAA/ANAME records with the records provided by new Deploy.
- Allow up to 48 hours for propagation before removing the domain from Deploy Classic.

References: official Deploy Classic migration guide at `https://docs.deno.com/deploy/migration_guide/`, Deno KV reference at `https://docs.deno.com/deploy/reference/deno_kv/`, and custom-domain tutorial at `https://docs.deno.com/examples/migrate_custom_domain_tutorial/`.

## Concrete KV Migration Plan

1. Inventory data before exporting. In the actual app repo, search for `Deno.openKv`, `kv.get`, `kv.set`, `kv.list`, `kv.delete`, `atomic`, and any prefix constants. Write down every key prefix that belongs to production data.
2. Freeze or reduce writes during final export. For low-write apps, schedule a short maintenance window. For write-heavy apps, either pause writes, run an initial export/import, then run a final delta export during cutover, or add app-specific dual-write logic temporarily.
3. In new Deploy, provision Deno KV and assign it to the target app/timeline.
4. Create a Deno Deploy organization or personal access token for the target KV and set it locally as `DENO_KV_ACCESS_TOKEN` only while running the import.
5. Obtain the source KV connect URL/token for the Classic KV. If this is unavailable through self-service, that is the point where support may be needed to get source access or an assisted export.
6. Export each selected prefix to JSONL from the source KV. First run preview mode with a small limit; then run the full export.
7. Dry-run import into the new KV. The import should count processed/imported/skipped records without writing anything.
8. Run the real import. Default to skip existing target keys. Use overwrite only if the target was intentionally reset and the operator explicitly accepts destructive behavior.
9. Verify representative keys, counts by prefix, login/session/user workflows, and any read-after-write behavior on the new app.
10. Cut DNS over only after the new app, env vars, KV reads/writes, and custom-domain TLS all pass.
11. Keep the Classic project and source KV intact until post-cutover validation passes.

Minimal script behavior to use:

- Export script: open source KV with `await Deno.openKv(KV_URL)`, iterate `kv.list({ prefix })`, serialize records as JSONL, and support `--preview`, `KV_PREFIX`, `KV_LIMIT`, and `KV_OUT`.
- Import script: open target KV with `await Deno.openKv("https://api.deno.com/v2/databases/<Database ID>/connect")`, read JSONL, support `--dry-run`, and import via `atomic().check({ key, versionstamp: null }).set(key, value).commit()` so existing target data is not overwritten by default.
- Preserve special Deno KV value types explicitly if used, including `Uint8Array`, `Deno.KvU64`, `bigint`, `Date`, `Map`, and `Set`.
- Important limitation: simple `kv.list()` export/import copies keys and values, not key expiration metadata. If the app relies on TTLs/expirations, reapply TTLs during import using app-specific rules.

Support is a fallback, not the primary path: contact `support@deno.com` only if source KV access/export cannot be obtained, remote connect is blocked, or verification shows data inconsistency that cannot be explained by app writes during migration.
