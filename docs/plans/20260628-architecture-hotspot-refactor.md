# Architecture Hotspot Refactor Plan

## Overview

Refactor the highest-ROI architecture hotspots found in the 2026-06-28 architecture review:

1. Versioned hook/session state-file contracts.
2. A live-session read contract over the volatile session core.
3. A smaller topic-creation flow by extracting draft, picker, and launch seams.
4. An injectable polling runtime bundle to reduce global-state/import-order risk.
5. A smaller Claude transcript parser core.

This plan is executable by Ralphex from the default plan directory:

```bash
ralphex docs/plans/20260628-architecture-hotspot-refactor.md
```

Ralphex parses checklist items as work items. Keep checkboxes only under `### Task N:` sections.

## Source artifact

Review/design sources:

- Session architecture review, 2026-06-28: scores and findings from the architecture review conversation.
- Existing review: `docs/architecture-review/2026-06-27-ccgram-full.md`.
- Target-state design: `docs/architecture-design/2026-05-23-ccgram-target.md`.
- Current architecture map: `docs/ai-agents/architecture-map.md`.
- Authoritative module inventory and invariants: `.claude/rules/architecture.md`.
- Enforced gates: `Makefile` targets `arch-guard`, `arch-check`, `check`.

Finding/risk IDs used by this plan:

- `R1` — Hook/session file contracts are load-bearing and schema-light.
- `R2` — Live-session state remains the highest-value coupling hotspot.
- `R3` — `directory_callbacks.py` concentrates too many topic-creation subflows.
- `R4` — Polling state uses module-level runtime singletons; import-order discipline is load-bearing.
- `R5` — `TranscriptParser.parse_entries` has high cyclomatic complexity and provider-format volatility.

## Scope

In scope:

- Production code under `src/ccgram/` for the five hotspots above.
- Unit, integration, and architecture-fitness tests that prove behavior and boundaries.
- Relevant docs: `docs/ai-agents/architecture-map.md`, `.claude/rules/architecture.md`, `docs/architecture.md`, and this plan.
- `.archfit.yaml` only for new advisory/fitness rules that make the new boundaries measurable.

Out of scope:

- No physical split of `WindowStateStore` or persisted `state.json` beyond compatible additive fields.
- No full handler rewrite.
- No Mini App frontend refactor except API-call-site updates forced by the new read contracts.
- No change to the Telegram topic/window identity model.
- No removal of lazy imports unless the replacement dependency direction is proven by tests.

## Success criteria

- Hook event and session-map files have explicit versioned parse/serialize contracts with backward-compatible readers.
- Session lifecycle consumers use a narrow read projection instead of reaching into broad session/task internals.
- Topic creation keeps `directory_callbacks.py` as a dispatcher; draft state and window launch sequencing have named contracts.
- Polling code can run against an injected `PollingRuntime`; module-level singletons remain only as the compatibility default.
- Claude transcript parsing is split into small event/message handlers without changing emitted `ParsedMessage` / `AgentMessage` behavior.
- Boundary and schema checks are executable in tests or archfit; no purely-prose architecture rule is introduced.
- `make arch-guard`, `make check`, and `make arch-check` pass or record an accepted advisory-only archfit warning.
- `npx gitnexus detect-changes --scope all --repo ccgram` shows affected flows matching the task scope before each task commit.

## Development approach

- Complete one `### Task N:` section per Ralphex iteration. Do not start the next task until the current task is green.
- Add or tighten tests before changing behavior-bearing code.
- Keep each task independently committable.
- Preserve public behavior unless a task explicitly names the behavior change.
- Before editing any production function, class, or method, run GitNexus impact analysis for that symbol:
  `npx gitnexus impact <symbol> --direction upstream --depth 3 --include-tests --repo ccgram`.
- If GitNexus reports HIGH or CRITICAL risk, state it in the task log before editing and keep changes inside the listed files.
- Before each task commit, run:
  `npx gitnexus detect-changes --scope all --repo ccgram`.
- Prefer new ports/contracts over broad rewrites.
- Keep compatibility adapters during migrations; remove only after callers have moved and tests prove the boundary.

## Validation commands

Focused commands appear in each task. Whole-plan final gate:

```bash
make arch-guard
make check
make arch-check
uv run pytest tests/integration/test_import_no_cycles.py -q
npx gitnexus detect-changes --scope all --repo ccgram
```

Useful focused gates:

