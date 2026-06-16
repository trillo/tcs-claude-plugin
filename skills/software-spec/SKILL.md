---
description: The SoftwareSpec step — generate the app's backend specification from the Input requirement and write it. Use after Input is complete, or when the user asks to generate, regenerate, or refine the spec (e.g. "add a requirement for X"). Drives step_guide → generate → md_update SoftwareSpec.
---

# Step: SoftwareSpec — generate the backend specification

Derives the structured backend spec from the `Input` requirement. Writing it
seeds the app's roles + onboarding matrix server-side and unlocks
`EntityModel`. Depends on `Input` being COMPLETED.

## Generate

1. **`step_guide({step:"SoftwareSpec"})`** → the live prompt
   (`modelRole`/`objective`/`instructions`/`constraints`) + the
   **`expectedOutputSchema`**. This is the authoritative contract — generate
   exactly to it; do not invent fields.
2. Read the requirement for grounding: `md_get modelClassName="Input"`.
3. Produce a **single JSON object** matching `expectedOutputSchema`. Per the
   prompt's rules:
   - personas **camelCase** (use `admin` for administrators),
   - entity names **PascalCase**, function names **camelCase**,
   - backend only — **no CRUD, no auth/login/signup, no UI wireframes, no
     infra/deploy** requirements (those are built in or handled elsewhere),
   - mark anything you inferred (not stated by the user) with `"inferred": true`.

## Write

```
md_update  modelClassName="SoftwareSpec"  appId=<appId>  content=<spec JSON>
```

`SoftwareSpec` is **`explicit_user_request_only`** — only create/rewrite it
when the user asks (generate the spec, or "add a requirement for X"), not as a
side effect.

## Confirm + hand off

Call `app_status` → confirm `SoftwareSpec: COMPLETED`, `workflowStatus: READY`,
and `EntityModel: READY`. Summarize the spec for the user (personas, key
entities, functions, agents, integrations), then ask whether to proceed to the
**entity-model** step.
