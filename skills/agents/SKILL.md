---
description: The Agents step — generate codeless conversational AI agents (and bind the function-tools they use) from the SoftwareSpec. Use after Functions, or when the user wants to add or edit an agent.
---

# Step: Agents — conversational AI agents

Generates the app's AI agents (`AgentM`) and binds the function-tools they
call. Depends on `Functions` being COMPLETED.

## Generate

1. `step_guide({step:"Agents"})` → prompt + `expectedOutputSchema` +
   `systemClasses`.
2. Ground: `md_get SoftwareSpec` (its `aiAgents` section), `md_list FunctionM`
   (the tools an agent can use), `md_list ClassM`.
3. Write each agent (camelCase names):
   `md_create modelClassName="AgentM" name="<agent>" content={name,
   instructions, tools, ...}`. If an agent needs a function-tool that doesn't
   exist yet, add it via the **functions** step first.

## Add one / confirm

Single agent: `step_guide({step:"Agents.add"})` (`AOSAddAgent`).
`app_status` → `Agents: COMPLETED`. Summarize, then ask whether to proceed to
**ui-scenario** or straight to **deploy**.