```bash
uv run ruff format --check src/ tests/
uv run ruff check src/ tests/
uv run pyright src/ccgram/ tests/
uv run deptry src
uv run python scripts/lint_lazy_imports.py
```

## Implementation steps

### Task 1: Version hook event and session-map file contracts

Justification: `R1`. `hook.py` writes `events.jsonl` and `session_map.json`; `event_reader.py`, `session_map.py`, `session_monitor.py`, and lifecycle code consume them. The seam crosses a short-lived hook process and the long-lived bot process, so drift must fail clearly or degrade in a controlled backward-compatible way.

Files:

- Create: `src/ccgram/hooks/state_files.py`
- Modify: `src/ccgram/hook.py`
- Modify: `src/ccgram/event_reader.py`
- Modify: `src/ccgram/session_map.py`
- Modify: `src/ccgram/monitor_events.py` only if event dataclasses need a version-aware constructor
- Create: `tests/ccgram/hooks/test_state_files.py`
- Modify: `tests/ccgram/test_hook.py`
- Modify: `tests/ccgram/test_session_map_primary.py`
- Modify: `tests/ccgram/test_session_monitor.py` only for changed construction helpers
- Modify: `docs/ai-agents/architecture-map.md`
- Modify: `.claude/rules/architecture.md`

Preconditions:

- Current `events.jsonl` and `session_map.json` readers support legacy files with no `schema_version`.
- No source writes outside the files listed above without updating this task.

Postconditions:

- One module owns state-file schemas, parse, serialize, validation, and backward compatibility.
- `hook.py` uses the serializer on write.
- `event_reader.py` and `session_map.py` use parsers on read.
- Versionless existing records still parse as schema v1.
- Invalid records are ignored with a logged reason, not treated as empty success.

Fitness gate:

- Add tests that fail on missing required fields for new-version records.
- Add tests that accept legacy versionless fixtures.
- Add a structural test, if simple, that production reads/writes of `events.jsonl` and `session_map.json` go through `hooks.state_files` except for path discovery.

Impact commands:

- `npx gitnexus impact _write_event --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact _update_session_map --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact read_new_events --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact parse_session_map --direction upstream --depth 3 --include-tests --repo ccgram`

Verification commands:

```bash
uv run pytest tests/ccgram/hooks/test_state_files.py tests/ccgram/test_hook.py tests/ccgram/test_session_map_primary.py tests/ccgram/test_session_monitor.py -q
uv run pyright src/ccgram/ tests/ccgram/hooks tests/ccgram/test_hook.py tests/ccgram/test_session_map_primary.py
make arch-guard
npx gitnexus detect-changes --scope all --repo ccgram
```

Manual checks:

- Inspect a freshly written `events.jsonl` line and `session_map.json` entry in a temp `$CCGRAM_DIR`; verify each has the expected schema version and no raw prompt/tool payload.

- [ ] Add `hooks.state_files` with version constants, `EventLogRecord`, `SessionMapEntry`, parse helpers, serialize helpers, and validation errors.
- [ ] Add legacy fixtures for current versionless `events.jsonl` and `session_map.json` records.
- [ ] Route `hook._write_event` and `hook._update_session_map` through the serializers while preserving file locking and corrupt-file backup behavior.
- [ ] Route `event_reader.read_new_events` through the event parser and keep offset handling unchanged.
- [ ] Route `session_map.read_session_map_raw` / `parse_session_map` through session-map parsing without changing backend-neutral key matching.
- [ ] Add tests for valid v1, valid legacy, missing required fields, malformed JSON, unknown future version, and extra ignored fields.
- [ ] Update architecture docs to name `hooks.state_files` as the state-file contract owner.
- [ ] Run the verification commands and mark this task complete only when all pass.

### Task 2: Add a live-session read contract for volatile session state

Justification: `R2`. The live-session core is the highest-value coupling hotspot. Consumers need task/session/transcript state, not the full shape of `SessionManager`, `SessionLifecycle`, `SessionMapSync`, `ClaudeTaskState`, or monitor internals.

Files:

