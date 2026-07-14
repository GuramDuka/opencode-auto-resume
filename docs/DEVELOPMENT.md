# Development Guide

## Quick Reference

```bash
bun install             # install dependencies (migrated from npm)
bun run build           # build src/index.ts → dist/index.js (Bun target)
bun test                # run all 125 tests concurrently across 5 files
bun run dev             # watch mode rebuild
```

## Project Layout

```
opencode-auto-resume/
├── src/
│   ├── index.ts                   # Plugin source (1392 lines, single file)
│   ├── index.test.ts              # Unit tests: short(), backoffMs(), getSid(), getError()
│   ├── index.integration.test.ts  # Integration tests: lifecycle, agent extraction, prompt flow
│   ├── index.plugin.test.ts       # Plugin unit tests: state tracking, resume, recovery, todos
│   ├── index.coverage.test.ts     # Pattern-coverage tests: tool-text detection patterns
│   └── index.it.test.ts           # Edge-case tests: state machine, event parsing, abort flow
├── dist/
│   └── index.js                   # Built output (not committed, in .gitignore)
├── docs/
│   ├── ARCHITECTURE.md            # Architecture deep-dive
│   ├── RECOVERY-MECHANISMS.md     # Recovery mechanism details
│   └── DEVELOPMENT.md             # This file
├── AGENTS.md                      # Agent guidance for OpenCode sessions
├── README.md                      # User-facing install, config, troubleshooting
├── package.json
├── tsconfig.json
├── bun.lock
└── .github/workflows/
    └── npm-publish.yml            # CI: publish to npm on releases
```

## Test Files — What Each Covers

| File | Tests | Scope |
|------|-------|-------|
| `index.test.ts` | 4 describe blocks | Pure functions: `short()`, `backoffMs()`, `getSid()`, `getError()` |
| `index.integration.test.ts` | 5 describe blocks | Plugin lifecycle, agent extraction, prompt API calling, realistic scenarios |
| `index.plugin.test.ts` | 13 describe blocks | State tracking, resume logic, tool-text recovery, loop detection, continue lock, idle flags, todo guard, task_complete, 🎉 detection, done claims |
| `index.coverage.test.ts` | 1 describe block | Verifies all tool-text patterns match expected XML/JSON formats |
| `index.it.test.ts` | 10 describe blocks | State machine, event parsing, hallucination loop, agent preservation, resume flow, abort flow, edge cases |

### Running Specific Tests

```bash
bun test src/index.test.ts         # just the unit tests
bun test src/index.plugin.test.ts  # just the plugin unit tests
bun test -t "Tool Text"            # run tests matching "Tool Text" in name
bun test -t "Resume"               # run tests matching "Resume" in name
```

## Testing Patterns

All tests use `bun:test` (Bun's built-in test runner). The project has no Jest, Vitest, or other test framework.

### Mock Context Pattern

Tests create mock OpenCode context objects. The most complete one is in `index.integration.test.ts`:

```typescript
const createRealisticContext = () => {
  const realSessions: Session[] = [/* ... */]
  const realMessages: Map<string, Message[]> = new Map()
  return {
    client: {
      app: { log: mock(() => {}) },
      session: {
        list: mock(async () => ({ data: realSessions })),
        messages: mock(async (path) => realMessages.get(path.id) ?? []),
        prompt: mock(async (config) => { promptCalls.push(config); return {} }),
        abort: mock(async () => ({})),
      },
    },
  }
}
```

The simpler mock in `index.plugin.test.ts` is also useful as a starting point for new tests.

### Test Gotchas

- **Test session IDs are plain strings** (`"session-1"`), unlike production which filters for `ses_` prefix
- **Backoff constants differ**: Tests reimplement backoff as `5000 * 2^attempt` capped at `160000` — production uses `baseBackoffMs * 2^(attempt-1)` capped at `maxBackoffMs`
- **Mock ordering**: `ctx.client.session.messages` can be set up to return different data for different sessions
- **Timers in tests**: The plugin uses `setInterval`/`setTimeout` internally. Tests generally avoid waiting by directly calling event handlers or checking state rather than waiting for timer ticks
- **No external services**: All tests are purely in-memory
- **Model filter tests**: When testing `modelFilter`, mock `ctx.client.session.messages` to return messages with a `model` field containing `{ providerID, modelID }`. The filter regex is tested against `providerID/modelID`. Pass `modelFilter: "anthropic/.*"` in options to test filtering.

## Adding a New Recovery Mechanism

1. Add fields to `SessionWatch` interface for state tracking
2. Add the detection logic in the timer loop (`startTimer()`) or event handler (`handleEvent()`)
3. Create a dedicated recovery function (like `tryResume`, `tryAbortAndResume`, `checkForToolCallAsText`)
4. Add recovery prompts as module-level constants
5. Write tests:
   - Unit tests for detection logic
   - Integration tests for the recovery flow
   - Edge-case tests for guard conditions

## Adding a New Event Handler

1. Add a `case` to the `switch (type)` in `handleEvent()`
2. Extract the session ID via `getSid(ev)`
3. Update `SessionWatch` state as needed
4. Write integration tests that fire the event through `hooks.event()`

## CI Pipeline

The GitHub Actions workflow (`npm-publish.yml`) runs on:

- **Release creation** (GitHub Releases page)
- **Manual trigger** (`workflow_dispatch`)

Steps:
```
bun install → bun run build → bun test → npm publish --provenance
```

The publish step uses Trusted Publishing (OIDC) and requires Node 24. The build and test steps use Bun.

### npm Package

Published as `opencode-auto-resume` on npm.

**Published files**: `dist/index.js`, `README.md`, `LICENSE`

## Module System

- **Format**: ESM (`"type": "module"` in package.json)
- **Build target**: `bun` (output uses Bun runtime APIs)
- **Entry point**: `dist/index.js` (built artifact)
- **TypeScript config**: Target ESNext, module bundler resolution, strict mode

## Dependencies

**Runtime**:
- `@opencode-ai/plugin` — plugin interface and `tool()` helper

**Dev**:
- `@opencode-ai/sdk` — SDK types (used in integration tests)
- `@types/bun` — Bun type definitions
- `typescript` — TypeScript compiler

## Repository

This is a **fork**. The upstream is [Mte90/opencode-auto-resume](https://github.com/Mte90/opencode-auto-resume). The current origin is at [GuramDuka/opencode-auto-resume](https://github.com/GuramDuka/opencode-auto-resume).

## Troubleshooting

### Build fails with "Could not resolve" errors
Run `bun install` first. The project once used npm but now uses Bun's lockfile.

### Tests fail on first run
Make sure `bun install` ran and created `bun.lock`. Tests import from `./index` which resolves via TypeScript source — Bun handles this natively.

### "Source code does not match any known pattern" when editing
The source uses 4-space indentation, not tabs. There's no formatter or linter configured — make sure your edits match the existing style.
