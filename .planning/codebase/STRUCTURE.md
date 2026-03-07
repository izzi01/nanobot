# Codebase Structure

**Analysis Date:** 2026-03-07

## Directory Layout

```text
nanobot/
├── nanobot/                # Python package: agent runtime, channels, config, providers
│   ├── agent/              # Core orchestration, context, memory, skills, tools
│   ├── bus/                # Inbound/outbound event models and async queues
│   ├── channels/           # Platform adapters (Telegram, Slack, Matrix, WhatsApp, etc.)
│   ├── cli/                # Typer CLI commands and runtime composition
│   ├── config/             # Pydantic schema and config load/save utilities
│   ├── cron/               # Scheduled task models and scheduler service
│   ├── heartbeat/          # Periodic heartbeat decision + execution trigger
│   ├── providers/          # LLM provider abstractions and implementations
│   ├── session/            # Session persistence and history retrieval
│   ├── skills/             # Built-in skill packs (each has SKILL.md)
│   ├── templates/          # Workspace bootstrap templates (AGENTS.md, TOOLS.md, etc.)
│   └── utils/              # Shared helpers (paths, filename safety, splitting)
├── bridge/                 # Node.js WhatsApp bridge sidecar (TypeScript)
│   └── src/                # Bridge server + Baileys WhatsApp client
├── tests/                  # Pytest suite and shell smoke test
├── case/                   # Demo assets (GIFs)
├── .planning/codebase/     # Generated architecture/quality/stack mapping docs
├── pyproject.toml          # Python packaging, dependencies, scripts, tool config
├── uv.lock                 # Locked Python dependency graph
├── README.md               # User-facing usage and setup documentation
└── docker-compose.yml      # Local container orchestration
```

## Directory Purposes

**`nanobot/`:**
- Purpose: Primary application package.
- Contains: All runtime logic, adapters, services, and abstractions.
- Key files: `nanobot/__main__.py`, `nanobot/cli/commands.py`, `nanobot/agent/loop.py`, `nanobot/config/schema.py`.

**`nanobot/agent/`:**
- Purpose: Core agent behavior and orchestration.
- Contains: Prompt context building (`context.py`), memory consolidation (`memory.py`), main loop (`loop.py`), skill loading (`skills.py`), background subagents (`subagent.py`), and tools in `tools/`.
- Key files: `nanobot/agent/loop.py`, `nanobot/agent/context.py`, `nanobot/agent/tools/registry.py`.

**`nanobot/channels/`:**
- Purpose: Channel adapter implementations.
- Contains: Shared base contract (`base.py`), runtime manager (`manager.py`), and one file per platform (`telegram.py`, `discord.py`, `slack.py`, `matrix.py`, `whatsapp.py`, etc.).
- Key files: `nanobot/channels/base.py`, `nanobot/channels/manager.py`.

**`nanobot/providers/`:**
- Purpose: LLM provider integration layer.
- Contains: Provider interface (`base.py`), concrete providers (`litellm_provider.py`, `custom_provider.py`, `azure_openai_provider.py`, `openai_codex_provider.py`), metadata registry (`registry.py`).
- Key files: `nanobot/providers/base.py`, `nanobot/providers/litellm_provider.py`, `nanobot/providers/registry.py`.

**`nanobot/config/`:**
- Purpose: Runtime configuration models and persistence.
- Contains: Pydantic schema (`schema.py`) and JSON loader/saver (`loader.py`).
- Key files: `nanobot/config/schema.py`, `nanobot/config/loader.py`.

**`nanobot/session/`:**
- Purpose: Session lifecycle and disk persistence.
- Contains: Session dataclass and JSONL manager.
- Key files: `nanobot/session/manager.py`.

**`nanobot/cron/`:**
- Purpose: Scheduled job domain model and scheduler engine.
- Contains: Dataclasses (`types.py`) and timer/execution service (`service.py`).
- Key files: `nanobot/cron/service.py`, `nanobot/cron/types.py`.

**`nanobot/heartbeat/`:**
- Purpose: Periodic task-checking service that can invoke agent execution.
- Contains: Heartbeat loop and decision/execution flow.
- Key files: `nanobot/heartbeat/service.py`.

