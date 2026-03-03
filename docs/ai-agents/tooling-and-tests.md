# Tooling and Tests

## Build and Validation Commands

Primary command set from `Makefile`:

- `make fmt`
- `make test`
- `make lint`
- `make typecheck`
- `make check` (runs fmt + lint + typecheck + test + integration)
- `make build`

Default local workflow for code changes:

1. `make fmt`
2. `make test`
3. `make lint`
4. `make typecheck`

Before considering work complete, run at least:

- `make check` (full gate: fmt + lint + typecheck + test)

## Toolchain and Libraries

- Python: `>=3.14`
- Package/dependency manager: `uv`
- Telegram framework: `python-telegram-bot`
- tmux integration: `libtmux`
- async/file IO: `aiofiles`
- logging: `structlog`
- terminal parsing: `pyte`
- screenshot rendering: `Pillow`

## Test Layout

- `tests/ccbot/`: unit tests mirroring source modules.
- `tests/integration/`: integration tests for monitor flow, dispatch, tmux manager, state roundtrips.
- `tests/conftest.py`: required test env setup before imports.
- Hypothesis property-based tests: `tests/ccbot/test_message_queue_properties.py`.

## Fast Test Targeting

Use focused test files that match changed modules first, then full test run.

Examples:

- session/state changes -> `tests/ccbot/test_session.py`, `tests/ccbot/test_state_migration.py`
- monitor/parsing changes -> `tests/ccbot/test_session_monitor.py`, `tests/ccbot/test_transcript_parser.py`
- handlers/UI changes -> `tests/ccbot/test_text_handler.py`, `tests/ccbot/test_status_polling.py`, `tests/ccbot/test_bot_callbacks.py`
- command changes -> `tests/ccbot/test_command_catalog.py`, `tests/ccbot/test_commands_command.py`, `tests/ccbot/test_cc_commands.py`
- hook/event changes -> `tests/ccbot/test_hook.py`, `tests/ccbot/test_hook_events.py`, `tests/ccbot/test_session_monitor_events.py`
- cleanup/lifecycle changes -> `tests/ccbot/test_cleanup.py`, `tests/ccbot/test_topic_emoji.py`
- provider changes -> `tests/ccbot/test_provider_contracts.py`, `tests/ccbot/test_jsonl_providers.py`

## Quality Constraints

- all hook/check issues are blocking.
- fix failing checks before proceeding to unrelated work.
- preserve existing architecture constraints (topic-window identity, provider boundaries, send-layer split).
