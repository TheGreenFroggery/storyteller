# Storyteller — System Architecture

## 1. System Context

```
┌──────────────────────────────────────────────────────────────┐
│                        Home Network                          │
│                                                              │
│   ┌──────────┐     WebSocket      ┌───────────────────────┐  │
│   │ Browser  │◄──────────────────►│  Docker Container      │  │
│   │ (Child)  │  audio + control   │  ┌─────────────────┐  │  │
│   │          │                    │  │  FastAPI Server  │  │  │
│   │ • OWW    │                    │  │  + Whisper       │  │  │
│   │ • MediaR │                    │  └────────┬────────┘  │  │
│   │ • Playbk │                    │           │            │  │
│   └──────────┘                    │           │            │  │
│                                   │   ┌───────┴────────┐  │  │
│   ┌──────────┐                    │   │  Gemini API    │  │  │
│   │ Browser  │◄───────────────────│   │  Google TTS    │  │  │
│   │ (Parent) │ HTTP               │   └────────────────┘  │  │
│   │          │                    │                       │  │
│   │ /admin   │                    │  Internal volumes:    │  │
│   │ ?token=  │                    │  ./stories/  ./logs/  │  │
│   └──────────┘                    │  ./storyteller.db     │  │
│                                   └───────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

## 2. Technology Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Backend** | Python 3.12 + FastAPI | Async-native, WebSocket support, simple deployment |
| **ASGI Server** | Uvicorn | Single worker sufficient for 1-2 sessions |
| **STT** | faster-whisper (small) | CTranslate2 backend, 4x faster than OpenAI Whisper. Int8 quantized, ~2GB RAM. Runs in-process via thread pool. |
| **VAD** | Silero VAD | Lightweight PyTorch model, ~1.5s configurable silence timeout |
| **Wake Word** | OpenWakeWord (browser, ONNX runtime) | Fully open-source, no API key, runs in-browser via ONNX.js |
| **Audio Transport** | WebSocket (RFC 6455) | Binary frames for audio, text frames for JSON control |
| **LLM** | Google Gemini (gemini-2.0-flash) Via `google-genai` SDK | Gemini Plus account gives generous rate limits. Good for creative storytelling. |
| **TTS** | Google Cloud TTS (Neural2) | Cloud (not local) — justified by vastly superior multi-voice expressiveness. Offline fallback via cached emergency stories. |
| **Voice Modulation** | Map child's description → closest Neural2 voice | Child describes desired voice ("deep and scary", "cheerful fairy"), server matches to best-fit Google TTS voice by pitch/rate/gender metadata |
| **Database** | SQLite (via aiosqlite) | WAL mode, zero-config |
| **Session State** | In-memory dict + SQLite checkpoint | SQLite for recovery after restart |
| **Deployment** | Single Docker container | Whisper runs in-process via faster-whisper. No separate service needed. |
| **Parent Admin** | Jinja2 templates, same FastAPI process | Shared-secret token on `/admin` |

## 3. Container Architecture

### 3.1 Single Container (Recommended)

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./stories:/app/stories
      - ./logs:/app/logs
      - storyteller.db:/app/storyteller.db
    environment:
      - GOOGLE_API_KEY=${GOOGLE_API_KEY}
      - STORYTELLER_SECRET=${STORYTELLER_SECRET:-changeme}
      - WHISPER_MODEL=small
    restart: unless-stopped

volumes:
  storyteller.db:
```

### 3.2 Why Single Container?

- **Simpler deployment**: one build, one service to manage
- **Whisper in-process**: `faster-whisper` runs inference in a thread pool (`run_in_executor`), which doesn't block the async event loop. A separate container adds HTTP overhead for no benefit at 1-2 sessions.
- **Resource efficiency**: One Python process sharing RAM (~2GB for Whisper + ~200MB for FastAPI) instead of duplicating the base image.
- **GPU optional**: If a GPU is available (WSL2 CUDA passthrough), Whisper uses it automatically. Same container, no reconfiguration.

## 4. Session Lifecycle & State Machine

### 4.1 States

```
        ┌──────────────────────────────────────────┐
        │                                          │
        v                                          │
   ┌─────────┐  wake word   ┌────────────┐         │
   │  IDLE   │─────────────►│  LISTENING │         │
   │         │              │  (command) │         │
   └─────────┘              └──────┬─────┘         │
        ▲                         │                │
        │                 ┌───────┴────────┐       │
        │                 │  PROCESSING    │       │
        │                 │  (LLM + TTS)   │       │
        │                 └───────┬────────┘       │
        │                         │                │
        │                  ┌──────┴──────┐         │
        │                  │  NARRATING  │─────────┘
        │                  │  (TTS play) │  wake word
        │                  └──────┬──────┘  (interrupt)
        │                         │
        │                  ┌──────┴──────┐
        │                  │  AWAITING   │
        │                  │  CHOICE     │
        │                  └──────┬──────┘
        │                         │  choice received
        │                         └─────────┘
        │
        │  ┌────────────┐
        │  │ COMPLETED  │  (5-10 min / child ends)
        │  └────────────┘
        │         │
        └─────────┘  resume via wake word + "let's continue"
```

