# AgentGuard — Implementation Master Plan

> **Discipline rule:** P0 must be complete and demo-stable before any P1 work begins. If a feature does not directly help the benchmark, the trace, the evaluation, or the final demo — it is optional and must be deferred.

---

## Executive Summary

AgentGuard is a reliability evaluation framework for a single tool-using LLM agent. The implementation spans **10 sequential phases** (P0) with **4 optional stretch phases** (P1). This plan maps directly to the system architecture development gates and phased delivery roadmap.

**Primary stack:** Python 3.11 · FastAPI · SQLAlchemy (async) · PostgreSQL 15 + pgvector · React 18 + Vite + Tailwind CSS · Ollama (local LLM) · sentence-transformers

**Design philosophy:**
- DRY: shared base classes for evaluators, providers, and tools — no copy-paste logic
- Modularity: each concern lives in exactly one package (`agent/`, `evaluation/`, `failure/`, `tracing/`, `providers/`)
- Optimized execution: async everywhere in the backend; sync only where truly unavoidable (embedding at seed time)
- Minimum viable lines: if a feature can be done in 30 lines, 80 "flexible" lines is a defect

---

## Phase 0 — Project Setup & Environment Verification
**Gate target:** Development environment is reproducible from a fresh clone

### Goals
- Confirm Docker Compose starts PostgreSQL 15 + pgvector cleanly
- FastAPI skeleton with `/health` endpoint
- Database session factory and settings loaded from `.env`
- Ollama smoke test: plain prompt AND structured-JSON prompt both parse correctly
- All secrets in `.env` (gitignored); `.env.example` committed

### Key Files
| File | Purpose |
|---|---|
| `docker-compose.yml` | PostgreSQL 15 + pgvector + backend + frontend orchestration |
| `backend/api/main.py` | FastAPI app factory, lifespan, CORS, router registration |
| `backend/db/config.py` | `pydantic-settings` `Settings` class — single source of all env vars |
| `backend/db/session.py` | Async SQLAlchemy engine + session factory |
| `scripts/smoke_ollama.py` | CLI script: plain prompt + structured-JSON prompt test |
| `.env.example` | Template for all required env vars |

### DRY/Optimization Notes
- Settings are loaded once at startup via `pydantic-settings` — no `os.getenv()` scattered across files
- Database session is a dependency-injected `AsyncSession` via FastAPI `Depends` — never a global singleton
- The Ollama smoke test runs the structured-JSON check specifically, not just "does it respond" — per blueprint risk row 2

### Acceptance Criteria
- `docker-compose up` starts Postgres; `GET /health` returns `{"status": "ok"}`
- Running `python scripts/smoke_ollama.py` prints PASS for both plain and JSON prompts
- `pytest backend/tests/unit/test_health.py` passes

---

## Phase 1 — Reference Agent (ReAct Loop)
**Gate target:** Gate 1 — Agent runs locally

### Goals
Build the minimal working agent: provider abstraction, three tools, hand-written ReAct reasoning loop. No persistence yet.

### Key Files
| File | Purpose |
|---|---|
| `backend/providers/base.py` | `LLMProvider` abstract interface: `complete(messages) -> str` |
| `backend/providers/ollama.py` | `OllamaProvider`: HTTP call to Ollama REST API via `httpx` |
| `backend/providers/gemini.py` | `GeminiProvider` stub (optional, raises `NotImplementedError` until P1) |
| `backend/agent/context.py` | `AgentContext` dataclass — per-run state, no cross-run memory |
| `backend/agent/tools/search.py` | `SearchTool` — stub returning empty list (replaced in Phase 4) |
| `backend/agent/tools/calculator.py` | `CalculatorTool` — safe `eval` wrapper for arithmetic |
| `backend/agent/tools/lookup.py` | `LookupTool` — structured key-value lookup from a dict |
| `backend/agent/executor.py` | `ToolExecutor` — dispatches tool calls, wraps errors cleanly |
| `backend/agent/runtime.py` | `AgentRuntime` — ReAct loop, step limit enforcement, prompt building |
| `scripts/run_agent.py` | CLI: run one question through the agent, print the trace to stdout |

