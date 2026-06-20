# Storyteller

A voice-only, web-based storytelling application for children aged 8-12. Children speak to the app and it tells them an interactive story using natural-sounding voices. Stories are shaped by the children's input in real time.

## Features

- **Wake-word activation** — Say "Storyteller" (customizable) to start or interrupt
- **Interactive narration** — Children can pause, redirect, or shape the story by speaking
- **Multiple voices** — Expressive TTS with distinct narrator and character voices
- **Choice-driven** — The app pauses to ask questions and adapts the story to responses
- **Story persistence** — Full story saved as `.txt` files; sessions can be resumed
- **Parent admin** — Simple dashboard to view stories and usage
- **Safety focused** — Age-appropriate content filtering, no data collection, no ads
- **Cost aware** — Usage tracking with optional daily/monthly caps

## Tech Stack

| Layer | Technology |
|---|---|
| **Backend** | Python 3.12 + FastAPI |
| **ASGI Server** | Uvicorn |
| **STT** | faster-whisper (small) |
| **VAD** | Silero VAD |
| **Wake Word** | OpenWakeWord (in-browser via ONNX.js) |
| **LLM** | Google Gemini (gemini-2.0-flash) |
| **TTS** | Google Cloud TTS (Neural2) |
| **Database** | SQLite (WAL mode) |
| **Deployment** | Single Docker container |

## Architecture

```
Device (browser)          Home Server              Cloud APIs
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  Microphone  │◄───────►│  FastAPI     │◄───────►│  Gemini API  │
│  Speaker     │         │  + Whisper   │         │  Google TTS  │
│  OpenWakeWord│         │  SQLite      │         └──────────────┘
│  Minimal UI  │         │  File Store  │
└──────────────┘         └──────────────┘
```

The app runs as a single Docker container on a home server. Audio is streamed via WebSocket, with all STT, LLM, and TTS handled server-side.

## Quick Start

1. Clone the repo and copy `.env.example` to `.env`, filling in your `GOOGLE_API_KEY`.
2. Run with Docker:
   ```bash
   docker compose up
   ```
3. Open `http://localhost:8000` in a browser on any device on your network.

## Repository Structure

```
├── CLAUDE.md           # Development context & conventions
├── architecture.md     # Detailed system architecture
├── specs.md            # Full product specification
└── README.md           # This file
```