- `IDLE`: No active session. Wake word triggers `LISTENING`.
- `LISTENING`: Mic open, VAD running. Audio buffered until silence (~1.5s), then sent to Whisper.
- `PROCESSING`: Whisper transcription → Gemini call → TTS generation.
- `NARRATING`: TTS audio streaming to browser. Wake word → interrupt → `LISTENING`.
- `AWAITING_CHOICE`: App asked a question. VAD runs, response → `PROCESSING`.
- `COMPLETED`: Story ended. Saved to SQLite + .txt file.

### 4.2 Session Object

```python
@dataclass
class Session:
    id: str
    device_id: str
    state: SessionState
    wake_word: str                       # Default "Storyteller"
    narrator_name: str                   # Child-renamed
    narrator_voice: str                  # Mapped Google TTS voice name
    narrator_description: str            # Child's original description ("deep scary")
    character_voices: dict[str, str]     # Character → voice mapping this session
    story_summary: str                   # Rolling summary for context
    recent_dialogue: list[dict]          # Last ~5 exchanges {"role":..., "content":...}
    story_title: str
    audio_buffer: bytearray
    created_at: datetime
    last_activity: datetime
    usage_count: int
```

## 5. WebSocket Protocol

### 5.1 Connection Lifecycle

```
Browser                                Server
   │                                      │
   │───── WebSocket handshake ───────────►│
   │     ws://host:8000/ws?device_id=X   │
   │                                      │
   │◄──── {"type":"connected","session_id":"s1"}────┘
   │                                      │
   │──── binary frame (Opus chunk) ──────►│  while speaking
   │──── binary frame (Opus chunk) ──────►│
   │──── {"type":"vad_end"} ─────────────►│  silence detected
   │                                      │
   │◄─── {"type":"transcript","text":""}──┘  Whisper done
   │                                      │
   │◄─── {"type":"tts_start"} ───────────┘
   │◄─── binary frame (TTS audio) ───────┘
   │◄─── binary frame (TTS audio) ───────┘
   │◄─── {"type":"tts_end"} ─────────────┘
   │                                      │
   │◄─── {"type":"choice","question":""}──┘  app asks child
```

### 5.2 Message Types

**Client → Server (text/JSON):**

| Type | Payload | When |
|---|---|---|
| `wake_word` | `{"device_id":"..."}` | OpenWakeWord detected wake word |
| `command` | `{"cmd":"pause\|continue\|slower\|louder"}` | Child voice command |
| `choice` | `{"text":"open the door"}` | Child answered a question |
| `voice_desc` | `{"text":"a deep scary voice"}` | Child describing desired narrator voice |

**Client → Server (binary):** Opus audio chunk every ~100ms while mic active.

**Server → Client (text/JSON):**

| Type | Payload | When |
|---|---|---|
| `connected` | `{"session_id":"...","state":"..."}` | On connect |
| `transcript` | `{"text":"...","final":true}` | Whisper result |
| `tts_start` | `{"voice":"...","text":"..."}` | Starting TTS segment |
| `tts_end` | `{}` | TTS segment complete |
| `choice` | `{"question":"...","options":[...]}` | App asking child |
| `state_change` | `{"state":"..."}` | State transition |
| `voice_options` | `{"options":["deep","cheerful","mysterious",...]}` | Prompt child to describe voice |
| `error` | `{"code":"...","message":"..."}` | Error |
| `session_end` | `{"reason":"..."}` | Session over |

**Server → Client (binary):** TTS audio chunk (MP3 or WAV, per TTS API output).

### 5.3 Audio Pipeline

```
Browser                              Server
  │                                     │
  │ OpenWakeWord (in-browser ONNX)      │
  │ ┌──────────────────────┐           │
  │ │ Detects "Storyteller" │──wake───►│  (immediate control message)
  │ └──────────────────────┘ word      │
  │                                     │
  │ ── binary (Opus chunk) ───────────►│  accumulates in buffer
  │ ── binary (Opus chunk) ───────────►│
  │ ── binary (Opus chunk) ───────────►│
  │ ── {"type":"vad_end"} ────────────►│
  │                         ┌──────────┴──────┐
  │                         │ Decode Opus→WAV │
  │                         │ (ffmpeg pipe)   │
  │                         └──────────┬──────┘
  │                         ┌──────────┴──────┐
  │                         │ Silero VAD      │
  │                         │ (validate       │
  │                         │  speech bounds) │
  │                         └──────────┬──────┘
  │                         ┌──────────┴──────┐
  │                         │ faster-whisper  │
  │                         │ (small, int8)   │
  │                         │ via thread pool │
  │                         └──────────┬──────┘
  │◄── {"type":"transcript","text":""}──┘
```

