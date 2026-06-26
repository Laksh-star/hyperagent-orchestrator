---
name: hyperagent-orchestrator
description: Drive Hyperagent (hyperagent.com) agents over its hosted MCP server — list agents, start background threads, poll for results, and chain or fan-out work across multiple agents. Use this whenever the user wants to run, dispatch, orchestrate, monitor, or chain Hyperagent agents from an MCP client (Claude Code, Claude desktop/web, Cowork, or any MCP host), or mentions "my Hyperagent agents", running an agent "in the background", building a multi-agent pipeline/chain/fan-out on Hyperagent, or a fleet/monitoring dashboard of their agents. Trigger even when the user doesn't say "MCP" explicitly — e.g. "kick off my research agent then feed it to my deck agent", "run these 5 agents in parallel and collect the results", or "show me what my agents are doing". Also use when the user asks why a Hyperagent run isn't behaving (stuck, no clarification, missing data).
---

# Hyperagent Orchestrator

## Mental model (read this first)

Hyperagent's MCP server is a **remote control for agents that already exist** on the user's Hyperagent account. You do **not** build or configure agents through MCP — you *dispatch work* to pre-built agents and collect results. Every agent runs **in the background**: you start a thread, it works asynchronously, and you poll for the result.

Internalizing this framing prevents the most common mistakes: trying to "create an agent" (you can't, via MCP), or waiting synchronously for a reply that only arrives by polling.

## Connecting

The server lives at `https://hyperagent.com/api/mcp` and authenticates via a one-time browser sign-in. Once the connector is added to the MCP host, its tools appear namespaced by the server (e.g. `mcp__<server-id>__list_agents`). If the tools are deferred/not loaded, discover and load them with ToolSearch (query the tool names below) before calling.

## The six tools

| Tool | What it does |
|---|---|
| `list_agents` | List reachable agents with `id`, `name`, `description`. Copy the `id` you need. |
| `create_thread` | Start an agent on a **self-contained** opening message. Returns a `threadId` immediately; the agent runs in the background. |
| `send_message` | Add a follow-up turn to an existing thread — runs now if idle, else after the current turn. The human-in-the-loop lever. |
| `get_thread` | Read a thread's messages plus an `isRunning` flag. **Poll this** to track progress and collect results. |
| `list_threads` | Browse threads (keyset-paginated; returns `nextCursor`). |
| `create_attachment_upload` | Get an upload URL for a file; PUT the bytes, then pass the returned `fileId` into `create_thread`/`send_message` — keeps file bytes out of your context. |

## The one rule that matters most

**The opening message must be fully self-contained.** It becomes the thread's first user turn, and the agent runs with no one to answer a mid-run clarifying question. If the task is ambiguous, resolve the ambiguity *before* dispatching — ask the user, gather the inputs, then compose one complete message. A vague opening message is the #1 cause of disappointing background runs.

## Core loop

ALWAYS follow this shape for a single dispatch:

1. `list_agents` → pick the agent's `id` (match on name/description).
2. `create_thread` with that `agentId` and a complete, self-contained `message` → capture the `threadId`.
3. Poll `get_thread` with that `threadId` until `isRunning` is `false`. Use a small `messageLimit` (e.g. 1) on poll calls — thread transcripts can be large and a full read may overflow context.
4. Read the final assistant message for the result, or `send_message` to continue.

Between polls, give the agent real time to work — many agents take minutes (research, media, deck-building). Don't hammer `get_thread` in a tight loop.

## Recipe: sequential chain (output → next input)

Use when one agent's output is the next agent's input.

1. Dispatch agent A; poll until done.
2. Extract A's final output.
3. Dispatch agent B with an opening message that **embeds** A's output (B cannot see A's thread — it only knows what you put in its message). Keep B's message self-contained.
4. Poll B; repeat down the chain.

**Example:** a research agent produces a sourced comparison → feed that comparison verbatim into a deck/report agent as its opening message.

## Recipe: fan-out (parallel dispatch)

