# opencode-auto-resume — Agent Guide

**OpenCode plugin** that auto-recovers stalled LLM sessions: stalls, broken tool calls, hallucination loops, orphan subagents.

First read this file. For deeper dives, see `docs/`.

## Quick start

```bash
bun install        # install dependencies (migrated from npm, uses bun.lock)
bun run build      # build src/index.ts → dist/index.js --target bun
bun test           # run all 125 tests across 5 files concurrently
bun run dev        # watch mode rebuild
```

## Documentation index

| File | When to read |
|------|-------------|
| `docs/ARCHITECTURE.md` | You need to understand the state machine, event flow, timer loop, or add a new event handler |
| `docs/RECOVERY-MECHANISMS.md` | You're debugging a recovery issue or need to understand a specific failure mode |
| `docs/DEVELOPMENT.md` | You're writing tests, adding a feature, or setting up the repo |
| `README.md` | You're installing or configuring the plugin as an end user |

## Architecture (lightning)

Single entry point: `src/index.ts` → `AutoResumePlugin` returns `{ event, config, tool }`.

- **Event-driven** — processes `session.status`, `session.created`, `session.updated`, `session.idle`, `session.interrupted`, `session.error`, `command.executed`, `todo.updated`
- **Timer loop** (every 5s) — checks busy sessions for stalls, loops, orphans
- **Tool-text detection** — on idle, fetches last messages, scans for raw XML/JSON tool calls
- **`task_complete` tool** — agents call this to signal done (replaces emoji heuristics)
- **All logging** via `ctx.client.app.log()` — never `console.log`

Core data structure per session: `SessionWatch` (24 fields tracking activity, retries, backoff, todos, tool calls, model filtering, etc.). See `docs/ARCHITECTURE.md` for the full state machine.

## 9 recovery mechanisms at a glance

| # | Mechanism | Trigger | Detection |
|---|-----------|---------|-----------|
| 1 | **Stall** | Stream goes silent >48s | Timer loop: `idle > chunkTimeoutMs + gracePeriodMs` |
| 2 | **Tool-as-text** | XML/JSON tool call printed as text | On idle: scan last 3 messages for 17 patterns |
| 3 | **Hallucination loop** | >3 continues in 10 min | `continueTimestamps` sliding window |
| 4 | **Orphan parent** | Subagent finishes, parent stays busy | busyCount drop: >1 → 1 |
| 5 | **Subagent stuck** | Subagent idle >1 min (>3 min if tool active) | `checkSubagentStatus()` in timer loop |
| 6 | **Todo guard** | Model claims done with open todos | "ready to continue" + `todos` check |
| 7 | **Interrupted retry** | Continue was interrupted | `continuing` flag + `lastRetryAt < 2s` |
| 8 | **🎉 completion** | Message ends with 🎉 | Text scan: `normalized.endsWith('🎉')` |
| 9 | **Spurious errors** | Error events on already-idle sessions | `busyCount === 0` at event time |

Recovery actions: send "continue" prompt (stall, tool-text, interrupted), abort+resume (hallucination, orphan), or mark recovered (🎉). See `docs/RECOVERY-MECHANISMS.md` for full details.

## Project structure

```
src/
  index.ts                      # plugin source (~1400 lines, single file)
  index.test.ts                 # short(), backoffMs(), getSid(), getError()
  index.integration.test.ts     # lifecycle, agent extraction, prompt flow
  index.plugin.test.ts          # state tracking, resume, recovery, todos (largest file)
  index.coverage.test.ts        # tool-text pattern matching
  index.it.test.ts              # state machine, event parsing, abort flow
docs/
  ARCHITECTURE.md               # state machine, event flow, timer loop
  RECOVERY-MECHANISMS.md        # each recovery type in detail
  DEVELOPMENT.md                # testing, adding features, CI, contribution
```

## Key configuration (plugin options)

| Option | Default | Effect |
|--------|---------|--------|
| `chunkTimeoutMs` | 45000 | Inactivity before stall recovery |
| `checkIntervalMs` | 5000 | Timer poll interval |
| `maxRetries` | 3 | Max auto-resume attempts |
| `subagentWaitMs` | 15000 | Wait before orphan-parent recovery |
| `loopMaxContinues` | 3 | Max continues in hallucination window |
| `baseBackoffMs` / `maxBackoffMs` | 1000 / 8000 | Retry backoff range |
| `modelFilter` | `""` (disabled) | Regex string — only recover sessions whose `providerID/modelID` matches (e.g. `"anthropic/.*"`) |

## Testing

```bash
bun test                              # all 125 tests
bun test src/index.plugin.test.ts     # specific file
bun test -t "Tool Text"               # by name pattern
```

- Tests heavily mock `ctx.client.session.{list,messages,prompt,abort}` — see `createMockContext()` in `index.plugin.test.ts` for a minimal example, or `createRealisticContext()` in `index.integration.test.ts` for a full one
- No external services, no fixtures, no snapshots
- All tests are purely in-memory

## Gotchas

- **Build needed before publish, not before tests** — `bun test` resolves imports from TypeScript source directly (Bun handles this). The build target `bun` outputs Bun-runtime-specific code.
- **`bun install` migrated from npm** — the original lockfile was `package-lock.json`. Running `bun install` replaced it with `bun.lock`. Both lockfile formats may be encountered.
- **Module format**: ESM only (`"type": "module"`).
- **Session ID prefix matters in prod** — production `getSid()` filters for `"ses_"` prefix. Tests use plain IDs like `"session-1"` and bypass this. If you write new tests, remember production will reject non-`ses_` IDs.
- **Backoff differs between prod and tests**: Production uses `baseBackoffMs * 2^(attempt-1)` (1s base, 8s cap). Tests reimplement with `5000 * 2^attempt` (5s base, 160s cap). Do not compare backoff values across prod and test output.
- **No linter/formatter configured** — no ESLint, Prettier, or biome config exists. Match the existing 4-space indentation.
- **Repository is a fork** — origin: `GuramDuka/opencode-auto-resume`, upstream: `Mte90/opencode-auto-resume`.

## CI / Publishing

GitHub Actions: `npm-publish.yml` triggers on releases + workflow_dispatch.
Steps: `bun install` → `bun run build` → `bun test` → `npm publish --provenance` (needs Node 24).