## 6. Data Model (SQLite)

```sql
PRAGMA journal_mode=WAL;

CREATE TABLE sessions (
    id              TEXT PRIMARY KEY,
    device_id       TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'active',
    wake_word       TEXT NOT NULL DEFAULT 'Storyteller',
    narrator_name   TEXT,
    narrator_voice  TEXT,
    narrator_desc   TEXT,                -- Child's voice description
    story_title     TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE stories (
    id          TEXT PRIMARY KEY,
    session_id  TEXT NOT NULL REFERENCES sessions(id),
    title       TEXT NOT NULL,
    content     TEXT NOT NULL,
    word_count  INTEGER,
    file_path   TEXT,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE dialogue (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id  TEXT NOT NULL REFERENCES sessions(id),
    role        TEXT NOT NULL,            -- 'child' | 'narrator' | 'character'
    content     TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE usage_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id  TEXT NOT NULL REFERENCES sessions(id),
    service     TEXT NOT NULL,            -- 'llm' | 'tts'
    model       TEXT,
    tokens_in   INTEGER,
    tokens_out  INTEGER,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## 7. Key Technical Decisions

### 7.1 Whisper — In-Process via Thread Pool

```python
# app/stt.py
from faster_whisper import WhisperModel

model = WhisperModel("small", device="cpu", compute_type="int8")

async def transcribe(audio_bytes: bytes) -> str:
    loop = asyncio.get_event_loop()
    segments, info = await loop.run_in_executor(
        None, lambda: model.transcribe(audio_bytes, beam_size=5)
    )
    return " ".join(seg.text for seg in segments)
```

- `run_in_executor` releases the event loop while Whisper processes
- One model instance, reused across sessions (inference is thread-safe)
- `int8` quantization: ~2GB RAM for small model

### 7.2 Voice Modulation (from Child Description)

When the child describes the narrator voice ("a deep scary voice", "a cheerful fairy"), the server maps it to the closest Google Cloud TTS Neural2 voice:

```python
# Voice profile metadata
VOICE_PROFILES = {
    "en-US-Neural2-A": {"gender": "FEMALE", "pitch": "medium", "style": "warm"},
    "en-US-Neural2-D": {"gender": "MALE",   "pitch": "low",    "style": "deep"},
    "en-US-Neural2-F": {"gender": "FEMALE", "pitch": "high",   "style": "cheerful"},
    "en-US-Neural2-J": {"gender": "MALE",   "pitch": "medium", "style": "calm"},
    # ... more
}

def match_voice(description: str) -> str:
    """Use Gemini to extract voice attributes, then cosine-similarity match."""
    # In practice: simple keyword matching for v1
    desc_lower = description.lower()
    if "deep" in desc_lower or "scary" in desc_lower or "big" in desc_lower:
        return "en-US-Neural2-D"
    if "cheer" in desc_lower or "happy" in desc_lower or "fairy" in desc_lower:
        return "en-US-Neural2-F"
    if "calm" in desc_lower or "gentle" in desc_lower or "soft" in desc_lower:
        return "en-US-Neural2-J"
    return random.choice(list(VOICE_PROFILES.keys()))
```

Character voices within the story are differentiated via Gemini-prompted tags + SSML voice switching (same approach as §7.4 below).

### 7.3 LLM: Gemini Integration

```python
# app/llm.py
from google import genai

client = genai.Client(api_key=settings.GOOGLE_API_KEY)

system_prompt = """You are a children's storyteller for ages 8-12.
Rules:
- Stories must be age-appropriate: no violence, horror, or mature themes.
- Keep paragraphs short (2-3 sentences).
- Tag character dialogue with [character:voice_description] markers.
- After each major story beat, you may ask the child a question.
- End the story naturally after 5-10 minutes of narration.
- Output format: plain text with character tags.
"""

async def generate_story(context: list[dict]) -> str:
    response = client.models.generate_content_stream(
        model="gemini-2.0-flash",
        contents=context,
        config=genai.types.GenerateContentConfig(
            system_instruction=system_prompt,
            max_output_tokens=1024,
            temperature=0.9,
        ),
    )
    # Stream response back to caller
    ...
