# How Multi-Agent Orchestration Works in OpenClaw

## The Big Picture

OpenClaw agents can spawn other agents as subagents to delegate work.
The parent orchestrates; children execute specialized tasks in parallel.

This is not a framework you configure in advance.
The LLM itself decides at runtime when to spawn a child, what task to give it,
and how to synthesize the results.


## Two Spawn Paths

| Path | Tool | How child runs |
|---|---|---|
| **Subagent spawn** | `sessions_spawn` | Same process, via Gateway `agent` RPC |
| **ACP spawn** | `sessions_spawn` with ACP routing | Separate process or machine via Agent Control Plane |

The common path is subagent spawn. ACP spawn is for cross-machine / cross-process federation.
This document focuses on subagent spawn.


## End-to-End Walkthrough

### Step 1 — Parent calls `sessions_spawn`

The parent's LLM decides it needs help and calls the `sessions_spawn` tool.
This triggers `spawnSubagentDirect()` in `agents/subagent-spawn.ts`.

Before anything launches, safety gates run in this order:

```
1. callerDepth >= maxSpawnDepth     → forbidden   (default max depth: 5)
2. activeChildren >= maxChildren    → forbidden   (default max children: 5)
3. agentId not in allowAgents       → forbidden   (cross-agent allowlist)
4. child unsandboxed + parent sandboxed → forbidden
```

If all checks pass, a unique child session key is minted:

```
agent:<targetAgentId>:subagent:<uuid>
```


### Step 2 — Child session is configured

The child session is patched via Gateway `sessions.patch` RPCs:

- `spawnDepth` — set to `parentDepth + 1`
- `model` — optional override specified by the parent
- `thinkingLevel` — optional thinking override
- Thread binding — if `thread=true`, a channel thread is bound to the child session


### Step 3 — Child's system prompt is injected

`buildSubagentSystemPrompt()` (`subagent-announce.ts:896`) generates a system prompt
that is injected into the child session. Key instructions:

- You are a **subagent**, not the main agent
- Your only job is the assigned task
- Your final message will be **auto-announced back to your parent** — do NOT poll
- No heartbeats, no cron jobs, no side quests
- Descendant results are push-based; do not busy-poll for status


### Step 4 — Child agent is launched

The parent calls Gateway `agent` RPC with:

```
deliver: false          ← do not deliver to end-user directly
lane: AGENT_LANE_SUBAGENT
extraSystemPrompt: <subagent system prompt>
```

This fires off the same `runEmbeddedPiAgent` ReAct loop, but in its own session.
The call returns immediately with `{ runId }`.

The child run is registered in the **subagent registry** (`registerSubagentRun`).

The parent's LLM is told:

> "Do NOT call sessions_list, sessions_history, exec sleep, or any polling tool.
> Wait for completion events to arrive as user messages."


### Step 5 — Child runs independently

The child executes its own full ReAct loop:

```
reason → tool calls → observe → reason → ... → final reply
```

It has no connection to the parent's loop.
It runs in its own session, with its own context window.


### Step 6 — Auto-announce on completion

When the child's loop ends, `runSubagentAnnounceFlow()` is triggered
(`subagent-announce.ts:1136`). It:

1. Calls `agent.wait` on the child's `runId` to confirm it settled
2. Reads the child's final assistant reply (`readLatestSubagentOutput`)
3. Decides how to deliver the result:

| Requester type | Delivery method |
|---|---|
| Another subagent (depth ≥ 1) | `queueEmbeddedPiMessage()` — injected directly into the parent's live ReAct loop as a new user message |
| Top-level user | Delivered via the messaging channel (Slack, Telegram, etc.) |

The parent's LLM then sees the child's result arrive as a new user message.
It can synthesize a final answer or spawn more children.


## Parallel Execution

Multiple children can run at the same time (up to `maxChildren`, default 5).

The parent is expected to:
1. Spawn all children in one turn
2. Track which `childSessionKey` values it is waiting for
3. Only send its final answer after ALL expected completion messages have arrived

This coordination is enforced through prompt instructions, not blocking code.
If a child completion event arrives after the parent already replied,
the parent is instructed to reply only with `NO_REPLY`.


## Full Picture

```
Parent agent (depth 0)
  │
  ├─ LLM calls sessions_spawn(task="...", agentId="worker-a")
  ├─ LLM calls sessions_spawn(task="...", agentId="worker-b")
  │         │
  │         └─ spawnSubagentDirect()
  │               ├─ safety checks (depth, children, allowlist, sandbox)
  │               ├─ mint child session key
  │               ├─ patch child session (depth, model, thinking)
  │               ├─ inject subagent system prompt
  │               └─ Gateway agent RPC → child ReAct loop starts async
  │
  │   (parent's LLM is idle, waiting for push-based completions)
  │
  ├─ Child A (depth 1) runs its own ReAct loop → produces final reply
  │     └─ runSubagentAnnounceFlow()
  │           └─ queueEmbeddedPiMessage → arrives as user message in parent
  │
  ├─ Child B (depth 1) runs its own ReAct loop → produces final reply
  │     └─ runSubagentAnnounceFlow()
  │           └─ queueEmbeddedPiMessage → arrives as user message in parent
  │
  └─ Parent's LLM sees both results, synthesizes final answer
```


## Depth and Guards

Subagent spawn depth is tracked in the session store.
`getSubagentDepthFromSessionStore()` walks up the `spawnedBy` chain to find the true depth.

A child at depth 1 can itself spawn children at depth 2, and so on,
up to `maxSpawnDepth` (default 5).

The subagent system prompt informs the child whether it is allowed to
spawn further children (`canSpawn = childDepth < maxSpawnDepth`).


## Cleanup

When a child run ends, its session can be:

| `cleanup` value | Behavior |
|---|---|
| `"delete"` | Session transcript is deleted after announce |
| `"keep"` | Session is kept (used when `mode="session"` for persistent thread-bound agents) |

Cleanup is attempted best-effort. Failures are logged but do not block the announce.


## Key Files

| File | Role |
|---|---|
| `src/agents/subagent-spawn.ts` | `spawnSubagentDirect()` — full spawn logic |
| `src/agents/subagent-announce.ts` | `runSubagentAnnounceFlow()` and `buildSubagentSystemPrompt()` |
| `src/agents/subagent-registry.ts` | Tracks active child runs per session |
| `src/agents/subagent-depth.ts` | Resolves true spawn depth from session store |
| `src/agents/acp-spawn.ts` | ACP (cross-machine) spawn path |
| `src/agents/subagent-announce-dispatch.ts` | Low-level announce delivery |
| `src/agents/subagent-announce-queue.ts` | Announce queue management |