**`nanobot/templates/`:**
- Purpose: Default workspace prompt/memory templates synced to user workspace.
- Contains: `AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `HEARTBEAT.md`, and `memory/MEMORY.md`.
- Key files: `nanobot/templates/AGENTS.md`, `nanobot/templates/memory/MEMORY.md`.

**`nanobot/skills/`:**
- Purpose: Built-in reusable skill definitions consumed by `SkillsLoader`.
- Contains: Per-skill directories each with `SKILL.md` (and optional scripts).
- Key files: `nanobot/skills/README.md`, `nanobot/skills/github/SKILL.md`, `nanobot/skills/tmux/scripts/find-sessions.sh`.

**`bridge/`:**
- Purpose: External Node.js sidecar for WhatsApp protocol handling.
- Contains: TypeScript source, package manifest, TypeScript config.
- Key files: `bridge/src/index.ts`, `bridge/src/server.ts`, `bridge/src/whatsapp.ts`, `bridge/package.json`.

**`tests/`:**
- Purpose: Automated tests for runtime and integration behaviors.
- Contains: Pytest modules and a shell docker smoke test.
- Key files: `tests/test_commands.py`, `tests/test_cron_service.py`, `tests/test_docker.sh`.

## Key File Locations

**Entry Points:**
- `nanobot/__main__.py`: Python module entrypoint (`python -m nanobot`).
- `nanobot/cli/commands.py`: Main Typer CLI app and runtime wiring.
- `bridge/src/index.ts`: WhatsApp bridge process entrypoint.

**Configuration:**
- `pyproject.toml`: Python dependencies, scripts, lint/test config.
- `nanobot/config/schema.py`: Typed config schema and provider matching helpers.
- `nanobot/config/loader.py`: Config path/load/save/migration.
- `bridge/tsconfig.json`: TypeScript compile settings for bridge.
- `bridge/package.json`: Bridge scripts and dependencies.

**Core Logic:**
- `nanobot/agent/loop.py`: Main execution loop and tool orchestration.
- `nanobot/agent/context.py`: Prompt + message assembly.
- `nanobot/channels/manager.py`: Channel lifecycle and outbound dispatching.
- `nanobot/providers/litellm_provider.py`: Primary provider implementation.
- `nanobot/session/manager.py`: Session persistence and retrieval.

**Testing:**
- `tests/*.py`: Unit/integration tests for CLI, channels, agent loop, cron, heartbeat, providers.
- `tests/test_docker.sh`: Container smoke test script.

## Naming Conventions

**Files:**
- Python modules use `snake_case.py`: `nanobot/agent/subagent.py`, `nanobot/channels/manager.py`.
- Tests use `test_*.py`: `tests/test_tool_validation.py`, `tests/test_message_tool.py`.
- Skills/templates use uppercase markdown names for canonical docs: `nanobot/templates/AGENTS.md`, `nanobot/skills/github/SKILL.md`.
- TypeScript bridge files use lowercase names: `bridge/src/server.ts`, `bridge/src/whatsapp.ts`.

**Directories:**
- Python package directories use lowercase single-word grouping by responsibility: `nanobot/providers/`, `nanobot/session/`, `nanobot/heartbeat/`.
- Tool implementations are grouped under `nanobot/agent/tools/`.
- Skill packs are directory-per-skill under `nanobot/skills/{skill_name}/`.

## Where to Add New Code

**New Feature (core runtime behavior):**
- Primary code: Add orchestration in `nanobot/agent/loop.py` and supporting service/module in the closest domain folder (`nanobot/cron/`, `nanobot/heartbeat/`, `nanobot/session/`, etc.).
- Config schema: Add new typed fields in `nanobot/config/schema.py` and load/save behavior in `nanobot/config/loader.py` only if migration is needed.
- Tests: Add `tests/test_<feature>.py` in `tests/`.

**New Channel Integration:**
- Implementation: Create `nanobot/channels/<channel>.py` implementing `BaseChannel` from `nanobot/channels/base.py`.
- Registration: Enable startup wiring in `ChannelManager._init_channels()` in `nanobot/channels/manager.py`.
- Config: Add channel config model to `nanobot/config/schema.py` and include it in `ChannelsConfig`.
- Tests: Add channel behavior tests in `tests/test_<channel>_channel.py`.

**New Tool Capability:**
- Implementation: Add tool class in `nanobot/agent/tools/<tool_name>.py` inheriting `Tool` from `nanobot/agent/tools/base.py`.
- Registration: Register in `AgentLoop._register_default_tools()` in `nanobot/agent/loop.py` (and `SubagentManager` if subagent access is required).
- Tests: Add tool validation/behavior tests in `tests/test_<tool_name>.py`.

**New Provider Integration:**
- Metadata first: Add `ProviderSpec` entry in `nanobot/providers/registry.py`.
- Config exposure: Add provider field in `ProvidersConfig` inside `nanobot/config/schema.py`.
- Runtime client: Add provider class in `nanobot/providers/` only when behavior cannot be expressed via `LiteLLMProvider` + registry metadata.
- Tests: Add provider tests in `tests/test_<provider>_provider.py`.

**New Skill Pack:**
- Implementation: Create `nanobot/skills/<skill_name>/SKILL.md` (and optional scripts/assets in that directory).
- Workspace override: Use `~/.nanobot/workspace/skills/<skill_name>/SKILL.md` for user-local customization (loaded first by `SkillsLoader` in `nanobot/agent/skills.py`).

**New Bridge Capability (WhatsApp-side):**
- Bridge logic: Add/modify TypeScript in `bridge/src/*.ts`.
- Python adapter changes: Mirror payload/contract updates in `nanobot/channels/whatsapp.py`.

## Special Directories

**`.planning/codebase/`:**
- Purpose: Generated architecture/stack/quality/concern mapping artifacts for GSD planner/executor workflows.
- Generated: Yes (by mapper commands).
- Committed: Yes.

**`nanobot/templates/`:**
- Purpose: Seed files copied into user workspace by `sync_workspace_templates()` in `nanobot/utils/helpers.py`.
- Generated: No (source-controlled templates).
- Committed: Yes.

**`nanobot/skills/`:**
- Purpose: Built-in skills packaged with distribution.
- Generated: No.
- Committed: Yes.

**`bridge/dist/`:**
- Purpose: Compiled JavaScript build output for bridge runtime.
- Generated: Yes (via `npm run build`).
- Committed: Not detected in repository root structure; runtime copies/builds into user path (`~/.nanobot/bridge`).

**`.venv/`:**
- Purpose: Local virtual environment.
- Generated: Yes.
- Committed: No (development artifact).

---

*Structure analysis: 2026-03-07*
