## Migration Status
- Overall: blocked
- Official guide: https://docs.deno.com/deploy/migration_guide/
- Wizard: `deno-deploy-classic-migration-wizard.sh`

## Completed
- Read the migration guide stage: used the skill's official guide and local wizard checklist as the migration source of truth.
- Audit HTTP server API: searched for `std/http/server.ts`, `@std/http/server`, and `serve(`. No app source file containing the described legacy import exists in this repository snapshot.
- Audit queues and KV: searched for `Deno.Kv.enqueue`, `listenQueue`, `Deno.openKv`, and related KV usage. No app source file containing the described queue call exists in this repository snapshot.
- Deploy the app: searched for `deployctl`, `deno deploy`, and GitHub workflow files. No `.github/workflows/*` files or deployctl workflow exists in this repository snapshot.

## Code Changes
- None. The task explicitly forbids modifying repository source files except under the requested outputs directory, and the Deno app files to migrate are not present.
- Safe edit that should be made when the app source is available: replace the legacy import and server start from:

```ts
import { serve } from "https://deno.land/std/http/server.ts";

serve(handler);
```

with:

```ts
Deno.serve(handler);
```

- Safe deploy edit that should be made when the GitHub Action is available: replace the deployctl workflow with either the new Deno Deploy GitHub integration configured in `https://console.deno.com`, or a Deno CLI based deployment using `deno deploy` if this project intentionally deploys from CI.

## Manual Dashboard Steps
- Create the new organization and app shell in `https://console.deno.com`.
- Recreate environment variables in the new app for the correct production, development, and build timelines.
- Provision and assign any required Deno KV database in the new Deploy console.
- Connect the GitHub repository through the new Deploy dashboard, or configure a new CLI deployment path that does not use `deployctl`.
- Smoke test the new app URL, logs, traces, metrics, and one important user workflow before changing DNS or retiring Classic.

## KV Plan
- KV data is not automatically migrated from Deploy Classic to new Deploy.
- No app KV key prefixes could be identified because the described app source is absent.
- If KV data exists, inspect the real app for key shapes and prefixes before export.
- For one-time local migration scripts, connect to the target database with `await Deno.openKv("https://api.deno.com/v2/databases/<Database ID>/connect")` using `DENO_KV_ACCESS_TOKEN`.
- Use preview/export, dry-run import, skip-existing writes by default, and representative-key verification before cutover.
- If the app relies on key expiration, simple `kv.list()` export/import will not preserve expiration metadata; reapply TTLs in app-specific migration logic.

## Risks
- Queue blocker: `Deno.Kv.enqueue` and `listenQueue` are unsupported on the new Deploy platform. The replacement needs a product/infrastructure choice, such as an external queue or a database-backed jobs table, and cannot be safely invented without the real job semantics.
- Deployment blocker: `deployctl` is being sunset. The real workflow must move to the new Deploy GitHub integration or `deno deploy`.
- Source blocker: the repository snapshot contains the migration wizard/skill and eval metadata, not the described Deno application files. Applying the HTTP and workflow edits would require the actual app source and workflow file.

## Verification
- `rg -n 'std(@[0-9.]+)?/http/server\.ts|from ["'"'"' ]?@std/http/server|serve\('`: found only wizard/skill/eval metadata references, not app source.
- `rg -n 'Deno\.Kv\.(enqueue|listenQueue)|\.enqueue\(|listenQueue\('`: found only eval metadata references, not app source.
- `rg -n 'Deno\.openKv|Deno\.Kv|openKv\('`: found only wizard/skill/eval references, not app source.
- `rg -n 'deployctl|deno deploy|Deno\.cron|cron\('`: found only wizard/skill/eval references, not a GitHub Action.
- `Glob .github/workflows/*`: no files found.
- `Glob **/deno.json*`: no files found.
- `Glob **/*.ts`: no files found.
