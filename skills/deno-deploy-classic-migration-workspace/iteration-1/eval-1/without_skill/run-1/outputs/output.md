# Deploy Classic Migration Report

No repository source files were changed. The checked-out workspace does not contain the described Deno app entrypoint or `.github/workflows` files; it only contains the migration wizard/evaluation files. Because the task explicitly forbids modifying repository source files except under this output directory, I saved the migration report here instead of editing the wizard files.

## Safe Edits To Apply When App Files Are Available

### 1. Replace legacy `std/http/server.ts` `serve` import

Deploy Classic apps often have code like:

```ts
import { serve } from "https://deno.land/std/http/server.ts";

serve((req) => {
  return new Response("ok");
});
```

Migrate this to the runtime-native API:

```ts
Deno.serve((req) => {
  return new Response("ok");
});
```

If the app currently passes an options object, preserve the same port/hostname where supported:

```ts
Deno.serve({ port: 8000 }, handler);
```

This is a safe code edit only after inspecting the real entrypoint, because some older `serve` call sites use signatures or listeners that need small manual translation.

### 2. Replace `deployctl` GitHub Action usage

The repo should stop relying on `deployctl`, which is being sunset for new Deno Deploy.

Preferred options:

- Use the new Deno Deploy dashboard GitHub integration for automatic deployments.
- Or update CI to install/use the Deno CLI and deploy with `deno deploy` once the target project is configured.

Blocked until the actual workflow file and deployment target details are available:

- Organization/project name on new Deno Deploy.
- Whether deploys should be dashboard-managed or CLI-managed.
- Required auth secret name and environment scoping.

### 3. Replace `Deno.Kv.enqueue` background jobs

`Deno.Kv.enqueue` / queue listeners are not supported on the new Deno Deploy platform. Do not silently translate this to another primitive without a product/infrastructure decision.

Blocked choices:

- Use an external queue service.
- Move background work to a scheduled/cron-driven polling model if latency requirements allow.
- Re-architect the work to happen inline, on webhook retry, or in a separate worker process outside new Deno Deploy.

Any safe migration should first inventory the queue payloads, retry behavior, idempotency keys, dead-letter expectations, and latency requirements.

## Blocked

- The described Deno app files are not present in this checkout.
- No `.github/workflows` directory exists in this checkout.
- No real `Deno.Kv.enqueue` call site exists in this checkout.
- New Deno Deploy project/deployment configuration is not available.
- Queue replacement requires a product/infrastructure decision and cannot be safely invented.

## Verification Performed

- Searched for `serve from`, `deployctl`, `Deno.Kv.enqueue`, and `.enqueue(`.
- Searched for `.github/**` workflows.
- Searched for TypeScript/JavaScript/YAML/Markdown files in the workspace.

Only migration wizard/evaluation files were found, not the app to migrate.