```

Gemini 2.0 Flash is fast, cheap, creative, and available under the Gemini Plus tier.

### 7.4 TTS & Voice Differentiation via SSML

Gemini output with character tags...

```
Luna [narrator] stepped into the dark cave.
[character:deep] "Who goes there?" [end]
Luna [narrator] trembled.
[character:young] "It's me, Luna!" [end]
```

...is parsed into SSML with voice switching:

```xml
<speak>
  <voice name="en-US-Neural2-A">
    Luna stepped into the dark cave.
  </voice>
  <voice name="en-US-Neural2-D">
    <prosody pitch="-2st">Who goes there?</prosody>
  </voice>
  <voice name="en-US-Neural2-A">
    Luna trembled.
  </voice>
  <voice name="en-US-Neural2-F">
    <prosody rate="fast">It's me, Luna!</prosody>
  </voice>
</speak>
```

Each character voice is selected per-session: the first time a character speaks, Gemini's voice descriptor is matched against the voice profile table. The mapping persists for the session.

### 7.5 Internet-Outage Fallback

When all cloud APIs are unreachable:

```
┌─────────────┐
│ 3 failures  │
│  (backoff)  │
└──────┬──────┘
       ▼
┌─────────────────────┐
│ "I'm having trouble │
│ connecting. Let me  │
│ tell you a story I  │
│ know by heart..."   │
└──────────┬──────────┘
           ▼
┌─────────────────────┐
│ Emergency story #3  │  (pre-written, bundled)
│ "The Brave Little   │
│  Kettle"            │
│ Read aloud via      │
│ local TTS (pyttsx3  │
│ or espeak fallback) │
└─────────────────────┘
```

3-5 pre-written stories bundled in the container. Read aloud using `pyttsx3` (offline, lower quality). Story text is still displayed/read.

### 7.6 Error Recovery

| Failure | Mechanism |
|---|---|
| Gemini API timeout | Retry ×3 with exponential backoff. Fallback: "Let me think of another way to tell this..." |
| Google TTS failure | Retry ×1. Fallback: narrator voice only (no character differentiation). |
| Whisper failure | Restart model. "I didn't quite catch that — could you say it again?" |
| SQLite contention | WAL mode handles 1-2 concurrent writers trivially. |
| Browser disconnect | Server keeps session for 60s. Reconnect with same `device_id` resumes it. |
| Power loss | Summary checkpointed to SQLite every exchange. Incomplete story saved as draft. |

## 8. Project Structure

```
storyteller/
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── README.md
│
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app, lifespan, CORS
│   ├── config.py            # Settings (pydantic-settings)
│   ├── session.py           # SessionManager, state machine
│   ├── audio.py             # Opus→WAV decode, Silero VAD
│   ├── stt.py               # faster-whisper wrapper
│   ├── llm.py               # Gemini client
│   ├── tts.py               # Google TTS client + SSML builder
│   ├── ws_handler.py        # WebSocket endpoint + protocol
│   ├── models.py            # Pydantic models
│   ├── db.py                # SQLite setup + queries
│   ├── admin.py             # Parent admin routes
│   └── emergency_stories.py # Bundled offline stories
│
├── templates/
│   ├── admin.html           # Parent dashboard
│   └── story.html           # Story viewer
│
├── stories/                 # Volume-mounted .txt output
├── logs/                    # Volume-mounted logs
└── requirements.txt
```

### 8.1 Dockerfile

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg libopus0 && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app/ ./app/
COPY templates/ ./templates/

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 8.2 Requirements

```
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
websockets>=12.0
aiosqlite>=0.20.0
faster-whisper>=1.1.0
google-genai>=1.0.0
google-cloud-texttospeech>=2.16.0
silero-vad>=5.1.0
pydantic-settings>=2.4.0
jinja2>=3.1.0
ffmpeg-python>=0.2.0
```

## 9. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **Children's voice accuracy** | Whisper degrades on child speech | Test early. Fallback: "Say it again, more clearly." Consider fine-tuning on child speech if needed. |
| **Opus decode complexity** | Audio pipeline breaks | `ffmpeg` pipe is battle-tested. Pure Python `opuslib` as alternative. |
| **TTS latency, voice switching** | Pauses between character dialogue | Pre-generate all segments in parallel. Keep voice connections warm. |
| **SQLite at scale** | Beyond 1-2 sessions | WAL mode sufficient. Migrate to PostgreSQL if ever needed. |
| **Browser echo/feedback** | TTS playback picked up by mic | `echoCancellation: true` in getUserMedia. Cooldown period after TTS stops before accepting speech. |
| **OpenWakeWord accuracy** | False positives/negatives | Tune threshold. Add 3s cooldown after activation. |
| **Gemini rate limits** | Story generation blocked | Queue requests. Fallback to shorter stories. Show "too many stories right now, let's try again later." |

## 10. Out of Scope (v1)

- **Usage caps**: Skipped. Gemini Plus provides sufficient rate protections.
- **Speaker recognition**: Device-based session resumption only.
- **Story illustrations**: v2 candidate.
- **Multi-device story rooms**: v2 candidate.
- **Internationalization / French**: v2 candidate.
- **User authentication**: Shared-secret token + internal network only.
