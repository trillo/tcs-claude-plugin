---
description: The UIScenario step — define per-role UI scenarios for the app. Optional. Use after SoftwareSpec, or when the user asks about the app's UI scenarios/pages.
---

# Step: UIScenario — UI scenarios

Defines high-level UI scenarios per persona/role. The AOS **adaptive /
Generative UI** renders the actual screens at runtime — you describe
scenarios, **not** wireframes or UI code. Depends on `SoftwareSpec`.

## Generate

1. `step_guide({step:"UIScenario"})` → prompt + `expectedOutputSchema`.
2. Ground: `md_get SoftwareSpec` (its `ui` + `personas` sections).
3. Write: `md_update modelClassName="UIScenario" content=<scenarios JSON>`.

## Confirm

`app_status` → `UIScenario: COMPLETED`. Summarize the scenarios, then ask
whether to proceed to **deploy**.
