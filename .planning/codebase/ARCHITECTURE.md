# Architecture

**Analysis Date:** 2026-03-07

## Pattern Overview

**Overall:** Event-driven modular monolith with plugin-style adapters.

**Key Characteristics:**
- Route all inbound/outbound communication through an internal async queue in `nanobot/bus/queue.py`.
- Keep platform-specific concerns in channel/provider adapters under `nanobot/channels/` and `nanobot/providers/`.
- Centralize orchestration in `AgentLoop` (`nanobot/agent/loop.py`) while delegating specialized responsibilities (memory, skills, cron, heartbeat, subagents).

## Layers

**CLI & Process Entrypoint Layer:**
- Purpose: Parse commands, load config, bootstrap runtime services, and run lifecycle.
- Location: `nanobot/__main__.py`, `nanobot/cli/commands.py`.
- Contains: Typer app, command handlers (`gateway`, `agent`, `status`, provider/channel management), runtime wiring.
- Depends on: `nanobot/config/*`, `nanobot/agent/*`, `nanobot/channels/manager.py`, `nanobot/cron/service.py`, `nanobot/heartbeat/service.py`, `nanobot/bus/queue.py`.
- Used by: Shell users invoking `nanobot` (script from `pyproject.toml`) or `python -m nanobot`.

**Configuration Layer:**
- Purpose: Define typed runtime settings and load/save config from disk.
- Location: `nanobot/config/schema.py`, `nanobot/config/loader.py`.
- Contains: Pydantic models for agents/channels/providers/tools/gateway, provider resolution logic, config migration.
- Depends on: `pydantic`, `pydantic-settings`, provider registry in `nanobot/providers/registry.py`.
- Used by: CLI bootstrap and any runtime component requiring typed settings.

**Messaging/Transport Core Layer:**
- Purpose: Decouple channels from agent processing.
- Location: `nanobot/bus/events.py`, `nanobot/bus/queue.py`.
- Contains: `InboundMessage`, `OutboundMessage`, async inbound/outbound queues.
- Depends on: Python stdlib dataclasses/asyncio.
- Used by: `nanobot/channels/*`, `nanobot/channels/manager.py`, `nanobot/agent/loop.py`, subagent announcements.

**Channel Adapter Layer:**
- Purpose: Integrate external chat platforms while exposing a common interface.
- Location: `nanobot/channels/base.py`, `nanobot/channels/manager.py`, concrete adapters in `nanobot/channels/*.py`.
- Contains: `BaseChannel` contract (`start`, `stop`, `send`), permission guard (`allow_from`), platform clients (Telegram, WhatsApp, Discord, Feishu, Slack, Matrix, etc.).
- Depends on: Bus layer, typed channel config from `nanobot/config/schema.py`, platform SDKs.
- Used by: Gateway runtime in `nanobot/cli/commands.py`.

**Agent Orchestration Layer:**
- Purpose: Execute the conversational/tool loop and persist session state.
- Location: `nanobot/agent/loop.py`.
- Contains: Tool registration, LLM call loop, tool-call execution, progress streaming, slash command handling, memory consolidation triggers, task cancellation.
- Depends on: `nanobot/agent/context.py`, `nanobot/session/manager.py`, `nanobot/agent/tools/*`, provider abstraction, bus events.
- Used by: `gateway` and `agent` commands; cron/heartbeat callbacks via `process_direct`.

**Context/Memory/Skill Composition Layer:**
- Purpose: Build prompt context from identity + templates + memory + skill metadata/content.
- Location: `nanobot/agent/context.py`, `nanobot/agent/memory.py`, `nanobot/agent/skills.py`, templates in `nanobot/templates/`.
- Contains: System prompt assembly, runtime context injection, multimodal message conversion, memory consolidation via virtual tool, skill discovery and availability checks.
- Depends on: Workspace files and provider interface.
- Used by: `AgentLoop` and subagent prompt construction.

**Tool Capability Layer:**
- Purpose: Define executable capabilities exposed to the LLM.
- Location: `nanobot/agent/tools/base.py`, `nanobot/agent/tools/registry.py`, implementations in `nanobot/agent/tools/*.py`.
- Contains: JSON-schema-based tool definitions/validation/casting plus filesystem, shell, web, message, spawn, cron, and MCP wrappers.
- Depends on: Internal services (`CronService`, `SubagentManager`, bus callback) and external libs (`httpx`, `mcp`).
- Used by: `AgentLoop` and `SubagentManager`.

