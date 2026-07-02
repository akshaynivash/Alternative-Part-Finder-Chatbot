# Engineer's Workday Assistant

A React + FastAPI app with three tools at one workbench — part sourcing, quick
chat, and workday help — each independently useful, none of them the "main"
feature. Part sourcing uses a tiered relaxation matching algorithm to find
alternative electronic parts (fuses); the other two pages cover general chat
and day-to-day assistant work (chat, study Q&A, meal planning, daily task
tracking). An async FastAPI backend (`backend/`) holds all the real logic;
a React/Vite/TypeScript frontend (`frontend/`) talks to it over HTTP.

## Features

- **Part Sourcing** — enter a part ID, get ranked alternatives via a
  5-tier relaxation algorithm (exact match → progressively relaxed current/
  breaking-capacity/mounting/fuse-type constraints). Explanations are
  rule-based by default, with an optional AI-generated mode (Phi-1.5).
- **Quick Chat** — general conversation via a locally running Ollama model.
- **Workday Help** — a LangChain tool-calling agent routes chat to the
  right capability (study assistant over a local FAISS index, daily schedule
  lookup, weekly meal planning) instead of matching on hardcoded keywords.
  Daily tasks are tracked in ChromaDB.

Each page degrades gracefully and independently if its dependencies aren't
installed/configured — the whole app doesn't crash because one model isn't
downloaded or one service isn't running.

## Setup

Requires Python 3.11+ (with [`uv`](https://docs.astral.sh/uv/)) and Node 18+.

```sh
cd backend && uv sync
cd ../frontend && npm install
```

### Download the local model (optional — only needed for AI-mode Part Sourcing explanations)

```sh
cd task-3
python model_install.py
```

This downloads `microsoft/phi-1.5` into `models/phi-1.5`.

### Quick Chat (optional)

Needs a locally running Ollama with any chat model pulled (defaults to
`mistral`; override via `OLLAMA_CHAT_MODEL`):

```sh
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull mistral
```

### Workday Help (optional)

Needs a locally running Ollama with a **tool-calling-capable** model (plain
`llama2` predates Ollama's tool-calling support and won't work reliably).
Defaults to `mistral`, which supports tool-calling and is small enough to
already be on hand; for more reliable tool adherence, use a model built
specifically for it instead:

```sh
curl -fsSL https://ollama.ai/install.sh | sh
ollama pull mistral   # or set OLLAMA_MODEL to llama3.1 / qwen2.5 for more reliable tool routing
```

No cloud account needed — the study-assistant RAG tool uses a local FAISS
index (`backend/app/services/vectorstore.py`), built automatically from PDFs
in `data/pdfs/` (falling back to `Report.pdf`, which already ships in this
repo) the first time it's used.

## Run

Two processes:

```sh
cd backend && uv run uvicorn app.main:app --reload --port 8000
cd frontend && npm run dev   # http://localhost:5173
```

Copy `frontend/.env.example` to `frontend/.env` (and `backend/.env.example`
to `backend/.env`) to override the defaults if needed.

## Data

`data/Partscleaned.csv` is a **synthetic** fuse catalog (see
`scripts/generate_synthetic_dataset.py`) — no public dataset matches this
schema, so one was generated with plausible real-world rating values to make
the app actually runnable.

## Tests

```sh
cd backend && uv run ruff check . && uv run pytest
cd frontend && npm run lint && npm run build
```

Backend tests cover the matching engine, ChromaDB-backed storage, and the API
layer (HTTP status codes, validation) — all run without any external services
or credentials. CI is split by path: `.github/workflows/backend-ci.yml` runs
on `backend/**` changes, `.github/workflows/ci.yml` runs on everything else.
