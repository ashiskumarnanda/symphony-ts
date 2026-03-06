# Symphony TypeScript Full Implementation Plan

Last updated: 2026-03-06

## 1. Goal

Build a TypeScript implementation of Symphony that targets full spec conformance against `openai/symphony` `main` as of 2026-03-06, not an MVP.

Primary source documents:

- `README.md`: https://github.com/openai/symphony/blob/main/README.md
- `SPEC.md`: https://github.com/openai/symphony/blob/main/SPEC.md
- Elixir reference README: https://github.com/openai/symphony/blob/main/elixir/README.md

## 2. Delivery Standard

We will treat "complete version" as:

1. Implement all `Core Conformance` items from `SPEC.md` Sections 17.1-17.7 and 18.1.
2. Also implement the two practical extensions that the reference implementation already ships:
   - Optional HTTP observability server (`13.7`)
   - Optional `linear_graphql` dynamic tool for Codex app-server sessions (`10.5`, `18.2`)
3. Keep restart persistence of retry queue/session metadata out of scope for the first 24h, because the spec lists it as future work, not conformance.

That gives us a "full spec implementation" for current conformance plus the most important real-world extensions, without inventing non-spec features.

## 3. Required Capability Map

The TS version must include these subsystems:

- CLI host
  - Optional workflow path argument
  - `--port`
  - `--logs-root`
  - Guardrails acknowledgement flag, matching the reference posture
- Workflow/config system
  - `WORKFLOW.md` loader
  - YAML front matter parser
  - strict prompt template handling
  - env/path/default resolution
  - file watching with last-known-good fallback on invalid reload
- Domain + orchestrator
  - single-authority in-memory scheduler state
  - poll loop
  - dispatch sort/claim rules
  - retry queue with continuation retry and exponential backoff
  - reconciliation for stall/terminal/non-active transitions
- Workspace manager
  - deterministic sanitized paths
  - root containment checks
  - hook execution with timeout policy
  - startup terminal cleanup
- Linear adapter
  - candidate fetch with pagination
  - terminal-state fetch
  - state refresh by issue IDs
  - normalization of labels, blockers, timestamps, priority
- Codex app-server client
  - `initialize -> initialized -> thread/start -> turn/start`
  - line-buffered stdout protocol parsing
  - stderr separation
  - timeout/error mapping
  - approval/tool/user-input handling policy
  - token/rate-limit extraction
- Agent runner
  - workspace + prompt + hooks + app-server session loop
  - continuation turns on the same thread
  - post-turn tracker refresh
- Observability
  - structured logs
  - runtime snapshot
  - HTTP dashboard/API
- Test suite
  - deterministic conformance coverage aligned to Sections 17.1-17.7

## 4. TypeScript Architecture

Recommended stack:

- Runtime: Node.js 22 LTS
- Package manager: `pnpm`
- Language: TypeScript 5.x, strict mode
- Test runner: `vitest`
- HTTP server: `fastify`
- File watch: `chokidar`
- YAML: `yaml`
- Template engine: `liquidjs` in strict mode
- Schema validation: `zod`
- Logging: `pino`
- GraphQL transport: native `fetch`

Recommended repository layout:

