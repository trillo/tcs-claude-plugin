---
description: The deploy step — push the app's metadata to Trillo AOS and watch it complete. Use when the user wants to deploy or redeploy, or after the model/functions are ready. Deploy is required before testing functions or seeding data.
---

# Step: deploy

Pushes the app's metadata (entities, functions, agents, spec) to Trillo AOS,
which provisions the schema + APIs and makes the app runnable.

## Deploy

1. Sanity-check with `app_status` — the app needs at least an entity model.
2. `deploy_app` (load the `deploy` group via `discovery_load_group` if it's
   not in your tool list) → returns a `taskId`.
3. Poll `task_events` until it ends; report success or the failure reason.

## After deploy

- `app_status` / `app_select` → `deployStatus: deployed`.
- AOS-touching tools now work: test functions (`function_test_sync`), seed
  data, run agents.
- The user can view the running app in the Trillo **adaptive UI** (outside the
  scope of code generation).

## Redeploy

After any metadata change (entities, functions, agents), redeploy to push it.
**Prod is locked — dev only**; promotion to production is a Trillo UI workflow.
