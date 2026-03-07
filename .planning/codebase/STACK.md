# Technology Stack

**Analysis Date:** 2026-03-07

## Languages

**Primary:**
- Python 3.11+ - Core application, CLI, agent loop, channels, and providers in `nanobot/**/*.py` (entrypoint at `nanobot/cli/commands.py`)

**Secondary:**
- TypeScript (ES2022 target) - WhatsApp bridge service in `bridge/src/*.ts` compiled to `bridge/dist` via `bridge/tsconfig.json`
- Shell - Developer utility script in `core_agent_lines.sh`

## Runtime

**Environment:**
- Python >=3.11 (project requirement in `pyproject.toml`)
- Node.js >=20.0.0 (bridge runtime requirement in `bridge/package.json`)

**Package Manager:**
- Python: `uv`/`pip` workflows (`uv.lock`, install flow in `Dockerfile`)
- Node: `npm` for bridge (`bridge/package.json` scripts)
- Lockfile: present (`uv.lock`); Node lockfile not detected under `bridge/`

## Frameworks

**Core:**
- Typer (`typer>=0.20.0`) - CLI command framework in `nanobot/cli/commands.py`
- Pydantic + pydantic-settings (`pydantic>=2.12.0`, `pydantic-settings>=2.12.0`) - config schema/validation in `nanobot/config/schema.py`
- LiteLLM (`litellm>=1.81.5`) - multi-provider LLM routing in `nanobot/providers/litellm_provider.py`

**Testing:**
- Pytest (`pytest>=9.0.0`) - tests under `tests/`
- pytest-asyncio (`pytest-asyncio>=1.3.0`) - async test support configured in `pyproject.toml`

**Build/Dev:**
- Hatchling - Python package build backend in `pyproject.toml`
- Ruff - linting configuration in `pyproject.toml`
- TypeScript compiler (`tsc`) - bridge build script in `bridge/package.json`
- Docker + Docker Compose - containerization in `Dockerfile` and `docker-compose.yml`

## Key Dependencies

**Critical:**
- `litellm` - unified provider interface and model routing in `nanobot/providers/litellm_provider.py`
- `openai` + `httpx` - direct provider and HTTP client functionality in `nanobot/providers/custom_provider.py`, `nanobot/providers/azure_openai_provider.py`, and channel clients
- `mcp` - Model Context Protocol client integration in `nanobot/agent/tools/mcp.py`
- `websockets` / `websocket-client` / `python-socketio` - realtime channel and bridge connectivity in `nanobot/channels/*.py`

**Infrastructure:**
- `loguru` - logging across runtime modules (e.g., `nanobot/channels/slack.py`, `nanobot/providers/transcription.py`)
- `oauth-cli-kit` - OAuth token acquisition for Codex provider in `nanobot/providers/openai_codex_provider.py`
- `readability-lxml` - web content extraction in `nanobot/agent/tools/web.py`
- `croniter` - scheduling support in `nanobot/cron/service.py`
- Platform SDKs: `python-telegram-bot`, `slack-sdk`, `dingtalk-stream`, `lark-oapi`, `qq-botpy`, optional `matrix-nio[e2e]` for channel integrations
- Bridge-specific: `@whiskeysockets/baileys`, `ws`, `qrcode-terminal`, `pino` in `bridge/package.json`

## Configuration

**Environment:**
- Primary user config file: `~/.nanobot/config.json` (read/write in `nanobot/config/loader.py`)
- Environment override model: `NANOBOT_` prefix with nested delimiter `__` via BaseSettings in `nanobot/config/schema.py`
- Provider credentials and endpoints are mapped through provider registry metadata in `nanobot/providers/registry.py`
- Runtime bridge environment variables: `BRIDGE_PORT`, `AUTH_DIR`, `BRIDGE_TOKEN` consumed in `bridge/src/index.ts`
- Optional tool/service environment keys are read at runtime (e.g., `BRAVE_API_KEY` in `nanobot/agent/tools/web.py`, `GROQ_API_KEY` in `nanobot/providers/transcription.py`)

**Build:**
- Python packaging and tool configs in `pyproject.toml`
- TypeScript build config in `bridge/tsconfig.json`
- Container build and dependency install in `Dockerfile`
- Service orchestration in `docker-compose.yml`

## Platform Requirements

**Development:**
- Python 3.11+ and Node.js 20+ installed
- `uv` (or pip) for Python dependency management
- `npm` for bridge dependency installation/build
- Writable home directory for runtime state under `~/.nanobot/` (config, history, sessions, media, channel state)

**Production:**
- Primary deployment target: long-running gateway/CLI process on Linux/macOS/Windows (`nanobot gateway` / `nanobot agent` from `nanobot/cli/commands.py`)
- Containerized deployment path available via `Dockerfile` and `docker-compose.yml` (`nanobot-gateway` service exposing port `18790`)

---

*Stack analysis: 2026-03-07*
