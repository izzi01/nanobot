# External Integrations

**Analysis Date:** 2026-03-07

## APIs & External Services

**LLM & AI Providers:**
- OpenRouter - Gateway LLM routing for multiple model families
  - SDK/Client: `litellm` via `nanobot/providers/litellm_provider.py`
  - Auth: `OPENROUTER_API_KEY`
- Anthropic - Claude model access through LiteLLM routing
  - SDK/Client: `litellm`
  - Auth: `ANTHROPIC_API_KEY`
- OpenAI - GPT model access through LiteLLM routing and direct OpenAI-compatible client paths
  - SDK/Client: `litellm`, `openai` (`nanobot/providers/custom_provider.py`)
  - Auth: `OPENAI_API_KEY`
- Azure OpenAI - Direct Azure deployment endpoint integration
  - SDK/Client: `httpx` in `nanobot/providers/azure_openai_provider.py`
  - Auth: provider config keys in `~/.nanobot/config.json` (`providers.azure_openai.apiKey` + `apiBase`)
- OpenAI Codex - OAuth-based Responses API integration
  - SDK/Client: `oauth-cli-kit` + `httpx` in `nanobot/providers/openai_codex_provider.py`
  - Auth: OAuth token flow (no static API key env var)
- GitHub Copilot - OAuth-based provider option in registry
  - SDK/Client: routed by `litellm` and provider registry in `nanobot/providers/registry.py`
  - Auth: OAuth-based (no static API key env var)
- DeepSeek / Gemini / Zhipu / DashScope / Moonshot / MiniMax / Groq / vLLM / AiHubMix / SiliconFlow / VolcEngine - provider registry-backed integrations
  - SDK/Client: `litellm` and provider registry metadata in `nanobot/providers/registry.py`
  - Auth: env keys defined per provider in `nanobot/providers/registry.py` (e.g., `DEEPSEEK_API_KEY`, `GEMINI_API_KEY`, `DASHSCOPE_API_KEY`, `MOONSHOT_API_KEY`, `MINIMAX_API_KEY`, `GROQ_API_KEY`, `HOSTED_VLLM_API_KEY`)

**Search & Web Data:**
- Brave Search API - web search tool backend
  - SDK/Client: `httpx` in `nanobot/agent/tools/web.py`
  - Auth: `BRAVE_API_KEY`
- Arbitrary HTTP websites - content fetch/extraction for agent tools
  - SDK/Client: `httpx` + `readability-lxml` in `nanobot/agent/tools/web.py`
  - Auth: none by default (supports optional proxy config)

**Chat Platform Integrations:**
- Telegram Bot API - inbound polling/outbound messaging
  - SDK/Client: `python-telegram-bot` in `nanobot/channels/telegram.py`
  - Auth: `channels.telegram.token` in `~/.nanobot/config.json`
- Discord Gateway + REST API - websocket events and message/file sends
  - SDK/Client: `websockets` + `httpx` in `nanobot/channels/discord.py`
  - Auth: `channels.discord.token`
- Slack Socket Mode - event stream and chat/file operations
  - SDK/Client: `slack-sdk` in `nanobot/channels/slack.py`
  - Auth: `channels.slack.botToken` + `channels.slack.appToken`
- Feishu/Lark - websocket long connection and message/media APIs
  - SDK/Client: `lark-oapi` in `nanobot/channels/feishu.py`
  - Auth: `channels.feishu.appId` + `channels.feishu.appSecret`
- DingTalk - stream mode receive + HTTP send/token APIs
  - SDK/Client: `dingtalk-stream` + `httpx` in `nanobot/channels/dingtalk.py`
  - Auth: `channels.dingtalk.clientId` + `channels.dingtalk.clientSecret`
- QQ - bot websocket/event integration
  - SDK/Client: `qq-botpy` in `nanobot/channels/qq.py`
  - Auth: `channels.qq.appId` + `channels.qq.secret`
- Matrix - homeserver sync/media APIs (optional extra)
  - SDK/Client: `matrix-nio[e2e]`, `mistune`, `nh3` in `nanobot/channels/matrix.py`
  - Auth: `channels.matrix.accessToken` (+ `channels.matrix.userId`, `channels.matrix.deviceId`)
- WhatsApp - local Node bridge + Baileys integration
  - SDK/Client: Python side `websockets` in `nanobot/channels/whatsapp.py`; bridge side `@whiskeysockets/baileys` in `bridge/src/whatsapp.ts`
  - Auth: `channels.whatsapp.bridgeToken` (propagated as `BRIDGE_TOKEN` in `nanobot/cli/commands.py`)
- Mochat - Socket.IO + HTTP API integration
  - SDK/Client: `python-socketio`, `httpx` in `nanobot/channels/mochat.py`
  - Auth: `channels.mochat.clawToken`