**Provider Abstraction Layer:**
- Purpose: Normalize model APIs to a single `chat` interface.
- Location: `nanobot/providers/base.py`, `nanobot/providers/litellm_provider.py`, `nanobot/providers/custom_provider.py`, `nanobot/providers/azure_openai_provider.py`, `nanobot/providers/openai_codex_provider.py`, `nanobot/providers/registry.py`.
- Contains: `LLMProvider` contract, provider registry metadata, request/response sanitation, model routing/prefixing.
- Depends on: LiteLLM/OpenAI clients and registry-driven metadata.
- Used by: `AgentLoop`, memory consolidation, heartbeat decision step, subagent loop.

**State & Scheduling Services Layer:**
- Purpose: Persist conversations and run asynchronous scheduled background workflows.
- Location: `nanobot/session/manager.py`, `nanobot/cron/service.py`, `nanobot/heartbeat/service.py`.
- Contains: JSONL session persistence, cron store/timer execution, heartbeat decision/execution lifecycle.
- Depends on: Workspace filesystem and provider callbacks.
- Used by: CLI bootstrap, AgentLoop, cron tool, gateway runtime.

**Bridge Sidecar Layer (Node.js):**
- Purpose: Handle WhatsApp Web protocol separately from Python runtime.
- Location: `bridge/src/index.ts`, `bridge/src/server.ts`, `bridge/src/whatsapp.ts`.
- Contains: WebSocket server bound to localhost, bridge token auth handshake, Baileys client.
- Depends on: Node runtime and `@whiskeysockets/baileys`.
- Used by: `nanobot/channels/whatsapp.py` over WebSocket (`bridge_url`).

## Data Flow

**Gateway Channel Message Flow:**

1. `nanobot/cli/commands.py` (`gateway`) loads config, creates `MessageBus`, `AgentLoop`, `ChannelManager`, `CronService`, and `HeartbeatService`.
2. Channel adapters (for example `nanobot/channels/telegram.py`) receive user input and call `BaseChannel._handle_message`, which publishes `InboundMessage` to `MessageBus`.
3. `AgentLoop.run()` in `nanobot/agent/loop.py` consumes inbound messages, resolves session context (`nanobot/session/manager.py`), and builds prompt via `ContextBuilder` (`nanobot/agent/context.py`).
4. Provider (`nanobot/providers/*`) returns either tool calls or final content; `ToolRegistry` executes tools in `nanobot/agent/tools/*`.
5. Agent posts progress/final `OutboundMessage` to `MessageBus`; `ChannelManager._dispatch_outbound()` forwards to matching channel adapter `send()`.
6. Channel adapter sends response to external platform SDK/API.

**Direct CLI Agent Flow:**

1. `nanobot/cli/commands.py` (`agent`) builds `AgentLoop` and optionally starts bus-driven interactive mode.
2. One-shot mode calls `AgentLoop.process_direct()` directly; interactive mode publishes `InboundMessage` through `MessageBus` then consumes outbound for rendering.
3. Session history persists via `SessionManager.save()` in `nanobot/session/manager.py`.

**Scheduled Task Flow (Cron):**

1. Cron jobs are created via `CronTool` in `nanobot/agent/tools/cron.py`.
2. `CronService` in `nanobot/cron/service.py` executes due jobs and invokes callback bound in `nanobot/cli/commands.py`.
3. Callback executes job instruction through `AgentLoop.process_direct()` and may publish outbound messages to user channel.

**Heartbeat Flow:**

1. `HeartbeatService` in `nanobot/heartbeat/service.py` reads `HEARTBEAT.md` from workspace.
2. It calls provider with a virtual `heartbeat` tool schema and expects `skip`/`run` action.
3. On `run`, callback in `nanobot/cli/commands.py` executes tasks through full `AgentLoop` and routes notification to a selected enabled channel.

**State Management:**
- Persist conversation history as per-session JSONL under workspace `sessions/` using `SessionManager` in `nanobot/session/manager.py`.
- Persist long-term memory and searchable history in workspace `memory/MEMORY.md` and `memory/HISTORY.md` via `MemoryStore` (`nanobot/agent/memory.py`).
- Persist cron jobs in JSON (`cron/jobs.json`) via `CronService` (`nanobot/cron/service.py`).
- Keep runtime queues and task registries in-memory (`MessageBus`, `AgentLoop._active_tasks`, subagent task maps).

