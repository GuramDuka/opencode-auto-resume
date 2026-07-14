# Architecture

**opencode-auto-resume** is a plugin for [OpenCode](https://github.com/anomalyco/opencode) that monitors all LLM sessions and auto-recovers from predictable failure modes. It is a single self-contained TypeScript module (~1400 lines) exporting a factory function conforming to the `@opencode-ai/plugin` interface.

## Plugin Interface

```typescript
export const AutoResumePlugin: Plugin = async (ctx, options) => {
  // ... setup, timers, event handlers ...
  return {
    event: ({ event }) => handleEvent(event),  // called on every SSE event
    config: async () => { /* ... */ },          // called when config reloads
    tool: { task_complete },                    // exposes a tool to agents
  }
}
```

The plugin returns three lifecycle hooks:

| Hook | Trigger | Purpose |
|------|---------|---------|
| `event` | Every SSE event | Track sessions, detect failures, trigger recovery |
| `config` | Config reload | Log confirmation |
| `tool` | Agent calls `task_complete` | Deterministic completion signal |

## Core State: `SessionWatch`

Every tracked session has a `SessionWatch` object. This is the full state machine:

```typescript
interface SessionWatch {
  // --- Activity tracking ---
  lastActivityAt: number        // timestamp of last SSE event for this session
  status: "busy" | "idle" | "retry" | "unknown"

  // --- Stall recovery ---
  resumeAttempts: number        // how many continues sent (capped by maxRetries)
  lastRetryAt: number           // backoff timestamp
  gaveUp: boolean               // true when maxRetries exhausted
  continuing: boolean           // lock: prevents concurrent continue sends

  // --- User cancellation ---
  userCancelled: boolean        // set on MessageAbortedError (ESC)
  aborting: boolean             // lock: prevents concurrent abort+resume

  // --- Orphan parent recovery ---
  orphanWatchStartAt: number | null
  isSubagent: boolean

  // --- Tool-text detection ---
  toolTextRecovered: boolean    // once true, no more tool-text checks
  toolTextAttempts: number      // (capped by maxRetries)
  checkingToolText: boolean     // lock: prevents concurrent checks
  toolTextTimer: ReturnType<typeof setTimeout> | null

  // --- Hallucination loop ---
  continueTimestamps: number[]  // sliding window of continues (10 min)
  interruptedContinueCount: number  // consecutive interrupted continues (cap 3)

  // --- Tool loop detection ---
  recentToolCalls: ToolCallRecord[]  // sliding 2-min window
  toolLoopAttempts: number

  // --- Subagent health ---
  lastSubagentCheckAt: number

  // --- Todo integration ---
  todos: Todo[]
  todoCheckAttempts: number

  // --- Idle cleanup ---
  idleSince: number | null
}
```

### State Machine

```
                ┌──────────┐
     event      │          │  session.list()
  ─────────────>│ unknown  │<─────────────────┐
                │          │                   │
                └────┬─────┘                   │
                     │ session.status: busy    │
                     v                         │
                ┌──────────┐                   │
                │          │                   │
                │  busy    │                   │
                │          │                   │
                └────┬─────┘                   │
                     │ session.status: idle    │
                     v                         │
                ┌──────────┐                   │
                │          │  idle > 10min     │
                │  idle    │ ──────────────> cleanup (deleted)
                │          │
                └──────────┘
```

## Event Flow

Every SSE event passes through `handleEvent()`. Here is how each type is processed:

### `session.status`
- **busy**: Reset all recovery flags (`resetSessionFlags`), touch activity timer.
- **idle**: Set `idleSince`. Schedule tool-text check (3s delay). Check orphan parent condition (did busyCount drop from >1 to 1?). If todos remain open on a lone busy session, send continue.
- **interrupted**: Same as idle, but check if a continue was just sent — retry immediately (up to 3x). Then schedule tool-text check with 500ms delay.
- **retry**: Touch activity timer only.

### `session.created` / `session.updated`
- Ensure a `SessionWatch` entry exists for the session.

### `session.idle`
- Legacy idle event. Same as `session.status` with idle type: tool-text check after 3s delay.

### `session.interrupted`
- Legacy interrupt event. Tool-text check after 500ms delay. Limited retry of interrupted continues (3 cap).

### `session.error`
- `MessageAbortedError` → mark ALL busy sessions as user-cancelled (ESC pressed), never resume them.
- Other errors → log and ignore (already-idle sessions may fire spurious errors).

### `command.executed`
- Reset all recovery flags across all sessions. A new command means fresh context.

### `todo.updated`
- Sync the todo list for the session. Used by the todo-based continue guard.

## Timer Loop (every `checkIntervalMs` = 5s)

```typescript
setInterval(() => {
  for each busy session:
    1. Sync real status from session.list() API
    2. Skip if: not busy | userCancelled | aborting
    3. Orphan watch active? → check subagent status → recover or abort+resume
    4. Multiple busy sessions? → skip (subagent running)
    5. Idle > chunkTimeoutMs + gracePeriodMs? → tryResume() or give up
  cleanupIdleSessions()
}, checkIntervalMs)
```

Key guard: when `busyCount > 1`, stall detection is paused entirely. Only lone busy sessions are candidates for recovery.

### Periodic Session Discovery

A separate timer runs `session.list()` every 60 seconds to pick up sessions that were missed by event tracking. Idle sessions older than 10 minutes or exceeding 50 total entries are cleaned up.

## Prompt Sending: `sendContinuePrompt`

Every continue goes through this function, which:
1. Acquires the `continuing` lock (prevents concurrent sends)
2. Fetches the last messages for the session
3. Extracts agent, model, and provider from the **last user message** (not assistant)
4. Calls `ctx.client.session.prompt()` with the extracted config
5. Records the continue timestamp (for hallucination detection)
6. Retries once on failure
7. Releases the `continuing` lock
8. Schedules a deferred check: verify the session went busy after the prompt

Model preservation is important — the plugin preserves the user's UI selection (agent, model, provider) by reading it from message history rather than the session `info` field.

## Abort + Resume: `tryAbortAndResume`

For hallucination loops and orphan parents, the plugin uses a two-step recovery:
1. Call `ctx.client.session.abort()` to kill the stuck generation
2. Wait 2 seconds
3. Call `sendContinuePrompt()` with "continue"

The `aborting` flag prevents concurrent aborts.

## Tool System

### `task_complete` tool

Agents can call this built-in tool to signal work is done:

```typescript
const taskCompleteTool = tool({
  description: "Signal that all work is complete.",
  args: {},
  execute: async (_args, ctx) => {
    // For parent sessions: set toolTextRecovered = true, clear pending timers
    // Subagents can call it without affecting parent recovery
    return "Task completion acknowledged."
  },
})
```

This is registered in the `tool` return hook and replaces fragile text-based heuristics.

## Logging

All logging goes through `ctx.client.app.log()` — never `console.log`. Log levels used: `debug`, `info`, `warn`, `error`. The service name is always `"auto-resume"`.

## Helper Functions

| Function | Purpose |
|----------|---------|
| `short(sid)` | Truncates session IDs for readable logs |
| `backoffMs(attempt)` | Exponential backoff: `baseBackoff * 2^(attempt-1)`, capped at `maxBackoffMs` |
| `getSid(ev)` | Extracts sessionID from various event shapes |
| `getError(ev)` | Extracts error object from event shapes |
| `getStatus(ev)` | Extracts status object from event shapes |
| `busyCount()` | Counts non-cancelled busy sessions |
| `getLoneBusySession()` | Returns the only busy session if exactly one exists |
| `touchSession(sid)` | Resets activity timer for a session |
| `ensureWatch(sid)` | Creates or returns SessionWatch entry |
| `resetSessionFlags(w)` | Clears all recovery flags (session went busy) |
| `resetIdleFlags(w)` | Clears idle-transient flags only |
| `extractMessages(response)` | Normalizes API response to message array |
