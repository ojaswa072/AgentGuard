# AgentGuard

> **A Reliability and Evaluation Framework for Tool-Using LLM Agents**

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?logo=python)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi)](https://fastapi.tiangolo.com)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)](https://react.dev)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+pgvector-336791?logo=postgresql)](https://postgresql.org)
[![Ollama](https://img.shields.io/badge/Ollama-local--first-000000)](https://ollama.ai)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## What Is AgentGuard?

Most agent demos answer the question **"did it work once?"**

AgentGuard answers something harder and more useful:

> **How reliable is a tool-using LLM agent when the same task is repeated and when controlled failures are injected?**

It is a small, fully-owned, traceable reliability evaluation system — not a Langfuse clone, not a DeepEval replacement, and not a multi-agent framework. It evaluates one reference agent it owns, on one controlled domain, end to end.

**Every metric is traceable to a real stored run. Every LLM-judge score is accompanied by a human-agreement calibration rate, never asserted as ground truth.**

---

## Core Features

| Feature | Description |
|---|---|
| 🤖 **Reference Agent** | Custom hand-written ReAct loop — no LangGraph, fully explainable |
| 🔍 **Execution Tracing** | Every LLM call, retrieval, and tool call stored as structured spans in PostgreSQL |
| 📊 **Deterministic Evaluation** | Task Success, Retrieval Relevance, Tool Selection/Argument Correctness, Latency |
| ⚖️ **Calibrated LLM Judge** | Groundedness judge with mandatory human-agreement calibration report |
| 💥 **Failure Injection** | Tool failure + retrieval failure — injected at a single controlled boundary |
| 🔁 **Repeated Execution** | k=2 trials per item to expose non-determinism (pass^k consistency) |
| 📈 **Reliability Profile** | Per-dimension metric profile — never a single "magic" composite score as the headline |
| 🖥️ **Minimal Dashboard** | Runs list, Trace detail, Metrics summary, Failure comparison |
| 📤 **CSV/JSON Export** | Full results dump for reporting |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│ FRONTEND — React + Vite + Tailwind + Recharts                        │
│  RunsView | TraceDetailView | MetricsDashboard | FailureExperimentView│
└───────────────────────────────┬──────────────────────────────────────┘
                                 │ REST / JSON
┌──────────────────────────────▼──────────────────────────────────────┐
│ BACKEND API — FastAPI                                                 │
│  /runs  /benchmark  /evaluations  /reliability  /export              │
└───────┬───────────────────────────────────────────┬─────────────────┘
        │                                             │
┌───────▼───────────────────────┐       ┌────────────▼────────────────┐
│ AGENT LAYER                    │       │ EVALUATION LAYER             │
│  AgentRuntime (ReAct, max 6)   │       │  TaskEvaluator               │
│  AgentContext (per-run)        │       │  RetrievalEvaluator          │
│  ToolExecutor                  │       │  ToolEvaluator               │
│   ├─ SearchTool (pgvector)     │       │  GroundednessEvaluator (LLM) │
│   ├─ CalculatorTool            │       │  ReliabilityAnalyzer (pass^k)│
│   └─ LookupTool                │       └────────────┬────────────────┘
│  FailureInjector (one point)   │                     │
└───────┬───────────────────────┘                      │
        │                                              │
┌───────▼────────────────────────────────────────────▼─────────────────┐
│ DATA LAYER — PostgreSQL 15 + pgvector                                  │
│  documents | benchmark_items | runs | spans | evaluations              │
└────────────────────────────────────────────────────────────────────────┘
        │
┌───────▼──────────────────────┐
│ PROVIDER LAYER                │
│  LLMProvider (abstract)       │
│   ├─ OllamaProvider (default) │
│   └─ GeminiProvider (P1 opt.) │
└───────────────────────────────┘
```

---

## Quick Start

### Prerequisites

- Docker Desktop (or Docker + Docker Compose)
- [Ollama](https://ollama.ai) installed locally with a 3–4B instruct model (e.g. `phi4-mini`)
- Python 3.11+ and Node 20+ (for local dev without Docker)

### 1. Clone and configure

```bash
git clone https://github.com/your-username/agentguard.git
cd agentguard
cp .env.example .env
# Edit .env — set OLLAMA_MODEL to whatever model you pulled with Ollama
```

### 2. Start with Docker

```bash
docker-compose up --build
```

This starts:
- **PostgreSQL 15 + pgvector** on port `5432`
- **FastAPI backend** on port `8000`
- **React/Vite frontend** on port `5173`

> **Ollama must be running on your host machine** — it uses your GPU, so it stays outside Docker.

### 3. Seed the database

```bash
# Inside the backend container (or in a local venv)
python scripts/seed_db.py
```

### 4. Verify

- Backend health: `GET http://localhost:8000/health`
- Frontend: `http://localhost:5173`
- API docs: `http://localhost:8000/docs`

---

## Local Development (without Docker)

```bash
# Backend
cd backend
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn api.main:app --reload

# Frontend
cd frontend
npm install
npm run dev
```

Run tests:

```bash
cd backend
pytest -v --cov=. --cov-report=term-missing
```

---

## Project Structure

```
agentguard/
├── backend/
│   ├── agent/          # AgentRuntime, AgentContext, ToolExecutor, tools
│   ├── providers/      # LLMProvider interface, OllamaProvider, GeminiProvider
│   ├── tracing/        # TraceCollector, span models
│   ├── evaluation/     # Evaluator base class, all metric implementations
│   ├── failure/        # FailureInjector (single injection point)
│   ├── api/            # FastAPI routers, main app, schemas
│   ├── db/             # SQLAlchemy models, Alembic migrations, session
│   ├── tests/
│   │   ├── unit/       # Per-module unit tests
│   │   └── integration/# End-to-end flow tests
│   ├── Dockerfile
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── api/        # Fetch wrappers for backend endpoints
│   │   ├── components/ # Shared UI components (MetricCard, SpanTimeline, etc.)
│   │   ├── pages/      # RunsView, TraceDetailView, MetricsDashboard, etc.
│   │   └── hooks/      # Custom React hooks
│   └── Dockerfile
├── dataset/
│   ├── corpus/         # Authored knowledge documents (100–400 words each)
│   └── benchmark/      # benchmark_items.json, labeling_notes.md
├── scripts/
│   ├── seed_db.py      # Load corpus + compute embeddings + load benchmark
│   └── run_calibration.py  # Judge vs. human agreement calculation
├── docs/
│   └── adr/            # Architecture Decision Records
├── .env.example
├── .gitignore
├── docker-compose.yml
├── IMPLEMENTATION_PLAN.md
├── ISSUES_BACKLOG.md
├── workflow_diff.md    # Append-only feature change log
└── README.md
```

---

## Evaluation Metrics

| Metric | Type | Description |
|---|---|---|
| **Task Success** | Deterministic | Answer matches labeled `expected_answer` |
| **Retrieval Relevance** | Deterministic | Retrieved doc IDs overlap with labeled `relevant_doc_ids` (precision + recall) |
| **Tool Selection** | Deterministic | Agent used the `expected_tool` |
| **Tool Argument Correctness** | Deterministic | Tool args are valid and schema-correct |
| **Groundedness** | LLM Judge | Final answer supported by retrieved evidence (calibrated against human labels) |
| **Latency** | Deterministic | Wall-clock time for run phases |
| **Consistency (pass^k)** | Derived | All k=2 trials agree on outcome for a benchmark item |

> The primary output is always the **Reliability Profile** (per-metric). A composite score is optional, weighted, and documented — never the headline finding.

---

## Failure Injection

Two controlled failure types are injected at a **single boundary** (`FailureInjector` wraps `ToolExecutor`):

| Type | Description |
|---|---|
| `tool_failure` | Tool raises an exception / returns a malformed result |
| `retrieval_failure` | Retrieval returns empty or irrelevant documents |

Comparing normal vs. failure-injected runs produces **ΔM** — the actual reliability signal.

---

## Benchmark Design

- **40–50 items** authored (not scraped) for one controlled domain
- **4 categories:** `answerable`, `unanswerable`, `ambiguous`, `conflicting`
- **Labeling process:** Two independent annotators label a 10–15 item calibration round, resolve disagreements, then label remaining items consistently (documented in `dataset/benchmark/labeling_notes.md`)
- **Groundedness calibration:** 30–40 run outputs independently human-labeled; agreement rate computed and reported

---

## Research Questions

**Primary:** How does the reliability of a tool-using LLM agent change when the same task is repeated and when controlled failures are introduced?

**Secondary:** How well does an LLM groundedness judge agree with human judgment on a labeled calibration set?

---

## Hypotheses

- **H1** — Single-run success is not a reliable measure of agent quality.
- **H2** — Controlled failure reduces task reliability in a measurable way.
- **H3** — Repeated execution reveals inconsistency hidden by a single pass.
- **H4** — LLM groundedness evaluation is useful but must be calibrated against human labels.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11, FastAPI, SQLAlchemy (async), asyncpg, Alembic |
| Database | PostgreSQL 15 + pgvector |
| Embeddings | sentence-transformers (all-MiniLM-L6-v2, 384-dim, CPU) |
| Primary LLM | Ollama (local, 3–4B quantized instruct model) |
| Optional LLM | Gemini Flash-class (P1 only, never load-bearing for P0) |
| Frontend | React 18, Vite, Tailwind CSS, Recharts |
| Testing | pytest, pytest-asyncio |
| Containerization | Docker Compose |

---

## Development Gates

| Gate | Milestone | Target |
|---|---|---|
| Gate 1 | Agent runs locally | Phase 1 |
| Gate 2 | Tracing works | Phase 2 |
| Gate 3 | Deterministic evaluation works | Phase 3 |
| Gate 4 | Groundedness judge + calibration | Phase 5 |
| Gate 5 | Failure injection works | Phase 6 |
| Gate 6 | Dashboard works | Phase 8 |
| Gate 7 | P0 complete — frozen and demo-ready | Phase 9 |

---

## Research & Methodological Context

This project is a **focused reliability evaluation study**, not a broad AI safety system. The framing is accurate and defensible:

- The contribution is in *benchmark design, controlled failure injection, evaluation pipeline, trace-based consistency analysis, and judge calibration* — not in building a new metrics library or tracing platform.
- Limitations are stated explicitly (see `docs/`): Task Success only applies to items with authored reference answers; the LLM judge is not ground truth; the benchmark is narrow by design.

---

## License

MIT — see [LICENSE](LICENSE).
