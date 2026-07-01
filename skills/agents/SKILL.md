---
description: The Agents step — generate AI agents (codeless conversational, codeful handler, or one-shot document processors) from the SoftwareSpec, bind their tools, generate + test handler code locally (MockCtx + role-play), and deploy. Use after Functions, when the user wants to add/edit/test/tune an agent.
---

# Step: Agents — AI agents (codeless / codeful / oneshot)

Generates the app's AI agents (`AgentM`), binds the function-tools they call,
and — for coded agents — generates and tests the Python handler. Depends on
`Functions` being COMPLETED.

**Three kinds** (the agent's `kind`, classified by discovery):
- **codeless** — a conversational agent the platform loop runs as-is. No code.
- **codeful** — needs imperative pre/post logic around the loop: a Python handler
  with `boot`/`restore`/`post_process` hooks + a thin `handle`.
- **oneshot** — a single-shot document/data processor: one `ctx.llm` call, no loop,
  no conversation. Runs **like a function**.

Handler code follows the canonical contract — read
`trillo-aos/docs/coded-agent-handler-contract.md` (the scaffolds per kind) and
ground every `ctx.*` call on `toolkit_stubs()` (now includes `ctx.llm`/`ctx.agent`).
Don't guess the API.

## Generate

1. `step_guide({step:"Agents"})` → prompt + `expectedOutputSchema` (carries
   `kind`) + `systemClasses`.
2. **`toolkit_stubs()`** → the typed `aos_toolkit` API incl. `ctx.agent`
   (`setup`/`execute`/`state`/`render`) and `ctx.llm`
   (`generate`/`process_document`). Ground handler code on these.
3. Ground: `md_get SoftwareSpec` (its `aiAgents` section), `md_list FunctionM`
   (the tools an agent can use), `md_list ClassM`.
4. Write each agent (camelCase `name`, snake_case file), **by kind**:
   - **codeless** → spec only: `agents/specs/<name>.json` + `md_create
     modelClassName="AgentM" name="<agent>" content={name, kind:"codeless",
     instructions, tools, ...}`. No `.py`.
   - **oneshot** → handler `agents/<snake_name>.py` (the `oneshot` scaffold —
     one `ctx.llm.process_document(agent=...)` call) + spec + `md_create
     content={..., kind:"oneshot", code, handlerName:"handle", responseSchema}`.
   - **codeful** → handler `agents/<snake_name>.py` (the **factored** scaffold:
     `boot`/`restore`/`post_process` + `handle`) + spec + `md_create
     content={..., kind:"codeful", code, handlerName:"handle"}`.
   The spec is `content` **minus** `code`; the `.py` is `content.code`. If an
   agent needs a function-tool that doesn't exist yet, add it via **functions**
   first.

## Test locally first (codeful / oneshot — offline, MockCtx)

Before deploying, unit-test the handler **logic** in seconds with `MockCtx` —
no pod, no model (mirrors the functions step). codeless agents skip this (no code).

- One-time: install the toolkit (`toolkit_install()` → write files + `pip install
  -e ".trillo/toolkit[test]"`) — gives `aos_toolkit` + `aos_toolkit_mock` + pytest.
- Create `agents/conftest.py` if missing — the `ctx` fixture that swaps a fresh
  `MockCtx` into the handler module (same shape as `functions/conftest.py`).
- Write `agents/tests/test_<snake_name>.py`. `MockCtx` mocks the loop + LLM:
  - **codeful** → drive the hooks: `ctx.agent.preload_result(
    stop_reason="completed"|"incomplete")` to exercise branches; assert on the
    result, `ctx.agent.state`, and recorded calls. (Worked example:
    `tcs-metadata/python/aos_toolkit/examples/{coded_agent_example.py,
    test_coded_agent_example.py}`.)
  - **oneshot** → `ctx.llm.preload({...})`; assert the handler persists/returns it
    (a function-style test).
  - Handlers are `async` → call with `asyncio.run(handle(params))`.
- **Run:** `pytest agents/tests/test_<snake_name>.py` (offline). Fix the `.py`,
  re-run — seconds, no deploy.

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

**Iterate:** tweak `instructions` (`md_update AgentM` / `agents/specs/<agent>.json`)
and re-run — no redeploy needed: call `agent_prompt` again with
`instructionsOverride` set to the edited body to get the freshly-composed prompt.
Redeploy when you want the canonical, exactly-as-deployed check.

In plain English: *"role-play the ListingAssistant agent as a homeowner so I can
tune its prompt."*

### Codeful agents — role-play the loop, run the hooks for real

A codeful handler is the loop **sandwiched between Python hooks**, so role-play
the whole handler by running the hooks as real Python around the loop role-play:
1. **`boot`/`restore`** — run as real Python (a one-liner harness: import
   `agents/<snake_name>.py`, call `boot(params)` / `restore(params, state)`; their
   `ctx.*` lookups hit the deployed app). Get `state`.
2. **The loop** — role-play exactly as above (adopt `agent_prompt`, drive turns,
   execute real tools) → a final markdown + a `stop_reason` you pick.
3. **`post_process`** — run as real Python with the role-played result + `state`
   (it persists via `ctx.*`).

> **If the AgentM declares an `outputSchema` (B6):** the deployed pod extracts the
> turn's structured output automatically and exposes it as `result.output`. The
> role-play loop does **not** run that step — so when you build the role-played
> `result`, also **produce the structured output yourself** (act as the extractor:
> map the conversation to the `outputSchema`) and set it as `result.output` before
> calling `post_process`. Otherwise `result.output` is `None` locally and you won't
> exercise the handler's output-handling branch. For MockCtx unit tests, use
> `ctx.agent.preload_output({...})`.

This runs the actual hook code (the part codeful adds) while you role-play the
loop — no agent redeploy. (This is why the hooks are factored into named
functions.) For **oneshot**, there's no loop: just produce the structured output
for the `ctx.llm` call (or use the MockCtx test above).

> **Testing layers** (full detail in `trillo-aos/docs/agent-testing-guide.md`):
> (1) MockCtx unit (above) → (2) role-play (here) → (3) local-pod fidelity
> (future) → (4) deployed `agent-test`. Layers 1–2 need no agent redeploy.