- Create: `src/ccgram/session_state_ports/__init__.py`
- Create: `src/ccgram/session_state_ports/live_session_state.py`
- Modify: `src/ccgram/session.py`
- Modify: `src/ccgram/session_lifecycle.py`
- Modify: `src/ccgram/session_monitor.py`
- Modify: `src/ccgram/claude_task_state.py` only to expose stable projection construction if needed
- Modify: `src/ccgram/handlers/messaging_pipeline/message_routing.py`
- Modify: `src/ccgram/handlers/messaging_pipeline/tool_batch.py` only if it reads task internals
- Modify: `src/ccgram/handlers/status/status_bubble.py` only if it reads task/session internals
- Modify: `src/ccgram/miniapp/api/transcript.py` only if it reads session internals
- Create: `tests/ccgram/session_state_ports/test_live_session_state.py`
- Modify: `tests/ccgram/test_session.py`
- Modify: `tests/ccgram/test_session_monitor.py`
- Modify: `tests/ccgram/test_query_layer_only_for_handlers.py` if the allowed read surface changes
- Modify: `.claude/rules/architecture.md`
- Modify: `docs/ai-agents/architecture-map.md`

Preconditions:

- Task 1 is merged, so session-map reads have a single contract.
- No persistence schema split is attempted.

Postconditions:

- Consumers read `LiveSessionSnapshot` / small projection functions for session id, cwd, provider name, transcript path, lifecycle/task summary, and last activity as needed.
- Writes still go through existing coordinating facades.
- Handler read-path tests still forbid direct broad `session_manager` reads.

Fitness gate:

- Extend or add an AST audit that forbids handlers and Mini App routes from importing `session_lifecycle`, `session_map_sync`, or `claude_task_state` directly for reads when a `session_state_ports` projection exists.
- Keep exceptions explicit and documented.

Impact commands:

- `npx gitnexus impact SessionManager --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact SessionMonitor --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact SessionLifecycle --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact ClaudeTaskState --direction upstream --depth 3 --include-tests --repo ccgram`

Verification commands:

```bash
uv run pytest tests/ccgram/session_state_ports/test_live_session_state.py tests/ccgram/test_session.py tests/ccgram/test_session_monitor.py tests/ccgram/test_query_layer_only_for_handlers.py -q
uv run pytest tests/ccgram/handlers/messaging_pipeline tests/ccgram/handlers/status -q
uv run pyright src/ccgram/ tests/ccgram/session_state_ports
make arch-guard
npx gitnexus detect-changes --scope all --repo ccgram
```

Manual checks:

- Review the final diff and confirm consumers know projection fields, not internal dictionaries or lifecycle caches.

- [ ] Add `session_state_ports.live_session_state` with frozen projection dataclasses and small read functions.
- [ ] Implement projections as thin adapters over existing session, lifecycle, map, and task-state owners.
- [ ] Migrate one consumer group at a time: message routing first, status/tool UI second, Mini App transcript route last.
- [ ] Keep public write/admin methods on `SessionManager`; do not duplicate save scheduling.
- [ ] Add projection tests for missing window/session, hookless provider, stale transcript path, last activity, and task-summary defaults.
- [ ] Add/extend architecture audit tests for direct session-core read imports.
- [ ] Update docs to name `session_state_ports` as the live-session read seam.
- [ ] Run the verification commands and mark this task complete only when all pass.

### Task 3: Split topic creation into draft, picker, and launch seams

Justification: `R3`. `handlers/topics/directory_callbacks.py` handles navigation, stale guards, worktree flow, workspace picker, provider/mode selection, and tmux/herdr launch sequencing. This keeps strongly related user flow close, but the file is too broad and hard to change safely.

Files:

- Modify: `src/ccgram/handlers/topics/directory_callbacks.py`
- Create: `src/ccgram/handlers/topics/topic_creation_draft.py`
- Create: `src/ccgram/handlers/topics/provider_mode_callbacks.py`
- Create: `src/ccgram/handlers/topics/workspace_callbacks.py`
- Create: `src/ccgram/handlers/topics/window_launch_service.py`
- Modify: `src/ccgram/handlers/topics/directory_browser.py` only for draft helper integration
- Modify: `src/ccgram/handlers/topics/topic_orchestration.py` only where launch sequencing already lives there
- Modify: `tests/ccgram/handlers/topics/test_directory_callbacks.py`
- Create: `tests/ccgram/handlers/topics/test_topic_creation_draft.py`
- Create: `tests/ccgram/handlers/topics/test_window_launch_service.py`
- Modify: `tests/ccgram/handlers/topics/test_workspace_picker.py`
- Modify: `tests/ccgram/handlers/topics/test_worktree.py`
- Modify: `tests/ccgram/handlers/topics/test_pending_creation_race.py`
- Modify: `docs/ai-agents/codebase-index.md`
- Modify: `.claude/rules/architecture.md`

