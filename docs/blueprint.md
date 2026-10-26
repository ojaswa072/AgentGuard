# AgentGuard — Final Project Blueprint

---

## 1. Executive Summary & Core Objectives

**What the system does:** AgentGuard runs a small, fully-owned, tool-using LLM agent against a hand-labeled benchmark, traces every step of execution, scores each run across multiple reliability dimensions (not just final-answer correctness), and deliberately injects controlled failures and repeated executions to measure how much the agent's performance degrades under realistic stress — producing a defensible, calibrated reliability report instead of a single-run success claim.

**Core value proposition:** most agent demos report "it worked" once. AgentGuard reports *how often* it works, *why* it fails when it fails, and *how much it can be trusted* — with every number traceable back to a real, stored run, and every LLM-judge score accompanied by a human-agreement calibration figure rather than asserted as ground truth.

**Non-goals (explicit):** AgentGuard is not a Langfuse/DeepEval replacement, not a multi-agent framework, not a production observability platform, not a general hallucination detector, and does not claim to evaluate arbitrary third-party agents. It evaluates one reference agent it owns, on one domain, end to end.

**Primary objectives, in priority order:**
1. A reference agent that reasons, retrieves, and calls tools — fully traced.
2. Deterministic evaluation of task success, retrieval relevance, and tool-use correctness.
3. One calibrated LLM-judge metric (groundedness), with a reported human-agreement rate.
4. Controlled failure injection (tool failure, retrieval failure) with baseline-vs-failure comparison.
5. Repeated-execution consistency (k=2 baseline) to expose non-determinism.
6. A minimal dashboard presenting the above — never a single composite score as the headline result.

---

## 2. System Architecture

### 2.1 Component Breakdown

```
┌──────────────────────────────────────────────────────────────────────┐
│ FRONTEND — React + Vite + Tailwind + Recharts                        │
│  RunsView | TraceDetailView | MetricsDashboard | FailureExperimentView│
└───────────────────────────────┬────────────────────────────────────┘
                                 │ REST / JSON
┌───────────────────────────────▼────────────────────────────────────┐
│ BACKEND API — FastAPI                                                │
│  /runs  /benchmark  /evaluations  /reliability  /export              │
└───────┬───────────────────────────────────────────┬─────────────────┘
        │                                             │
┌───────▼───────────────┐                 ┌───────────▼─────────────┐
│ AGENT LAYER             │                 │ EVALUATION LAYER         │
│  AgentRuntime (ReAct)   │                 │  TaskEvaluator            │
│  AgentContext            │                 │  RetrievalEvaluator       │
│  ToolExecutor             │                │  ToolEvaluator             │
│   ├─ SearchTool            │               │  GroundednessEvaluator     │
│   ├─ CalculatorTool         │              │   (LLM judge, calibrated)  │
│   └─ LookupTool              │             │  ReliabilityAnalyzer       │
│  FailureInjector               │           │   (k-trial consistency,    │
└───────┬─────────────────────────┘          │    failure-delta compare)  │
        │                                     └───────────┬────────────────┘
┌───────▼─────────────────┐                               │
│ PROVIDER LAYER            │                              │
│  LLMProvider (interface)   │                             │
│   ├─ OllamaProvider (default)│                           │
│   └─ GeminiProvider (optional)│                          │
└───────┬────────────────────────┘                         │
        │                                                   │
┌───────▼───────────────────────────────────────────────────▼────────┐
│ DATA LAYER — PostgreSQL 15 + pgvector                                │
│  documents (vector embeddings) | benchmark_items | runs | spans      │
│  evaluations                                                          │
└────────────────────────────────────────────────────────────────────┘
```

### 2.2 Data Flow (One Run)

```
benchmark_item
   → AgentRuntime.run(item, provider, failure_config)
     → [reasoning step: LLMProvider.complete()]
     → [retrieval step: pgvector similarity search over documents] (via ToolExecutor, wrapped by FailureInjector)
     → [tool step: calculator / lookup] (via ToolExecutor, wrapped by FailureInjector)
     → ... repeat until final_answer or step limit
   → TraceCollector flushes Run + Spans to Postgres
   → EvaluatorEngine reads the closed trace + benchmark_item ground truth
     → writes Evaluation rows (one per metric)
   → Frontend reads /runs, /runs/{id}, /runs/{id}/evaluations for display
```

