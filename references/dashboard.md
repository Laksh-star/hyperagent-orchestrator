# Building a Hyperagent fleet/monitoring dashboard

A dashboard over Hyperagent is a **read-only join**: `list_agents` (id → name) joined with `list_threads` (threads tagged by `namedAgentId`, `invocationSource`, timestamps). Optionally `get_thread` per recent thread for a live running/idle flag and a last-message preview. No writes — safe to refresh freely.

## What you can show from MCP data

- Counts: total agents, total/recent threads, breakdown by trigger (`invocationSource`: mcp / scheduled / manual).
- Per-agent thread counts (group threads by `namedAgentId`). Note these are *threads observed*, which differ from Hyperagent's internal "run" ledger — say so rather than implying parity.
- Recent threads list with name, agent, trigger tag, relative time.
- Running/idle status + last-message preview, if you poll `get_thread` on the newest N threads (`isRunning`; pull the latest readable text block from `contentBlocks`).

## What you CANNOT show from MCP data

Scores, costs, quality trends, "pending improvements" — none are on the MCP surface (Command Center UI only). Either omit them, render them blank with an honest "n/a via MCP" label, or read the Command Center page with browser tools as a separate snapshot.

## The critical runtime caveat: the in-artifact data bridge

If you render the dashboard as a persisted host artifact (e.g. Cowork's `create_artifact`), the artifact's JS calls MCP tools through a host-injected bridge (e.g. `window.cowork.callMcpTool`). **In some runtimes that bridge is NOT injected** — the connector works fine at the session level, but code *inside* the artifact cannot reach it, and a naive dashboard throws `Cannot read properties of undefined (reading 'callMcpTool')`.

Design for this from the start with **snapshot-first + graceful upgrade**:

1. Fetch current data yourself (session-level MCP calls) and **embed it as a snapshot** literal in the artifact HTML, with a visible "as of <timestamp>" label.
2. On load, render the embedded snapshot **immediately** — no error path, no blank screen.
3. Optionally attempt the bridge with a short timeout; if present, upgrade to live (status/preview); if absent, stay on the snapshot. Never let a missing bridge produce an error.
4. Refresh the snapshot **out of band**: have the assistant re-pull and rewrite the embedded data on request, or wire an ad-hoc scheduled task that does the same. (A button that calls `runScheduledTask` also needs the bridge, so it can't be the only refresh path.)

A helper to wait for the bridge before using it:

```js
function ensureBridge(timeout=8000){
  return new Promise((res,rej)=>{ const t0=Date.now();
    (function chk(){
      if(window.cowork && typeof window.cowork.callMcpTool==="function") return res();
      if(Date.now()-t0>timeout) return rej(new Error("bridge_unavailable"));
      setTimeout(chk,150);
    })();
  });
}
```

Parsing inside an artifact: `callMcpTool` returns `{content, structuredContent, isError}` — read `r.structuredContent ?? JSON.parse(r.content[0].text)`.

## Pagination

`list_threads` is keyset-paginated (default ~20, max ~100, returns `nextCursor`). For accurate per-agent counts, page through several times (cap it, e.g. 5 pages) — but remember even a full sweep counts threads, not Hyperagent's internal runs.

## Honest-labeling rules

- Mark snapshot views with their capture time.
- Label MCP-derived run counts as "threads observed", not "runs".
- Show score/cost cells as blank/"n/a via MCP" rather than guessing.
