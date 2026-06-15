# Clean-room test — Trillo Claude Code plugin

Verify the **plugin** (not a hand-edited `~/.claude.json`) registers the
`trillo-ai` MCP server, connects over TLS, completes OAuth, and exposes the
tools + `trillo:trillo-overview` skill.

## 0. Prereqs
- Trillo AI running locally; gateway on `https://localhost:9020`.
  - Sanity: `curl -sk -o /dev/null -w "%{http_code}\n" https://localhost:9020/api/v2.0/mcp` → `401` (alive + protected). Connection refused → start Trillo AI first.
- Plugin repo: `github.com/trillo/tcs-claude-plugin` (local dir `/Users/anil/workspace2/trillo-claude-plugin`).

## 1. Remove any old manual MCP config (avoid `trillo-ai` name collision)
- Edit `~/.claude.json`; under the relevant project's `"mcpServers"`, delete any hand-added `"trillo-ai": { … }` entry (leave `"mcpServers": {}`).
- Why: the plugin registers a server *also* named `trillo-ai`; two sources collide and you can't tell which is active.

## 2. Add the marketplace + install the plugin
- From GitHub: `claude plugin marketplace add trillo/tcs-claude-plugin` then `claude plugin install trillo@trillo`
- Or local path: `claude plugin marketplace add /Users/anil/workspace2/trillo-claude-plugin` then `claude plugin install trillo@trillo`

## 3. Relaunch Claude Code trusting the dev (self-signed) cert
The dev gateway's self-signed cert blocks the MCP connection **and** the OAuth discovery call, so trust it **before** authenticating:
```bash
NODE_TLS_REJECT_UNAUTHORIZED=0 claude        # blunt, dev-only
# or: NODE_EXTRA_CA_CERTS=~/trillo-dev-gateway.pem claude   # trust only the dev cert
```

## 4. Authenticate
- `/plugin` (or `/mcp`) → `trillo-ai` → **Authenticate** → browser opens Trillo AI login → sign in (+ pick tenant) → status flips to ✔ connected.

## 5. Verify
- `/plugin` → `trillo` · **enabled**; `trillo-ai` · **connected** (source = the plugin, `Config location: Dynamically configured` — NOT `~/.claude.json`).
- Skill `trillo:trillo-overview` available.
- E2E: ask Claude to "list my Trillo apps" → it calls `app_list` → returns your tenant's apps.

## 6. Fail signals
- `✘ self signed certificate` → relaunch with the TLS env var (step 3) before auth.
- `△ Enter to auth` → connected over TLS, just needs OAuth (step 4).
- Server shows under "Local MCPs (~/.claude.json)" → old manual entry not removed (step 1).
- Server absent → plugin not installed/enabled (`/plugin`) or marketplace not added.

## 7. Iterate / cleanup
- After plugin edits: push, then `claude plugin marketplace update trillo` → restart.
- Remove: `claude plugin uninstall trillo@trillo`.

## Notes
- Production: point `.mcp.json` `url` at the real Trillo AI HTTPS endpoint (valid cert → no step 3 needed).
- `clientId` `trillo-claude-code` is a public PKCE client (no secret) — safe to ship.