```text
.
├── src/
│   ├── cli/
│   │   └── main.ts
│   ├── config/
│   │   ├── workflow-loader.ts
│   │   ├── config-resolver.ts
│   │   ├── workflow-watch.ts
│   │   └── defaults.ts
│   ├── domain/
│   │   ├── issue.ts
│   │   ├── run-attempt.ts
│   │   ├── live-session.ts
│   │   ├── retry-entry.ts
│   │   └── orchestrator-state.ts
│   ├── orchestrator/
│   │   ├── orchestrator.ts
│   │   ├── dispatch.ts
│   │   ├── retry-queue.ts
│   │   ├── reconcile.ts
│   │   └── snapshot.ts
│   ├── workspace/
│   │   ├── workspace-manager.ts
│   │   ├── path-safety.ts
│   │   └── hooks.ts
│   ├── tracker/
│   │   ├── tracker.ts
│   │   ├── linear-client.ts
│   │   ├── linear-queries.ts
│   │   ├── linear-normalize.ts
│   │   └── errors.ts
│   ├── codex/
│   │   ├── app-server-client.ts
│   │   ├── protocol.ts
│   │   ├── line-reader.ts
│   │   ├── event-normalizer.ts
│   │   ├── token-accounting.ts
│   │   └── dynamic-tools/
│   │       └── linear-graphql.ts
│   ├── agent/
│   │   ├── agent-runner.ts
│   │   └── prompt-builder.ts
│   ├── observability/
│   │   ├── logger.ts
│   │   ├── dashboard-server.ts
│   │   ├── rest-api.ts
│   │   └── presenter.ts
│   └── index.ts
├── test/
│   ├── config/
│   ├── workspace/
│   ├── tracker/
│   ├── orchestrator/
│   ├── codex/
│   ├── observability/
│   └── fixtures/
└── WORKFLOW.md
```

## 5. Key Design Decisions

### 5.1 Orchestrator Model

Use one in-process orchestrator service object as the sole mutator of runtime state. All worker/app-server events should flow back through an internal queue so state mutations stay serialized.

This is the cleanest TS equivalent to the Elixir single-authority model and is mandatory for claim/idempotency correctness.

### 5.2 Worker Concurrency Model

Use:

- one top-level scheduler tick loop
- one child process for each Codex app-server session
- one async task per worker run attempt
- one timer per retry entry

Do not use a database in v1. The spec explicitly allows in-memory recovery driven by tracker state and filesystem.

### 5.3 Strict Prompt Rendering

Use `liquidjs` configured to fail on:

- unknown variables
- unknown filters

This is required by the prompt contract and prevents silent drift between workflow intent and runtime behavior.

### 5.4 Safety Model

Default posture should match the reference implementation's high-trust preview stance:

- explicit CLI acknowledgement required before startup
- agent cwd must equal the issue workspace
- workspace path must be inside configured root
- workspace key sanitization enforced everywhere
- hook output truncated in logs
- secrets never logged

### 5.5 Dynamic Tool Extension

Implement `linear_graphql` as a narrow client-side tool exposed only when:

- `tracker.kind === "linear"`
- valid auth is present

Unsupported tool calls must fail in-session without stalling the turn.

## 6. 24-Hour Execution Plan

This only works if we optimize for vertical slices and parallel-safe module boundaries.

### Phase 0: First 2 hours

- Initialize repo tooling:
  - `package.json`
  - `tsconfig.json`
  - `vitest`
  - lint/format scripts
- Create the domain types and config schemas first.
- Freeze runtime defaults directly from `SPEC.md`.
- Add fixtures for:
  - workflow parsing
  - Linear GraphQL payloads
  - Codex protocol transcripts

Exit criteria:

- Project boots
- Tests run
- Config/domain types compile

### Phase 1: Hours 2-6

- Implement workflow loader, config resolver, path/env/default coercion.
- Implement `WORKFLOW.md` watcher with last-known-good semantics.
- Implement prompt builder with strict Liquid rendering.
- Implement workspace manager and hook runner.

Exit criteria:

- Sections 17.1 and 17.2 mostly green

### Phase 2: Hours 6-10

- Implement Linear adapter:
  - candidate fetch
  - state refresh
  - terminal-state fetch
  - pagination
  - normalization
- Build test coverage from canned GraphQL responses.

Exit criteria:

- Section 17.3 green

### Phase 3: Hours 10-15

- Implement Codex app-server client:
  - spawn
  - startup handshake
  - stdout JSON line buffering
  - stderr handling
  - turn completion/failure detection
  - timeout mapping
  - token/rate-limit extraction
- Implement dynamic tool plumbing and `linear_graphql`.

Exit criteria:

- Section 17.5 green from transcript-driven tests

### Phase 4: Hours 15-20

- Implement orchestrator core:
  - state object
  - claim/dispatch logic
  - retry scheduling
  - continuation retry after normal exit
  - reconciliation
  - stall handling
  - startup terminal cleanup
