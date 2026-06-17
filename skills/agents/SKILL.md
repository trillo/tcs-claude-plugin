---
description: The Agents step — generate codeless conversational AI agents (and bind the function-tools they use) from the SoftwareSpec, and role-play an agent locally to iterate on its prompt before deploying. Use after Functions, when the user wants to add or edit an agent, or to test/tune/role-play an agent's prompt.
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

## Role-play to iterate on the prompt (post-deploy)

Agents run on the execution pod at runtime. Before the slow **deploy → open the
chat UI → converse** loop, iterate on an agent's **instructions + tool bindings**
fast by **role-playing the agent yourself** — adopt its system prompt, let the
user drive turns, and **actually call its tools** against the deployed app.

> This is *prompt/behavior* iteration, not bit-fidelity. You are not the agent's
> real model (Gemini) and there's no selection-narrowing step or memory here — so
> it catches prompt, tool-binding, and output-format issues, **not** model-specific
> quirks. The deployed agent in the chat UI is the integration truth.

**Setup (needs the app deployed):**
1. `md_get AgentM name=<agent>` → `instructions`, `tools` (globs: `data_*`,
   `fn:*`, function names), `model`, `scope`.
2. **Get the real system prompt** — `agent_prompt({appId, name:"<agent>"})`
   (activities group) returns `{prompt}`: the agent's instructions wrapped in the
   platform common-preamble + common-appendix, placeholders substituted, exactly
   as the runtime composes it. **Adopt that `prompt` as your operating prompt for
   the role-play.** To preview *edited, not-yet-deployed* instructions without a
   redeploy, pass them: `agent_prompt({appId, name, instructionsOverride:"<edited
   instructions>"})`.

**Resolve each bound tool to a real call:**
- **FunctionM tools** (`fn:*` / a function name) → `function_test_sync({appId,
  name, params})`.
- **`data_*` tools** → `curl {aosUrl}/api/v2.0/data/...` with an `aos_token`.
  Agents act as a **delegated user**, so mint the token for the **role the agent
  serves** (not admin) so ownership/ACL behave realistically. Multi-tenant app →
  pick/create the tenant first (`tenant_list` / `tenant_create`), then mint in it.
- **discovery tools** (`list_classes`, `describe_class`) → answer from
  `md_list`/`md_get`. **`save_memory`** → stub (acknowledge, don't persist).

**The loop:** under the agent's system prompt, answer each user turn the developer
types; when the agent would call a tool, **execute the real call**, append the
result, and continue (cap ~10 tool hops, like the pod). Show the turns + tool
calls so the developer can judge the behavior.

**Iterate:** tweak `instructions` (`md_update AgentM` / `agents/<agent>.json`) and
re-run — no redeploy needed: call `agent_prompt` again with
`instructionsOverride` set to the edited body to get the freshly-composed prompt.
Redeploy when you want the canonical, exactly-as-deployed check.

In plain English: *"role-play the ListingAssistant agent as a homeowner so I can
tune its prompt."*
