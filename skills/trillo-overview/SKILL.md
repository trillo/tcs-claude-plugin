---
description: Start here for any Trillo app work. Explains how to authenticate to Trillo AI, pick or create the app you'll author, and what the MCP tools do. Use at the beginning of a Trillo session or whenever the user mentions a Trillo app, AOS, functions/entities/agents, or deploying.
---

# Authoring Trillo AOS apps from Claude Code

You build **Trillo AOS applications** by talking to the **Trillo AI** MCP
server (`trillo-ai`). Server-side metadata (the app's spec, entities,
functions, agents) lives in Trillo AI; deployed code runs on Trillo AOS.
Your job is to drive generation and write/refine function code locally,
persisting it back through the MCP tools.

## 1. Authenticate (one time per machine)

Auth is **OAuth**, handled by Claude Code's MCP client — you do not manage
tokens. If the `trillo-ai` server isn't authenticated:

1. Run `/mcp`, select **trillo-ai**, choose **authenticate**.
2. A browser opens Trillo AI's login. Sign in with your Trillo AI account
   and (if prompted) pick the workspace/tenant to author in.
3. Claude Code stores the tokens; the connection flips to authenticated.

There is **no token file and no `auth_refresh`** to call — ignore any
older bundle-based instructions. If a tool call ever returns
"unauthorized", re-run `/mcp` → authenticate.

## 2. Pick or create the app

Every `md_*` / `deploy` / `activity` tool needs a numeric **`appId`** to
scope its work. (In Trillo, an "app" is identified by this appId.)

1. **Resuming?** If `.trillo/session.json` exists in the working
   directory, read it — it has your `appId`, `name`, `env`, `tenantId`.
   You're set; skip to authoring. (Re-run `app_select` with that appId
   for a fresh `deployStatus`.)
2. **Otherwise** call **`app_list`** → `{apps: [{appId, name,
   displayName, description, env, deployStatus, role}]}`, scoped to your
   tenant.
3. **Open an existing app** → `app_select` with the chosen `appId`.
   **Start a new app** → `app_create` with `{name, description?}` (names
   start with a letter; letters/numbers/`_`/`-`; unique in the tenant;
   created in `dev`). If `app_list` is empty, go straight to `app_create`.
4. **Record it:** write the returned context object
   `{appId, name, displayName, description, env, deployStatus, tenantId,
   role}` to **`.trillo/session.json`** in the working directory. That is
   how app context persists across turns — the server keeps no session
   state (one app per working directory; it contains no secrets, safe to
   commit).
5. **Pass `appId`** to every subsequent `md_*` / `deploy` / `activity`
   tool call.

## 3. The tool surface (progressive disclosure)

Tier 0 tools are always available: `app_list`, `app_select`, `app_create`,
`app_status` (what's done / what's next for an app), `step_guide` (the live
prompt + output schema for an authoring step), `aos_capabilities` (the
platform coverage map), `toolkit_stubs` (the `aos_toolkit` API for function
code), `md_list_model_classes`, `discovery_list_groups`,
`discovery_load_group`, `discovery_list_skills`, `task_events`.

To build an app step by step, use the **building-an-app** skill — it drives
Input → SoftwareSpec → EntityModel → Functions → Agents → UIScenario → deploy
using `app_status` + `step_guide`, asking before each step.

Tier 1 tool groups load on demand — call `discovery_list_groups`, then
`discovery_load_group(name)`:
- **md-crud** — read/write the app's artifacts (`md_get`/`md_create`/
  `md_update`/`md_list` over `FunctionM`, `ClassM` (entity model),
  `AgentM`, `SoftwareSpec`, `UIScenario`). Call `md_list_model_classes`
  first to see valid `modelClassName` values and how each is addressed.
- **activities** — exercise the deployed app (test functions against real
  data); requires `deployStatus == "deployed"`.
- **deploy** — push the app's metadata to Trillo AOS; returns a `taskId`,
  poll with `task_events` until done.

## 4. Deploy status gates what works

`deployStatus` (from `app_select`/`app_create`) tells you which groups are
usable now:
- Metadata authoring (entities, functions, agents, UI specs) and **deploy**
  work in **any** state.
- AOS-touching tools (function testing, agent runs, AOS queries) need
  `deployStatus == "deployed"`.

It's a snapshot — after a deploy, re-run `app_select` (or re-read the
deploy result) to refresh it.

## 5. Prod is not authorable here

Claude Code authors apps in **dev** only. Promotion to production is a
Trillo UI workflow, enforced server-side — don't attempt prod mutations.

## Typical flow

`app_list` → `app_select`/`app_create` → write `.trillo/session.json` →
author the spec/entities/functions (`md-crud`) → `deploy` → poll
`task_events` → test (`activities`). Run `discovery_list_skills` to find
deeper guides as they ship.
