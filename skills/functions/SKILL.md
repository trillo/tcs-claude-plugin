---
description: The Functions step — generate serverless function specs AND Python code from the SoftwareSpec/EntityModel, test them locally (offline pytest + MockCtx) and against the deployed app, and (post-deploy) seed data. Use after EntityModel, or when the user wants to add, edit, or test a function.
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
4. Write each function — spec **and** Python:
   `md_create modelClassName="FunctionM" name="<camelCaseName>" content={name,
   functionName, description, params, returns, code, runtime, ...}`.
   **Generate both names yourself, in the right case** — don't rely on the
   server to convert:
   - **`name`** — the **camelCase** linkage key, matching the function's name in
     `SoftwareSpec.functions` / agent tool bindings / `/fn/{name}` **exactly**.
     It's the join key across artifacts, so the casing must be stable — always
     camelCase, never snake_case (a snake_case name would be normalized and an
     acronym like `draftHTMLDoc` could drift to `draftHtmlDoc`, breaking the
     match with the spec).
   - **`functionName`** — the **snake_case** form of `name` (functions are
     Python). This is the workspace file/symbol: `functions/<functionName>.py`
     holds the `def handler(...)`, so the code reads naturally snake_case while
     the linkage key stays camelCase. (NodeJS later: `functionName == name`.)
   Example: `name: "draftHTMLDoc"`, `functionName: "draft_html_doc"`,
   file `functions/draft_html_doc.py`.

## Test locally first (offline, MockCtx — fast, no deploy)

Run logic tests in seconds with the toolkit's `MockCtx`. **Default-on: do this
before deploying.** These are *logic* tests against mocked AOS — they catch most
bugs in seconds; the deployed test (next section) is the integration truth.

**One-time workspace setup:**
- `pip install aos-toolkit` — provides `aos_toolkit` + `aos_toolkit_mock` +
  pytest. (The user owns the toolkit; don't modify it.)
- If `.trillo/<appId>/functions/conftest.py` is missing, create it — it supplies
  the `ctx` fixture and puts the functions dir on `sys.path`:

  ```python
  """Workspace test conftest — `ctx` (MockCtx) for local function tests."""
  from __future__ import annotations
  import sys
  from pathlib import Path
  import pytest
  import aos_toolkit
  from aos_toolkit_mock import MockCtx

  _FUNCTIONS = Path(__file__).resolve().parent
  if str(_FUNCTIONS) not in sys.path:
      sys.path.insert(0, str(_FUNCTIONS))

  @pytest.fixture
  def ctx(monkeypatch):
      mock = MockCtx()
      monkeypatch.setattr(aos_toolkit, "ctx", mock)
      for mod in list(sys.modules.values()):  # patch any loaded function module's ctx
          f = getattr(mod, "__file__", None)
          if f and str(_FUNCTIONS) in str(f) and hasattr(mod, "ctx"):
              monkeypatch.setattr(mod, "ctx", mock, raising=False)
      return mock
  ```

**Write the test** `functions/tests/test_<functionName>.py` — **happy + negative
branches** (mirror the toolkit examples): preload the reads the handler makes,
call `handler({...})`, assert on the result and on writes/effects. (`<functionName>`
is the snake_case file name; import the module by that name.)

```python
from <functionName> import handler

def test_happy(ctx):
    ctx.data.preload_get("Product", 1, {"id": 1, "inventory": 10})
    result = handler({"productId": 1, "quantity": 2})
    assert result["success"] is True
    assert ctx.data.was_called("create", class_name="Order")

def test_rejects_missing_param(ctx):
    assert handler({})["success"] is False  # missing required params
```
Assertion surface: `ctx.data.preload_get/preload_create`, `ctx.data.last_call(...)`,
`ctx.data.was_called(...)`, `ctx.audit.was_called(...)`, `ctx.email.was_called(...)`.
Cover happy path + missing/invalid params + not-found + guard failures.

**Run:** `pytest functions/tests/test_<functionName>.py` (offline). Read
failures, fix `functions/<functionName>.py`, re-run — seconds, no deploy.

> Get local tests green **before** deploying. If the user says deploy anyway,
> warn and proceed — local pass is a quality signal, not a hard gate (some
> functions need real data; the deployed test is the real check).

## Test against AOS (integration truth, post-deploy)

After local tests pass → push (`md_update`) → `deploy_app` → `function_test_sync`
(load the `activities` group via `discovery_load_group` if needed): runs the
real function against live AOS data — the check `MockCtx` can't give. Fix
(`md_update FunctionM`) → re-test (no redeploy between code edits; the test sends
your current code).

**Test as a specific role.** AOS behavior is often role- and identity-dependent
(ownership filters, the onboarding matrix, per-role ACL). `function_test_sync`
runs as tenant-admin; to exercise the app **as any role**, mint a role-scoped
token and call AOS directly:
1. **Pick a tenant.** Single-tenant app → use tenant 0 (the default). For a
   **multi-tenant** app (`AppConfig.multiTenant`), `tenant_list({appId})` to see
   tenants; `tenant_create({appId, name})` to make a dev test tenant (multi-tenant
   apps only — single-tenant apps return `APP_NOT_MULTI_TENANT`). Note its
   `tenantId`.
2. `aos_token({appId, role, tenantId})` (activities group) → `{aosToken, aosUrl,
   role, tenantId, expiresAt}`. AOS lazily creates a reserved `_user_<role>`
   test user in that tenant. `role` must be one of the app's `AppRole`s (or
   `admin`); list via `md_list(modelClassName="AppRole", appId=...)`. `tenantId`
   defaults to 0.
3. Exercise with Bash + curl: `POST {aosUrl}/api/v2.0/fn/{name}` (or `/data/...`)
   with header `Authorization: Bearer {aosToken}`. Compare what each role can do.
   Re-mint when the token expires (short TTL). **Dev-only** — refused for
   prod-promoted apps.

## Seed data (optional, post-deploy)

If the app needs seed/reference data, populate it with the AOS data tools
**after deploy** (data tools are deploy-gated).

## Add one / confirm

Single function: `step_guide({step:"Functions.add"})` (`AOSAddFunction`).
`app_status` → `Functions: COMPLETED`, `Agents: READY`. Summarize, ask to
proceed.