### 2.3 Trace Entity Relationships

```
Run (1) ──< Span (N, parent_span_id self-referencing, one level deep)
              ├─ span_type = llm_call
              ├─ span_type = retrieval
              └─ span_type = tool_call ──(optional)── Failure (injected fault, if any)
Run (1) ──< Evaluation (N, one per metric_name)
```

---

## 3. Tech Stack

| Layer | Technology | Justification |
|---|---|---|
| Backend framework | Python 3.11, FastAPI | Async-native, typed, minimal boilerplate, easy to test |
| ORM / DB driver | SQLAlchemy (async) + asyncpg | Mature, typed models, avoids hand-rolled SQL in app code |
| Database | PostgreSQL 15 + pgvector extension | One service for both structured trace data and vector search — avoids running a second vector-DB service at this corpus scale (80–150 docs) |
| Embeddings | sentence-transformers (MiniLM-class, ~384-dim) | CPU-friendly, no paid API, optimized for local CPU execution |
| Primary LLM provider | Ollama (local REST) | Zero quota ceiling, offline-capable, guarantees the full benchmark run completes regardless of external service availability |
| Optional LLM provider | Gemini (Flash-class, free tier) | Fast iteration during dev, small cross-provider comparison only — never load-bearing for P0 |
| Frontend framework | React + Vite | Fast dev loop, minimal config |
| Styling | Tailwind CSS | Utility-first styling without heavy UI library maintenance overhead |
| Charts | Recharts | Covers bar/line distribution needs without a heavier charting library |
| Testing | pytest, pytest-asyncio | Standard, async-compatible, works with FastAPI's test client |
| Containerization | Docker Compose | One-command local environment; no Kubernetes — unjustified at this scale |
| CI (P1, optional) | GitHub Actions | Runs the test suite + a benchmark smoke subset on push |

**Explicitly rejected:** dedicated vector database (pgvector is sufficient at this corpus size); LangGraph (hand-written ReAct loop is more explainable and defensible); real OpenTelemetry Collector + Grafana stack; Redis/Kafka (asyncio background task suffices).

---

## 4. Data Models & Schemas

### 4.1 Database Schema (PostgreSQL DDL)

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  metadata JSONB DEFAULT '{}',
  embedding VECTOR(384) NOT NULL
);
CREATE INDEX documents_embedding_idx ON documents USING ivfflat (embedding vector_cosine_ops);

CREATE TABLE benchmark_items (
  id SERIAL PRIMARY KEY,
  question TEXT NOT NULL,
  category TEXT NOT NULL CHECK (category IN ('answerable','unanswerable','ambiguous','conflicting')),
  expected_answer TEXT,
  expected_tool TEXT,
  expected_behavior TEXT,
  relevant_doc_ids INT[] DEFAULT '{}',
  notes TEXT
);

CREATE TABLE runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  benchmark_item_id INT NOT NULL REFERENCES benchmark_items(id),
  provider TEXT NOT NULL,
  model_name TEXT NOT NULL,
  trial_index INT NOT NULL DEFAULT 0,
  failure_config JSONB,
  status TEXT NOT NULL CHECK (status IN ('running','completed','max_steps_exceeded','error')),
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  ended_at TIMESTAMPTZ
);
CREATE INDEX runs_item_provider_idx ON runs(benchmark_item_id, provider);

CREATE TABLE spans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  run_id UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  parent_span_id UUID REFERENCES spans(id),
  span_type TEXT NOT NULL CHECK (span_type IN ('agent_run','llm_call','retrieval','tool_call')),
  name TEXT NOT NULL,
  started_at TIMESTAMPTZ NOT NULL,
  ended_at TIMESTAMPTZ,
  attributes JSONB NOT NULL DEFAULT '{}'
);
CREATE INDEX spans_run_idx ON spans(run_id);

