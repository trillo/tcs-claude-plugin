# Trillo — Claude Code plugin

Author **Trillo AOS** applications from Claude Code. The plugin registers
one OAuth-protected MCP server (**Trillo AI**) and ships skills that guide
a session through login → app select/create → generation → deploy.

Distributed as a self-hosted marketplace: this repo is **both** the
marketplace catalog and the plugin.

## Install

```bash
# 1. Add this repo as a marketplace
claude plugin marketplace add trillo/tcs-claude-plugin

# 2. Install the plugin
claude plugin install trillo@trillo
```

(Or interactively: `/plugin marketplace add trillo/tcs-claude-plugin`
then `/plugin install trillo@trillo`.)

## First use

1. `/mcp` → select **trillo-ai** → **authenticate**. A browser opens
   Trillo AI's login; sign in with your Trillo AI account.
2. Ask Claude to start — the `trillo:trillo-overview` skill drives the
   rest (`app_list` → `app_select`/`app_create` → author → deploy).

## Layout

```
tcs-claude-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest (name "trillo")
│   └── marketplace.json     # marketplace catalog (lists this plugin)
├── .mcp.json                # Trillo AI MCP server: http url + OAuth client
├── skills/
│   └── trillo-overview/SKILL.md   # session bootstrap (auth + app lifecycle)
└── README.md
```

More skills (function-codegen, testing, deploy, building-an-app) and a
`prompts/` set are added incrementally (plan-59.B).

## ⚠️ Environment URL

`.mcp.json` currently points the `trillo-ai` server at
`https://localhost:9020/api/v2.0/mcp` — the **local dev** Trillo AI
gateway. Before distributing the plugin publicly, change that `url` to the
public Trillo AI URL. The OAuth `clientId` (`trillo-claude-code`) is a
public client (PKCE, no secret) and is safe to ship; `callbackPort` is the
loopback port Claude Code uses for the OAuth redirect.

## Local development — self-signed certificate

The dev gateway serves a **self-signed TLS cert**, which Claude Code's MCP
client (and the OAuth discovery call) reject by default — the server shows
`✘ failed · self signed certificate`. This is a **dev-only** issue; a
production URL with a valid cert won't see it.

Relaunch Claude Code with TLS verification relaxed **before** authenticating
(the cert blocks the OAuth discovery call too, so it must come first):

```bash
# blunt, dev-only — skip cert verification entirely
NODE_TLS_REJECT_UNAUTHORIZED=0 claude

# or cleaner — trust only the dev gateway cert
openssl s_client -connect localhost:9020 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > ~/trillo-dev-gateway.pem
NODE_EXTRA_CA_CERTS=~/trillo-dev-gateway.pem claude
```

Then `/plugin` (or `/mcp`) → `trillo-ai` → **Authenticate**. See
`TESTING.md` for the full clean-room procedure.

## Plan

See `trillo-aos/docs/plan-59-claude-code-authoring.md` (slice 59.B) for the
plugin design and roadmap.