### Agent Loop Contract
```
while step_index < MAX_STEPS:
    action = llm_decide(context)       # → {"action": "tool"|"answer", "tool": ..., "args": ..., "answer": ...}
    if action == "answer":
        break
    result = tool_executor.run(action)
    context.append(result)
step_index += 1
```

### DRY/Optimization Notes
- `LLMProvider` is a protocol/ABC — `AgentRuntime` calls `provider.complete()` only; never imports `OllamaProvider` directly
- Tool schema is a `TypedDict` — small, no extra library
- The step limit is enforced by a `for` loop with a `break`, not a `while True` — eliminates the unbounded loop risk
- `CalculatorTool` uses `ast.literal_eval` + operator dispatch (not raw `eval`) to prevent code injection

### Acceptance Criteria
- 3 sample questions run end-to-end via CLI, each terminates with `final_answer` or `max_steps_exceeded`
- No unhandled exceptions on tool error (tool failure returns an error result, loop continues)
- `pytest backend/tests/unit/test_tools.py` passes (calculator, lookup edge cases)

---

## Phase 2 — Tracing Infrastructure
**Gate target:** Gate 2 — Tracing works

### Goals
Every agent step is stored as a `Span` in PostgreSQL. A `Run` record wraps the whole execution. The trace is queryable via API.

### Key Files
| File | Purpose |
|---|---|
| `backend/db/models.py` | SQLAlchemy ORM models: `Run`, `Span`, `Evaluation`, `BenchmarkItem`, `Document` |
| `backend/db/migrations/` | Alembic `env.py` + `versions/` — one migration per schema change |
| `backend/tracing/collector.py` | `TraceCollector` — buffers span dicts, flushes to DB on `close()` |
| `backend/tracing/models.py` | Pydantic span/run schemas for serialization |
| `backend/api/routers/runs.py` | `POST /runs`, `GET /runs`, `GET /runs/{id}` |
| `backend/api/schemas.py` | Request/response Pydantic models |

### Trace Schema (per blueprint Section 4.1)
```
Run → many Spans (flat list, parent_span_id links one level deep)
Span types: agent_run | llm_call | retrieval | tool_call
attributes: JSONB (gen_ai.*-style keys)
```

### DRY/Optimization Notes
- `TraceCollector` is injected into `AgentRuntime` — the runtime does not know about the DB directly
- Spans are buffered in memory during a run, flushed in one DB transaction on `close()` — avoids per-step round-trips
- `attributes` uses `gen_ai.*` key naming (OTel-GenAI-inspired) for portability without a real OTel collector
- A run that crashes mid-execution has its `status` set to `error` in a `try/finally` — never left as `running`

### Acceptance Criteria
- `POST /runs` creates a run + spans; `GET /runs/{id}` returns the full trace tree
- A 3-step run produces exactly the expected number of span rows in Postgres
- A run that crashes mid-execution shows `status=error` in the DB
- `pytest backend/tests/integration/test_tracing.py` passes

---

## Phase 3 — Deterministic Evaluators
**Gate target:** Gate 3 — Deterministic evaluation works

### Goals
Five metrics computed against a closed run+trace, without any LLM call. Results stored in the `evaluations` table.

### Key Files
| File | Purpose |
|---|---|
| `backend/evaluation/base.py` | `Evaluator` ABC: `evaluate(run, item) -> EvaluationResult` |
| `backend/evaluation/task.py` | `TaskEvaluator` — exact + normalized string match against `expected_answer` |
| `backend/evaluation/retrieval.py` | `RetrievalEvaluator` — precision/recall of retrieved vs. labeled `relevant_doc_ids` |
| `backend/evaluation/tool.py` | `ToolEvaluator` — selection correctness (exact match) + argument schema validation |
| `backend/evaluation/latency.py` | `LatencyEvaluator` — wall-clock ms from run start to end, per-phase breakdown |
| `backend/evaluation/engine.py` | `EvaluatorEngine` — runs all evaluators on a closed run, writes `Evaluation` rows |
| `backend/api/routers/evaluations.py` | `POST /runs/{id}/evaluate`, `GET /runs/{id}/evaluations` |

