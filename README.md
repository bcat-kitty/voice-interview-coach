# Voice Interview Coach

An AI-powered technical interviewer that conducts voice-based interviews about your project. It reads your project documentation and asks informed, specific questions in a natural spoken conversation. Built with LiveKit for real-time voice and RAG (Retrieval-Augmented Generation) for context-aware questioning.

## Features

- 🎤 **Real-time voice interview** — natural conversation through LiveKit's voice pipeline
- 📄 **RAG-powered questions** — the agent reads your project docs to ask specific, informed questions
- 📝 **Live transcription** — see the conversation transcript as you speak
- 🎯 **Intelligent follow-ups** — challenges vague answers and probes for technical depth
- 📊 **Structured feedback** — strengths and areas for improvement
- 🔒 **Capacity management** — controlled concurrency (default max 5 concurrent sessions) to manage cost

## Tech Stack

**Backend:** Python 3.11+ (`uv`), LiveKit Agents SDK, OpenAI GPT-4o-mini, AssemblyAI (STT), Cartesia Sonic (TTS), Silero VAD, LangChain, ChromaDB, FastAPI

**Frontend:** React + TypeScript, Vite, LiveKit React SDK

## Prerequisites

- Python 3.11+ with the [`uv`](https://github.com/astral-sh/uv) package manager
- Node.js 18+ and npm
- A free [LiveKit Cloud](https://cloud.livekit.io) account
- API keys: OpenAI, and LiveKit (URL, API key, API secret)

## Setup

```bash
# Backend
cd backend
uv sync

# Frontend
cd frontend
npm install
```

Create `backend/.env.local` (see `backend/.env.example` for a template):

```bash
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=your-api-key
LIVEKIT_API_SECRET=your-api-secret
OPENAI_API_KEY=sk-your-openai-key
```

Ingest your project documentation — place a PDF at `backend/data/project_doc_long.pdf`, then:

```bash
cd backend
uv run ingest.py
```

## Running

Run three processes simultaneously:

```bash
# Terminal 1 — token server
cd backend && uv run token_server.py

# Terminal 2 — AI agent
cd backend && uv run agent.py dev

# Terminal 3 — frontend
cd frontend && npm run dev
```

Open `http://localhost:5173`, click **Start Interview**, allow microphone access, and answer the agent's questions as you would in a real interview.

## How It Works

**Voice pipeline:**

```
User speaks → VAD detects speech → STT transcribes → LLM processes → TTS speaks → User hears
                                                          ↓
                                                     RAG system
```

1. **VAD** (Silero) detects when you start/stop speaking
2. **STT** (AssemblyAI) transcribes your speech
3. **RAG** — the agent calls a `search_project_docs` tool that does semantic search over your docs (ChromaDB + OpenAI embeddings) on demand
4. **LLM** (GPT-4o-mini) generates contextual follow-up questions
5. **TTS** (Cartesia Sonic) synthesizes the reply, streamed back via LiveKit

Documents are chunked (1000 chars, 200 overlap), embedded into ChromaDB, and deduplicated via content fingerprinting. See [DESIGN.md](DESIGN.md) for architecture and trade-offs.

## Configuration

Key settings live in `agent.py` (concurrency, idle timeout, voice models) and `rag.py` (chunk size, overlap, retrieval `k`, deduplication).

## Security Notes

This is a development setup. Before any production use, add authentication to the token server, rate limiting, CORS restriction, input validation, managed secrets, and HTTPS.

## License

For educational purposes. Review the license terms of LiveKit, OpenAI, AssemblyAI, and Cartesia for their respective services.
