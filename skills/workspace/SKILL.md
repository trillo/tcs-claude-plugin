---
description: Mirror a Trillo app's artifacts to a local directory so you can read/grep/edit them as files, and keep them in sync with Trillo AI. Use at the start of work on an app (pull), when the user says "check code with Trillo AI" / "resync", and after you edit a local file (push). Trillo AI (the DB) is the source of truth.
---

# Local workspace (pull / edit / sync)

Work on an app's artifacts as **files** under `.trillo/<appId>/`, then sync with
Trillo AI. **The DB is authoritative**; the local copy is a working mirror.

## Directory layout

```
.trillo/
  session.json                      # appId, name, env, tenantId (already there)
  <appId>/
    requirement.html                # Input
    software-spec.json              # SoftwareSpec
    ui-scenario.json                # UIScenario (if present)
    entities/<ClassName>.json       # custom entities (ClassM)
    entities/_system/<ClassName>.json   # built-in entities — READ ONLY, reference only
    functions/<fnName>.py           # the function's Python (content.code)
    functions/specs/<fnName>.json   # the function spec (content without code)
    functions/tests/test_<fnName>.py  # local MockCtx tests (see the functions skill)
    functions/conftest.py             # `ctx` fixture for local tests (see the functions skill)
    agents/<agentName>.json         # AgentM
    .sync.json                      # sync baseline — git-ignored, do not edit by hand
    .backups/<name>.<ts>            # pre-overwrite backups — git-ignored
```

Add `.trillo/<appId>/.sync.json` and `.trillo/<appId>/.backups/` to `.gitignore`.

## Artifact ↔ file mapping

| Class | MCP | File(s) |
|---|---|---|
| Input | `md_get Input` | `requirement.html` (the HTML) |
| SoftwareSpec | `md_get SoftwareSpec` | `software-spec.json` (the content) |
| UIScenario | `md_get UIScenario` | `ui-scenario.json` |
| ClassM | `md_list`/`md_get ClassM` | `entities/<name>.json`; `systemClasses` → `entities/_system/<name>.json` (read-only) |
| FunctionM | `md_list`/`md_get FunctionM` | `functions/<name>.py` (= `content.code`) + `functions/specs/<name>.json` (= content minus `code`) |
| AgentM | `md_list`/`md_get AgentM` | `agents/<name>.json` |

`.sync.json` records a baseline per artifact so we can tell what changed:
```json
{ "appId": <id>, "pulledAt": "<iso>",
  "artifacts": { "<Class>" or "<Class>/<name>": { "serverUpdatedAt": <ms>, "localHash": "<sha256 of the file content>" } } }
```
Compute `localHash` with `shasum -a 256` (Bash). For a function, hash the
`.py` + `specs/*.json` pair (concatenated) so an edit to either counts.

## Initial pull (start of work on an app)

When you select/resume an app and `.trillo/<appId>/.sync.json` doesn't exist (or
the user asks to pull):
1. For each singleton (`Input`, `SoftwareSpec`, `UIScenario`) → `md_get` → write
   its file (skip if the artifact doesn't exist yet).
2. For `ClassM`/`FunctionM`/`AgentM` → `md_list` → `md_get` each → write file(s).
   Write the `systemClasses` block to `entities/_system/` (read-only reference).
3. Record each artifact's `serverUpdatedAt` (the `updatedAt` from the response)
   and `localHash` in `.sync.json`. Stamp `pulledAt`.

## Resync ("check code with Trillo AI")

Pull only what's **newer in the DB** — auto-apply, but never lose local work:
1. `md_list` each class (cheap — carries `updatedAt`). An artifact is
   **server-newer** if its `updatedAt` > the baseline `serverUpdatedAt`, or it
   has no baseline (new on server).
2. For each server-newer artifact, before overwriting:
   - compute the current `localHash`; if it **differs from the baseline**
     (you have **unpushed local edits**), first **copy the local file(s) to
     `.backups/<name>.<ts>`** and tell the user "kept your local <name> at
     `.backups/...`".
   - then `md_get` and overwrite the local file(s); update the baseline.
3. Report what was pulled and any backups made. (Resync is **pull-only** — it
   never deletes local files or pushes.)

## Push (after you edit a local file)

When you change a local artifact (the normal authoring flow), save it back:
- Read the file(s); for a function, **recombine** `functions/<name>.py` into
  `content.code` alongside `functions/specs/<name>.json`.
- `md_update` (or `md_create` for a new artifact). This bumps the server
  `updatedAt`.
- Refresh that artifact's baseline (`serverUpdatedAt` from the response +
  recomputed `localHash`) so a later resync sees it as in-sync.

Never edit `entities/_system/*` — those are built-ins; reference only.

## Deleting an artifact

Deletes are **explicit and confirmed** — a missing local file is NOT a delete.
- The user deleting one thing → call `md_delete <Class> name` directly, then
  remove the local file + its baseline.
- A bulk "push deletions" → **list first**: show every artifact that has a
  baseline but no local file, and require an explicit **yes** before any
  `md_delete`. Never delete on the server silently from a missing file.