### Evaluator Interface
```python
class EvaluationResult(TypedDict):
    metric_name: str
    score: float          # 0.0–1.0 for all except Latency (ms)
    rationale: str
    is_llm_judge: bool

class Evaluator(ABC):
    @abstractmethod
    async def evaluate(self, run: Run, item: BenchmarkItem) -> EvaluationResult: ...
```

### DRY/Optimization Notes
- One base class, five concrete implementations — zero duplicated score-storage logic
- `EvaluatorEngine` runs all evaluators in a single pass and writes all results in one DB transaction
- `TaskEvaluator` normalizes both strings (strip, lowercase, remove punctuation) before comparing — handles minor formatting differences without an LLM
- `RetrievalEvaluator` returns `(precision + recall) / 2` — the harmonic mean (F1) is a P1 refinement

### Acceptance Criteria
- 3 fixture traces with known scores produce correct outputs for all 5 evaluators
- Evaluators handle `max_steps_exceeded` runs (no spans) gracefully — return `score=0.0` with a rationale explaining why
- `pytest backend/tests/unit/test_evaluators.py` passes

---

## Phase 4 — Benchmark Dataset & Real Retrieval
**Gate target:** Complete benchmark pipeline

### Goals
Real pgvector similarity search replaces the stub. A 40–50 item benchmark is authored, labeled, and loaded. The corpus of 80–150 documents is seeded.

### Key Files
| File | Purpose |
|---|---|
| `dataset/corpus/*.md` | Authored knowledge documents (100–400 words each, one per file) |
| `dataset/benchmark/benchmark_items.json` | 40–50 labeled benchmark items |
| `dataset/benchmark/labeling_notes.md` | Labeling process, calibration disagreements, edge-case decisions |
| `scripts/seed_db.py` | Loads corpus → computes embeddings → inserts documents; loads benchmark items |
| `backend/agent/tools/search.py` | REPLACE stub: real pgvector cosine similarity search, top-k |
| `backend/api/routers/benchmark.py` | `GET /benchmark`, `POST /benchmark/run` |

### Retrieval Implementation
```python
# pgvector cosine similarity query (SQLAlchemy ORM)
results = await session.execute(
    select(Document)
    .order_by(Document.embedding.cosine_distance(query_embedding))
    .limit(top_k)
)
```

### Benchmark Item Schema
```json
{
  "id": "Q010",
  "question": "What is the refund deadline for hostel fee requests?",
  "category": "answerable",
  "expected_answer": "30 September",
  "expected_tool": "search",
  "expected_behavior": "answer",
  "relevant_doc_ids": [12, 18],
  "notes": "Answer appears in refund policy and fee notice."
}
```

### Benchmark Categories
| Category | Description | Task Success scoring |
|---|---|---|
| `answerable` | Has a single defined correct answer | Match against `expected_answer` |
| `unanswerable` | Answer is not in the corpus | Agent should abstain or state "not found" |
| `ambiguous` | Multiple defensible answers exist | Score process metrics, not `expected_answer` match |
| `conflicting` | Corpus contains contradictory information | Score process metrics; expected behavior is to flag conflict |

### DRY/Optimization Notes
- `seed_db.py` is idempotent: truncate-and-reload on every run — never hand-edit seeded rows in DB
- Embedding model is imported once as a module-level singleton in `seed_db.py` — no repeated model loads
- `relevant_doc_ids` is validated against real DB document IDs at seed time — fails loud if any reference is broken