- Implement agent runner loop around workspace + prompt + codex session.

Exit criteria:

- Section 17.4 green

### Phase 5: Hours 20-22

- Implement structured logging, snapshot builder, REST API, and simple HTML dashboard.
- Wire `--port` and `server.port` precedence.

Exit criteria:

- Section 17.6 complete
- HTTP extension usable

### Phase 6: Hours 22-24

- Full conformance sweep against Sections 17 and 18.
- Run end-to-end smoke test with a local fake tracker + fake codex server.
- If credentials are available, run the real integration profile for Linear.
- Produce `README`, sample `WORKFLOW.md`, and operational notes.

Exit criteria:

- All core conformance tests passing
- Extension tests passing
- Manual smoke test evidence captured

## 7. Build Order by Critical Path

If we need to sequence tightly, this is the dependency order:

1. Types + config schema
2. Workflow loader + prompt engine
3. Workspace safety + hooks
4. Linear tracker client
5. Codex app-server client
6. Agent runner
7. Orchestrator
8. Observability surfaces
9. CLI wiring
10. End-to-end validation

Reason: the orchestrator depends on stable contracts from config, tracker, workspace, and codex layers. Building it earlier increases rework.

## 8. Test Strategy

We should mirror the spec instead of inventing ad hoc tests.

### 8.1 Required automated suites

- `test/config/*`
  - workflow path precedence
  - YAML parse errors
  - `$VAR` resolution
  - reload keeps last good config
- `test/workspace/*`
  - sanitization
  - root containment
  - hook timeout/failure behavior
- `test/tracker/*`
  - pagination
  - normalization
  - error mapping
- `test/codex/*`
  - startup handshake
  - partial-line buffering
  - approval/input-required/tool-call handling
  - token/rate-limit extraction
- `test/orchestrator/*`
  - dispatch order
  - blocker gating
  - retry backoff
  - stall termination
  - terminal cleanup
- `test/observability/*`
  - snapshot correctness
  - API payload shape

### 8.2 Integration harnesses

- Fake Linear HTTP server
- Fake Codex app-server subprocess
- Temporary workspace root
- Golden transcript fixtures for session/event flows

### 8.3 Real integration

Run only when `LINEAR_API_KEY` is available and explicitly enabled.

## 9. Risks

### Risk 1: Codex app-server protocol drift

The spec is logical, not bound to one exact payload version. We need a tolerant parser and transcript fixtures for multiple equivalent payload shapes.

### Risk 2: Linear schema drift

The spec explicitly warns that Linear GraphQL details can drift. Keep query construction isolated and test normalization separately from transport.

### Risk 3: Concurrency bugs in JS runtime

Node is single-threaded, but async timers and worker exits can still interleave badly. Keep every state mutation behind one orchestrator event queue.

### Risk 4: Hook safety

Hooks are trusted shell scripts and can hang or flood logs. Enforce timeout, cwd, output truncation, and explicit event logging.

### Risk 5: Scope creep from “complete version”

Do not expand into non-spec features such as persistent scheduler DB, tracker write APIs in orchestrator, or multi-tracker plugins during the first 24h.

## 10. Done Definition

The TypeScript port is done when all are true:

1. Every Section 18.1 item is implemented.
2. The HTTP server extension and `linear_graphql` extension are implemented.
3. All conformance-oriented tests for Sections 17.1-17.7 pass.
4. The service can:
   - load `WORKFLOW.md`
   - poll Linear
   - create per-issue workspaces
   - launch Codex app-server
   - continue turns on the same thread
   - retry/reconcile correctly
   - expose runtime state over logs and HTTP
5. A local end-to-end run shows one issue moving through a real orchestration cycle.

## 11. Immediate Next Step

Start implementation by scaffolding the repo and locking the contracts first:

1. Initialize the TS project and strict tooling.
2. Encode the spec defaults and typed config schema.
3. Add failing tests copied directly from the conformance checklist categories.

This is the fastest path to a full version in 24 hours without accidentally drifting back to an MVP.
