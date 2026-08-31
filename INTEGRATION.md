# Multi-client integration (Claude Code · Cursor · Codex)

This plugin is built so the **MCP core is truly cross-client** and the **harness
layer ports via the cross-vendor Agent Skills standard**. Only one piece is
genuinely Claude-Code-specific (the batch *dynamic workflow*); on Cursor and
Codex that role is filled by the sequential harness or subagents. Every path is
autonomous after the single `/rca-build` gate — no host ever prompts mid-run. The
setup interview is a phase of that same skill: it runs on a repo's first invocation
and never again, so it is the one interactive surface and it is not per build.

## What transfers, what doesn't

| Layer | Claude Code | Cursor | Codex |
|---|---|---|---|
| `bstack` MCP server (`listTestIds` + `tfaRcaTurn` + `triggerRcaReport`) | `.mcp.json` (auto-discovered) | `.cursor-mcp.json` / `.cursor/mcp.json` | `~/.codex/config.toml` `[mcp_servers.bstack]` |
| `rca-build` skill (`SKILL.md`) | plugin `skills/` | Agent Skills (`.cursor/skills/` or cursor-plugin `"skills":"./skills/"`) | Agent Skills (`.agents/skills/`) |
| `ai-tfa-coordinator` agent | plugin `agents/` | `.cursor/agents/` (also reads `.claude/agents/`) | `.codex/agents/` |
| Per-test RCA **loop** | `agents/ai-tfa-coordinator.md` | same skill/agent | same skill/agent |
| Batch orchestration | dynamic workflow `workflows/rca-batch.mjs` (or subagents) | subagents, or **sequential** `lib/loop.mjs` | subagents, or **sequential** `lib/loop.mjs` |

The dynamic workflow (`workflows/rca-batch.mjs`) uses Claude Code's Workflow
runtime, which Cursor/Codex don't have. The same batch still runs there via
**subagents** (both hosts support subagents) or the **sequential thin-client
harness** `lib/loop.mjs` (`runRcaLoop`) — the conformance-tested loop that
drives `tfaRcaTurn` over the same contract without any host-specific
orchestration. On every host the run finishes the same way: glimpse table →
`triggerRcaReport(buildUuid)` → "Full report on the Test Observability UI:
<viewReport>". No local report file is ever written.

## Claude Code

```bash
claude --plugin-dir ./
/tfa-rca:rca-build <build-id>
```

No credentials to set: `.mcp.json` points at the hosted server and Claude Code runs
the OAuth flow on first connect. `/mcp` shows the connection and re-triggers sign-in.

`.claude-plugin/plugin.json` + root `.mcp.json` + `skills/` + `agents/` are
auto-discovered. (No `commands/rca-build.md` on purpose — a command and skill
with the same name collide and the skill body fails to load.)

## Cursor

The repo ships Cursor parity files mirroring `slack-mcp-plugin`:
`.cursor-plugin/plugin.json` (points at `../.cursor-mcp.json` and `./skills/`)
and `.cursor-mcp.json` (the hosted `bstack` server over HTTP).

**Wire the MCP server** — either:
- copy `.cursor-mcp.json`'s `bstack` entry into your project `.cursor/mcp.json`
  (top-level `mcpServers`), or
- Cursor → Settings → Cursor Settings → **MCP** → paste the same JSON, or
- use an **Add to Cursor** deeplink:
  `cursor://anysphere.cursor-deeplink/mcp/install?name=bstack&config=<base64-of-the-bstack-entry-body>`

Nothing else to set — the entry carries only `type` and `url`, and Cursor performs the
OAuth sign-in itself. There are no environment variables and no placeholders to fill.

**Skill + agent discovery** — Cursor reads `.cursor/skills/` and `.cursor/agents/`
(and also `.claude/agents/`). The simplest no-duplication setup is to symlink the
shared trees:

```bash
mkdir -p .cursor
ln -s ../skills  .cursor/skills
ln -s ../agents  .cursor/agents
```

Then drive it from Agent chat: invoke the `rca-build` skill with a build id.

## Codex

Codex reads the global `~/.codex/config.toml` (no per-project MCP file).

**Wire the MCP server** — either copy the block from `codex-mcp.example.toml`
into `~/.codex/config.toml`, or:

```bash
codex mcp add bstack --url "https://mcp.browserstack.com/mcp?isTfaPlugin=true"
```

Codex's own key for a streamable-HTTP server is `url`, and `auth = "oauth"` is its
documented fallback when no bearer token or static header is configured — which is this
case. No experimental flag is needed; `experimental_environment = "remote"` gates remote
*stdio executors*, a different feature.

**Skill + agent discovery** — Codex reads `.agents/skills/` (skills) and
`.codex/agents/` (subagents). Symlink the shared trees:

```bash
mkdir -p .agents .codex
ln -s ../skills  .agents/skills
ln -s ../agents  .codex/agents
```

Then run the `rca-build` skill; the coordinator + `tfaRcaTurn` loop are identical.

## Notes

- The `bstack` server is **remote HTTP with OAuth** — `type`/`url` in Claude Code and
  Cursor, `url`/`auth` in Codex. No `command`, no `args`, no `env`, and no credential in
  any config file.
- **`?isTfaPlugin=true` is load-bearing.** The server registers the TFA RCA
  collaboration tools per-request only when that query parameter is present. Drop it and
  the plugin loads with nothing to call, which looks like a broken install rather than a
  missing flag. Query parameters in the `url` are supported by all three clients.
- Env-var interpolation (`${VAR}`) is honored by Claude Code's `.mcp.json`; on
  Cursor/Codex, replace the placeholders with literals if your client doesn't
  expand them.
- Everything in `lib/` and the `SKILL.md`/agent prose is host-agnostic — only the
  MCP wiring file and the dynamic workflow are host-specific.
