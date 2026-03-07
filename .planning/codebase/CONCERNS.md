# Codebase Concerns

**Analysis Date:** 2026-03-07

## Tech Debt

**Monolithic channel/CLI modules:**
- Issue: Multiple channel implementations and CLI orchestration are large multi-responsibility files, mixing transport handling, parsing, retry logic, formatting, and policy checks.
- Files: `nanobot/channels/feishu.py`, `nanobot/channels/mochat.py`, `nanobot/channels/matrix.py`, `nanobot/cli/commands.py`
- Impact: High change risk, hard reviewability, and frequent regression surface in channel-specific patches.
- Fix approach: Split each file into protocol client, message parser, policy/authorization, and delivery adapters with per-module tests.

**Inconsistent channel policy semantics in config comments vs runtime behavior:**
- Issue: `QQConfig.allow_from` comment states empty list means public access, but runtime enforces empty allow list as deny-all and startup failure for enabled channels.
- Files: `nanobot/config/schema.py`, `nanobot/channels/base.py`, `nanobot/channels/manager.py`
- Impact: Misconfiguration risk and confusing production behavior when enabling QQ or other channels.
- Fix approach: Align schema docs/comments with runtime policy and add explicit validation tests for every channel config.

**Session persistence rewrites entire history every save:**
- Issue: Each save writes full JSONL metadata + all messages, not append/incremental writes.
- Files: `nanobot/session/manager.py`
- Impact: I/O grows with session size, slower turns over time, and higher corruption blast radius on interrupted writes.
- Fix approach: Introduce append-only write path + periodic compaction, and atomic temp-file replace semantics.

## Known Bugs

**Enabled channel exits on empty allow list (including defaults):**
- Symptoms: Gateway startup aborts with `SystemExit` when a channel is enabled without `allowFrom` entries.
- Files: `nanobot/channels/manager.py`, `nanobot/config/schema.py`
- Trigger: Set any channel `enabled: true` while `allow_from` remains default `[]`.
- Workaround: Set `allow_from` to specific IDs or `[*]` before enabling.

**WhatsApp voice messages are acknowledged but not transcribed:**
- Symptoms: Voice messages become placeholder text rather than user content.
- Files: `bridge/src/whatsapp.ts`, `nanobot/channels/whatsapp.py`
- Trigger: Receive WhatsApp `audioMessage` events.
- Workaround: Ask users to send text/manual transcription.

## Security Considerations

**TLS downgrade fallback in Codex provider:**
- Risk: On certificate validation failure, request retries with `verify=False`, allowing insecure transport fallback.
- Files: `nanobot/providers/openai_codex_provider.py`
- Current mitigation: Warning log is emitted before insecure retry.
- Recommendations: Remove insecure fallback by default; gate behind explicit opt-in config for controlled environments.

**Shell command guard is pattern-based, not capability-based:**
- Risk: Dangerous operations can bypass regex deny patterns and shell execution remains broadly available.
- Files: `nanobot/agent/tools/shell.py`, `nanobot/config/schema.py`, `SECURITY.md`
- Current mitigation: Deny pattern list, timeout, optional workspace restriction.
- Recommendations: Add strict allowlist mode defaults, argument-level parsing, and role-based tool enablement per channel/session.

**Secrets stored as plain text in config:**
- Risk: Credential exposure from host compromise or weak file permissions.
- Files: `nanobot/config/loader.py`, `SECURITY.md`
- Current mitigation: Documentation recommends restrictive file permissions.
- Recommendations: Add keyring integration + startup permission checks for `~/.nanobot/config.json`.

## Performance Bottlenecks

**Global single-message processing lock:**
- Problem: All inbound messages are serialized behind one global lock.
- Files: `nanobot/agent/loop.py`
- Cause: `_dispatch()` wraps processing with one shared `self._processing_lock`.
- Improvement path: Use per-session locks and bounded worker pools for cross-session concurrency.

**Unbounded in-memory message queues:**
- Problem: Inbound/outbound queues have no max size or backpressure strategy.
- Files: `nanobot/bus/queue.py`
- Cause: `asyncio.Queue()` default unbounded allocation.
- Improvement path: Add bounded queues, drop policies, and queue-depth alarms.

