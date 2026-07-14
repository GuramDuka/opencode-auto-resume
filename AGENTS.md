# opencode-auto-resume — Agent Guide

**OpenCode plugin** that auto-recovers stalled LLM sessions: stalls, broken tool calls, hallucination loops, orphan subagents.

## Quick start

```bash
bun install        # install dependencies
bun run build      # build: bun build src/index.ts --outdir dist --target bun
bun test           # run all tests (5 test files in src/)
bun run dev        # watch mode rebuild
```

## Architecture

Single entry point: `src/index.ts` → exports `AutoResumePlugin` function returning `{ event, config, tool }` hooks.

- **Event-driven**: processes `session.status`, `session.created`, `session.updated`, `session.idle`, `session.interrupted`, `session.error`, `command.executed`, `todo.updated`
- **Timer loop** (every 5s): checks busy sessions for stall (>48s idle), hallucination loops (>3 continues/10min), orphan parents
- **Tool-text detection**: on idle, fetches last messages and scans raw text for unexecuted XML/JSON tool calls
- **Exposes `task_complete` tool** — agents call this to signal done (replaces fragile emoji heuristics)
- All logging via `ctx.client.app.log()` — never `console.log`

## Project structure

```
src/
  index.ts                      # plugin source (one file, ~1400 lines)
  index.test.ts                 # unit tests: short(), backoffMs(), getSid(), getError()
  index.integration.test.ts     # integration tests: plugin lifecycle, agent extraction
  index.plugin.test.ts          # plugin unit tests: state tracking, resume logic, tool recovery
  index.coverage.test.ts        # pattern-coverage tests for tool-text detection
  index.it.test.ts              # edge-case tests: state machine, event parsing, abort flow
```

## Key configuration (plugin options)

| Option | Default | Effect |
|---|---|---|
| `chunkTimeoutMs` | 45000 | Inactivity before stall recovery |
| `checkIntervalMs` | 5000 | Timer poll interval |
| `maxRetries` | 3 | Max auto-resume attempts |
| `subagentWaitMs` | 15000 | Wait before orphan-parent recovery |
| `loopMaxContinues` | 3 | Max continues in hallucination window |
| `baseBackoffMs` / `maxBackoffMs` | 1000 / 8000 | Retry backoff range |

## Testing

- `bun test` — runs all `src/*.test.ts` files concurrently
- Tests heavily mock `ctx.client.session.{list,messages,prompt,abort}` — see `createMockContext()` in `index.plugin.test.ts`
- No external services, no fixtures, no snapshot tests
- Real session IDs start with `ses_` but tests use short IDs like `"session-1"`

## Gotchas

- **`bun build` is required before testing** — tests import from `./index`, which resolves via TypeScript source (Bun handles this natively), but the published npm package uses the built `dist/index.js`.
- **Module format**: ESM only (`"type": "module"`). The build target is `bun` — the output uses Bun runtime APIs.
- **Session ID prefix matters**: Production code filters session IDs starting with `"ses_"`. Tests use arbitrary IDs and bypass this check.
- **Backoff differs between prod and tests**: Production uses `baseBackoffMs * 2^(attempt-1)` capped at `maxBackoffMs`; test files independently reimplement backoff with different constants (`5000 * 2^attempt` capped at 160000).
- **Repository is a fork** — origin at `GuramDuka/opencode-auto-resume`. Upstream: `Mte90/opencode-auto-resume`.

## CI / Publishing

GitHub Actions: `npm-publish.yml` triggers on releases + workflow_dispatch.
Steps: `bun install` → `bun run build` → `bun test` → `npm publish --provenance` (needs Node 24).
