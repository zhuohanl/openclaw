# How the Agent Loop Works in OpenClaw

## The Big Picture

OpenClaw does not implement the core ReAct loop itself.
The actual loop lives inside an external package:

```
@mariozechner/pi-agent-core  (v0.55.3)
@mariozechner/pi-coding-agent
```

OpenClaw's job is to:
1. Set up the session and tools
2. Fire off the loop with one call
3. Subscribe to events as the loop runs
4. Bridge those events to OpenClaw's own output streams


## Entry Points

```
Gateway RPC  →  agent / agent.wait
CLI          →  agent command
```

The full call chain:

```
Gateway RPC (server-methods/agent.ts)
  └─ agentCommand (commands/agent.ts)
       └─ runEmbeddedPiAgent (agents/pi-embedded-runner/run.ts)
            └─ runEmbeddedAttempt (agents/pi-embedded-runner/run/attempt.ts)
                 ├─ createAgentSession()        ← sets up pi-agent-core session
                 ├─ subscribeEmbeddedPiSession() ← listens to events
                 └─ activeSession.prompt()      ← fires the loop
```

Everything from `activeSession.prompt()` downward runs inside pi-agent-core.
That single async call only resolves when the loop is completely done.


## What Happens Before the Loop Fires

`runEmbeddedAttempt` does a lot of preparation before calling `prompt()`:

- Acquires a **session write lock** (one run per session at a time)
- Opens the **SessionManager** to load conversation history
- Resolves **model + auth profile** (with failover support)
- Loads **skills** and injects them into the system prompt
- Builds the **system prompt** from base prompt + skills + bootstrap context
- Sanitizes and validates the **message history** (turn ordering, tool pairing, image pruning)
- Detects and loads **images** referenced in the prompt
- Runs **before_prompt_build** plugin hooks
- Starts an **abort timer** (default 600s timeout)


## The ReAct Loop — Three Phases as Events

Once `prompt()` is called, pi-agent-core drives the loop and emits events.
OpenClaw subscribes to these via `subscribeEmbeddedPiSession()`.

The event handler (`pi-embedded-subscribe.handlers.ts`) dispatches to:

```
message_start / message_update / message_end   →  Reason phase
tool_execution_start / update / end            →  Act phase
agent_start / agent_end                        →  Loop lifecycle
auto_compaction_start / end                    →  Compaction
```

### Reason Phase

`message_update` fires repeatedly with sub-events:

| Sub-event | Meaning |
|---|---|
| `thinking_start/delta/end` | Model's hidden reasoning (inside `<think>` tags) |
| `text_delta` | Streaming visible text, character by character |
| `text_end` | Final text for this assistant message turn |

Hidden reasoning is stripped by `stripBlockTags()` and never sent to the user.
Visible text is streamed out on the `assistant` event stream in real time.

If `enforceFinalTag` is enabled, only content inside `<final>` tags is treated
as the answer. Everything before `<final>` is considered thinking-out-loud
and is suppressed.

### Act Phase

When the model emits a tool call, pi-agent-core fires:

```
tool_execution_start  →  OpenClaw emits tool stream { phase: "start" }
                          runs before_tool_call plugin hooks
tool_execution_update →  partial results streamed out
tool_execution_end    →  result captured, after_tool_call hooks run
                          result written back into session history
```

The tool result goes back into the model's context automatically.
This feeds the next Reason cycle.

### Observe Phase

There is no explicit "Observe" event. It is implicit:
the tool result is appended to the session messages by pi-agent-core,
which then triggers the next `message_start` in the same loop.

The cycle continues until the model produces a response with no tool calls.


## The Three Output Streams

While the loop runs, OpenClaw pushes events to three named streams:

```
assistant  →  streamed text deltas and final text
tool       →  tool start / update / result events
lifecycle  →  phase: "start" | "end" | "error"
```

These are emitted via `emitAgentEvent()` and consumed by the Gateway,
which delivers them to WebSocket clients, messaging channels, etc.


## Compaction

When the context window fills up, pi-agent-core triggers auto-compaction.
OpenClaw handles this by:

1. Catching `auto_compaction_start` → pauses block reply flushing
2. Waiting for `auto_compaction_end` → resets in-memory buffers
3. The loop then retries with a compacted context

Compaction events are emitted on the `lifecycle` stream so callers can observe them.


## How the Loop Terminates

`agent.wait` polls for a `lifecycle end` or `lifecycle error` event on the given `runId`.

The loop ends when any of these happen:

| Cause | How |
|---|---|
| Model sends no more tool calls | pi-agent-core resolves `prompt()` naturally |
| Timeout | `setTimeout` fires → `abortRun(true)` → `activeSession.abort()` |
| External cancel | `AbortSignal` → same abort path |
| Compaction failure | Caught as `promptErrorSource = "compaction"` |
| Gateway disconnect | RPC timeout propagates as abort |


## Key Files

| File | Role |
|---|---|
| `src/gateway/server-methods/agent.ts` | Gateway RPC entry point |
| `src/commands/agent.ts` | `agentCommand` orchestrator |
| `src/agents/pi-embedded-runner/run.ts` | `runEmbeddedPiAgent` — queuing + model resolution |
| `src/agents/pi-embedded-runner/run/attempt.ts` | `runEmbeddedAttempt` — the full setup + loop launch |
| `src/agents/pi-embedded-subscribe.ts` | `subscribeEmbeddedPiSession` — event subscription setup |
| `src/agents/pi-embedded-subscribe.handlers.ts` | Event dispatcher |
| `src/agents/pi-embedded-subscribe.handlers.messages.ts` | Reason phase handlers |
| `src/agents/pi-embedded-subscribe.handlers.tools.ts` | Act phase handlers |
