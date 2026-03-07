# Testing Patterns

**Analysis Date:** 2026-03-07

## Test Framework

**Runner:**
- pytest (configured in `pyproject.toml` under `[tool.pytest.ini_options]`)
- Async plugin: `pytest-asyncio` declared in `pyproject.toml` dev dependencies
- Config: `pyproject.toml`

**Assertion Library:**
- Built-in `assert` statements in pytest style (examples across `tests/test_tool_validation.py`, `tests/test_commands.py`, `tests/test_matrix_channel.py`).

**Run Commands:**
```bash
pytest                       # Run all tests from testpaths
pytest -k <pattern>          # Run selected tests by name expression
pytest tests/test_matrix_channel.py  # Run a single file
```

## Test File Organization

**Location:**
- Tests are in a dedicated top-level `tests/` directory (not co-located with source).
- Pytest discovery is pinned to `tests` via `testpaths = ["tests"]` in `pyproject.toml`.

**Naming:**
- File naming uses `test_*.py` (examples: `tests/test_cron_service.py`, `tests/test_cli_input.py`).
- Test function naming uses `test_<behavior>` style for plain tests.
- Related tests are grouped in `Test*` classes where useful (examples: `TestHandleStop` in `tests/test_task_cancel.py`, `TestMemoryConsolidationTypeHandling` in `tests/test_memory_consolidation_types.py`).

**Structure:**
```
tests/
├── test_commands.py
├── test_task_cancel.py
├── test_tool_validation.py
├── test_matrix_channel.py
└── ...
```

## Test Structure

**Suite Organization:**
```python
import pytest
from unittest.mock import AsyncMock, MagicMock, patch


def _make_loop():
    ...


class TestHandleStop:
    @pytest.mark.asyncio
    async def test_stop_no_active_task(self):
        ...
        assert "No active task" in out.content
```

Pattern reflected in `tests/test_task_cancel.py`.

**Patterns:**
- Setup pattern:
  - Factory/helper functions for SUT creation (examples: `_make_loop` in `tests/test_task_cancel.py`, `_make_config` in `tests/test_email_channel.py`).
  - Fixtures for reusable patches/state (examples: `mock_paths` in `tests/test_commands.py`, `mock_prompt_session` in `tests/test_cli_input.py`).
- Teardown pattern:
  - Context manager patch teardown (`with patch(...)` blocks).
  - Temp path lifecycle managed by pytest `tmp_path` fixture.
  - Explicit cleanup for temp dirs when not using `tmp_path` (example: `shutil.rmtree` in `tests/test_commands.py`).
- Assertion pattern:
  - Prefer direct value assertions and substring checks.
  - Use `pytest.raises(...)` for validation error paths (`tests/test_cron_service.py`, `tests/test_azure_openai_provider.py`).

## Mocking

**Framework:**
- `unittest.mock` (`patch`, `MagicMock`, `AsyncMock`) plus pytest `monkeypatch` fixture.

**Patterns:**
```python
with patch("nanobot.agent.loop.ContextBuilder"), \
     patch("nanobot.agent.loop.SessionManager"), \
     patch("nanobot.agent.loop.SubagentManager") as MockSubMgr:
    MockSubMgr.return_value.cancel_by_session = AsyncMock(return_value=0)
```
Pattern from `tests/test_task_cancel.py`.

```python
monkeypatch.setattr("nanobot.channels.email.smtplib.SMTP", _smtp_factory)
```
Pattern from `tests/test_email_channel.py`.

**What to Mock:**
- External boundaries: HTTP clients, SMTP/IMAP, SDK clients, websocket clients (`tests/test_email_channel.py`, `tests/test_matrix_channel.py`, `tests/test_azure_openai_provider.py`).
- Expensive or long-running async internals with `AsyncMock` (`tests/test_message_tool_suppress.py`, `tests/test_task_cancel.py`).
- Global singleton-ish objects in CLI with `patch` (`tests/test_cli_input.py`).

**What NOT to Mock:**
- Pure validation/casting logic and deterministic helpers should run real code (examples: `tests/test_tool_validation.py`, `tests/test_feishu_table_split.py`, `tests/test_context_prompt_cache.py`).

## Fixtures and Factories

**Test Data:**
```python
@pytest.fixture
def mock_paths():
    with patch("nanobot.config.loader.get_config_path") as mock_cp, \
         patch("nanobot.config.loader.save_config") as mock_sc, \
         patch("nanobot.config.loader.load_config") as mock_lc, \
         patch("nanobot.utils.helpers.get_workspace_path") as mock_ws:
        ...
        yield config_file, workspace_dir
```
Pattern from `tests/test_commands.py`.

**Location:**
- Fixtures are currently local to test modules (no shared `tests/conftest.py` detected).
- Reusable private builders are defined in-file with `_make_*` naming.

## Coverage

**Requirements:**
- No explicit minimum coverage threshold detected in repository configuration (`pyproject.toml` has no coverage section).

**View Coverage:**
```bash
pytest --cov=nanobot --cov-report=term-missing
```
(Usable command pattern; coverage plugin configuration is not committed in current config.)

## Test Types

**Unit Tests:**
- Dominant type. Most files validate single-module behavior with mocked dependencies.
- Examples:
  - Tool parameter casting/validation in `tests/test_tool_validation.py`
  - Prompt/session behavior in `tests/test_context_prompt_cache.py` and `tests/test_loop_save_turn.py`

**Integration Tests:**
- Lightweight integration-style tests exist around service orchestration with in-memory or fake dependencies.
- Examples:
  - `MessageBus` + `AgentLoop` dispatch/cancel paths in `tests/test_task_cancel.py`
  - `CronService` persistence/reload behavior in `tests/test_cron_service.py`

**E2E Tests:**
- Not used (no browser/system end-to-end harness detected).

## Common Patterns

**Async Testing:**
```python
@pytest.mark.asyncio
async def test_trigger_now_executes_when_decision_is_run(tmp_path) -> None:
    ...
    result = await service.trigger_now()
    assert result == "done"
```
Pattern from `tests/test_heartbeat_service.py`.

**Error Testing:**
```python
with pytest.raises(ValueError, match="unknown timezone 'America/Vancovuer'"):
    service.add_job(...)
```
Pattern from `tests/test_cron_service.py`.

Additional observed pattern:
```python
result = await provider.chat(messages)
assert result.finish_reason == "error"
assert "Azure OpenAI API Error 401" in result.content
```
Pattern from `tests/test_azure_openai_provider.py`.

---

*Testing analysis: 2026-03-07*
