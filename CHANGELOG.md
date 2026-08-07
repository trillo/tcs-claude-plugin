# Changelog — Trillo Claude Code plugin

Changes to the plugin's tools and to how Claude Code talks to Trillo. Read the **⚠️ breaking**
items before your next session so nothing surprises you.

---

## 2026-08-08 — New `scenario-to-a2ui` skill + skill-guidance fixes (plugin v0.4.0)

### New

- **`scenario-to-a2ui`** skill — author adaptive-UI screens from a scenario: `UIView`/`UIFlow` via
  `md_*` (A2UISpec + Direct-API actions + `bindings`), `newFunctions[]` for backend gaps, and a
  per-role sidebar. Distinct from the `ui-scenario` step (which only *defines* scenarios).

### Skill-guidance fixes (behavior, read if you author functions/agents)

- **Agents** — the sub-agent tool is **`run_agent`**, not `call_agent` (binding `call_agent` is
  silently ignored). **`responseSchema` is a Gemini `Schema`, not JSON Schema** — express nullable as
  `{"type":"string","nullable":true}`, not a `["string","null"]` union. Added `capabilities`
  ("Permitted APIs") derivation guidance, rewired doc refs to the content API, and `ctx.task.log`
  logging notes.
- **Functions** — the toolkit needs **Python 3.11+** (use `python3.11` on macOS); pointers to the
  user-management and files-and-folders developer guides.
- **Entity model** — assorted doc corrections.

---

## 2026-08-05 — Toolkit installs via pip; new `toolkit_status` (plugin v0.3.0)

### ⚠️ Breaking / behavior changes — read first

1. **`toolkit_install` and `toolkit_stubs` are gone.** The `aos_toolkit` SDK now ships as a
   **versioned wheel on its public GitHub release**, so you install it with `pip` instead of
   fetching it over MCP (the old replies exceeded the MCP response cap). Run the new
   **`toolkit_status`** tool for the exact command — it reports the toolkit version this platform
   build deploys plus a `pip install "aos-toolkit[test] @ <release-wheel-url>"` to run in a venv.
   Ground function/agent code on the **installed** toolkit (its typed stubs come with it).
   - *Needs the Trillo AI server redeployed with `toolkit_status`; reconnect `/mcp` and restart
     Claude Code afterward so the refreshed tool list loads.*

### Other

- **`toolkit_status`** (new) — one call returning the deployed toolkit version, the install command,
  and how to check your local install, so you can spot and fix version drift.
- Task-event tracing URL corrected to `GET /api/v2.0/tasks/{taskId}/events` (for an agent, the task
  id *is* the conversation id).
- `data_seed` / `data_query` "as a role" note clarified: class-level `acl` is **not** enforced for a
  role token on the direct data API, so seeding/querying as a role does not verify that role's
  permissions.

---

## 2026-07-30 — Tool cleanup + new dev-time tools (plugin v0.2.0)

### ⚠️ Breaking / behavior changes — read first

1. **`auth_refresh` is gone.** Token refresh is now handled **automatically** by Claude Code's
   OAuth connection — there is no tool to call. If your session's auth fully expires or is revoked,
   run **`/mcp` → reconnect** (browser sign-in). The old `.trillo/tokens.json` bundle and the
   `/auth/claude-code/refresh` endpoint have been removed.
   - *If you had any habit or script that called `auth_refresh`, drop it.*

2. **`discovery_list_skills` is gone.** It listed stale skill files that pointed at bundle-era paths
   which no longer exist. The plugin's own skills (loaded by Claude Code automatically) are the
   guidance now — nothing for you to do.

3. **`aos_token` role hint corrected.** It used to suggest discovering roles via
   `md_list(modelClassName='AppRole')`, which errored. Your app's roles live in its **SoftwareSpec**
   (`md_get modelClassName='SoftwareSpec'`).

**What to do:** once the update is live, **restart Claude Code** so the tool list refreshes.

### ✨ New tools — the `activities` group (dev-only, available after `deploy_app`)

You can now exercise and administer your **deployed dev app** directly from Claude Code — no more
hand-rolled `curl`:

| Tool | What it does |
|---|---|
| **`aos_call`** | Call **any** deployed-app REST API (as tenant-admin or a chosen role). The general-purpose way to do anything without a dedicated tool. |
| **`secret_set`** / **`secret_list`** | Set an encrypted secret (so functions that call third-party APIs can run) / list secret names + metadata. Values are write-only — never returned. |
| **`data_seed`** / **`data_query`** | Create / query records, as tenant-admin **or a given role** (runs as that role's identity). *Note: class `acl` isn't yet enforced for role tokens on the direct data API, so this doesn't prove the role's permissions.* |
| **`agent_invoke`** | Run a deployed agent and get its reply (waits briefly, else returns a handle to fetch the reply later). |
| **`knowledge_ingest`** | Load a text document into a knowledge container for RAG. |

All of these are **dev-only** (enforced by the runtime) and require the app to be deployed.

You mostly won't call these by name — ask Claude in plain language ("set the Stripe key as a
secret", "seed three test orders", "run the triage agent on this ticket") and it picks the tool.

The model is **hybrid**: reach for a dedicated tool when one fits, and **`aos_call`** is the
catch-all for everything else the platform's API exposes — so you're never blocked waiting for a
new tool to ship. A few concrete examples (you just ask in plain language; Claude picks the tool):

- *"Set the SendGrid API key as a secret"* → the dedicated **`secret_set`** tool.
- *"Seed three test orders, then show them back to me as the `sales` role"* → **`data_seed`** ×3,
  then **`data_query`** with `role: "sales"` (so you also see exactly what that role is allowed to read).
- *"Create a test user qa@acme.test with the manager role"* → there's no dedicated user tool, so Claude
  falls back to **`aos_call`** → `POST /api/v2.0/users/provision` with the user body.
- *"What external services are configured on this app?"* → **`aos_call`** →
  `POST /api/v2.0/admin/external-services/list`.

`aos_call` runs against your **dev** app as tenant-admin (or a role you name), so anything the app's
REST API can do — app config, users, external services, webhooks, scheduled jobs, audit — is reachable
even without a dedicated tool. Not sure of a path? Ask Claude to check **`aos_capabilities`** or the
API reference first. Rule of thumb: **dedicated tool if one exists (cleaner, safer), `aos_call` for
the long tail.**

### TL;DR

- **Stop calling `auth_refresh`** — refresh is automatic; reconnect `/mcp` if fully expired.
- **Restart Claude Code** after the update to pick up the new tools.
- **New:** secrets, data seed/query, agent invoke, knowledge ingest, and `aos_call` for everything
  else — all dev-only, after deploy.
