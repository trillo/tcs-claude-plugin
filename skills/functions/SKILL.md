---
description: The Functions step — generate serverless function specs AND Python code from the SoftwareSpec/EntityModel, test them, and (post-deploy) seed data. Use after EntityModel, or when the user wants to add, edit, or test a function.
---

# Step: Functions — specs + code

Functions are serverless handlers beyond CRUD. You generate the spec **and**
the Python, ground the code on the real toolkit API, and test it. Depends on
`EntityModel` being COMPLETED.

## Generate

1. `step_guide({step:"Functions"})` → prompt + `expectedOutputSchema` +
   `systemClasses` + a toolkit note.
2. **`toolkit_stubs()`** → the typed `aos_toolkit` API (`data`, `files`,
   `email`, `um`, `audit`, `responses`, `memory`). **Ground all code on these
   signatures — do not guess the API.**
3. Ground on the model: `md_get SoftwareSpec`, `md_list ClassM` (custom +
   system entities).
4. Write each function — spec **and** Python (camelCase names):
   `md_create modelClassName="FunctionM" name="<fn>" content={name, description,
   params, returns, code, runtime, ...}`.

## Test

Requires the app **deployed** (`deployStatus == "deployed"`). Use
`function_test_sync` (load the `activities` group via `discovery_load_group`
if it isn't in your tool list) → run with sample inputs, read the result, fix
the Python (`md_update FunctionM`), re-test until green.

## Seed data (optional, post-deploy)

If the app needs seed/reference data, populate it with the AOS data tools
**after deploy** (data tools are deploy-gated).

## Add one / confirm

Single function: `step_guide({step:"Functions.add"})` (`AOSAddFunction`).
`app_status` → `Functions: COMPLETED`, `Agents: READY`. Summarize, ask to
proceed.
