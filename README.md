# Hyperagent Orchestrator (Claude skill)

A Claude skill for driving [Hyperagent](https://hyperagent.com) agents over its hosted MCP server — list agents, start background threads, poll for results, and chain or fan-out work across multiple agents.

Works in any MCP host (Claude Code, the Claude desktop/web apps, Cowork, or other MCP clients) that has the Hyperagent connector added.

## What it gives Claude

- The **mental model**: the MCP is a remote control for agents that already exist — you dispatch work and poll for results, you don't build agents through it.
- The **six tools** and the one rule that matters most: the opening message must be fully self-contained (agents can't ask a clarifying question mid-run).
- The **core loop**: `list_agents` → `create_thread` → poll `get_thread` until `isRunning` is false → read / `send_message`.
- Three recipes: **sequential chain** (output → next input), **fan-out** (parallel dispatch), and **human-in-the-loop refine** (`send_message` on an existing thread).
- Verified response shapes, an honest "what's *not* available over MCP" section (scores/costs/run counts live only in the Command Center UI), and a monitoring-dashboard pattern (`references/dashboard.md`) including the in-artifact bridge caveat and a snapshot-first fallback.

## Install

- **Cowork / Claude desktop:** download [`dist/hyperagent-orchestrator.skill`](dist/hyperagent-orchestrator.skill) (or grab it from the [latest release](../../releases/latest)) and use "Save skill", or add it via Settings → Capabilities.
- **Claude Code / manual:** copy the `hyperagent-orchestrator/` folder into your skills directory (e.g. `~/.claude/skills/`).

## Prerequisite

Add the Hyperagent MCP connector (`https://hyperagent.com/api/mcp`) to your MCP host and complete the one-time browser sign-in. See Hyperagent's MCP docs for details.

## Contents

```
hyperagent-orchestrator/
├── SKILL.md                 # skill instructions + frontmatter
└── references/
    └── dashboard.md         # monitoring-dashboard pattern + bridge caveat
```

## License

MIT — see [LICENSE](LICENSE).
