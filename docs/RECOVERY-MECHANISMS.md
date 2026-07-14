# Recovery Mechanisms

The plugin detects and recovers from 9 distinct failure modes. Each is documented below with trigger conditions, detection logic, and recovery action.

---

## 1. Stall Recovery

**What it detects**: The LLM stream goes silent mid-generation. The UI shows a blinking cursor with no new tokens.

**Trigger**: A session stays `busy` but emits no SSE events for `chunkTimeoutMs + gracePeriodMs` (default 48s).

**Detection** (in timer loop):
```
w.lastActivityAt > 0
&& (now - w.lastActivityAt) >= chunkTimeoutMs + gracePeriodMs
&& !hasActiveTool(sid)
&& w.resumeAttempts < maxRetries
```

**Action**: `tryResume()` → `sendContinuePrompt()` with `"continue"`.

**Backoff**: `baseBackoffMs * 2^(attempt-1)` capped at `maxBackoffMs`. Default: 1s, 2s, 4s, 8s.

**Exhaustion**: After `maxRetries` attempts, `gaveUp = true` and no further resumes are sent for that session.

**Guards**:
- Skipped if `busyCount > 1` (subagents running — activity may not hit this session's timer)
- Skipped if session has an active tool call (tool execution may take longer than timeout)
- Skipped if within backoff window since last retry

---

## 2. Tool-Call-as-Text Recovery

**What it detects**: The model prints XML/JSON tool invocations as raw text (e.g., `<function=edit>` or `{"type":"function"}`) instead of executing them. The session goes idle but the tool was never run.

**Trigger**: Session goes idle → scheduled check after 3s (or 500ms for interrupted sessions).

**Detection** (`checkForToolCallAsText`):
1. Fetch last 3 messages
2. Scan for pattern matches against `TOOL_TEXT_PATTERNS` (17 patterns):
   - Raw XML tags: `<function=`, `<invoke`, `<tool_call`, `<parameter`
   - Truncated XML: open tag without close tag
   - Plain tool names as XML: `<edit>`, `<write>`, `<read>`, etc.
   - JSON formats: `{"type":"function"}`, `{"name":"read"}`
3. Check for truncated XML (open tag present, close tag absent)
4. Also check for "ready to continue" text patterns

**Recovery prompts** (priority order):
| Priority | Source | Prompt |
|----------|--------|--------|
| 0 | tool-text (reasoning) | "Your last message contained a raw tool call in reasoning. Please execute it." |
| 0 | tool-text (text) | "Your last message contained a raw tool call printed as text instead of being executed." |
| 0 | tool-loop (detected) | "You've been calling the same tool repeatedly — step back and reassess." |
| 1 | ready-to-continue | "continue" |
| 1 | done-claim (no emoji) | "I need you to verify more carefully that you have actually completed all the required tasks." |
| 1 | tool-use (part) | "continue" |
| 2 | idle-with-open-todos | "continue" |

**Backoff**: Same exponential backoff as stall recovery, shared `toolTextAttempts` counter.

**Guards**:
- Max `maxRetries` attempts per session
- Once `toolTextRecovered = true`, no further checks
- Skipped if session went busy between schedule and check
- Skipped if session was active within last 3 seconds

### Tool Loop Detection

During tool-text scanning, the plugin tracks the last 15 tool calls (within 2 minutes) and detects:
- **Consecutive same tool**: 3+ identical tools in a row
- **Pattern loop**: repeating patterns of length 2–5 (e.g., `edit-read-edit-read-edit-read`)

When detected, sends `TOOL_LOOP_RECOVERY_PROMPT` instead of the standard prompt (up to 2 loop recovery attempts).

---

## 3. Hallucination Loop Detection

**What it detects**: The model generates the same broken output repeatedly. Each "continue" picks up the broken generation, creating an infinite loop.

**Trigger**: A session exceeds `loopMaxContinues` (default 3) continue prompts within `loopWindowMs` (default 10 minutes).

**Detection**:
```
continueTimestamps.length >= loopMaxContinues
// within sliding window: only timestamps in the last 10 min
```

**Action**: `tryAbortAndResume()` — abort the stuck request, wait 2 seconds, then send a fresh "continue".

**Guards**:
- Check if session has an active tool call before aborting (don't abort mid-tool)
- `aborting` flag prevents concurrent aborts

---

## 4. Orphan Parent Recovery

**What it detects**: A subagent finishes (goes idle) but the parent session stays stuck in "busy" forever. OpenCode's UI shows the parent as processing but no subagent is actually running.

**Trigger**: `busyCount` drops from >1 to exactly 1 after a session goes idle. The remaining busy session is the orphan candidate.

**Detection**:
```typescript
// On session.status: idle event:
if (prevBusyCount > 1 && currentBusy === 1) {
  lone.orphanWatchStartAt = Date.now()
}
```

**Action** (in timer loop, after `subagentWaitMs + gracePeriodMs` = 18s):
1. `checkSubagentStatus(sid)` — looks for any busy subagents
2. If a subagent is crashed: try `recoverSubagent(stuckSid)` → if fails, abort+resume parent
3. If no subagents exist (status === "idle"): abort+resume parent
4. If subagent still running: wait more

**Guards**:
- `orphanWatchStartAt` starts the timer only after busyCount drops
- `subagentWaitMs` prevents premature recovery
- Will not retry orphan recovery if `resumeAttempts >= maxRetries`

---

## 5. Subagent Stuck Detection

**What it detects**: A subagent hasn't produced new text for >1 minute (>3 minutes if a tool call is in progress).

**Trigger**: Checked as part of orphan parent flow and timer loop.

**Detection** (`checkSubagentStatus`):
1. List all sessions
2. For each session that is NOT the parent:
   - If busy: check last message timestamp
   - No new text in `SUBAGENT_STUCK_MS` (60s), or `SUBAGENT_STUCK_MS * 3` if tool call in progress → crashed
   - If last message has an error → crashed

**Action**: `recoverSubagent(sid)` — sends a recovery prompt to the stuck subagent. If recovery fails, trigger abort+resume on the parent.

---

## 6. Todo-Based Continue Guard

**What it does**: Prevents the plugin from sending "continue" when all tasks are legitimately done, while also preventing premature abandonment when there's still work to do.

**Detection**: During tool-text scanning, when the model says "ready to continue":
1. Check session's `todos` list (synced from `todo.updated` events)
2. If open todos exist → send "continue"
3. If all todos completed/cancelled → skip continue (first 2 attempts)
4. On 3rd attempt with all todos completed → send a nudge: "Please close all completed todos and finish your message."
5. If open todos remain and session is idle → send "continue" (idle-with-open-todos)

Also detects "done" claims without emoji (`task done`, `all done`, `finished`, `complete`) and checks if todos are still open. If work is claimed done but todos remain, sends `DONE_WITHOUT_WORK_PROMPT`.

---

## 7. Interrupted Continue Retry

**What it detects**: The plugin sends a "continue" but the session immediately gets interrupted (network issue, race condition).

**Detection**: On `session.interrupted` event, check if `continuing` flag was recently set or `lastRetryAt < 2s ago`.

**Action**: Retry the continue immediately. Capped at 3 consecutive interrupted continues to prevent infinite loops.

---

## 8. 🎉 Completion Detection

**What it detects**: The model signals completion with 🎉 at the end of its message.

**Detection**: During tool-text scanning, normalize the text (trim, strip trailing punctuation) and check if it ends with `🎉`.

**Action**: Set `toolTextRecovered = true` — stop all future recovery attempts for this session. Never send a continue.

---

## 9. Spurious Error Suppression

**What it handles**: OpenCode fires `session.error` events after normal completion. These are not actual failures.

**Detection**: If `busyCount === 0` when a non-abort error arrives, ignore it.

**Action**: Log at `debug` level only. No state changes.
