# Storyteller — Project Context for AI Agents

## Project Overview

A voice-only, web-based interactive storytelling app for children aged 8-12. Runs on a home network via a single Docker container. Children speak to the app via a browser, and it tells interactive stories using expressive TTS voices, driven by a cloud LLM.

**Status: Design phase — no source code yet.** Spec and architecture docs exist at `specs.md` and `architecture.md`.

---

## Tech Stack (Decided)

| Layer | Technology | Rationale |
|---|---|---|
| Backend | Python 3.12 + FastAPI | Async-native, WebSocket support |
| ASGI Server | Uvicorn (single worker) | Sufficient for 1-2 concurrent sessions |
| STT | faster-whisper (small, int8 quantized) | CTranslate2 backend, ~2GB RAM, in-process via thread pool |
| VAD | Silero VAD | Lightweight PyTorch model, ~1.5s configurable silence timeout |
| Wake Word | OpenWakeWord (browser, ONNX runtime) | Fully offline, no API key, runs in-browser |
| Audio Transport | WebSocket | Binary frames for audio, text frames for JSON control |
| LLM | Google Gemini (gemini-2.0-flash) via `google-genai` SDK | Creative storytelling, generous rate limits with Gemini Plus |
| TTS | Google Cloud TTS (Neural2) | Multi-voice expressiveness; offline fallback via bundled emergency stories |
| Database | SQLite via aiosqlite (WAL mode) | Zero-config, sufficient for 1-2 sessions |
| Deployment | Single Docker container | Whisper runs in-process; no separate services |

---

## Architecture

### Session State Machine

```
IDLE → (wake word) → LISTENING → (VAD end) → PROCESSING → (TTS) → NARRATING
                                                                     ↓ (wake word interrupt)
                                                                   LISTENING
                                                                     ↓ (question asked)
                                                                   AWAITING_CHOICE → (response) → PROCESSING
COMPLETED → (wake word + "let's continue") → LISTENING
```

- **IDLE**: No session. Wake word → LISTENING.
- **LISTENING**: Mic open. Audio buffered until silence (~1.5s), then Whisper.
- **PROCESSING**: Whisper → Gemini → TTS generation.
- **NARRATING**: TTS streaming. Wake word interrupts → LISTENING.
- **AWAITING_CHOICE**: App asked a question. VAD runs, response → PROCESSING.
- **COMPLETED**: Story ended. Saved to SQLite + .txt.

### WebSocket Protocol

**Connection**: `ws://host:8000/ws?device_id=X`

**Client → Server (binary)**: Opus audio chunks every ~100ms while mic active.

**Client → Server (text/JSON)**:
| Type | Payload | When |
|---|---|---|
| `wake_word` | `{"device_id":"..."}` | Wake word detected |
| `command` | `{"cmd":"pause\|continue\|slower\|louder"}` | Voice command |
| `choice` | `{"text":"open the door"}` | Answered a question |
| `voice_desc` | `{"text":"a deep scary voice"}` | Describe narrator voice |

**Server → Client (text/JSON)**:
| Type | Key Fields | When |
|---|---|---|
| `connected` | `session_id`, `state` | On connect |
| `transcript` | `text`, `final` | Whisper result |
| `tts_start` | `voice`, `text` | Starting TTS segment |
| `tts_end` | `{}` | TTS done |
| `choice` | `question`, `options` | Asking child |
| `state_change` | `state` | State transition |
| `error` | `code`, `message` | Error |
| `session_end` | `reason` | Session over |

**Server → Client (binary)**: TTS audio chunks (MP3 or WAV).

### Audio Pipeline (Server-Side)

```
binary (Opus) → ffmpeg pipe (Opus→WAV) → Silero VAD (validate speech) → faster-whisper (transcribe)
```

### Data Model (SQLite)

Four tables: `sessions`, `stories`, `dialogue`, `usage_log`. Sessions checkpointed on every exchange for recovery after power loss. Full schema in `architecture.md:232-275`.

### Planned Project Structure

```
storyteller/
├── docker-compose.yml
├── Dockerfile (python:3.12-slim + ffmpeg, libopus0)
├── .env.example
├── app/
│   ├── main.py              # FastAPI app, lifespan, CORS
│   ├── config.py            # pydantic-settings
│   ├── session.py           # SessionManager, state machine
│   ├── audio.py             # Opus→WAV decode, Silero VAD
│   ├── stt.py               # faster-whisper wrapper (run_in_executor)
│   ├── llm.py               # Gemini client (streaming)
│   ├── tts.py               # Google TTS client + SSML builder
│   ├── ws_handler.py        # WebSocket endpoint + protocol
│   ├── models.py            # Pydantic models
│   ├── db.py                # SQLite setup + queries
│   ├── admin.py             # Parent admin routes (Jinja2)
│   └── emergency_stories.py # Bundled offline stories
├── templates/
│   ├── admin.html
│   └── story.html
├── stories/                 # Volume-mounted .txt output
└── logs/                    # Volume-mounted logs
```

---

## Key Design Decisions

1. **Whisper in-process via `run_in_executor`** — releases event loop, single model instance reused across sessions (thread-safe). Int8 quantization ~2GB RAM.