**Cron due jobs executed sequentially:**
- Problem: Long-running job execution delays subsequent due jobs.
- Files: `nanobot/cron/service.py`
- Cause: `_on_timer()` awaits each `_execute_job()` in series.
- Improvement path: Execute due jobs concurrently with per-job timeout and bounded concurrency.

## Fragile Areas

**Matrix channel end-to-end flow complexity:**
- Files: `nanobot/channels/matrix.py`
- Why fragile: Handles sync, thread metadata, media download/decrypt/upload, HTML sanitization, typing keepalive, and policy gates in one class.
- Safe modification: Change one concern at a time (policy vs media vs sync), and run targeted Matrix tests after each edit.
- Test coverage: Strong Matrix test presence (`tests/test_matrix_channel.py`), but still high branch complexity.

**Mochat real-time + fallback polling coordination:**
- Files: `nanobot/channels/mochat.py`
- Why fragile: Combines socket transport, polling fallback, cursor persistence, dedupe state, and delayed message buffering.
- Safe modification: Isolate transport changes from message aggregation logic; add regression tests for reconnection and delayed flush behavior.
- Test coverage: No direct Mochat test module detected under `tests/`.

## Scaling Limits

**Agent throughput concurrency limit:**
- Current capacity: Effectively one active message-processing critical section at a time.
- Limit: Multi-channel traffic queues behind single processing lock.
- Scaling path: Shard processing by `session_key` and run multiple workers with cooperative cancellation.

**Queue memory growth under load:**
- Current capacity: No explicit cap on queued inbound/outbound messages.
- Limit: Memory pressure can grow unbounded during producer bursts.
- Scaling path: Introduce queue bounds + channel-side rate limiting + overflow handling.

## Dependencies at Risk

**Release-candidate WhatsApp protocol dependency:**
- Risk: `@whiskeysockets/baileys` pinned to RC version may change behavior/API unexpectedly.
- Impact: Bridge startup/connect/message parsing instability.
- Migration plan: Track stable Baileys release and add compatibility test matrix in `bridge/` before upgrade.

**Unbounded major upgrade surface for OpenAI SDK:**
- Risk: Python dependency `openai>=2.8.0` has no upper bound.
- Impact: Future major/minor API shifts can break provider behavior unexpectedly.
- Migration plan: Add upper bound and scheduled dependency update cadence with provider regression tests.

## Missing Critical Features

**No built-in message rate limiting / abuse controls:**
- Problem: Current system allows unlimited inbound message volume per sender/channel.
- Blocks: Safe multi-user deployment and predictable provider cost control.

**No first-class secret manager integration:**
- Problem: Credentials remain file-based plaintext by default.
- Blocks: Hardened production posture in regulated environments.

## Test Coverage Gaps

**Channel reliability gaps outside Matrix/Email/Feishu helpers:**
- What's not tested: Direct behavior of `telegram`, `whatsapp`, `discord`, `slack`, `qq`, `dingtalk`, and `mochat` channels.
- Files: `nanobot/channels/telegram.py`, `nanobot/channels/whatsapp.py`, `nanobot/channels/discord.py`, `nanobot/channels/slack.py`, `nanobot/channels/qq.py`, `nanobot/channels/dingtalk.py`, `nanobot/channels/mochat.py`
- Risk: Reconnect, auth, and platform-specific formatting regressions can ship unnoticed.
- Priority: High

**Bridge runtime has no direct test coverage:**
- What's not tested: Node bridge auth handshake, websocket command handling, and media download behavior.
- Files: `bridge/src/server.ts`, `bridge/src/whatsapp.ts`, `bridge/src/index.ts`
- Risk: WhatsApp integration failures appear only at runtime.
- Priority: High

**Security-critical exec guard lacks full adversarial test suite:**
- What's not tested: Comprehensive bypass attempts for shell safety patterns.
- Files: `nanobot/agent/tools/shell.py`, `tests/test_tool_validation.py`
- Risk: Pattern-bypass commands could evade guard assumptions.
- Priority: High

---

*Concerns audit: 2026-03-07*
