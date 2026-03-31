# Modularity Review

**Scope**: Full codebase — `src/ccgram/` (~28,500 LOC across 84 Python modules)
**Date**: 2026-03-30
**Context**: Third review. Previous reviews (2026-03-28, 2026-03-29) resolved 4 of 5 original issues. This review covers the new inter-agent messaging feature and reassesses persistent concerns.

## Executive Summary

ccgram bridges Telegram Forum topics to tmux windows running AI coding agents (Claude Code, Codex, Gemini, Shell). The previous modularity refactoring achieved strong results: `bot.py` was halved, polling was decomposed into strategies, the provider protocol was cleaned up, and the callback dispatch was replaced with a self-registering registry. Three concerns persist — `SessionManager` remains a god object (1,526 lines, 50 methods, 24 consumers), per-topic state is fragmented across 14+ modules with inconsistent key types, and the new inter-agent messaging feature introduces [intrusive coupling](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) via private-interface imports across 5 modules. The messaging subsystem's `mailbox.py` core is excellently isolated (zero internal imports), but the handler-layer modules (`msg_broker`, `msg_spawn`) bypass encapsulation boundaries to share mutable state and private functions.

## Coupling Overview Table

| Integration                                                   | [Strength](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | [Distance](https://coupling.dev/posts/dimensions-of-coupling/distance/) | [Volatility](https://coupling.dev/posts/dimensions-of-coupling/volatility/) | [Balanced?](https://coupling.dev/posts/core-concepts/balance/) |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Hook System → Session Monitor                                 | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | High (separate processes, JSONL files)                                  | Low                                                                         | Yes                                                            |
| Provider Protocol → Consumers                                 | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | Low (same package)                                                      | Medium                                                                      | Yes                                                            |
| LLM / Whisper → Consumers                                     | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | Low (same package)                                                      | Low                                                                         | Yes                                                            |
| Callback Registry → Handlers                                  | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | Low (same package)                                                      | Low                                                                         | Yes                                                            |
| ThreadRouter → Consumers                                      | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | Low (same package)                                                      | Low                                                                         | Yes                                                            |
| Shell PromptMatch → Consumers                                 | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | Low (same package)                                                      | Low                                                                         | Yes                                                            |
| `mailbox.py` Core → Consumers                                 | [Contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)   | Low (same package)                                                      | Medium                                                                      | Yes                                                            |
| Polling Strategies → Coordinator                              | [Model](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)      | Low (same package)                                                      | Medium                                                                      | Borderline                                                     |
| `msg_broker` → `mailbox` internals                            | [Intrusive](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)  | Low (same package)                                                      | High                                                                        | **No**                                                         |
| `msg_broker` + `msg_spawn` → `_pending_requests`              | [Intrusive](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)  | Low (same package)                                                      | High                                                                        | **No**                                                         |
| `msg_spawn` → `topic_orchestration` / `msg_telegram` privates | [Intrusive](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)  | Low (same package)                                                      | High                                                                        | **No**                                                         |
| `msg_cmd` → `state.json` schema                               | [Intrusive](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)  | Medium (CLI process vs bot process)                                     | High                                                                        | **No**                                                         |
| `SessionManager` → 24 consumers                               | [Functional](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) | Low (same package)                                                      | High                                                                        | **No**                                                         |
| `cleanup.py` → 14 modules                                     | [Intrusive](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)  | Low (same package)                                                      | High                                                                        | **No**                                                         |
| `TopicLifecycle` → `Terminal._states`                         | [Intrusive](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/)  | Low (same file)                                                         | Medium                                                                      | Borderline                                                     |

## Issue: Inter-Agent Messaging Private-Interface Coupling

**Integration**: `msg_broker.py` / `msg_spawn.py` / `msg_cmd.py` → `mailbox.py` / `spawn_request.py` / `topic_orchestration.py` internals
**Severity**: Significant

### Knowledge Leakage

The messaging feature's handler layer reaches across module boundaries via private interfaces in five specific ways:

1. **`msg_broker.py`** imports `_sanitize_dir_name` and `_validate_no_traversal` from `mailbox.py` — private functions that encode the mailbox's internal directory structure (`inbox_dir / "tmp" / "deliver-{msg_id}.txt"`). The broker re-implements mailbox path construction logic rather than calling a public API.

2. **`msg_broker.py`** and **`msg_spawn.py`** both import `_pending_requests` — a private `dict` from `spawn_request.py`. Three modules co-own a single in-process mutable dict with no accessor protocol. Any cache structure change requires synchronized edits across all three files.

3. **`msg_spawn.py`** imports `_resolve_topic` (private) from `msg_telegram.py`, and `_collect_target_chats` / `_create_topic_in_chat` (private) from `topic_orchestration.py`. These are implementation details of sibling handler modules, not stable contracts.

4. **`msg_cmd.py`**'s `_load_window_states()` reads `state.json` directly and hardcodes the field names `window_states`, `cwd`, `window_name`, `provider_name`, `external` — the private serialization schema of `SessionManager`. This exists because the CLI cannot instantiate the bot, but it creates a silent [contract](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) that breaks if `WindowState.to_dict()` is ever changed.

5. **`msg_broker` ↔ `msg_telegram`** form a bidirectional dependency cycle. Both defer their cross-imports to function scope to avoid circular import errors, but the logical cycle means neither module can be tested or reasoned about independently.

### Complexity Impact

A developer modifying `mailbox.py`'s internal directory structure (e.g., changing the temporary delivery path) must also update `msg_broker.py`, which silently duplicates that knowledge. The `_pending_requests` shared dict is particularly dangerous: three modules assume they can `.pop()` from it concurrently, with no locking or ownership protocol. The [complexity](https://coupling.dev/posts/core-concepts/complexity/) here exceeds what a developer can hold in working memory — the implicit contracts between these five modules are invisible at the function signature level.

### Cascading Changes

- **Renaming `_pending_requests` or changing its value type** in `spawn_request.py` forces changes in both `msg_broker.py` and `msg_spawn.py`.
- **Changing `state.json` field names** in `session.py` silently breaks `msg_cmd.py`'s peer discovery — no import error, no type error, just wrong runtime behavior.
- **Refactoring `topic_orchestration.py`'s internal `_create_topic_in_chat`** breaks `msg_spawn.py`'s spawn-approval flow.
- **Restructuring `mailbox.py`'s directory layout** requires updating `msg_broker.py`'s `write_delivery_file` function.

### Recommended Improvement

To reduce [integration strength](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) from intrusive to contract:

1. **Expose a `create_delivery_path(msg_id)` method on `Mailbox`** or move `write_delivery_file` into `mailbox.py` entirely. The broker should call a public method, not re-implement internal path logic.
2. **Replace `_pending_requests` with accessor functions** in `spawn_request.py`: `get_pending(request_id) -> SpawnRequest | None`, `pop_pending(request_id) -> SpawnRequest | None`, `iter_pending() -> Iterator`. The dict remains private; consumers go through a stable API.
3. **Promote `_resolve_topic`, `_collect_target_chats`, `_create_topic_in_chat` to public functions** (drop the underscore prefix) with stable signatures. These are used across module boundaries and should be treated as published interfaces.
4. **Add a `load_window_info()` function to `msg_discovery.py`** that reads `state.json` via `SessionManager.to_dict()` serialization contract — or better, define a `WindowInfoSnapshot` export on `session.py` that `msg_cmd.py` can import for the CLI path. This makes the schema dependency explicit.
5. **Break the `msg_broker` ↔ `msg_telegram` cycle** by extracting the shared `delivery_strategy` into a third module (e.g., `msg_delivery.py`) that both import without cross-referencing each other.

Trade-off: five small API additions across three modules. The cost is low — most changes are promoting existing private functions to public — and the benefit is that each module's public interface becomes the single source of truth for its behavior.

## Issue: SessionManager God Object

**Integration**: `SessionManager` (1,526 lines, ~50 public methods) → 24 consumer modules
**Severity**: Significant

### Knowledge Leakage

`SessionManager` bundles six distinct concerns into a single class, exposing all of them through one import. Every consumer — whether it needs to send a keystroke, check a user preference, or resolve a session — gets the full 50-method API surface. The concerns are:

1. **Window state store**: `window_states` dict, CRUD operations (8 methods)
2. **Per-window mode configuration**: approval, notification, batch mode getters/setters/cyclers (9 methods)
3. **State persistence and lifecycle**: startup resolution, shutdown flush, pruning (4 methods)
4. **Session I/O**: transcript resolution, `send_to_window`, `get_recent_messages` (3 methods)
5. **User favorites/MRU/offsets**: starred directories, most-recently-used, read positions (6 methods)
6. **Thread routing pass-through**: 9 methods that delegate directly to `thread_router` with zero added logic

The `protocols.py` file that was intended to provide [interface segregation](https://coupling.dev/posts/core-concepts/modularity/) no longer exists — it was removed after the previous refactoring. All 24 consumers import the concrete `session_manager` singleton directly with no abstraction boundary.

### Complexity Impact

When a developer adds a new per-window setting (e.g., a notification preference), the change lands in the same 1,526-line file alongside unrelated concerns like tmux keystroke sending and user MRU tracking. The [cognitive load](https://coupling.dev/posts/core-concepts/complexity/) of modifying `SessionManager` is high because any of its 50 methods might interact with any of its 8 internal data structures. The 24-consumer fan-in means that any signature change, even to a rarely-used method, must be checked against a quarter of the codebase.

### Cascading Changes

- **Adding a new preference type** (getter + setter + cycler) adds 3 methods to an already-overloaded class and may require persistence format changes in `to_dict`/`from_dict`.
- **Changing `send_to_window`'s behavior** (e.g., adding rate limiting) affects 9 callers that import `session_manager` solely for this method.
- **Renaming `window_states`** cascades to 4 modules that access the raw dict directly (`polling_coordinator`, `sync_command`, `msg_spawn`, `providers/__init__`).

### Recommended Improvement

Interface segregation without physical decomposition — the same recommendation from review #1, now with a clearer path since `ThreadRouter` proves the pattern works:

1. **Drop the 9 pass-through delegation methods** (`bind_thread`, `unbind_thread`, `get_window_for_thread`, etc.). Every caller already has access to `thread_router` — the delegation exists only for backward compatibility that a feature branch doesn't need.
2. **Extract `UserPreferences`** (starred, MRU, offsets — 6 methods) into a small standalone class, following the `ThreadRouter` pattern. Only 2 modules use these methods (`directory_browser`, `directory_callbacks`).
3. **Define narrow `Protocol` interfaces** for the remaining concerns: `WindowStateStore` (8 methods), `SessionIO` (3 methods), `WindowModeConfig` (9 methods). Consumers type-annotate against the protocol they actually need, not the full `SessionManager`.

Trade-off: the physical class stays unified (avoiding serialization headaches), but the logical API is partitioned. Removing 9 pass-through methods and extracting `UserPreferences` immediately drops the public method count from ~50 to ~35 and removes 2 consumers from `SessionManager`'s fan-in.

## Issue: Fragmented Per-Topic State with Intrusive Cleanup

**Integration**: `cleanup.py` → 14 stateful modules via lazy imports
**Severity**: Significant

### Knowledge Leakage

Per-topic runtime state is scattered across 14+ modules as module-level mutable dicts, each with its own key type. `cleanup.py` acts as a centralized teardown orchestrator that must know about every stateful module's internal cleanup function. It currently contains 14 lazy imports to avoid circular dependencies — each one is an [intrusive coupling](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) point where `cleanup.py` reaches into a module's private state management.

The key type inconsistency makes the knowledge leakage worse:

| Key Type                      | Used By                                                            |
| ----------------------------- | ------------------------------------------------------------------ |
| `(user_id, thread_id)`        | message_queue, interactive_ui, command_history, polling lifecycle  |
| `(chat_id, thread_id)`        | topic_emoji, shell_commands                                        |
| `window_id` (bare string)     | polling terminal, subagents, shell_capture, pane_alerts, vim_state |
| `qualified_id` (`session:@N`) | mailbox, delivery, spawn, declared                                 |

`cleanup.py` must translate between all four schemes on every call, using `thread_router.resolve_chat_id()` to bridge the `user_id` ↔ `chat_id` gap. If the thread binding is deleted before cleanup resolves `chat_id`, the `(chat_id, thread_id)` state is silently orphaned.

### Complexity Impact

Every new feature that adds per-topic mutable state must also add a lazy import and cleanup call to `cleanup.py` — or the state leaks. This is an implicit contract with no enforcement mechanism. The 14 lazy imports are evidence of at least 4 structural dependency cycles in the handlers package. A developer adding a simple per-window counter must understand the cleanup system, choose the correct key type, and wire up the teardown — exceeding the [cognitive capacity](https://coupling.dev/posts/core-concepts/complexity/) required for what should be a local change.

Three active state leaks remain unaddressed:

| Leaked State                            | Module              | Risk                                                            |
| --------------------------------------- | ------------------- | --------------------------------------------------------------- |
| `_bash_capture_tasks`                   | `text_handler.py`   | Active `asyncio.Task` objects never cancelled on topic deletion |
| `_loop_alert_pairs`                     | `msg_telegram.py`   | Bounded by LRU (100 entries) but never per-topic cleaned        |
| `_last_send_time` / `_rate_limit_locks` | `message_sender.py` | Grow unbounded with `chat_id` keys, never evicted               |

### Cascading Changes

- **Adding a new per-topic feature** (e.g., a per-window message counter) requires changes in the feature module _and_ `cleanup.py` — a mandatory two-file change that is easy to forget.
- **Changing `thread_router.resolve_chat_id()` semantics** silently breaks cleanup for all `(chat_id, thread_id)`-keyed state.
- **Deleting a handler module** requires also removing its lazy import from `cleanup.py`, or cleanup crashes on the missing import.

### Recommended Improvement

Consolidate per-topic state into a single registry with a uniform key:

1. **Create a `TopicStateRegistry`** with a `register(key, cleanup_fn)` API. Modules register their cleanup functions at import time (like `callback_registry`'s `@register` pattern). `cleanup.py` becomes a one-liner: `registry.clear(topic_key)`.
2. **Standardize on `window_id` as the universal key** for window-scoped state, and `(user_id, thread_id)` for topic-scoped state. Modules that currently use `(chat_id, thread_id)` switch to `(user_id, thread_id)` — the `resolve_chat_id` translation moves to the point of use (emoji rendering, shell command display) rather than cleanup.
3. **Fix the three leaks**: cancel `_bash_capture_tasks` in `clear_topic_state`, add `_loop_alert_pairs` cleanup keyed by window_id, and add TTL eviction to `_last_send_time`.

Trade-off: the registry adds a small abstraction layer (~50 lines). The payoff is that new stateful features wire themselves into cleanup via a one-line registration call instead of a cross-module lazy-import edit, and the cleanup path becomes verifiable (the registry knows all registered state).

## Issue: Cross-Strategy Encapsulation Violations in Polling

**Integration**: `TopicLifecycleStrategy` → `TerminalStatusStrategy._states` (private dict)
**Severity**: Minor

### Knowledge Leakage

`TopicLifecycleStrategy` directly accesses `self._terminal._states` — the private state dict of `TerminalStatusStrategy` — in five methods: `reset_autoclose_state`, `clear_probe_failures`, `reset_probe_failures_state`, `clear_seen_status`, and `reset_seen_status_state`. This means `TopicLifecycleStrategy` knows the internal data structure of its sibling strategy, bypassing the public method interface that `TerminalStatusStrategy` provides.

Additionally, `polling_coordinator.py` imports five underscore-prefixed constants (`_ACTIVITY_THRESHOLD`, `_MAX_PROBE_FAILURES`, `_PANE_COUNT_TTL`, `_STARTUP_TIMEOUT`, `_TYPING_INTERVAL`) from `polling_strategies.py` and uses them to drive threshold logic inline. The coordinator co-authors the decision logic that strategies should own.

### Complexity Impact

This is a contained issue. Both the coordinator and all strategies live in the same package, and the affected logic is cohesive (all related to terminal status polling). The [distance](https://coupling.dev/posts/dimensions-of-coupling/distance/) is very low — same package, same developer, same deployment unit. The practical impact is limited to making the strategy classes harder to refactor independently: renaming `_states` or changing its structure in `TerminalStatusStrategy` requires updating `TopicLifecycleStrategy` in lockstep.

### Cascading Changes

- **Changing `WindowPollState` fields** in `TerminalStatusStrategy._states` cascades to 5 methods in `TopicLifecycleStrategy`.
- **Changing threshold values** requires updating both the constant definition in `polling_strategies.py` and the comparison logic in `polling_coordinator.py`.

### Recommended Improvement

Add public methods to `TerminalStatusStrategy` for the operations that `TopicLifecycleStrategy` currently performs by reaching into `_states`:

- `clear_probe_failures(window_id)`, `reset_all_probe_failures()`
- `clear_seen_status(window_id)`, `reset_all_seen_status()`
- `clear_unbound_timer(window_id)`, `reset_all_unbound_timers()`

Move threshold comparisons (`>= _MAX_PROBE_FAILURES`, `>= _STARTUP_TIMEOUT`) inside strategy methods so the coordinator delegates rather than evaluates.

Trade-off: ~30 lines of method additions. The [volatility](https://coupling.dev/posts/dimensions-of-coupling/volatility/) of this area is medium, so this is lower priority than Issues 1–3 but straightforward to fix.

## Well-Balanced Integrations

Several integrations demonstrate good [modularity](https://coupling.dev/posts/core-concepts/modularity/):

- **Hook System → Session Monitor**: [Contract coupling](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) via structured JSONL files between separate processes. High [distance](https://coupling.dev/posts/dimensions-of-coupling/distance/), low [volatility](https://coupling.dev/posts/dimensions-of-coupling/volatility/). The gold standard in this codebase.
- **Provider Protocol → Consumers**: [Contract coupling](https://coupling.dev/posts/dimensions-of-coupling/integration-strength/) via `AgentProvider` protocol and `ProviderCapabilities` dataclass. Capability flags gate behavior without name-string checks.
- **LLM / Whisper Subsystems**: Protocol + factory pattern with zero cross-coupling. Adding a new LLM provider requires one file and one registry entry.
- **Callback Registry**: Self-registering `@register` decorator. Adding a callback requires zero changes to `bot.py`.
- **ThreadRouter**: Zero internal ccgram imports, 19 clean public methods, `to_dict`/`from_dict` serialization. The cleanest extraction from the original `SessionManager`.
- **`mailbox.py` Core**: Zero internal imports. Pure data model (`Message` dataclass) with file I/O. The messaging feature's strongest module.
- **Shell `PromptMatch` Contract**: Named dataclass fields replaced fragile `re.Match.group(N)` access. Low volatility, explicitly typed.

## Modularity Scorecard

| Dimension                          | Review #1 (Mar 28) | Review #2 (Mar 29) | Review #3 (Mar 30)                   | Trend                        |
| ---------------------------------- | ------------------ | ------------------ | ------------------------------------ | ---------------------------- |
| Largest module (lines)             | 2,018 (bot.py)     | 1,476 (session.py) | 1,526 (session.py)                   | Stable                       |
| Modules > 1,000 lines              | 3                  | 2                  | 2                                    | Stable                       |
| Max import count                   | 41 (bot.py)        | ~20 (bot.py)       | ~27 (polling_coordinator)            | Slight regression            |
| Critical issues                    | 3                  | 0                  | 0                                    | Maintained                   |
| Significant issues                 | 2                  | 1                  | 3                                    | New feature added coupling   |
| Minor issues                       | 0                  | 2                  | 1                                    | Improved                     |
| Balanced integrations              | 2                  | 6                  | 7                                    | Improved                     |
| Late import workarounds            | Not measured       | 7 (cleanup.py)     | 14 (cleanup.py)                      | Worsened (new feature state) |
| Modules with zero internal imports | Not measured       | Not measured       | 3 (mailbox, msg_skill, ThreadRouter) | Baseline                     |

## Priority Recommendations

1. **P1**: Fix inter-agent messaging private-interface coupling — promote private functions to public APIs, replace shared mutable dict with accessor functions, break the broker↔telegram cycle.
2. **P1**: Fix the three active state leaks (`_bash_capture_tasks`, `_loop_alert_pairs`, `_last_send_time`).
3. **P2**: Remove SessionManager's 9 pass-through thread-routing methods; extract `UserPreferences` (6 methods, 2 consumers).
4. **P2**: Introduce a `TopicStateRegistry` to replace cleanup.py's 14 lazy imports with a self-registration pattern.
5. **P3**: Add public methods to `TerminalStatusStrategy` to eliminate `TopicLifecycleStrategy`'s 5 private-state accesses.

---

_This analysis was performed using the [Balanced Coupling](https://coupling.dev) model by [Vlad Khononov](https://vladikk.com)._