Preconditions:

- Current `/new` and directory browser behavior is characterized by existing tests.
- Task 2 does not need to be complete unless this task reads the new live-session projection.

Postconditions:

- `directory_callbacks.py` is a thin callback dispatcher and stale-flow guard.
- `TopicCreationDraft` owns `context.user_data` keys for pending thread/text, selected directory, worktree, workspace, provider, and launch mode.
- Provider/mode selection, workspace selection, and launch service are separately testable.
- Window launch sequencing remains centralized and atomic from the caller perspective.

Fitness gate:

- Add or extend a handler layering test that prevents new topic-creation submodules from importing concrete multiplexer backends or raw window state.
- Existing `test_multiplexer_boundary.py`, `test_window_store_import_boundary.py`, and query-layer tests must pass.

Impact commands:

- `npx gitnexus impact handle_directory_callback --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact _handle_provider_select --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact _handle_mode_select --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact create_worktree --direction upstream --depth 3 --include-tests --repo ccgram`

Verification commands:

```bash
uv run pytest tests/ccgram/handlers/topics/test_directory_callbacks.py tests/ccgram/handlers/topics/test_topic_creation_draft.py tests/ccgram/handlers/topics/test_window_launch_service.py tests/ccgram/handlers/topics/test_workspace_picker.py tests/ccgram/handlers/topics/test_worktree.py tests/ccgram/handlers/topics/test_pending_creation_race.py -q
uv run pytest tests/ccgram/test_multiplexer_boundary.py tests/ccgram/test_window_store_import_boundary.py tests/ccgram/test_query_layer_only_for_handlers.py -q
uv run pyright src/ccgram/ tests/ccgram/handlers/topics
make arch-guard
npx gitnexus detect-changes --scope all --repo ccgram
```

Manual checks:

- Run a local bot smoke test for `/new`: normal cwd, dirty git repo, new worktree, workspace picker, provider picker, cancel, and stale callback.

- [ ] Add `TopicCreationDraft` with typed accessors over `context.user_data`; keep all existing key names during migration.
- [ ] Move stale-thread/topic validation to one draft-aware helper.
- [ ] Extract provider/mode callback handling into `provider_mode_callbacks.py` without changing callback data values.
- [ ] Extract workspace callback handling into `workspace_callbacks.py` without changing herdr/tmux capability gates.
- [ ] Extract create-window/bind/persist/forward-pending-text sequencing into `window_launch_service.py` with `WindowLaunchRequest` and `WindowLaunchResult`.
- [ ] Keep `directory_callbacks.py` dispatching to the extracted modules; no behavior changes beyond internal structure.
- [ ] Add tests for draft cleanup, stale callback fail-closed behavior, launch success, launch partial failure, and pending text forwarding.
- [ ] Update docs/indexes so future topic-creation edits go to the new modules.
- [ ] Run the verification commands and mark this task complete only when all pass.

### Task 4: Introduce an injectable polling runtime bundle

Justification: `R4`. Polling already has a good pure-decision kernel, but stateful strategies live as module-level singletons in `polling_state.py`. A runtime bundle lowers global coupling and makes tests more explicit without changing polling behavior.

Files:

- Create: `src/ccgram/handlers/polling/polling_runtime.py`
- Modify: `src/ccgram/handlers/polling/polling_state.py`
- Modify: `src/ccgram/handlers/polling/polling_coordinator.py`
- Modify: `src/ccgram/handlers/polling/periodic_tasks.py`
- Modify: `src/ccgram/handlers/polling/window_tick/__init__.py`
- Modify: `src/ccgram/handlers/polling/window_tick/observe.py`
- Modify: `src/ccgram/handlers/polling/window_tick/apply.py`
- Modify: `src/ccgram/bootstrap.py` only if runtime wiring belongs at startup
- Create: `tests/ccgram/handlers/polling/test_polling_runtime.py`
- Modify: `tests/ccgram/handlers/polling/test_polling_strategies.py`
- Modify: `tests/ccgram/handlers/polling/test_window_tick.py`
- Modify: `tests/ccgram/handlers/polling/test_status_polling.py`
- Modify: `tests/ccgram/handlers/polling/test_polling_types_purity.py` only if imports change
- Modify: `.claude/rules/architecture.md`
- Modify: `docs/ai-agents/architecture-map.md`

Preconditions:

