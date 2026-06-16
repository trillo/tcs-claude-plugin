---
description: Drives building a Trillo AOS app step by step (Input → SoftwareSpec → EntityModel → Functions → Agents → UIScenario → deploy). Use whenever the user wants to build, continue, or resume a Trillo app, asks "what do I do next", or right after an app is selected. Checks app_status, runs the current step, and asks before advancing.
---

# Building a Trillo AOS app — the development process

You guide the user through a **stepwise refinement** — the same process the
Trillo AI UI follows. The generation knowledge (prompts, schemas, the toolkit
API) is served live by Trillo AI; your job is to orchestrate the steps and
write each result back. Keep the user in the loop — this is a conversation,
not a batch job.

## The step chain

```
Input → SoftwareSpec → EntityModel → Functions → Agents → UIScenario → deploy
```

Each step depends on the previous one, and the **server enforces it** — a step
stays `NOT_READY` until its predecessor has output. So you never have to track
ordering yourself: **`app_status` is the source of truth.**

## On every turn (and whenever the user asks "what's next?")

1. Make sure an app is selected — see the **trillo-overview** skill
   (`app_list` / `app_select` / `app_create` + `.trillo/session.json`).
2. Call **`app_status`** → `{activities:[{name, status, statusInfo}], nextActivity, deployStatus}`.
3. Briefly tell the user where things stand — a short checklist:
   `COMPLETED ✓ · READY (do next) · NOT_READY (blocked — statusInfo says on what)` —
   and what `nextActivity` is.

## Running a step

For the current step (`nextActivity`) or a step the user names:

1. **Fetch its guide:** `step_guide({step})` → the live prompt
   (`modelRole`/`objective`/`instructions`/`constraints`) + `expectedOutputSchema`.
   Generate **exactly** to that schema — do not invent fields.
   Steps: `SoftwareSpec`, `EntityModel`, `Functions`, `Agents`, `UIScenario`;
   add-one variants: `EntityModel.add`, `Functions.add`, `Agents.add`.
   (`Input` has no `step_guide` — use the **requirements** skill.)
2. **Generate** the artifact. Show the user a short summary, not a wall of JSON.
3. **Write** it with the right model class:
   - `SoftwareSpec` / `UIScenario` → `md_update` (one activity_output per app)
   - `ClassM` (EntityModel) / `FunctionM` (Functions) / `AgentM` (Agents) →
     `md_create` per item, `md_update` to edit one
4. **End of step:** summarize what was produced, then **ASK before moving to
   the next step.** Do not auto-run the whole chain.

## Per-step skills

When a focused skill exists for the step, follow it: **requirements** (Input),
**software-spec**, **entity-model**, **functions**, **agents**,
**ui-scenario**, **deploy**. (They ship incrementally; if one isn't present,
fall back to this skill + `step_guide`.)

## Revisiting / adding

The user can go back to a completed step to regenerate or refine it, or add a
single item — use the `.add` `step_guide` variant (`EntityModel.add`, etc.) for
one entity/function/agent. After any write, re-check `app_status`; the server
re-derives downstream readiness.

## Capability gaps

If a requirement needs something AOS doesn't support yet, ground yourself with
`aos_capabilities` (the coverage map: In place / Partial / Gap), tell the user
plainly rather than faking it, and — with their OK — file it via
`request_aos_capability`. See the **request-aos-capability** skill.

## Guardrails

- Authoring is **dev only** — prod is locked server-side.
- DB seeding / function testing need the app **deployed** (`deployStatus ==
  "deployed"`); do those after `deploy`.