- Email - IMAP polling + SMTP sending
  - SDK/Client: stdlib `imaplib` / `smtplib` in `nanobot/channels/email.py`
  - Auth: config credentials (`channels.email.imapUsername`/`imapPassword` and `smtpUsername`/`smtpPassword`)

**Protocol/Tooling Integrations:**
- MCP servers (stdio/SSE/streamable HTTP) - dynamic tool federation
  - SDK/Client: `mcp` + `httpx` in `nanobot/agent/tools/mcp.py`
  - Auth: per-server headers/env configured under `tools.mcpServers` in `~/.nanobot/config.json`

## Data Storage

**Databases:**
- Not detected (no ORM or relational/NoSQL client usage in runtime code under `nanobot/`)
  - Connection: Not applicable
  - Client: Not applicable

**File Storage:**
- Local filesystem only
  - Runtime root: `~/.nanobot/` via `nanobot/utils/helpers.py`
  - Config: `~/.nanobot/config.json` via `nanobot/config/loader.py`
  - Session data: JSONL under `~/.nanobot/sessions` managed by `nanobot/session/manager.py`
  - Media downloads/uploads cache: `~/.nanobot/media` used by channel modules (e.g., `nanobot/channels/discord.py`, `nanobot/channels/telegram.py`)
  - Matrix store: `~/.nanobot/matrix-store` via `nanobot/channels/matrix.py`

**Caching:**
- In-process memory caches only (e.g., message dedup buffers in channel modules like `nanobot/channels/whatsapp.py`, `nanobot/channels/qq.py`)
- No Redis/Memcached service detected

## Authentication & Identity

**Auth Provider:**
- Mixed provider-specific auth (no single centralized identity provider)
  - Implementation: API keys and OAuth tokens in provider/channel configs (`nanobot/config/schema.py`, `nanobot/providers/registry.py`, `nanobot/providers/openai_codex_provider.py`)

## Monitoring & Observability

**Error Tracking:**
- None detected (no Sentry/Honeycomb/etc. integration)

**Logs:**
- `loguru` structured app logging across channels/providers/tools (e.g., `nanobot/channels/slack.py`, `nanobot/providers/litellm_provider.py`)
- Bridge logs via Node console + `pino` in `bridge/src/whatsapp.ts`

## CI/CD & Deployment

**Hosting:**
- Local process runtime and Dockerized deployment
  - Container build in `Dockerfile`
  - Compose service definitions in `docker-compose.yml`

**CI Pipeline:**
- Not detected (`.github/workflows/` not present)

## Environment Configuration

**Required env vars:**
- Core override namespace: `NANOBOT_*` (from `ConfigDict(env_prefix="NANOBOT_", env_nested_delimiter="__")` in `nanobot/config/schema.py`)
- Provider/auth vars used in runtime code paths:
  - `OPENROUTER_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `DEEPSEEK_API_KEY`, `GEMINI_API_KEY`, `ZAI_API_KEY`, `ZHIPUAI_API_KEY`, `DASHSCOPE_API_KEY`, `MOONSHOT_API_KEY`, `MOONSHOT_API_BASE`, `MINIMAX_API_KEY`, `HOSTED_VLLM_API_KEY`, `GROQ_API_KEY` (registry in `nanobot/providers/registry.py`)
  - `BRAVE_API_KEY` (`nanobot/agent/tools/web.py`)
  - Bridge runtime vars: `BRIDGE_PORT`, `AUTH_DIR`, `BRIDGE_TOKEN` (`bridge/src/index.ts`)

**Secrets location:**
- Primary secret storage: user-local config file `~/.nanobot/config.json` (`nanobot/config/loader.py`)
- Optional process env vars for providers/tools/bridge
- `.env` files were not detected in repository root scan

## Webhooks & Callbacks

**Incoming:**
- Slack webhook path configurable as `channels.slack.webhookPath` (`/slack/events`) in `nanobot/config/schema.py`, but runtime implementation uses Socket Mode in `nanobot/channels/slack.py`
- No HTTP webhook server endpoints implemented in repo runtime modules; channel ingress is primarily websocket/polling based

**Outgoing:**
- Brave Search API: `https://api.search.brave.com/res/v1/web/search` in `nanobot/agent/tools/web.py`
- Codex Responses SSE: `https://chatgpt.com/backend-api/codex/responses` in `nanobot/providers/openai_codex_provider.py`
- Groq transcription API: `https://api.groq.com/openai/v1/audio/transcriptions` in `nanobot/providers/transcription.py`
- DingTalk token/media/message endpoints in `nanobot/channels/dingtalk.py`
- Discord REST base: `https://discord.com/api/v10` in `nanobot/channels/discord.py`
- Matrix homeserver and media APIs via configured `channels.matrix.homeserver` in `nanobot/channels/matrix.py`
- WhatsApp bridge websocket default `ws://localhost:3001` from `nanobot/config/schema.py` and `bridge/src/server.ts`

---

*Integration audit: 2026-03-07*