### Acceptance Criteria
- `python scripts/seed_db.py` runs without error and prints document count + embedding dimension
- Spot-checking 5 benchmark items: the labeled `relevant_doc_ids` contain the document that answers the question
- A real retrieval query for a benchmark item question returns the expected top document in position 1 or 2
- `pytest backend/tests/integration/test_retrieval.py` passes

---

## Phase 5 — Groundedness Judge + Human Calibration
**Gate target:** Gate 4 — Groundedness judge works (calibrated)

### Goals
One LLM-judge metric: does the final answer cite the retrieved evidence? Calibrated against 30–40 human-labeled outputs. Agreement rate recorded and never elided.

### Key Files
| File | Purpose |
|---|---|
| `backend/evaluation/groundedness.py` | `GroundednessEvaluator` — judge prompt + `LLMProvider.complete()` + response parsing |
| `backend/evaluation/judge_prompt.py` | The judge prompt template (separate file for easy iteration) |
| `scripts/run_calibration.py` | Loads 30–40 run outputs, runs judge, computes agreement vs. CSV human labels |
| `dataset/benchmark/calibration_labels.csv` | Human labels: `run_id, human_label (0|0.5|1)` |

### Judge Prompt Contract
```
System: You are evaluating whether an AI assistant's answer is supported by the evidence it retrieved.
User:
  Evidence: {retrieved_documents_text}
  Answer: {final_answer}
  Is the answer supported by the evidence?
  Respond with JSON only: {"label": "SUPPORTED"|"PARTIALLY_SUPPORTED"|"UNSUPPORTED", "score": 0.0-1.0, "reason": "..."}
```