- Current polling tests pass.
- No behavior-bearing status-transition change is bundled into this refactor.

Postconditions:

- `PollingRuntime` owns `TerminalPollState`, `TerminalScreenBuffer`, `InteractiveUIStrategy`, `TopicLifecycleStrategy`, and `PaneStatusStrategy`.
- Module-level objects remain as a default runtime adapter for existing callers during migration.
- `tick_window`, observe, apply, periodic tasks, and coordinator accept an explicit runtime or use the default.
- `polling_types.py` stays pure.

Fitness gate:

- Extend `test_polling_types_purity.py` if needed so pure types still import only stdlib + provider status contract.
- Add a test proving `PollingRuntime()` can be constructed in isolation and reset without touching the default singleton runtime.

Impact commands:

- `npx gitnexus impact PaneStatusStrategy --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact TerminalPollState --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact tick_window --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact status_poll_loop --direction upstream --depth 3 --include-tests --repo ccgram`

Verification commands:

```bash
uv run pytest tests/ccgram/handlers/polling/test_polling_runtime.py tests/ccgram/handlers/polling/test_polling_strategies.py tests/ccgram/handlers/polling/test_window_tick.py tests/ccgram/handlers/polling/test_status_polling.py tests/ccgram/handlers/polling/test_polling_types_purity.py -q
uv run pytest tests/integration/test_import_no_cycles.py -q
uv run python scripts/lint_lazy_imports.py
uv run pyright src/ccgram/ tests/ccgram/handlers/polling
make arch-guard
npx gitnexus detect-changes --scope all --repo ccgram
```

Manual checks:

- Local bot smoke test: status emoji, typing indicator, live view tick, pane prompt alert, dead-window banner.

- [ ] Add `PollingRuntime` dataclass/factory with owned strategy instances and a reset method.
- [ ] Replace direct module-level singleton access inside coordinator/window_tick paths with runtime attributes.
- [ ] Keep compatibility exports in `polling_state.py` for existing callers; route them to the default runtime.
- [ ] Update tests to construct isolated runtimes instead of mutating global state where practical.
- [ ] Confirm `polling_types.py` remains pure and `window_tick/decide.py` remains side-effect free.
- [ ] Update docs to describe `PollingRuntime` as the stateful polling owner.
- [ ] Run the verification commands and mark this task complete only when all pass.

### Task 5: Split Claude transcript parsing into small handlers

Justification: `R5`. `TranscriptParser.parse_entries` is the highest-complexity parser hotspot and sits on volatile external transcript formats. The goal is a behavior-preserving split with table-driven tests, not a new transcript model.

Files:

- Modify: `src/ccgram/transcript_parser.py`
- Create: `src/ccgram/transcript_events.py` only if helper functions need a cohesive home; otherwise keep private helpers in `transcript_parser.py`
- Modify: `src/ccgram/providers/claude.py` only if it can call a clearer parser surface without behavior change
- Modify: `tests/ccgram/test_transcript_parser.py`
- Modify: `tests/ccgram/test_terminal_parser.py` only if shared fixtures move
- Modify: `tests/ccgram/providers/test_claude.py`
- Modify: `docs/ai-agents/codebase-index.md`

Preconditions:

- Existing transcript parser tests pass.
- Representative Claude JSONL fixtures exist or are added before changing parser logic.

Postconditions:

- `parse_entries` delegates to small handlers by entry/message/tool type.
- Pending-tool state transitions are explicit and covered by tests.
- Public parser behavior is unchanged for assistant/user/tool use/tool result/thinking/history/task-update entries.

Fitness gate:

- Add regression tests for each parsed entry kind before refactoring.
- Keep `radon`/complexity improvement as advisory; do not gate on an exact number unless the project already supports it.

Impact commands:

- `npx gitnexus impact TranscriptParser --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact parse_entries --direction upstream --depth 3 --include-tests --repo ccgram`
- `npx gitnexus impact ClaudeProvider.parse_transcript_entries --direction upstream --depth 3 --include-tests --repo ccgram`

Verification commands:

```bash
uv run pytest tests/ccgram/test_transcript_parser.py tests/ccgram/providers/test_claude.py -q
uv run pytest tests/ccgram/test_session_monitor.py tests/ccgram/handlers/messaging_pipeline -q
uv run pyright src/ccgram/ tests/ccgram/test_transcript_parser.py
uv run ruff check src/ccgram/transcript_parser.py tests/ccgram/test_transcript_parser.py
npx gitnexus detect-changes --scope all --repo ccgram
```