## Key Abstractions

**Message Bus Contract:**
- Purpose: Isolate agent computation from chat transport implementation.
- Examples: `nanobot/bus/events.py`, `nanobot/bus/queue.py`.
- Pattern: Producer/consumer queues with typed dataclass envelopes.

**Channel Plugin Contract:**
- Purpose: Add a platform adapter without changing agent internals.
- Examples: `nanobot/channels/base.py`, `nanobot/channels/telegram.py`, `nanobot/channels/whatsapp.py`.
- Pattern: Template method + polymorphic adapter (`start/stop/send` + shared `_handle_message`).

**Provider Contract:**
- Purpose: Make model invocation provider-agnostic.
- Examples: `nanobot/providers/base.py`, `nanobot/providers/litellm_provider.py`.
- Pattern: Strategy pattern (`LLMProvider.chat`) plus registry-driven routing in `nanobot/providers/registry.py`.

**Tool Contract + Registry:**
- Purpose: Expose constrained, schema-validated side effects to the model.
- Examples: `nanobot/agent/tools/base.py`, `nanobot/agent/tools/registry.py`.
- Pattern: Plugin registry with schema validation/casting and standardized execution surface.

**Session Aggregate:**
- Purpose: Maintain append-only turn history and consolidation pointer.
- Examples: `nanobot/session/manager.py` (`Session`, `SessionManager`).
- Pattern: Aggregate root persisted as JSONL plus in-memory cache.

## Entry Points

**Python Module Entrypoint:**
- Location: `nanobot/__main__.py`
- Triggers: `python -m nanobot`
- Responsibilities: Delegate directly to Typer app in `nanobot/cli/commands.py`.

**CLI Script Entrypoint:**
- Location: `pyproject.toml` (`[project.scripts] nanobot = "nanobot.cli.commands:app"`)
- Triggers: `nanobot ...`
- Responsibilities: Expose all operational commands (`onboard`, `gateway`, `agent`, `status`, `channels`, `provider`).

**Gateway Runtime Entrypoint:**
- Location: `nanobot/cli/commands.py` (`gateway` command)
- Triggers: `nanobot gateway`
- Responsibilities: Initialize bus/provider/agent/channels/cron/heartbeat and run long-lived async services.

**Agent Runtime Entrypoint:**
- Location: `nanobot/cli/commands.py` (`agent` command)
- Triggers: `nanobot agent`.
- Responsibilities: Start direct or interactive local chat flow with same agent/tool stack as gateway.

**WhatsApp Bridge Entrypoint (Node):**
- Location: `bridge/src/index.ts`
- Triggers: `npm start` in `bridge/`.
- Responsibilities: Start localhost bridge server and WhatsApp client lifecycle.

## Error Handling

**Strategy:** Fail soft at boundaries, propagate user-safe error strings through tool and channel layers, and keep event loop alive.

**Patterns:**
- Wrap most long-running loops with broad exception guards and continue (`nanobot/agent/loop.py`, `nanobot/channels/manager.py`, `nanobot/heartbeat/service.py`).
- Convert provider/tool exceptions to textual error responses rather than hard-crashing (`nanobot/providers/litellm_provider.py`, `nanobot/agent/tools/registry.py`).
- Keep cancellation explicit for stop semantics and shutdown (`nanobot/agent/loop.py`, `nanobot/cron/service.py`, `nanobot/channels/manager.py`).

## Cross-Cutting Concerns

**Logging:** Structured runtime logging via `loguru` across agent, channels, provider, cron, and heartbeat (`nanobot/agent/loop.py`, `nanobot/channels/manager.py`, `nanobot/providers/litellm_provider.py`, `nanobot/cron/service.py`).

**Validation:**
- Config validation via Pydantic schema (`nanobot/config/schema.py`).
- Tool input validation and type casting via base `Tool` class (`nanobot/agent/tools/base.py`).
- URL validation in web fetch tool (`nanobot/agent/tools/web.py`).

**Authentication:**
- Channel-level allowlist gate in `BaseChannel.is_allowed()` (`nanobot/channels/base.py`).
- Provider auth via API key/OAuth selection in config/provider factory (`nanobot/cli/commands.py`, `nanobot/providers/registry.py`).
- WhatsApp bridge token handshake in `bridge/src/server.ts`.

---

*Architecture analysis: 2026-03-07*