### Calibration Protocol
1. Run the judge on 30–40 completed run outputs
2. Two independent human annotators label the same outputs: `SUPPORTED / PARTIALLY_SUPPORTED / UNSUPPORTED`
3. Annotators do not view the judge's output before completing their labels
4. Run `scripts/run_calibration.py` to compute agreement percentage (and optionally Cohen's kappa)
5. Record the agreement rate in `agentcontext.md` Section 3 and technical documentation

### DRY/Optimization Notes
- Judge prompt is in a single file (`judge_prompt.py`) — one edit point if the prompt needs iteration
- `GroundednessEvaluator` inherits the same `Evaluator` base class — no special-cased storage logic
- Score mapping: `SUPPORTED → 1.0`, `PARTIALLY_SUPPORTED → 0.5`, `UNSUPPORTED → 0.0`
- JSON parse failure (malformed judge output) returns `score=0.0` with rationale `"parse_error"` — never crashes

### Acceptance Criteria
- `GroundednessEvaluator` returns a valid `EvaluationResult` for 10 test run fixtures
- Judge output JSON always parses (test malformed output handling)
- `scripts/run_calibration.py` produces a printed agreement rate (e.g., `Judge agreement: 78% (31/40 items)`)
- Agreement rate is written to `agentcontext.md` (not just console)
- `pytest backend/tests/unit/test_groundedness.py` passes

---

## Phase 6 — Failure Injection
**Gate target:** Gate 5 — Failure injection works

### Goals
Two controlled failure types injected at a single boundary. Baseline vs. failure-injected runs are comparable. Failures are visible in the trace.

### Key Files
| File | Purpose |
|---|---|
| `backend/failure/injector.py` | `FailureInjector` — wraps `ToolExecutor` calls; applies configured failure mode |
| `backend/failure/modes.py` | `FailureMode` enum: `NONE`, `TOOL_FAILURE`, `RETRIEVAL_FAILURE` |
| `backend/failure/config.py` | `FailureConfig` dataclass: `mode`, `target_tool`, `probability` |

### Failure Injection Contract
```python
class FailureInjector:
    def __init__(self, config: FailureConfig | None): ...

    def apply(self, tool_name: str, result: ToolResult) -> ToolResult:
        """Called after tool.execute(), before result enters AgentContext.
        Returns a modified result if failure is configured, else returns result unchanged."""
```

### Failure Types
| Type | What happens | Trace record |
|---|---|---|
| `TOOL_FAILURE` | Tool result is replaced with `{"error": "tool_timeout", "success": false}` | `attributes.failure_injected = true`, `attributes.failure_mode = "tool_failure"` |
| `RETRIEVAL_FAILURE` | Retrieval result is replaced with empty list `[]` | `attributes.failure_injected = true`, `attributes.failure_mode = "retrieval_failure"` |

### DRY/Optimization Notes
- `FailureInjector.apply()` is called in exactly one place in `ToolExecutor` — grep-verified single injection point
- `failure_config=None` bypasses all injection with zero overhead (no conditional branching in the agent loop)
- Injected failures are tagged `failure_injected=true` in the span attributes — never silently indistinguishable from real bugs
- The `POST /runs` request accepts `failure_config` as a nullable JSON field — no separate endpoint

### Acceptance Criteria
- Running 5 items with `failure_config=None` and then with each failure type: trace shows `failure_injected=true` on the relevant span
- Downstream evaluation scores move (e.g., `task_success` drops from 80% to <40% under `TOOL_FAILURE`)
- Grepping the entire codebase for `failure` shows only `backend/failure/` and the one call site in `ToolExecutor`
- `pytest backend/tests/unit/test_failure_injector.py` passes (8+ test cases including edge cases)

---

## Phase 7 — Repeated Execution & Reliability Analyzer
**Gate target:** pass^k consistency metric

### Goals
Run k=2 trials per benchmark item (same item, same condition, different `trial_index`). Compute per-item and aggregate pass^k consistency.

### Key Files
| File | Purpose |
|---|---|
| `backend/evaluation/reliability.py` | `ReliabilityAnalyzer` — reads existing runs, computes pass^k per item |
| `backend/api/routers/reliability.py` | `GET /reliability?benchmark_item_id=&condition=` |

### pass^k Formula (from τ-bench)
```
pass^k = (items where ALL k trials succeed) / (total items with exactly k trials)

Success definition per category:
  answerable:    task_success score = 1.0 on all k trials
  unanswerable:  agent abstained on all k trials
  ambiguous:     same tool selected on all k trials (process consistency)
  conflicting:   same tool selected on all k trials
```

### DRY/Optimization Notes
- `ReliabilityAnalyzer` is purely read-only — it never triggers new runs (single responsibility)
- The computation is a SQL GROUP BY query + Python aggregate, not a per-run loop
- k=1 returns `None` with a rationale string — never raises, never divides by zero

### Acceptance Criteria
- One benchmark item with 2 trial runs: computed consistency matches hand-calculation
- k=1 returns `None` without erroring
- `GET /reliability` with a batch of 15 items returns per-item and aggregate consistency
- `pytest backend/tests/unit/test_reliability.py` passes

---

## Phase 8 — Minimal Dashboard
**Gate target:** Gate 6 — Dashboard works

### Goals
Four views in React. All metric computation happens in the backend — frontend only renders. Every view handles empty/loading/error states.

### Pages
| Page | Route | Backend endpoint |
|---|---|---|
| Runs List | `/runs` | `GET /runs` (paginated, filterable by `condition`, `status`) |
| Trace Detail | `/runs/:id` | `GET /runs/:id` (full span tree) |
| Metrics Summary | `/metrics` | `GET /runs/:id/evaluations` aggregated across a batch |
| Failure Experiment | `/compare` | Two runs side-by-side, ΔM displayed |

### Component Hierarchy
```
App
 ├── RunsView
 │    ├── RunFilter (condition, status)
 │    └── RunCard (question preview, pass/fail badge, metric scores)
 ├── TraceDetailView
 │    ├── SpanTimeline (ordered spans, duration bars)
 │    └── SpanDetail (expandable attributes JSON)
 ├── MetricsDashboard
 │    ├── MetricCard (task_success, retrieval_relevance, tool_correctness, groundedness)
 │    └── ReliabilityBarChart (Recharts)
 └── FailureExperimentView
      ├── RunSelector (pick two run IDs)
      └── DeltaMetricTable (ΔM per metric, colored red if degraded)
```

### DRY/Optimization Notes
- `MetricCard` is a single reusable component — all four metric cards are rendered from the same component with different props
- One shared `useApi(url)` hook handles fetch + loading + error state — no duplicate fetch logic in each page
- No metric computation in JavaScript — all numbers come from backend API responses
- Animations: fade-in on mount, 300ms, `prefers-reduced-motion` respected via Tailwind's `motion-safe:` prefix

### Acceptance Criteria
- All four views render with real seeded data — every number matches a direct DB query
- Empty state shown when no runs exist; loading spinner shown while fetching; error message shown on API failure
- `MetricCard` re-renders correctly for all 5 metric names
- No metric computation in `*.jsx` files (grep-verified)

---

## Phase 9 — Integration, Hardening & Final Freeze
**Gate target:** Gate 7 — P0 complete

### Goals
Full end-to-end run of the complete benchmark. All test coverage gaps filled. Export endpoint. Scope frozen. `p0-frozen` tag.

### Key Files
| File | Purpose |
|---|---|
| `backend/api/routers/export.py` | `GET /export?format=csv\|json` — full results dump |
| `scripts/run_full_benchmark.py` | Orchestrates full 40–50 item × k=2 × normal + failure conditions |

### Final Validation Checklist
- [ ] Full benchmark (40–50 items × Ollama × k=2) runs end-to-end without error
- [ ] Failure injection subset (10 items × 2 failure types) runs and metrics degrade as expected
- [ ] All 4 dashboard views show correct data from a freshly seeded DB
- [ ] `GET /export?format=csv` produces a parseable CSV with all run + evaluation data
- [ ] `pytest -v` passes 100% — zero skipped, zero xfail hiding real issues
- [ ] Cold-start `docker-compose up` on a clean checkout works end-to-end
- [ ] `git tag p0-frozen` applied

---

## Stretch Goals (P1 — Only After Gate 7 Passes)

> **P1 rule:** Do not begin any of these until the `p0-frozen` tag exists and the full P0 demo has been rehearsed successfully.

| Stretch Goal | Description | Value |
|---|---|---|
| **Benchmark expansion** | Grow from 40–50 to 80+ items; add more varied categories | Stronger statistical confidence in reliability findings |
| **Third failure type** | Corrupted tool result (malformed JSON response from tool) | Richer failure taxonomy |
| **Provider comparison** | Run a small Gemini subset on 10–15 controlled items | Cross-provider reliability comparison section in report |
| **k=3 consistency** | Upgrade repeated-execution trials from k=2 to k=3 | Tighter consistency bound |
| **Judge calibration expansion** | Grow calibration set from 30–40 to 60–80 outputs | Lower uncertainty on reported agreement rate |
| **CI pipeline** | GitHub Actions: run smoke tests + benchmark subset on every push | Prevents regressions during P1 work |
| **Dashboard polish** | Better charting, waterfall latency view, mobile responsiveness | Demo quality |
| **Composite score** | Optional weighted composite (see blueprint Section 28); displayed below the profile, never replacing it | Convenience summary for report |

---

## Adherence to Key Principles

| Principle | How This Plan Enforces It |
|---|---|
| **DRY** | Single `Evaluator` base class; single `LLMProvider` interface; one `FailureInjector` call site; one `useApi` hook |
| **Modularity** | Each package has one responsibility: `agent/`, `tracing/`, `evaluation/`, `failure/`, `providers/`, `api/` |
| **Optimized execution** | Async SQLAlchemy throughout; embeddings on CPU only; spans buffered per-run (one DB transaction on flush) |
| **Minimum required lines** | No speculative abstractions; stub tools upgraded in-place in Phase 4 rather than building a plugin architecture |
| **Local-first** | Ollama is default and P0-critical; Gemini is never imported unconditionally |
| **Testable by design** | Evaluators are pure functions over data; `FailureInjector` is injected not global; provider is an interface |
