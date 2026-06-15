# Trillo — Claude Code plugin

Build and ship **Trillo AOS applications** without leaving Claude Code.
Install the plugin, sign in to your Trillo account, and Claude can discover
or create an app, generate its entities/functions/agents, write and refine
the code, and deploy — all over a single secure connection.

## What it does

The plugin registers one MCP server (**Trillo AI**) and ships a guide skill.
From a Claude Code session you can:

- **Pick or create an app** to work on (`app_list`, `app_select`, `app_create`).
- **Author the app's metadata** — software spec, entity model, functions,
  agents, UI specs — reading and writing through Trillo AI.
- **Generate and refine function code** locally, persisted back to Trillo.
- **Deploy** to your Trillo AOS dev environment and test the running app.

The `trillo:trillo-overview` skill walks Claude through the flow at the
start of a session.

## Prerequisites

- A **Trillo AI account** — sign up at [trillo.ai](https://trillo.ai).
- Claude Code (recent version with plugin support).

## Install

```bash
claude plugin marketplace add trillo/tcs-claude-plugin
claude plugin install trillo@trillo
```

(Or interactively: `/plugin marketplace add trillo/tcs-claude-plugin`,
then `/plugin install trillo@trillo`.)

## First use

1. Run **`/mcp`**, select **trillo-ai**, choose **Authenticate**. A browser
   opens the Trillo sign-in page; log in and (if prompted) choose the
   workspace to work in. Claude Code stores your credentials securely and
   refreshes them automatically.
2. Ask Claude to get started — e.g. *"list my Trillo apps"* or *"create a
   new Trillo app for …"*. The `trillo:trillo-overview` skill guides the
   rest.

That's it — no tokens to copy, no files to download.

## How it works

- **Secure sign-in (OAuth).** Authentication is standard OAuth handled by
  Claude Code; the plugin ships only a public client id, never a secret.
  Tokens live in your OS keychain.
- **One connection.** Everything — app discovery, metadata, code, deploy —
  flows through the Trillo AI MCP server. Tools load progressively
  (`discovery_list_groups` / `discovery_load_group`) so the surface stays
  small until you need more.
- **Dev-scoped.** Claude Code authors apps in your **dev** environment;
  promotion to production is done from the Trillo UI.

## Support

- Docs & sign-up: [trillo.ai](https://trillo.ai)
- Issues: please file them in this repository.

## License

© Trillo Inc. All rights reserved.