2. **Voice modulation** — child describes desired voice ("deep scary") → server keyword-matches to closest Google TTS Neural2 voice. Character voices differentiated via Gemini-prompted `[character:...]` tags parsed into SSML with `<voice>` switching.

3. **Context management** — rolling story summary + last ~5 exchanges sent to LLM (not full history). Keeps costs down and coherence up.

4. **Internet-outage fallback** — after 3 retries with backoff, serve pre-written emergency stories via offline `pyttsx3` TTS.

5. **Browser disconnect** — server keeps session for 60s, reconnect with same `device_id` resumes it.

6. **Parent admin** — shared-secret token auth (`/admin?token=...`), lists stories with word count, click to read. Internal network only.

---

## Python & FastAPI Conventions

### Python Standards

- **Type hints everywhere**: Every function signature must have typed parameters and return annotations. Use `from __future__ import annotations` at the top of every module for deferred evaluation.
- **Async-first**: Use `async def` for all I/O-bound functions (DB queries, API calls, file reads). CPU-bound work (Whisper inference) goes in `run_in_executor` — never `asyncio.to_thread` for consistency.
- **pydantic v2 for boundaries**: All API request/response schemas, config, and WebSocket messages use pydantic BaseModel (not dataclasses) for validation and serialization.
- **dataclasses for internals**: Internal state objects (Session, state machine transitions) use `@dataclass` — faster and simpler when validation isn't needed.
- **Path handling**: Always `from pathlib import Path`. Use `Path` objects, never `os.path` or raw strings for file paths.
- **No bare `try/except`**: Catch specific exception types only. Use `try/except/else` or `contextlib.suppress` for expected failures.
- **Structured logging**: Use `logging.getLogger(__name__)` per module. Log structured data as `key=value` in message strings, not as separate arguments. Never log PII (session IDs are OK, child names are not).
- **`if __name__ == "__main__"` guard** on every executable module.
- **Linting**: Ruff for linting and formatting. Single quotes for strings, no trailing commas in multi-line collections, 100-char line limit. Align code with PEP 8. No Option or Union. Use type annotations everywhere if not clearly inferred from context.
- **Line alignment**: Align : and = for multiple lines in lists, dicts, and function call arguments.
- e.g. not
- `
- a: dict = {}
- bb: tuple = ()
- ccc: int | None = None
- `
- but this:
- `
- a  : dict       = {}
- bb : tuple      = ()
- ccc: int | None = None
- `

### FastAPI Patterns

- **Route organization**: Routes live close to their resource — `admin.py` owns `/admin/*`, `ws_handler.py` owns the WebSocket endpoint. `main.py` only creates the `FastAPI()` app, mounts routers, and configures lifespan/middleware.
- **Lifespan, not on_event**: Startup/shutdown logic uses FastAPI's `@asynccontextmanager lifespan` pattern. The deprecated `on_event` is never used.
- **Dependency injection for everything**: Authentication (`Depends(verify_token)`), DB access, session access — always via `Depends()`, never as globals.
- **pydantic-settings for config**: A single `Settings` class via `pydantic-settings.BaseSettings` reads from environment / `.env`. Loaded once at module level and accessed as a singleton.
- **Exception handlers**: Register custom `@app.exception_handler` for `HTTPException`, `WebSocketException`, and validation errors. Never let raw exceptions reach the ASGI layer. Use a consistent JSON error body: `{"error": {"code": "...", "message": "..."}}`.
- **WebSocket error handling**: Wrap the main dispatch loop in `try/except WebSocketDisconnect` + `except Exception` with a text-frame error message before closing. Never leave a WebSocket hanging on unhandled errors.
- **Background work**: Use `BackgroundTasks` for logging/metrics after a response. Use a dedicated `asyncio.Task` (managed via lifespan) for periodic cleanup (session GC, usage rollups).
- **Admin routes**: Use `Depends()` for token verification. Token passed as query param `?token=`. No session cookies — internal-network-only simplifies auth.
- **CORS**: Configured once in `main.py` lifespan. Restrict origins to internal network range if possible (`10.*.*.*`, `192.168.*.*`).
- **Response models**: Every route declares `response_model=` on the decorator. No raw dict returns from route functions.

---

## Gotchas & Non-Obvious Patterns

- **Whisper runs in thread pool**: `loop.run_in_executor(None, lambda: model.transcribe(...))` — this is critical because faster-whisper is CPU-bound and synchronous.
- **Echo cancellation**: `getUserMedia` with `echoCancellation: true`, plus a cooldown period after TTS stops before accepting speech input.
- **No speaker recognition**: Session resumption is device-based only (v1).
- **15.5s silence timeout** is configurable but 1.5s is the default.
- **OpenWakeWord runs in-browser** (ONNX.js), not on the server. The server only receives a `wake_word` message.
- **SSML voice switching** is used for character differentiation within a single TTS response.
- **Gemini output**: plain text with `[character:voice_description]` and `[end]` markers, parsed on server side.
- **Dependency graph**: ffmpeg + libopus0 are apt packages, not pip packages.

---

## Status

- [x] Product spec written (`specs.md`)
- [x] Architecture designed (`architecture.md`)
- [ ] Source code — not yet started
- [ ] Build/run commands — undefined until code exists

When source code is added, run `generate AGENTS.md` to document build, test, and lint commands, code patterns, and conventions.
