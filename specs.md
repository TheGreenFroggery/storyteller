# Storyteller — Voice-Interactive Story App (Spec v1)

## Overview
A voice-only, web-based storytelling application for children aged 8-12. Children speak to the app, and it tells them an interactive story using natural-sounding voices. Stories are shaped by the children's input in real time.

## Core Experience
1. The app listens for a **wake word** (default: "Storyteller"). The children can rename it; the new name is persisted.
2. Upon hearing the wake word, the app greets the children and asks what kind of story they'd like.
3. The app offers suggestions if the children are unsure (e.g. "Would you like a story about a brave explorer, a magical kingdom, or a mystery in space?").
4. The children may **choose a narrator voice** from a shortlist, or one is selected at random.
5. The app begins telling the story using **TTS** with expressive, character-differentiated voices.
6. At any point, the children can **interrupt** by speaking the wake word again. The app stops narrating immediately and enters a command-listening mode, awaiting a question or direction change. The wake word alone (without a follow-on command) acts as a pause.
7. The app may also **pause voluntarily** to ask the children questions or offer choices ("What should Luna do next — open the door or climb the wall?"). The app detects the end of the child's response via **Voice Activity Detection (VAD)** with a configurable silence timeout (~1.5s).
8. After **5-10 minutes**, unless the children indicate they want to continue, the app brings the story to a natural close.
9. Children can say the wake word to **resume** a previous story ("Storyteller, let's continue!"). The app resumes the most recent session from that device (device-based; no speaker recognition in v1).
10. The app generates a **story title** shortly after the child describes the desired story (used for the filename), and announces it again at the end.
11. A **2-3 sentence recap** is offered after the title ("Today, Luna and the dragon flew to the Crystal Mountains and discovered...").
12. The full story text is saved as a `.txt` file in a designated folder using the title.

## Voice Commands (beyond wake word)

| Command | Behaviour |
|---|---|
| *Wake word* (start) | Begin a new story session |
| *Wake word* + "let's continue" | Resume the most recent story for this device |
| "Say that again" / "Slower" | Adjust pacing |
| "What happens next?" | Continue narrating |
| "Louder" / "Softer" | Adjust volume |

The app listens **continuously** (barge-in mode). Voice commands are recognised even while TTS is playing — the mic stays active and uses echo cancellation to filter out its own speech.

## Voices

| Role | Details |
|---|---|
| **Narrator** | 3-5 narrator voices (varying tone, pace, gender). Children may choose, or one is randomised. The children can rename the narrator; this name persists across sessions. |
| **Characters** | The LLM tags character dialogue with voice attributes (e.g. "deep voice", "young, cheerful"). TTS renders each character with a distinct voice. |

## Safety & Guardrails
- All LLM responses are filtered for age-appropriateness (8-12).
- No personal data collection beyond what's needed for the session.
- No external links, ads, or upsells.
- Content topics: no violence, horror, or mature themes beyond standard children's content.
- Parent-accessible story logs (plain text files).

## Error Handling & Resilience

| Failure | Response |
|---|---|
| Internet down | "I'm having trouble connecting. Let me try again in a moment." Retries with backoff. After 3 failures, offers a locally cached "emergency story" — a short, pre-written generic story bundled with the app that requires no API calls. |
| STT mishears | "I didn't quite catch that — could you say it again?" Max 3 retries before prompting a different phrasing. |
| LLM inappropriate response | Guardrail filter blocks it. App responds with a safe redirect ("Let me think of something better..."). |
| TTS failure | Retry once, then fall back to a simpler voice. |

## Observability & Logging
- Key events (session start/end, API calls, errors, wake word detection) are logged with timestamps.
- Logs are plain text files, rotated daily, stored alongside story files.
- No personally identifiable information (PII) is logged — only session IDs and event types.
- Logs aid debugging and parent oversight; no external telemetry service is used.

## Architecture

```
Device (browser)          Home Server              Cloud APIs
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  Microphone  │◄───────►│   Web App    │◄───────►│  LLM API     │
│  Speaker     │         │  STT Engine  │         │  TTS API     │
│  Minimal UI  │         │  Session     │         └──────────────┘
└──────────────┘         │  Manager     │
                         │  File Store  │
Device 2 (browser)◄────►│              │
                         └──────────────┘
```

## Concurrent Sessions
- Multiple devices can use the app simultaneously.
- Each session is **fully isolated**: its own story context, wake word listener, TTS stream, and state.
- No shared state between devices (v1). A "story room" concept (multiple devices joining the same story) is a v2 possibility.

## Context Management
- Full conversation history is **not** passed to the LLM on every request.
- A rolling **story summary** is maintained and updated after each exchange.
- Only the summary + recent dialogue are sent as context, keeping costs down and coherence up.

## Technical Requirements

| Component | Preference | Notes |
|---|---|---|
| **STT** | On home server (e.g. Whisper) | Free. Audio streamed from browser to server. Must handle children's voices reasonably well. |
| **LLM** | Cloud API, free/low-cost | e.g. OpenAI GPT-4o-mini, Google Gemini Flash, Mistral. System prompt enforces guardrails. |
| **TTS** | Low-cost cloud service | e.g. Google Cloud TTS (Neural/Studio), Azure Neural TTS, Amazon Polly Neural. Must support multiple voices + SSML for expressiveness. |
| **Wake Word** | Lightweight local detection | Small model (Porcupine, OpenWakeWord) or keyword-spotting. |
| **Hosting** | Internal network only | Accessible via IP + port on home network. No public exposure. |
| **Storage** | Plain text files in a folder | One file per story. Filename: `YYYY-MM-DD_HH-MM_<title>.txt` |

## Cost Awareness
- A **usage counter** tracks LLM and TTS API calls per session and per day.
- A simple **parent dashboard** (minimal web page or CLI) shows:
  - Total API calls / estimated cost
  - Daily usage trend
  - Option to set a daily/monthly cap
- Common phrases and responses are **cached** where possible.

## Languages
- **v1**: English
- **v2**: French (spec does not preclude this)

## Parent Admin
- Saved stories are plain text files — the source of truth.
- A minimal **parent web page** lists all stories with:
  - Filename, timestamp, word count
  - Click to read full text
- Simple shared-secret access in v1 (a configurable token in the URL, e.g. `/admin?token=abc123`). No full auth system — internal network only provides baseline security.

## Future (v2+)
- Story illustrations (AI-generated images alongside narration)
- French support
- Structured "choose your own adventure" mode
- Parent story queue (pre-load topics for kids to discover)
- "Story room" — multiple devices joining the same story
