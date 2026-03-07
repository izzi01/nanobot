# Coding Conventions

**Analysis Date:** 2026-03-07

## Naming Patterns

**Files:**
- Use snake_case for Python module files in `nanobot/` and `tests/` (examples: `nanobot/agent/loop.py`, `nanobot/config/schema.py`, `tests/test_task_cancel.py`).
- Use `test_*.py` naming for Python tests in `tests/` (examples: `tests/test_commands.py`, `tests/test_tool_validation.py`).
- Use kebab-case-free, lower-case TypeScript filenames in `bridge/src/` with one concept per file (examples: `bridge/src/server.ts`, `bridge/src/whatsapp.ts`).

**Functions:**
- Use snake_case for Python functions and methods (examples: `_read_interactive_input_async` in `nanobot/cli/commands.py`, `_compute_next_run` in `nanobot/cron/service.py`).
- Use leading underscore for non-public helpers (examples: `_sanitize_messages` in `nanobot/providers/litellm_provider.py`, `_extract_post_content` in `nanobot/channels/feishu.py`).
- Use camelCase for TypeScript methods/functions (examples: `handleCommand`, `sendMessage` in `bridge/src/server.ts` and `bridge/src/whatsapp.ts`).

**Variables:**
- Use snake_case for Python locals/attributes (examples: `session_key_override` in `nanobot/bus/events.py`, `_processed_uids` in `nanobot/channels/email.py`).
- Use UPPER_SNAKE_CASE for Python constants (examples: `TYPING_NOTICE_TIMEOUT_MS`, `MATRIX_ALLOWED_HTML_TAGS` in `nanobot/channels/matrix.py`).

**Types:**
- Use PascalCase for classes/dataclasses/Pydantic models (examples: `AgentLoop` in `nanobot/agent/loop.py`, `OutboundMessage` in `nanobot/bus/events.py`, `Config` in `nanobot/config/schema.py`).
- Use explicit type hints on public APIs and most internal helpers (examples throughout `nanobot/agent/tools/base.py`, `nanobot/session/manager.py`).

## Code Style

**Formatting:**
- Tool used: Ruff configuration in `pyproject.toml` (`[tool.ruff]`).
- Key settings from `pyproject.toml`:
  - `line-length = 100`
  - `target-version = "py311"`
  - Lint families enabled: `E`, `F`, `I`, `N`, `W`
  - `E501` ignored (line-length overflow checks disabled in lint phase)
- TypeScript bridge code is strict-typed via `bridge/tsconfig.json` with `"strict": true`.

**Linting:**
- Tool used: Ruff (`[tool.ruff]` and `[tool.ruff.lint]` in `pyproject.toml`).
- Keep imports sorted/grouped and naming convention checks enabled by including Ruff `I` and `N` rules.

## Import Organization

**Order:**
1. Standard library imports
2. Third-party imports
3. Local `nanobot.*` imports

Observed examples:
- `nanobot/channels/email.py` (stdlib → `loguru` → local imports)
- `nanobot/agent/context.py` (stdlib → local imports)
- `tests/test_commands.py` (stdlib → `pytest`/`typer` → local imports)

**Path Aliases:**
- Python aliasing is not used; import project modules with full absolute package paths like `from nanobot.config.schema import Config`.
- TypeScript alias paths are not configured in `bridge/tsconfig.json`; use relative imports like `./whatsapp.js` in `bridge/src/server.ts`.

## Error Handling

**Patterns:**
- Use `try/except` with graceful fallbacks in runtime-critical code (examples: `_connect_mcp` in `nanobot/agent/loop.py`, `load_config` in `nanobot/config/loader.py`).
- Use structured log messages for failures with `logger.error(...)` or `logger.exception(...)` (examples in `nanobot/agent/loop.py`, `nanobot/channels/email.py`, `nanobot/cron/service.py`).
- In tool implementations, prefer returning explicit error strings instead of raising when user-facing recovery is expected (examples: `nanobot/agent/tools/message.py`, `nanobot/agent/tools/shell.py`, `nanobot/agent/tools/registry.py`).
- Raise typed exceptions for hard validation boundaries (examples: `ValueError` in `nanobot/providers/azure_openai_provider.py`, cron timezone validation in `nanobot/cron/service.py`).

## Logging

**Framework:** `loguru` (primary), with direct `print` usage in CLI/config bootstrap paths.

**Patterns:**
- Use Loguru brace formatting style: `logger.info("Cron: added job '{}' ({})", name, job.id)` in `nanobot/cron/service.py`.
- Use `logger.exception(...)` for stack traces in catch-all handlers (example: `_dispatch` in `nanobot/agent/loop.py`).
- Keep user-visible warnings/errors in CLI with `rich`/`console.print` and fallback `print` in config loader (`nanobot/cli/commands.py`, `nanobot/config/loader.py`).

## Comments

**When to Comment:**
- Add module-level docstrings describing purpose and subsystem boundaries (examples: `nanobot/agent/loop.py`, `nanobot/channels/matrix.py`, `nanobot/heartbeat/service.py`).
- Use inline comments for non-obvious safety/security behavior (examples: command guards in `nanobot/agent/tools/shell.py`, WebSocket auth flow in `bridge/src/server.ts`).

**JSDoc/TSDoc:**
- Python uses docstrings on modules/classes/methods consistently.
- TypeScript bridge uses block comments and interface declarations rather than heavy TSDoc tags (examples in `bridge/src/server.ts`, `bridge/src/whatsapp.ts`).

## Function Design

**Size:**
- Keep helpers focused and composable (`_extract_message_bytes`, `_extract_uid` in `nanobot/channels/email.py`).
- Use larger orchestration functions only for lifecycle/control flow (`AgentLoop._run_agent_loop` in `nanobot/agent/loop.py`, `ChannelManager._init_channels` in `nanobot/channels/manager.py`).

**Parameters:**
- Use typed parameters with defaults for optional behavior (examples: `AgentLoop.__init__` in `nanobot/agent/loop.py`, `CronService.add_job` in `nanobot/cron/service.py`).
- Prefer explicit keyword-style configs over positional bags (examples: `MatrixChannel.__init__` and provider constructors).

**Return Values:**
- Return structured dataclasses/Pydantic models for domain entities (`LLMResponse` in `nanobot/providers/base.py`, `Config` in `nanobot/config/schema.py`).
- Return `str` payloads from tools and include explicit error text when failing (`nanobot/agent/tools/message.py`, `nanobot/agent/tools/shell.py`).

## Module Design

**Exports:**
- Keep package-level exports minimal using `__all__` in lightweight `__init__.py` files (examples: `nanobot/channels/__init__.py`, `nanobot/agent/tools/__init__.py`).
- Put concrete logic in leaf modules and import by absolute path from call sites.

**Barrel Files:**
- Barrel-style files are used sparingly only for tiny top-level exports (`nanobot/channels/__init__.py`, `nanobot/agent/tools/__init__.py`).
- Do not route core runtime logic through barrels; import implementation modules directly for clarity and static analysis.

---

*Convention analysis: 2026-03-07*
