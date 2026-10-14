# Voice Interview Coach

An AI-powered technical interviewer that conducts **real-time voice interviews about your project**. It reads your project documentation and asks informed, specific questions in a natural spoken conversation — challenging vague answers and probing for depth, just like a real interviewer. Built on LiveKit for the voice pipeline and RAG for context-aware questioning.

## What it is

Point it at a PDF describing your project, start an interview, and talk. The agent greets you, asks questions grounded in your docs, follows up when answers are thin, and produces structured feedback at the end. It's useful for practicing project/technical interviews, and as a reference implementation of a production-shaped voice agent with retrieval and cost controls.

## Features

- 🎤 **Real-time voice conversation** through LiveKit's voice pipeline
- 📄 **RAG-powered questions** — the agent reads your project docs to ask specific, informed questions
- 📝 **Live transcription** as you speak
- 🎯 **Intelligent follow-ups** — it challenges vague answers and probes for technical depth
- 📊 **Structured feedback** on strengths and areas to improve
- 🔒 **Capacity management** — controlled concurrency (default max 5 sessions) to cap cost

## Architecture

### Voice pipeline

```
User speaks → VAD detects speech → STT transcribes → LLM processes → TTS speaks → User hears
                                                          │
                                                          ▼
                                                    RAG system
                                              (on-demand doc search)
```

1. **VAD** (Silero) detects when you start and stop speaking (~500ms silence = end of turn).
2. **STT** (AssemblyAI streaming) transcribes your speech.
3. **RAG** — when the agent needs project context, the LLM **calls a `search_project_docs` function tool** that does semantic search over your docs (ChromaDB + OpenAI embeddings); retrieved chunks are injected into the context. Retrieval is on-demand via function calling, not stuffed into every prompt.
4. **LLM** (GPT-4o-mini) generates the next question or follow-up.
5. **TTS** (Cartesia Sonic) synthesizes the reply, streamed back to the browser over LiveKit WebRTC.

### RAG ingestion

```
project_doc_long.pdf
  → chunk (1000 chars, 200 overlap)
  → embed (OpenAI)
  → store in ChromaDB
          │
 query ───┴─► semantic search (k=4) → dedupe (content fingerprint, first 150 chars) → context
```

### Components

```
backend/                     # Python (uv)
├── agent.py                 # main AI agent + voice pipeline, session/concurrency mgmt
├── token_server.py          # FastAPI server issuing LiveKit access tokens
├── tools.py                 # function tools (RAG search, feedback)
├── rag.py                   # vector store + retrieval logic
├── prompts.py               # interviewer system prompts
├── ingest.py                # load docs into the vector DB
└── data/
    ├── project_doc_long.pdf # your project documentation (input)
    └── chroma_db/           # vector store (created by ingest.py)

frontend/                    # React + TypeScript (Vite)
└── src/
    ├── App.tsx              # connection logic
    └── components/          # CallInterface, Transcript, AudioVisualizer, VideoRenderer
```

Three processes cooperate at runtime: the **token server** (auth), the **agent** (the interviewer), and the **frontend** (the call UI). See [DESIGN.md](DESIGN.md) for detailed design decisions and trade-offs.

### Token server endpoints (FastAPI, `:8000`)

| Endpoint | Purpose |
|---|---|
| `GET /livekit-url` | Returns the LiveKit WebSocket URL. |
| `GET /token?room=&username=` | Issues a LiveKit access token (HTTP 429 when at capacity). |
| `GET /capacity-check` | Reports active vs. max sessions. |

### Capacity & cost controls

`MAX_CONCURRENT_SESSIONS = 5`, a 15-minute idle timeout, a 60s activity check, and a 3s empty-room grace period keep a public demo from running up unbounded API bills.

## What stands out

- **RAG via function calling, not prompt-stuffing.** The agent decides *when* it needs project context and fetches it on demand, which keeps prompts lean and answers specific.
- **A real voice pipeline, not a chatbot.** VAD + streaming STT + low-latency TTS over WebRTC make it feel like a live conversation, with barge-in and natural turn-taking.
- **Built for cost from day one.** Concurrency caps, idle timeouts, and a capacity endpoint are first-class, not afterthoughts.
- **Interviewer behavior, not just Q&A.** Prompts are tuned to detect vague answers and push for technical depth, producing structured end-of-session feedback.

## Tech stack

**Backend:** Python 3.11+ (`uv`), LiveKit Agents SDK, OpenAI GPT-4o-mini (+ embeddings), AssemblyAI (STT), Cartesia Sonic (TTS), Silero VAD, LangChain, ChromaDB, FastAPI.
**Frontend:** React + TypeScript, Vite, LiveKit React SDK.

## How to run

### Prerequisites

- Python 3.11+ with the [`uv`](https://github.com/astral-sh/uv) package manager
- Node.js 18+ and npm
- A free [LiveKit Cloud](https://cloud.livekit.io) account
- API keys: OpenAI, and LiveKit (URL, API key, API secret)

### Setup

```bash
# Backend
cd backend
uv sync

# Frontend
cd frontend
npm install
```

Create `backend/.env.local` (template in `backend/.env.example`):

```bash
LIVEKIT_URL=wss://your-project.livekit.cloud
LIVEKIT_API_KEY=your-api-key
LIVEKIT_API_SECRET=your-api-secret
OPENAI_API_KEY=sk-your-openai-key
```

Ingest your documentation — put a PDF at `backend/data/project_doc_long.pdf`, then:

```bash
cd backend
uv run ingest.py
```

### Run (three terminals)

```bash
# Terminal 1 — token server
cd backend && uv run token_server.py

# Terminal 2 — AI agent
cd backend && uv run agent.py dev

# Terminal 3 — frontend
cd frontend && npm run dev
```

Open `http://localhost:5173`, click **Start Interview**, allow microphone access, and answer the agent's questions. Use Chrome or Edge for the best WebRTC support.

## Security notes

This is a development setup. Before any production use, add authentication to the token server, rate limiting, CORS restriction (change `*` to specific domains), input validation, managed secrets, and HTTPS.

## License

For educational purposes. Review the license terms of LiveKit, OpenAI, AssemblyAI, and Cartesia for their respective services.