CREATE TABLE evaluations (
  id SERIAL PRIMARY KEY,
  run_id UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  metric_name TEXT NOT NULL,
  score FLOAT NOT NULL,
  is_llm_judge BOOLEAN NOT NULL DEFAULT FALSE,
  judge_model TEXT,
  rationale TEXT,
  human_label FLOAT
);
CREATE INDEX evaluations_run_idx ON evaluations(run_id);
```

No `users` table (single-user local tool, no auth in scope) and no `agents` table (one fully-owned reference agent, not a pluggable multi-agent platform) — deliberate scope exclusions.

### 4.2 API Contracts

**`POST /runs`** — trigger a single run
```json
// Request
{ "benchmark_item_id": 12, "provider": "ollama", "model_name": "phi4-mini", "failure_config": null }
// Response (201)
{ "run_id": "a1b2c3d4-...", "status": "running" }
```

**`GET /runs/{run_id}`** — full run + trace tree
```json
{
  "run_id": "a1b2c3d4-...", "benchmark_item_id": 12, "provider": "ollama",
  "status": "completed", "started_at": "2026-10-04T10:00:00Z", "ended_at": "2026-10-04T10:00:03Z",
  "spans": [
    { "span_id": "sp1", "span_type": "retrieval", "attributes": { "query": "...", "retrieved_doc_ids": [4,9], "scores": [0.81,0.63] } },
    { "span_id": "sp2", "span_type": "tool_call", "attributes": { "tool_name": "calculator", "arguments": {"expression":"12*4"}, "success": true } }
  ]
}
```

**`GET /runs/{run_id}/evaluations`**
```json
[
  { "metric_name": "task_success", "score": 1.0, "is_llm_judge": false },
  { "metric_name": "groundedness", "score": 0.82, "is_llm_judge": true, "judge_model": "phi4-mini", "rationale": "..." }
]
```

**`GET /benchmark`** — list benchmark items (paginated, filterable by `category`)

**`POST /benchmark/run`** — batch-run a benchmark subset
```json
// Request
{ "benchmark_item_ids": [1,2,3], "provider": "ollama", "k": 2, "failure_config": null }
// Response (202, async job)
{ "batch_id": "batch_01J9...", "status": "queued", "total_runs": 6 }
```

**`GET /reliability?benchmark_item_id=12`** — consistency across trials
```json
{ "benchmark_item_id": 12, "k": 2, "agreement_rate": 0.5, "runs": ["run_id_1","run_id_2"] }
```

**`GET /export?format=csv|json`** — full results dump for the report

---

## 5. Risk Analysis & Mitigations

| Risk | Category | Impact | Mitigation |
|---|---|---|---|
| Gemini model name/free-tier terms change mid-build | Availability | Medium | Model name is a config value; Ollama is default, never fallback; Week-1 model-swap smoke test |
| Local 3–4B model produces malformed tool-call JSON | Reliability | High | Test structured-output in Week 1; fail loud (parse_error span) rather than swallow silently |
| LLM judge is biased or inconsistent | Evaluation validity | High | Mandatory human-labeled calibration subset (30–40 runs) with reported agreement rate |
| Benchmark labeling takes longer than estimated | Schedule | High | Fixed floor of 40–50 items; calibration round (10–15 items) before full labeling |
| Failure injection logic scattered across codebase | Maintainability | Medium | Single injection point: `FailureInjector` wraps `ToolExecutor`/`LLMProvider` only |
| Uncontrolled context growth in `AgentContext` | Performance | Low–Medium | Hard step limit (4–6); only doc IDs/scores in context, not full text |
| Dashboard scope creep eats backend time | Schedule | Medium | Dashboard is explicitly minimal-first; polish is P1, gated behind stable backend |
| SQL injection / unsafe query construction | Security | Low | SQLAlchemy parameterized queries only; no raw string-interpolated SQL |
| Composite reliability score misread as ground truth | Methodological validity | Medium | Reliability Profile is always the primary output; composite is optional and explicitly documented |
