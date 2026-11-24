# AGENTS.md

Project guidelines and engineering standards for `voice-agent-platform`.

## 1. Top Priority: Latency & Performance

- **End-to-end latency is the primary constraint.** Optimize for minimal voice-to-voice delay across transport, VAD/turn detection, STT, LLM, and TTS.
- **Stream everywhere.** Prefer streaming pipelines, non-blocking async I/O, and incremental token/audio processing over buffered or batch steps.
- **Keep overhead low.** Avoid unnecessary abstractions, heavy dependencies, synchronous blocking calls on hot paths, or extra network hops.
- **Measure with built-in telemetry.** Enable and use each framework's native metrics and OpenTelemetry export rather than writing custom timers:
  - **LiveKit:** Use `ChatMessage.metrics`, `EOUMetrics`, `session_usage_updated` / `session.usage`, and OpenTelemetry (`livekit.agents.telemetry.set_tracer_provider()`); avoid deprecated session-level `metrics_collected` and `UsageCollector`.
  - **Pipecat:** Use `PipelineParams(enable_metrics=True, enable_usage_metrics=True)` and `MetricsFrame`, with `enable_tracing=True` and `enable_turn_tracking=True` passed to `PipelineWorker(...)` (paired with `setup_tracing()`)—never pass `enable_tracing` to `PipelineParams`, which silently drops unknown fields in Pipecat 1.0+.
  - Verify there are no regressions in TTFT, TTFB, or turn-taking responsiveness.

## 2. Documentation & Community-First Workflow (MCP + GitHub)

Before creating, modifying, or refactoring any LiveKit or Pipecat code, services, functions, or configuration:

- **Query the docs MCP first** (`livekit-docs` for LiveKit, `pipecat-context-hub` for Pipecat).
- **Check GitHub Issues, Discussions, and Official Forums** for both LiveKit and Pipecat (each project's repository issues/discussions as well as their individual community forums) when investigating edge cases, performance tuning, bugs, or undocumented behaviors—these often contain proven tips, workarounds, and gotchas.
- **Verify exact APIs and parameters** from the retrieved documentation—never guess signatures or rely on deprecated or redundant functions/parameters.
- **Follow current recommended patterns** for streaming, interruptions/barge-in, and resource cleanup as documented by each framework.

## 3. Barge-In, Cancellation & Lifecycle Hygiene

- **Immediate interruption handling.** On user barge-in, immediately cancel in-flight LLM streams, halt TTS synthesis, and flush downstream audio playback buffers using current documented mechanisms:
  - **LiveKit:** Configure `TurnHandlingOptions` / `InterruptionOptions` (and `session.interrupt()` for manual interruption; with realtime models using server-side turn detection, set `turn_detection=None` if disabling interruptions).
  - **Pipecat (1.0+):** Use `PipelineWorker` / `WorkerRunner`, `InterruptionFrame`, and `UserTurnStrategies` inside `LLMUserAggregatorParams` (never deprecated `PipelineTask` / `PipelineRunner`, `StartInterruptionFrame`, or `allow_interruptions`).
- **Clean resource teardown.** Explicitly close WebRTC rooms, SIP sessions, HTTP/WebSocket clients, and background `asyncio` tasks on disconnect or error—never leave orphaned tasks or leaked connections. Use `session.shutdown(drain=True)` / `await session.aclose()` with bounded `ctx.add_shutdown_callback` hooks in LiveKit, and `EndFrame` (graceful) or `CancelFrame` / `await worker.cancel()` (immediate) in Pipecat.

## 4. Async & Event-Loop Safety

- **Never block the audio event loop.** Forbid synchronous disk I/O, blocking HTTP/SDK calls, or heavy CPU work on the `asyncio` event loop. Use async clients (`httpx.AsyncClient`, `aiohttp`, async SDKs) and offload unavoidable CPU-bound work to executors/processes.
- **Keep hot paths lock-free and non-blocking.** Avoid long-held locks or synchronous sleeps inside frame processors, VAD callbacks, and room event handlers.

## 5. Python Tooling & Testing (`uv`)

- **Use Python `>= 3.11` and `uv` for all environment and package management.** Manage dependencies, virtual environments, and script/test execution with `uv` (`uv add`, `uv sync --locked`, `uv run`) rather than `pip` or `poetry`.
- **Always use the latest stable versions.** Target the latest stable releases of `livekit-agents` and `pipecat-ai` (avoid alpha/beta/pre-release pins), verifying current releases before adding or upgrading dependencies and locking them in `uv.lock`.
- **Test pipelines with mocked streams.** Write fast unit and integration tests for voice pipelines by mocking STT, LLM, and TTS streams, verifying interruption/barge-in handling and clean task cancellation.

## 6. Secrets & Configuration

- **Never hardcode secrets.** Keep all API keys, LiveKit/Pipecat credentials, and SIP/telephony tokens in environment variables (`.env` locally, `--env-file` or secret stores in containers/cloud).
- **Keep `.env` gitignored** and maintain an up-to-date `.env.example` with required variable names (no real values).

## 7. Local Sandbox & Docker Deployment

- **Support local dev and local Docker sandboxes.** Ensure agents run both in fast local dev mode (`lk agent dev` / `pipecat.runner.run`—local dev only) and inside a local Docker container (`Dockerfile` using glibc-based images like `ghcr.io/astral-sh/uv:python*-bookworm-slim` or `dailyco/pipecat-base`, non-root user, `UV_COMPILE_BYTECODE=1`, and `uv sync --locked`; Alpine/`musl` is unsupported by LiveKit). Test locally via `docker run --env-file .env` or `docker compose` before deploying to any server.
- **Keep entry points environment-agnostic.** Separate configuration/secrets from agent logic so the exact same code runs identically in local dev, local Docker sandboxes, and production.
