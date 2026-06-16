---
description: The EntityModel step — generate the app's custom business entities from the SoftwareSpec (and add/edit individual ones), and view the full model including built-in system entities. Use after SoftwareSpec, or when the user wants to add or change an entity.
---

# Step: EntityModel — the data model

Generates the app's **custom** business entities (`ClassM`). The platform's
**system entities** (`User`, `Tenant`, etc.) are built-in — you **reference**
them, never define them. Depends on `SoftwareSpec` being COMPLETED.

## See the full model first

`md_list modelClassName="ClassM"` returns:
- `rows` — this app's **custom** entities (editable here),
- `systemClasses` — the **built-in** platform entities (read-only, from
  `app-classes`; not editable via `md_*`).

The full model = custom + system. Always account for the system entities so you
don't duplicate them.

## Generate

1. `step_guide({step:"EntityModel"})` → prompt + `expectedOutputSchema` + the
   live `systemClasses`. Generate to the schema; **reference** system entities
   by name in relationships; do **not** redefine them.
2. Ground on the spec: `md_get modelClassName="SoftwareSpec"`.
3. Write each custom entity (PascalCase names):
   `md_create modelClassName="ClassM" name="<Entity>" content=<def>`.
   Edit one with `md_update`.

## Add a single entity

When the model already exists and the user wants one more, use
`step_guide({step:"EntityModel.add"})` (the `AOSAddEntity` prompt) →
`md_create ClassM`.

## Confirm + hand off

`app_status` → `EntityModel: COMPLETED`, `Functions: READY`. Summarize the
entities (custom + the system ones in play), then ask whether to proceed to
the **functions** step.