Manual checks:

- Compare a small real Claude transcript before/after with a throwaway script or focused test fixture; emitted message count and key fields should match.

- [ ] Add table-driven parser tests for every current entry/message kind handled by `parse_entries`.
- [ ] Extract handlers for user text, assistant text, tool use, tool result, thinking/history, task update, and unknown/ignored entries.
- [ ] Make pending-tool state updates explicit in helper return values or a tiny local state object.
- [ ] Keep `TranscriptParser.parse_entries` as the public entry point.
- [ ] Run existing session-monitor and message-pipeline tests to catch downstream formatting drift.
- [ ] Update docs/indexes if the parser helper module or edit locations changed.
- [ ] Run the verification commands and mark this task complete only when all pass.

### Task 6: Final verification, docs, and re-review handoff

Justification: Every architecture refactor needs a final evidence pass. This task proves the new seams are enforced, docs are current, and GitNexus/archfit see the expected scope.

Files:

- Modify: `docs/ai-agents/architecture-map.md`
- Modify: `docs/ai-agents/codebase-index.md`
- Modify: `.claude/rules/architecture.md`
- Modify: `docs/architecture.md` if module inventory or generated architecture map is maintained manually
- Modify: `.archfit.yaml` only if new module boundaries or rules were added in earlier tasks
- Modify: `docs/plans/20260628-architecture-hotspot-refactor.md`

Preconditions:

- Tasks 1–5 are merged or explicitly marked deferred with rationale.

Postconditions:

- Docs describe the new owners and edit locations.
- All task success criteria are either complete or explicitly deferred.
- Final validation passes.
- A scoped follow-up `architecture-review` is recommended.

Fitness gate:

- `make arch-guard` passes.
- `make arch-check` passes or exits only with accepted advisory warnings.
- `tests/integration/test_import_no_cycles.py` passes.

Impact commands:

- `npx gitnexus detect-changes --scope all --repo ccgram`
- If the new ports/runtimes are indexed, run focused impacts for `EventLogRecord`, `LiveSessionSnapshot`, `WindowLaunchRequest`, `PollingRuntime`, and `TranscriptParser.parse_entries` before final commit.

Verification commands:

```bash
uv run ruff format --check src/ tests/
uv run ruff check src/ tests/
uv run pyright src/ccgram/ tests/
uv run deptry src
uv run python scripts/lint_lazy_imports.py
uv run pytest tests/ -m "not integration and not e2e" --tb=short -v --timeout=30
uv run pytest tests/integration/ -m "not llm" --tb=short -v --timeout=30
make arch-guard
make arch-check
npx gitnexus detect-changes --scope all --repo ccgram
```

Manual checks:

- Smoke-test a real bot session: `/new`, provider selection, worktree creation, normal text send, hook event display, status polling, live view, `/last`, Mini App transcript route, and one Claude transcript update.
- Run a scoped `architecture-review` for: hook contracts, session-state ports, topic creation, polling runtime, transcript parser.

- [ ] Update docs to name all new contracts and edit locations.
- [ ] Update `.archfit.yaml` module labels/rules only for boundaries that now have executable checks.
- [ ] Re-run whole-plan validation commands.
- [ ] Run GitNexus detect-changes and record affected symbols/flows in this task log.
- [ ] Mark completed task checkboxes and leave any deferred items with a reason.
- [ ] Record the re-review recommendation and stop; implementation follow-up belongs to a new plan after re-review.

## Acceptance criteria

- Every completed task has code, tests, and docs updated together.
- No task broadens a public facade without adding or updating a boundary test.
- Runtime behavior is preserved except for clearer validation/failure handling around newly versioned state-file records.
- The final gate passes or has a documented advisory-only archfit warning.
- The final GitNexus detect-changes output matches the plan scope.

## Safety notes

- State-file parsing changes are compatibility-sensitive. Never delete or rewrite user `session_map.json` / `events.jsonl` during tests except under a temp `$CCGRAM_DIR`.
- Topic creation and polling changes are user-visible. Keep compatibility adapters until tests and smoke checks pass.
- Do not split persistence or introduce a migration unless a task is explicitly rewritten and approved.
- Do not chase archfit coupling numbers mechanically; use pytest architecture gates as the enforcement source and archfit as whole-graph drift signal.
- After implementation, run a scoped architecture re-review before starting another broad refactor.