Use when the same work applies to many inputs, or many agents apply to one brief. This is the biggest speed win the MCP offers.

1. Fire several `create_thread` calls **in the same turn** (same agent over N inputs, or N agents on one brief). Collect all the `threadId`s.
2. Poll each with `get_thread` until all are `false`.
3. Aggregate. Optionally feed the aggregate into a final "reduce" agent (chain + fan-out combined).

## Recipe: human-in-the-loop refine

When the user wants to correct or extend a finished run, use `send_message` on the **existing** thread rather than starting over — the agent keeps its context, so you only describe the delta ("tighten section 3", "strip the unsourced claims"). Poll again for the revised result.

## Agent delegation (server-side) vs. MCP orchestration (client-side)

Hyperagent supports **agent delegation**: agents can invoke each other directly, handing off work based on specialization (configured per agent under the Model tab; the owner controls which agents may invoke which). This overlaps with the sequential-chain recipe — but it runs *inside* Hyperagent, not from your client. Choose deliberately:

- **Let agents delegate (server-side) when** the team is fixed and pre-wired, you want the agents to decide routing, and you only care about the final deliverable. Just `create_thread` on the lead agent and poll `get_thread` — the internal handoffs happen within that run.
- **Orchestrate from the client (MCP) when** you need deterministic sequencing, human approval gates between steps, **parallel fan-out**, cross-connector pipelines (pull from Gmail/Slack/Drive, route results back out), or you want to combine agents the owner hasn't enabled for delegation. MCP can reach any agent you can; delegation only spans the enabled pairs.
- **Combine them:** trigger a delegating *lead* agent over MCP, let it hand off internally, and poll for the assembled result. This is cleaner than client-side relay for fixed teams while keeping MCP for triggering, fan-out, and result collection.

Either way the orchestration primitives are unchanged — `create_thread` to start, `get_thread` (`isRunning`) to track, `send_message` to refine. Delegation just moves some handoffs from your code into the run.

## Response shapes (so your parsing is right)

- `list_threads` → `{ threads: [{ id, name, namedAgentId, invocationSource, isArchived, createdAt, updatedAt }], nextCursor }`. `invocationSource` is `"mcp"`, `"scheduled"`, or `null` (manual/UI).
- `get_thread` → `{ thread: {…}, messages: [...], isRunning: boolean }`. The human-readable text usually lives in a message's `contentBlocks` (a JSON string of typed blocks); plain `content` may just be a placeholder like `"[1 tool result]"`.
- Poll with `messageLimit: 1` to read `isRunning` cheaply; only pull more messages once the run is done.

## What is NOT available over MCP (set expectations honestly)

The MCP surface exposes agents, threads, and messages — **nothing else**. Quality scores, per-run costs, total/agent run counts, and "pending improvements" exist only inside Hyperagent's own app (the Command Center). If the user wants those numbers in an external view, tell them plainly: they aren't reachable via MCP. The options are (a) read the Command Center page with browser tools, or (b) wait for a public read/REST API. Don't fabricate or estimate these values.

## Building a fleet/monitoring dashboard

If the user wants a live board of their agents and threads, read `references/dashboard.md` for the full pattern — including the important runtime caveat about whether the host injects an in-artifact data bridge, and the snapshot-first fallback that keeps the dashboard working regardless.

## Attachments

To hand an agent a file (PDF, dataset, transcript): call `create_attachment_upload` to get an `uploadUrl` + `fileId`, PUT the bytes to `uploadUrl`, then pass `fileId` in `attachmentIds` on `create_thread`/`send_message`. Never read large file bytes into your own context just to forward them.

## Quick gotcha checklist

- Opening message not self-contained → agent guesses or stalls. Fix the message, not the agent.
- Synchronous expectation → there is none; always poll `get_thread`.
- Huge `get_thread` payloads → poll with `messageLimit: 1`.
- Asking for scores/costs over MCP → not available; see the section above.
- "Create an agent" via MCP → not supported; agents are built in the Hyperagent app.
