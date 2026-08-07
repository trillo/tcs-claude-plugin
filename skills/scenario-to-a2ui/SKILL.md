---
description: The UI-Scenario step — turn a scenario (a role + a task) into stored, data-bound adaptive-UI screens. Author UIView/UIFlow via md_*, wire Direct-API actions + bindings, emit newFunctions[] for missing endpoints, and register a role sidebar. Use to "build the guided UI / screens / dashboard for a scenario", "generate a2ui Views/Flow", or "add a screen".
---

# Step: UI Scenario → adaptive-UI Views & Flows

Turn a scenario (**a role + a task**) into **stored, deterministic, data-bound screens** the
Adaptive UI serves **without an LLM call**. You author two metadata classes — **`UIView`** (one
screen) and **`UIFlow`** (a guarded sequence of screens) — plus any **`newFunctions[]`** the screens
call. Runs after `EntityModel`/`Functions` for a slice; **deploy gates the UI** (the serve-API reads
the deployed rows).

Guided-first: the sidebar + in-screen **Direct-API** actions are the primary path; chat is the
fallback. One task per screen (a small card/panel); depth comes from **sequencing** screens in a
Flow, not cramming one.

> **Distinct from the `ui-scenario` step.** `ui-scenario` *defines* scenarios (a role + a task, as a
> spec); this skill *builds* the concrete, stored a2ui screens from one. Run this after a scenario
> exists (or straight from a role + task) when the user wants actual screens/Views/Flows.

## Vocabulary

- **A2UISpec** — a flat screen definition: `{ "root": "<id>", "elements": { "<id>": { "type", "props"?, "children"?: ["<id>"] } } }`. Unknown `type` renders a visible fallback.
- **primitive** (`type`) — `Card, Heading, Text, Table, List, KeyValue, Stats, Chart, Progress, Badge, Alert, Divider, Image, Link, Code, Button, Form, FileUpload, EditableTable`.
- **UIView** — a stored A2UISpec (one screen), served role-scoped by `uid`.
- **UIFlow** — an ordered, guarded graph over Views (the guided experience).

## 1. Ground on the deployed catalog (don't guess)

- `md_list` / `md_get` the app's **ClassM** (entities), **FunctionM**, **AgentM** — screens bind to
  these. `aos_capabilities` for the platform surface. `md_list_model_classes` to confirm addressing.
- **Reuse before invent** — compose from the primitives above; propose a new renderer only for a
  genuine widget-level gap, not a one-off composition.

## 2. Decompose the scenario into a Flow of screens

Identify the screens, their order, and any conditional branches. A single-screen scenario is a
one-screen Flow.

## 3. Author each screen as a `UIView`

`md_create(modelClassName="UIView", name="<uid>", content={…})` — **`name` IS the `uid`** (the stable
key that `UIFlow.entryViewUid` and `UIView.parentUID` reference).

```jsonc
{
  "title": "Open Invoices",
  "role": "user",                 // the role this screen serves (sidebar is one View per role)
  "appRoles": ["billing"],        // optional extra app roles
  "placement": "content",         // navigation | content | modal | overlay
  "parentUID": "addInvoiceFlow",  // owning UIFlow uid — omit for a standalone View
  "sortOrder": 10,
  "icon": "receipt",              // navigation Views only
  "spec": { "root": "card", "elements": { "card": { "type": "Card", "children": ["tbl"] },
            "tbl": { "type": "Table", "props": { "dataSource": { "className": "Invoice",
                     "filters": [{ "field": "status", "op": "eq", "value": "open" }],
                     "sort": [{ "field": "dueDate", "dir": "asc" }] } } } } },
  "bindings": [ { "api": "fn:openInvoiceStats", "map": { "kpis.value": "count" } } ]
}
```

- **See** — pick data renderers: list → `Table`+`DataSource` (`{className, filters[], sort[]}`);
  KPIs → `Stats`; detail → `KeyValue`; trend → `Chart`. `Table`/`EditableTable` carry their **own**
  `DataSource`; every other renderer gets live values from the View-level **`bindings`** block
  (`[{ api | query, params?, map: { "<elementId>.<propPath>": "<resultPath>" } }]`; `api` = an MCP
  tool name or `fn:<name>`).
- **Do** — `Button` carries a **Direct-API** action `{ api, data }` + optional
  `then { goto: "<viewUid>", condition? }` (fast, auditable). Use an **agent-bounce** (invoke a
  deployed agent) for edit/open-ended screens and as the chat fallback.

## 4. Author the `UIFlow`

`md_create(modelClassName="UIFlow", name="<uid>", content={ title, role, appRoles?, entryViewUid,
transitions })`. The flow's step Views are the `UIView`s with `parentUID = <this uid>`.

- `transitions`: a guarded graph — `[{ from, on, to, condition? }]`. A Direct-API action's
  `then.goto` jumps to a stored View (deterministic); an agent-bounce generates the next screen.
- `condition`: a declarative boolean over the action **result** / flow state — atom `{ field, op,
  value }` (`op ∈ eq,neq,gt,gte,lt,lte,like,in`), combined with `{and:[…]} | {or:[…]} | {not:…}`.
  No arbitrary code.

## 5. Register the role sidebar (entry points)

Each role gets **one navigation surface**: author a `placement:"navigation"` `UIView` per role whose
`spec` links into that role's Views/Flows, ordered by `sortOrder`. The Adaptive UI bootstraps it via
`GET /api/v2.0/ui/sidebar?role=<role>`; individual screens load via `/ui/view/{uid}` and
`/ui/flow/{uid}`.

## 6. Missing backend → `newFunctions[]`

If a screen's action or binding needs a function that isn't built, author it in the **Functions**
step (`md_create FunctionM`) as part of this scenario, and point the screen's `api` at `fn:<name>`.
Never author identity/auth functions (the platform owns that surface).

## 7. Deploy gates the UI, then reconcile

The serve-API reads **deployed** rows, so **deploy to dev** after authoring — then the Adaptive UI
can serve the sidebar → Views → Flows. **Reconcile** any endpoint you referenced before it existed:
confirm it deployed, match params/response. Promotion to prod is just a redeploy (the `UIView`/
`UIFlow` rows travel with the app).

## Renderer support in flight (`[[ext]]`)

Three spec features depend on renderer work owned by the **adaptive-UI (React) session** — author
against them (forward-looking and correct at rest), but a screen relying on them won't fully render
until they land:

1. **View-level `bindings`** — live values in non-table renderers (`Stats`/`KeyValue`/`Chart`/
   `Progress`). The big one; most guided screens need it.
2. **Direct-API `action`/`then` on `Form`/`EditableTable`/`FileUpload`** (today only `Button` has it).
3. **`{param}`** flow-context substitution in `DataSource` filters / `bindings` (e.g. `{salesOrderId}`).

**Stage-1 screens** (a `Table` bound to a `DataSource`, plus a `Button`) need none of these — they
render today. Reach for `[[ext]]` features only for the live/guided Stage-2 path.
