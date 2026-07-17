# Deno Deploy Classic Migration Wizard

Interactive bash wizard for migrating a Deploy Classic project to the new Deno Deploy platform.

## Methodology

This tool was generated using Matt Pocock's `wizard` skill from [`mattpocock/skills`](https://github.com/mattpocock/skills/tree/272f99b22574f50e4266791c86b9302682970e23/skills/in-progress/wizard). The skill provides the wizard template, stage structure, and UX contract; the per-project migration stages were authored against Deno's [migration guide](https://docs.deno.com/deploy/migration_guide/).

LLM used: `GPT-5.5`.

## Run

```bash
chmod +x deno-deploy-classic-migration-wizard.sh
./deno-deploy-classic-migration-wizard.sh
```

It records captured values in `.env.deno-deploy-migration`.

## KV migration

When a project uses Deno KV, the wizard now offers a self-serve path before
falling back to support. It references the official
[Deno KV on Deploy](https://docs.deno.com/deploy/reference/deno_kv/) docs,
captures the source and target KV connect URLs, and prints traceable one-time
export/import scripts that can be saved as files or piped into `deno eval`.

The generated workflow defaults to preview/dry-run steps and skips existing
target keys unless an explicit overwrite flag is used. If an app relies on KV
key expiration, adapt the one-time script to reapply TTLs during import.
