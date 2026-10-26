# AgentGuard — Issues Backlog

> **Workflow:** Each issue maps to one feature branch (`issue-N-short-name`), one implementation unit, one commit, and one PR. Work through issues in numeric order. Do not begin issue N+1 until issue N is merged and tests pass.
>
> **Branch naming:** `issue-N-short-description` (e.g., `issue-3-fastapi-skeleton`)
> **Commit format:** `phase-X: <what> — <why>`

---

## Phase 0 — Project Setup

---

### Issue #1 — Docker Compose + PostgreSQL/pgvector service

**Title:** `[P0] Bootstrap Docker Compose with PostgreSQL 15 + pgvector`

**Description & Context:**
All backend services depend on a PostgreSQL 15 database with the `pgvector` extension. The `docker-compose.yml` must start this service cleanly, expose port 5432, and pass a health check before any dependent service starts. The `pgvector/pgvector:pg15` Docker image bundles both Postgres and the extension — no manual extension installation needed.

**Acceptance Criteria:**
- [ ] `docker-compose up postgres` starts without errors
- [ ] `pg_isready -U agentguard` succeeds inside the container
- [ ] `docker-compose down -v` cleanly destroys the volume
- [ ] No hardcoded passwords — all credentials from `.env`
- [ ] A cold-start from a clean clone (no `.env`) fails with a clear error, not a silent misconfiguration

**Technical Specs:**
- Image: `pgvector/pgvector:pg15`
- Health check: `pg_isready`, 5s interval, 10 retries
- Volume: named `postgres_data` (persists across restarts)
- The `hf_cache` volume is declared here too (used by backend for HuggingFace model cache)
- **Do not** use `depends_on` without `condition: service_healthy` — the backend must wait for Postgres to be ready

**Optimization:** Use the official `pgvector/pgvector:pg15` image rather than installing pgvector manually — saves ~15 lines of Dockerfile and eliminates a build step.

---

### Issue #2 — Settings & database session factory

**Title:** `[P0] Implement pydantic-settings config and async SQLAlchemy session`

**Description & Context:**
All environment variables (database URL, Ollama URL, model name, etc.) must flow through a single `Settings` object loaded from `.env`. The async SQLAlchemy engine and session factory must be created once at startup and injected into FastAPI route handlers via `Depends`.

**Acceptance Criteria:**
- [ ] `backend/db/config.py` exports a `get_settings()` cached function returning a `Settings` instance
- [ ] `backend/db/session.py` exports `get_db()` — an async generator yielding an `AsyncSession`
- [ ] No `os.getenv()` calls anywhere outside `config.py`
- [ ] Starting the app without a `.env` file raises a `ValidationError` with a clear message listing missing fields

**Technical Specs:**
- `pydantic-settings==2.7.0` with `model_config = SettingsConfigDict(env_file=".env")`
- `create_async_engine` from `sqlalchemy.ext.asyncio`
- Session factory: `async_sessionmaker(engine, expire_on_commit=False)`
- `get_db()` used as `Depends(get_db)` in route functions — never a global session

**Libraries:** `pydantic-settings`, `sqlalchemy[asyncio]`, `asyncpg`

---

### Issue #3 — FastAPI application skeleton + /health endpoint

**Title:** `[P0] FastAPI app factory, lifespan, CORS, and /health endpoint`

**Description & Context:**
The FastAPI application needs a clean factory function, async lifespan (for startup/shutdown hooks), CORS configured for the frontend dev server, and a `/health` endpoint that returns `{"status": "ok"}`. All routers will be registered here as they're built in later phases.

**Acceptance Criteria:**
- [ ] `GET /health` returns `200 {"status": "ok"}`
- [ ] CORS allows `http://localhost:5173` (Vite dev server)
- [ ] `GET /docs` serves the Swagger UI
- [ ] `pytest backend/tests/unit/test_health.py` passes using `httpx.AsyncClient`
- [ ] App starts in < 3 seconds (no blocking imports at startup)

**Technical Specs:**
- `backend/api/main.py`: `create_app()` factory function, not a module-level `app = FastAPI()`
- Lifespan context manager handles any startup logic (e.g., DB connection check)
- Router registration pattern: `app.include_router(runs_router, prefix="/runs")`

**Libraries:** `fastapi`, `uvicorn[standard]`

---

### Issue #4 — Ollama smoke test script

**Title:** `[P0] Ollama structured-output smoke test — verify plain + JSON prompts before any agent work`

**Description & Context:**
Per blueprint Section 18 and risk row 2, the biggest hidden risk is the local 3–4B model producing malformed tool-call JSON. This must be verified in Phase 0, not discovered in Phase 1 or later. The smoke test script must test BOTH a plain text prompt and a structured JSON output prompt.

**Acceptance Criteria:**
- [ ] `python scripts/smoke_ollama.py` prints `[PASS] plain prompt` and `[PASS] JSON prompt`
- [ ] If the JSON prompt fails to parse, it prints `[FAIL] JSON prompt — <error>` and exits with code 1
- [ ] The script accepts `--model` and `--base-url` CLI args (defaults from `.env`)
- [ ] The structured JSON test uses the same schema the agent will use: `{"action": "tool|answer", "tool": "...", "args": {}, "answer": "..."}`

**Technical Specs:**
- Uses `httpx` to call Ollama's `/api/chat` endpoint directly (not through the `OllamaProvider` class, which doesn't exist yet)
- JSON parse attempt uses `json.loads()` wrapped in a try/except — no regex parsing

**Libraries:** `httpx`, `json`, `argparse`

---

## Phase 1 — Reference Agent

---

### Issue #5 — LLMProvider interface + OllamaProvider

**Title:** `[P1-Agent] Implement LLMProvider ABC and OllamaProvider`

**Description & Context:**
The agent must never import a specific LLM SDK directly — all calls go through the `LLMProvider` interface. `OllamaProvider` is the concrete implementation. `GeminiProvider` is a stub that raises `NotImplementedError` until P1. This abstraction is the single place where provider-specific logic lives.

**Acceptance Criteria:**
- [ ] `LLMProvider.complete(messages: list[dict]) -> str` is the only method the agent calls
- [ ] `OllamaProvider` makes an `httpx` POST to Ollama's `/api/chat` endpoint
- [ ] `OllamaProvider` raises a typed `ProviderError` on HTTP failure or timeout — never lets `httpx` exceptions leak
- [ ] `GeminiProvider` raises `NotImplementedError("GeminiProvider is a P1 feature")` on any call
- [ ] `pytest backend/tests/unit/test_providers.py` passes (mocked HTTP calls)

**Technical Specs:**
- `backend/providers/base.py`: `LLMProvider(ABC)` with `@abstractmethod complete()`
- `backend/providers/ollama.py`: sync `httpx.Client` (not async — called from within the agent loop which manages its own async context)
- Timeout: 30s (configurable from `Settings`)
- Model name: read from `Settings.ollama_model` — never hardcoded

**Libraries:** `httpx`, `abc`

---

### Issue #6 — AgentContext dataclass

**Title:** `[P1-Agent] Implement AgentContext — per-run state container`

**Description & Context:**
`AgentContext` is the single mutable state object for one agent run. It holds messages, retrieved document IDs (not full text), tool results, and the current step index. It must never accumulate full document text (only IDs and scores) and must never carry state across runs.

**Acceptance Criteria:**
- [ ] `AgentContext` is a `dataclass` (not a `dict`, not a `Pydantic` model — the simplest structure that works)
- [ ] Fields: `run_id`, `benchmark_item_id`, `provider`, `model_name`, `step_index`, `messages`, `retrieved_doc_ids`, `tool_results`, `state`, `failure_config`
- [ ] `retrieved_doc_ids` is a `list[tuple[int, float]]` (id, score) — never stores full document text
- [ ] A new instance is created per run — no class-level mutable state
- [ ] `pytest backend/tests/unit/test_context.py` passes

**Technical Specs:**
- `backend/agent/context.py`
- `state` field: `Literal["running", "completed", "max_steps_exceeded", "error"]`

---

### Issue #7 — Three agent tools: Calculator, Lookup, Search (stub)

**Title:** `[P1-Agent] Implement CalculatorTool, LookupTool, and SearchTool (stub)`

**Description & Context:**
The agent needs exactly 3 tools. `CalculatorTool` handles arithmetic safely (no raw `eval`). `LookupTool` does deterministic key-value lookup against a predefined dict (e.g., fee deadlines, policy dates). `SearchTool` is a stub returning an empty list — it will be replaced with real pgvector search in Issue #17 (Phase 4).

**Acceptance Criteria:**
- [ ] `CalculatorTool.run(expression: str) -> float` handles `+`, `-`, `*`, `/`, `**` with operator precedence
- [ ] `CalculatorTool` raises `ToolError("division_by_zero")` on `/0`, never a raw `ZeroDivisionError`
- [ ] `LookupTool.run(key: str) -> str | None` returns `None` cleanly for missing keys (agent handles gracefully)
- [ ] `SearchTool.run(query: str) -> list` returns `[]` — clearly marked as stub in code
- [ ] All three implement a common `Tool` protocol: `name: str`, `description: str`, `run(**kwargs) -> Any`
- [ ] `pytest backend/tests/unit/test_tools.py` passes (edge cases: division by zero, missing key, expression parse error)

**Technical Specs:**
- `backend/agent/tools/calculator.py`, `lookup.py`, `search.py`
- Calculator: use Python's `ast` module (`ast.parse` + `ast.NodeVisitor`) — no `eval()`, no `exec()`
- Safe operators: `Add`, `Sub`, `Mult`, `Div`, `Pow`, `USub` (unary minus)

**Libraries:** `ast`, `operator`

---

### Issue #8 — ToolExecutor

**Title:** `[P1-Agent] Implement ToolExecutor — dispatches tool calls, wraps errors`

**Description & Context:**
`ToolExecutor` is the single place that dispatches a tool call by name to the correct `Tool` instance, wraps exceptions into a typed `ToolResult`, and is the single point where `FailureInjector` will be wired in Phase 6. It must never crash the agent loop — all tool errors become error results.

**Acceptance Criteria:**
- [ ] `ToolExecutor.run(tool_name: str, args: dict) -> ToolResult`
- [ ] Unknown tool name → `ToolResult(success=False, error="unknown_tool")`
- [ ] Tool raises exception → `ToolResult(success=False, error=str(e))`
- [ ] Tool succeeds → `ToolResult(success=True, output=<result>)`
- [ ] `pytest backend/tests/unit/test_executor.py` passes

**Technical Specs:**
- `backend/agent/executor.py`
- `ToolResult` is a `TypedDict`: `{"success": bool, "output": Any | None, "error": str | None}`
- Tool registry is a `dict[str, Tool]` passed at construction — not a global registry, not auto-discovered

---

### Issue #9 — AgentRuntime ReAct loop

**Title:** `[P1-Agent] Implement AgentRuntime — hand-written ReAct reasoning loop`

**Description & Context:**
The core of the project: a hand-written ReAct loop that runs at most `MAX_STEPS` steps. Each step asks the LLM to decide whether to call a tool or produce a final answer. No LangGraph, no agent framework — every line of decision logic is written and owned by the team.

**Acceptance Criteria:**
- [ ] Loop terminates after at most `MAX_STEPS` steps (default 6) — never runs indefinitely
- [ ] Each step produces exactly one of: tool call, final answer, or error
- [ ] A tool error result is added to the context and the loop continues (no crash)
- [ ] `status` is set correctly on termination: `completed`, `max_steps_exceeded`, or `error`
- [ ] `pytest backend/tests/unit/test_runtime.py` passes with a mocked `LLMProvider`

**Technical Specs:**
- `backend/agent/runtime.py`
- LLM decision schema (parsed from LLM JSON output):
  ```json
  {"action": "tool", "tool": "calculator", "args": {"expression": "12*4"}}
  {"action": "answer", "answer": "The answer is 48."}
  ```
- On JSON parse failure from LLM: log a `parse_error` span attribute, increment step, retry (up to 2 retries per step before giving up)
- Prompt format: system prompt + message history (question, tool results) — kept under 1024 tokens
- CLI script: `scripts/run_agent.py --question "..." --provider ollama`

**Libraries:** `json`, `dataclasses`

---

## Phase 2 — Tracing Infrastructure

---

### Issue #10 — SQLAlchemy ORM models + Alembic initial migration

**Title:** `[P2-Tracing] Database ORM models and initial Alembic migration`

**Description & Context:**
All five tables from the blueprint schema (`documents`, `benchmark_items`, `runs`, `spans`, `evaluations`) must be defined as SQLAlchemy ORM models. Alembic manages schema migrations — no manual `CREATE TABLE` statements in application code.

**Acceptance Criteria:**
- [ ] All five ORM models defined in `backend/db/models.py`
- [ ] `alembic upgrade head` creates all tables + indexes cleanly on a fresh DB
- [ ] `alembic downgrade -1` reverts without error
- [ ] UUID primary keys use `gen_random_uuid()` server-side default (PostgreSQL 13+ built-in)
- [ ] pgvector `VECTOR(384)` column on `Document` model is correctly mapped via the `pgvector` SQLAlchemy type
- [ ] `pytest backend/tests/integration/test_models.py` passes (create + query round-trip for each model)

**Technical Specs:**
- `backend/db/models.py`: `Document`, `BenchmarkItem`, `Run`, `Span`, `Evaluation`
- `backend/db/migrations/` with `alembic.ini` and `env.py` using async engine
- `runs.status` is a `CheckConstraint` — validated at the DB level, not just application level
- `spans.span_type` is a `CheckConstraint` — same pattern

**Libraries:** `sqlalchemy[asyncio]`, `asyncpg`, `alembic`, `pgvector`

---

### Issue #11 — TraceCollector

**Title:** `[P2-Tracing] Implement TraceCollector — buffers spans, flushes to DB on run close`

**Description & Context:**
`TraceCollector` is the single interface between the agent runtime and the database for trace data. It buffers span events in memory during a run and writes them all in one transaction when the run closes. This avoids per-step DB round-trips. It must ensure that even a crashed run has its `status=error` persisted via a `try/finally` pattern.

**Acceptance Criteria:**
- [ ] `TraceCollector.start_run(...)` creates a `Run` row with `status=running`
- [ ] `TraceCollector.add_span(...)` buffers span in memory (no DB write yet)
- [ ] `TraceCollector.close(status)` writes all buffered spans + updates `Run.status` in one transaction
- [ ] If `close()` is never called (crash), a `try/finally` in the caller sets `status=error`
- [ ] No raw provider SDK objects appear in `attributes` — only JSON-serializable primitives
- [ ] `pytest backend/tests/unit/test_collector.py` passes with a mock DB session

**Technical Specs:**
- `backend/tracing/collector.py`
- Span `attributes` key naming: `gen_ai.prompt.tokens`, `gen_ai.completion.tokens`, `gen_ai.tool.name`, `gen_ai.retrieval.query`, `gen_ai.retrieval.doc_ids`
- `parent_span_id`: the `agent_run` span is the parent of all other spans — one level of nesting only

---

### Issue #12 — Runs API endpoints

**Title:** `[P2-Tracing] POST /runs, GET /runs, GET /runs/{id} endpoints`

**Description & Context:**
Three REST endpoints expose run creation and trace retrieval. `POST /runs` triggers an agent run asynchronously and returns immediately with the `run_id`. `GET /runs/{id}` returns the full run + span tree.

**Acceptance Criteria:**
- [ ] `POST /runs` returns `{"run_id": "...", "status": "running"}` immediately (run executes in background)
- [ ] `GET /runs` returns a paginated list of runs with filters: `condition` (normal/tool_failure/retrieval_failure), `status`
- [ ] `GET /runs/{id}` returns the full run with its spans sorted by `started_at`
- [ ] `GET /runs/{id}` on a non-existent ID returns `404 {"detail": "run not found"}`
- [ ] `pytest backend/tests/integration/test_runs_api.py` passes

**Technical Specs:**
- `backend/api/routers/runs.py`
- Background task: `fastapi.BackgroundTasks` (not a separate thread pool — asyncio task is sufficient per ADR-010)
- Response schema uses Pydantic models defined in `backend/api/schemas.py`
- `GET /runs` default page size: 20; max: 100

---

## Phase 3 — Deterministic Evaluators

---

### Issue #13 — Evaluator base class + EvaluatorEngine

**Title:** `[P3-Eval] Evaluator ABC and EvaluatorEngine`

**Description & Context:**
All evaluators share the same interface. `EvaluatorEngine` runs all registered evaluators on a closed run and writes results to the `evaluations` table in one transaction. This is the single place where evaluation is triggered — no ad-hoc per-metric API calls.

**Acceptance Criteria:**
- [ ] `Evaluator.evaluate(run: Run, item: BenchmarkItem) -> EvaluationResult` is the only required method
- [ ] `EvaluatorEngine.run_all(run_id)` fetches the run + spans + item, runs all evaluators, writes all `Evaluation` rows
- [ ] If an evaluator raises, `EvaluatorEngine` catches it, writes `score=0.0` with `rationale="evaluator_error: <msg>"`, and continues with the next evaluator — never aborts the whole evaluation
- [ ] `pytest backend/tests/unit/test_engine.py` passes

**Technical Specs:**
- `backend/evaluation/base.py`, `backend/evaluation/engine.py`
- `EvaluationResult = TypedDict` with fields: `metric_name`, `score`, `rationale`, `is_llm_judge`
- `EvaluatorEngine` is initialized with a list of `Evaluator` instances — not hardcoded, not auto-discovered

---

### Issue #14 — TaskEvaluator

**Title:** `[P3-Eval] Implement TaskEvaluator — deterministic answer correctness`

**Description & Context:**
Task Success checks whether the agent's final answer matches the benchmark item's `expected_answer`. For `unanswerable` items, success means the agent abstained or said "not found." For `ambiguous` and `conflicting` items, `task_success` is not scored on answer correctness — see Issue #15 for process metrics.

**Acceptance Criteria:**
- [ ] `answerable` items: normalized string match (lowercase, strip, remove punctuation) → `1.0` or `0.0`
- [ ] `unanswerable` items: checks if `final_answer` contains abstention keywords (`"not found"`, `"I don't know"`, `"no information"`) → `1.0` or `0.0`
- [ ] `ambiguous`/`conflicting` items: returns `score=None` with `rationale="task_success not scored for ambiguous/conflicting items — use process metrics"`
- [ ] Empty/missing `final_answer` in trace → `0.0` with clear rationale
- [ ] `pytest backend/tests/unit/test_task_evaluator.py` passes (10+ cases)

**Technical Specs:**
- `backend/evaluation/task.py`
- Normalization function is a pure utility in `backend/evaluation/utils.py` — shared by other evaluators that need string comparison

---

### Issue #15 — RetrievalEvaluator + ToolEvaluator

**Title:** `[P3-Eval] Implement RetrievalEvaluator and ToolEvaluator`

**Description & Context:**
`RetrievalEvaluator` measures whether the agent retrieved the documents the benchmark labels as relevant. `ToolEvaluator` checks two things: (1) was the correct tool selected, and (2) were the tool arguments valid? These are the core process metrics that work even for ambiguous/unanswerable items.

**Acceptance Criteria:**
- `RetrievalEvaluator`:
  - [ ] Computes `precision = |retrieved ∩ relevant| / |retrieved|`
  - [ ] Computes `recall = |retrieved ∩ relevant| / |relevant|`
  - [ ] Returns `(precision + recall) / 2` as the score (0.0–1.0)
  - [ ] Empty retrieval result → `score=0.0`
  - [ ] Item with `relevant_doc_ids=[]` → `score=1.0` (no retrieval expected, none done)
- `ToolEvaluator`:
  - [ ] Selection score: `1.0` if first tool called matches `expected_tool`, else `0.0`
  - [ ] Argument score: `1.0` if args are non-empty and JSON-valid, `0.5` if empty, `0.0` if unparseable
  - [ ] Overall tool score: `(selection + argument) / 2`
  - [ ] `pytest backend/tests/unit/test_retrieval_evaluator.py` and `test_tool_evaluator.py` pass

**Technical Specs:**
- `backend/evaluation/retrieval.py`, `backend/evaluation/tool.py`
- `RetrievalEvaluator` reads `retrieved_doc_ids` from the `retrieval` span's `attributes` JSONB
- `ToolEvaluator` reads `tool_name` and `arguments` from the first `tool_call` span's `attributes`

---

### Issue #16 — LatencyEvaluator + evaluations API endpoint

**Title:** `[P3-Eval] LatencyEvaluator and POST /runs/{id}/evaluate endpoint`

**Description & Context:**
`LatencyEvaluator` records wall-clock timing for the full run and per-phase breakdown. Unlike other evaluators, its "score" is a duration in milliseconds — not 0.0–1.0. The `POST /runs/{id}/evaluate` endpoint triggers `EvaluatorEngine.run_all()` and returns the evaluation results.

**Acceptance Criteria:**
- [ ] `LatencyEvaluator` returns `score` = total run duration in milliseconds (float)
- [ ] Per-phase breakdown stored in `rationale` as JSON string: `{"llm_ms": 1200, "retrieval_ms": 340, "tool_ms": 80}`
- [ ] `POST /runs/{id}/evaluate` triggers all evaluators and returns all `EvaluationResult` objects
- [ ] `GET /runs/{id}/evaluations` returns all stored evaluations for a run
- [ ] `pytest backend/tests/unit/test_latency_evaluator.py` passes

**Technical Specs:**
- `backend/evaluation/latency.py`
- `backend/api/routers/evaluations.py`
- Duration = `run.ended_at - run.started_at` in milliseconds (if `ended_at` is None, return `0.0` with rationale `"run still in progress"`)

---

## Phase 4 — Benchmark & Real Retrieval

---

### Issue #17 — pgvector similarity search (replace SearchTool stub)

**Title:** `[P4-Benchmark] Replace SearchTool stub with real pgvector cosine similarity search`

**Description & Context:**
Phase 1 left a stub `SearchTool` returning `[]`. This issue replaces it with a real async pgvector query. The embedding of the query is computed using the same `sentence-transformers` model used at seed time (pinned in `Settings`).

**Acceptance Criteria:**
- [ ] `SearchTool.run(query: str) -> list[dict]` returns top-k documents with `{"id": int, "title": str, "content": str, "score": float}`
- [ ] The embedding model is loaded once at process startup (not per-query) — uses `functools.lru_cache` or module-level singleton
- [ ] `top_k` is configurable from `Settings` (default 3)
- [ ] `pytest backend/tests/integration/test_search_tool.py` passes against a real seeded test DB

**Technical Specs:**
- `backend/agent/tools/search.py` (edit in place — no new file)
- pgvector query: `ORDER BY embedding <=> :query_vec LIMIT :k` (cosine distance via `<=>` operator)
- Embedding: `SentenceTransformer(settings.embedding_model).encode(query)` → `numpy.ndarray` → `list[float]`

**Libraries:** `sentence-transformers`, `pgvector`, `sqlalchemy`

---

### Issue #18 — Corpus authoring and DB seed script

**Title:** `[P4-Benchmark] Author knowledge corpus and implement seed_db.py`

**Description & Context:**
The corpus is 80–150 authored documents (100–400 words each) covering the chosen domain (college policy / internal knowledge base). `seed_db.py` loads them, computes embeddings, and inserts into the `documents` table. It is idempotent (truncate-and-reload).

**Acceptance Criteria:**
- [ ] At least 80 documents authored in `dataset/corpus/*.md` format
- [ ] `python scripts/seed_db.py` runs without error and prints `Seeded N documents (dim=384)`
- [ ] Re-running `seed_db.py` on an already-seeded DB clears and reloads without duplicate rows
- [ ] Embedding dimension matches `Settings.embedding_dim` (384)
- [ ] `pytest backend/tests/integration/test_seed.py` passes (verify document count + embedding shape)

**Technical Specs:**
- `scripts/seed_db.py`
- Process: `read .md files → encode text → bulk insert via SQLAlchemy core (not ORM) for speed`
- Use `asyncio.run()` as the entry point — the seeding logic is async
- Batch insert: 20 documents per batch to avoid memory spikes with large embedding tensors

---

### Issue #19 — Benchmark items authoring, labeling, and loader

**Title:** `[P4-Benchmark] Author 40–50 benchmark items with calibration process`

**Description & Context:**
40–50 benchmark items authored and labeled per the documented process. The labeling calibration round (10–15 items, two independent annotators label and resolve disagreements) must be completed before the remaining items are labeled. `labeling_notes.md` documents edge cases and disagreements.

**Acceptance Criteria:**
- [ ] `dataset/benchmark/benchmark_items.json` contains 40–50 items
- [ ] Distribution: at least 20 `answerable`, 10 `unanswerable`, 5 `ambiguous`, 5 `conflicting`
- [ ] Every item has valid `relevant_doc_ids` that correspond to real seeded document IDs
- [ ] `dataset/benchmark/labeling_notes.md` documents the calibration round
- [ ] `python scripts/seed_db.py` loads benchmark items (alongside corpus) without error
- [ ] `pytest backend/tests/integration/test_benchmark.py` passes (spot-check 5 items: labeled docs exist + contain the answer)

**Technical Specs:**
- Item schema per blueprint Section 11 (see `docs/blueprint.md`)
- `scripts/seed_db.py` validates referential integrity: every `relevant_doc_id` must exist in `documents` — fails loud if broken

**Libraries:** `json`

---

### Issue #20 — Benchmark API endpoints

**Title:** `[P4-Benchmark] GET /benchmark and POST /benchmark/run endpoints`

**Description & Context:**
`GET /benchmark` lists benchmark items for the dashboard. `POST /benchmark/run` runs a subset of items with optional `k` trials and `failure_config`, executing as an async background task.

**Acceptance Criteria:**
- [ ] `GET /benchmark?category=answerable&page=1` returns paginated items
- [ ] `POST /benchmark/run` returns `{"batch_id": "...", "status": "queued", "total_runs": N}` immediately
- [ ] The background task runs each `benchmark_item_id` × `k` trials sequentially (not in parallel — avoids Ollama queue issues)
- [ ] `GET /benchmark/run/{batch_id}` returns batch progress and completion status
- [ ] `pytest backend/tests/integration/test_benchmark_api.py` passes

**Technical Specs:**
- `backend/api/routers/benchmark.py`
- Batch status tracked in a simple in-memory dict (not Redis — per ADR-010)
- Background task timeout: if a single run exceeds 120s, mark it `error` and continue with next

---

## Phase 5 — Groundedness Judge

---

### Issue #21 — GroundednessEvaluator with LLM judge

**Title:** `[P5-Judge] Implement GroundednessEvaluator — LLM-as-judge for answer groundedness`

**Description & Context:**
`GroundednessEvaluator` prompts the local LLM (via `LLMProvider`) to judge whether the agent's final answer is supported by its retrieved evidence. This is the only LLM-judge metric. It is calibrated against human labels in Issue #22.

**Acceptance Criteria:**
- [ ] Judge prompt is in `backend/evaluation/judge_prompt.py` (separate from evaluator logic)
- [ ] `GroundednessEvaluator` calls `LLMProvider.complete()` — never imports Ollama SDK directly
- [ ] Structured JSON output: `{"label": "SUPPORTED|PARTIALLY_SUPPORTED|UNSUPPORTED", "score": 0.0-1.0, "reason": "..."}`
- [ ] JSON parse failure → `score=0.0`, `rationale="parse_error"`, `is_llm_judge=True` — no crash
- [ ] `pytest backend/tests/unit/test_groundedness.py` passes (mocked LLM provider, including malformed response)

**Technical Specs:**
- `backend/evaluation/groundedness.py`, `backend/evaluation/judge_prompt.py`
- Score mapping: `SUPPORTED → 1.0`, `PARTIALLY_SUPPORTED → 0.5`, `UNSUPPORTED → 0.0`
- Judge prompt injects `retrieved_documents_text` (top-3 document contents joined) + `final_answer`
- Keep the judge prompt under 512 tokens for the local model's context budget

**Libraries:** (none new — uses existing `LLMProvider` and `httpx`)

---

### Issue #22 — Human calibration script and agreement rate

**Title:** `[P5-Judge] Calibration script — judge vs. human agreement rate`

**Description & Context:**
30–40 run outputs must be independently human-labeled, then compared against the judge's output. The agreement rate is the primary validation that the judge is useful. This number must be recorded in `agentcontext.md` and will appear in the final report.

**Acceptance Criteria:**
- [ ] `scripts/run_calibration.py` reads `dataset/benchmark/calibration_labels.csv` and runs the judge on the same 30–40 outputs
- [ ] Prints agreement percentage: `Judge agreement: X% (N/M items)`
- [ ] Optionally computes Cohen's kappa (if `scipy` is available)
- [ ] Agreement rate is written to `agentcontext.md` Section 3 (as a RESOLVED blocker entry)
- [ ] `dataset/benchmark/calibration_labels.csv` template committed with `run_id, label_dev1, label_dev2` columns

**Technical Specs:**
- `scripts/run_calibration.py`
- `calibration_labels.csv` format: `run_id, human_label` (0=UNSUPPORTED, 0.5=PARTIALLY_SUPPORTED, 1.0=SUPPORTED)
- Agreement = exact match between human label and judge label (same 3-class scale)

**Libraries:** `csv`, `json` (optionally `scipy.stats.cohen_kappa_score`)

---

## Phase 6 — Failure Injection

---

### Issue #23 — FailureInjector and failure modes

**Title:** `[P6-Failure] Implement FailureInjector — single-boundary failure injection`

**Description & Context:**
`FailureInjector` wraps `ToolExecutor` calls. When a failure is configured, it intercepts the tool result after execution and replaces it with the configured failure output. This is the ONLY place failure simulation exists in the codebase — grep-verified in the acceptance criteria.

**Acceptance Criteria:**
- [ ] `FailureInjector(config=None)` is a no-op — zero overhead on normal runs
- [ ] `TOOL_FAILURE`: replaces tool result with `{"error": "tool_timeout", "success": False}`
- [ ] `RETRIEVAL_FAILURE`: replaces retrieval result with `[]`
- [ ] Failure is tagged in the affected span: `attributes["gen_ai.failure.injected"] = True`, `attributes["gen_ai.failure.mode"] = "tool_failure"`
- [ ] **Grep check:** `rg "failure" backend/ --include="*.py" -l` shows only `backend/failure/` and `backend/agent/executor.py` — nowhere else
- [ ] `pytest backend/tests/unit/test_failure_injector.py` passes (8+ test cases)

**Technical Specs:**
- `backend/failure/injector.py`, `backend/failure/modes.py`, `backend/failure/config.py`
- `FailureConfig = dataclass(mode: FailureMode, target_tool: str | None, probability: float = 1.0)`
- `probability < 1.0` enables probabilistic failure (for future stress tests); P0 always uses `probability=1.0`

---

### Issue #24 — Baseline vs. failure comparison endpoint

**Title:** `[P6-Failure] Run failure experiment and add comparison data to API`

**Description & Context:**
Run the same 10-item subset in normal condition and in both failure conditions. Compute ΔM (metric delta) between baseline and failure runs. Expose comparison data via API for the dashboard.

**Acceptance Criteria:**
- [ ] `GET /runs?condition=tool_failure` and `GET /runs?condition=normal` return filterable subsets
- [ ] `GET /runs/{id1}/compare/{id2}` returns per-metric delta: `{"metric": "task_success", "baseline": 0.8, "failure": 0.3, "delta": -0.5}`
- [ ] Running 10 items with `TOOL_FAILURE` causes `task_success` to drop measurably vs. baseline
- [ ] `pytest backend/tests/integration/test_failure_experiment.py` passes

**Technical Specs:**
- `backend/api/routers/runs.py` (extend existing router)
- Comparison: fetch evaluations for both runs, align by `metric_name`, compute `delta = failure_score - baseline_score`

---

## Phase 7 — Repeated Execution

---

### Issue #25 — ReliabilityAnalyzer (pass^k consistency)

**Title:** `[P7-Reliability] Implement ReliabilityAnalyzer — pass^k consistency metric`

**Description & Context:**
`ReliabilityAnalyzer` computes pass^k (τ-bench definition): the fraction of benchmark items where ALL k trials produce the same outcome. It reads existing runs — it never triggers new ones. For the project baseline, k=2.

**Acceptance Criteria:**
- [ ] `ReliabilityAnalyzer.compute(benchmark_item_id, k)` returns `{"per_item": [...], "aggregate_pass_k": float}`
- [ ] k=1 → returns `{"aggregate_pass_k": None, "reason": "cannot measure consistency with k=1"}`
- [ ] Items with fewer than k runs → excluded from aggregate with a warning, not an error
- [ ] Success definition per category is correctly implemented (see Implementation Plan Phase 7)
- [ ] `pytest backend/tests/unit/test_reliability.py` passes (manual calculation verified)

**Technical Specs:**
- `backend/evaluation/reliability.py`
- Core query: `SELECT * FROM runs WHERE benchmark_item_id = :id AND condition = :cond ORDER BY trial_index`
- Read-only — zero DB writes

---

### Issue #26 — /reliability endpoint

**Title:** `[P7-Reliability] GET /reliability endpoint`

**Description & Context:**
Expose the `ReliabilityAnalyzer` via a REST endpoint. Supports querying for a single item or for all items in a batch.

**Acceptance Criteria:**
- [ ] `GET /reliability?benchmark_item_id=12` returns per-item consistency for item 12
- [ ] `GET /reliability?batch_id=batch_01J9` returns per-item and aggregate consistency for all items in the batch
- [ ] `pytest backend/tests/integration/test_reliability_api.py` passes

**Technical Specs:**
- `backend/api/routers/reliability.py`

---

## Phase 8 — Minimal Dashboard

---

### Issue #27 — React project setup + shared API hook

**Title:** `[P8-Dashboard] Vite + React 18 + Tailwind CSS setup and shared useApi hook`

**Description & Context:**
Initialize the React/Vite/Tailwind frontend. The shared `useApi` hook handles fetch, loading state, and error state for every page — no duplicate fetch logic in components.

**Acceptance Criteria:**
- [ ] `npm run dev` starts Vite dev server at `localhost:5173`
- [ ] `useApi(url)` hook returns `{ data, loading, error }` — handles all three states
- [ ] Tailwind CSS is configured and a simple test component renders with Tailwind classes
- [ ] `VITE_API_BASE_URL` is loaded from `.env` via `import.meta.env`

**Technical Specs:**
- `frontend/src/hooks/useApi.js`
- Tailwind config: default color palette + custom `primary` color (e.g., indigo)
- No component library — Tailwind only

**Libraries:** `react@18`, `vite`, `tailwindcss`

---

### Issue #28 — RunsView page

**Title:** `[P8-Dashboard] RunsView — paginated runs list with filters`

**Description & Context:**
The main landing page. Displays all runs as cards with key metrics at a glance. Filterable by condition and status. Clicking a run navigates to its trace detail.

**Acceptance Criteria:**
- [ ] Renders run cards: benchmark question (first 80 chars), condition badge, status badge, task_success score
- [ ] Filter dropdowns for `condition` and `status` update the displayed list without a page reload
- [ ] Empty state: "No runs yet. Trigger a run from the API." shown when list is empty
- [ ] Loading spinner shown while fetching
- [ ] Clicking a card navigates to `/runs/:id`

**Technical Specs:**
- `frontend/src/pages/RunsView.jsx`
- Fetch: `GET /runs?condition=&status=&page=1`
- Condition badge colors: `normal`=green, `tool_failure`=red, `retrieval_failure`=orange

---

### Issue #29 — TraceDetailView page

**Title:** `[P8-Dashboard] TraceDetailView — span timeline for a single run`

**Description & Context:**
The trace detail page shows every span in the run as an ordered timeline. Each span is expandable to show its full `attributes`. Timing bars show relative duration. This is the primary debugging tool.

**Acceptance Criteria:**
- [ ] Spans displayed in chronological order (`started_at` ascending)
- [ ] Each span shows: `span_type` icon, `name`, duration in ms
- [ ] Clicking a span expands its `attributes` as formatted JSON
- [ ] Spans tagged `gen_ai.failure.injected=true` are highlighted in red
- [ ] Breadcrumb navigation back to runs list

**Technical Specs:**
- `frontend/src/pages/TraceDetailView.jsx`
- `frontend/src/components/SpanTimeline.jsx` (reusable timeline component)
- Duration bar width: `span_duration / total_run_duration * 100%`

---

### Issue #30 — MetricsDashboard page

**Title:** `[P8-Dashboard] MetricsDashboard — aggregate metric cards and bar chart`

**Description & Context:**
The metrics page shows the Reliability Profile: per-metric aggregate scores across all runs in a selected condition. The primary display is metric cards (not a composite score). An optional bar chart shows the profile visually.

**Acceptance Criteria:**
- [ ] Four metric cards: task_success, retrieval_relevance, tool_correctness, groundedness (each shows mean ± std)
- [ ] Consistency (pass^k) displayed as a fifth card
- [ ] Recharts bar chart with all 5 metrics (excluding latency) side by side
- [ ] Condition selector (normal / tool_failure / retrieval_failure) filters the displayed metrics
- [ ] Latency shown separately as "P50: Xms / P95: Xms" — not on the 0–1 chart

**Technical Specs:**
- `frontend/src/pages/MetricsDashboard.jsx`
- `frontend/src/components/MetricCard.jsx` (reusable — all 5 cards use same component)
- Recharts `BarChart` with `Bar`, `XAxis`, `YAxis`, `Tooltip`

**Libraries:** `recharts`

---

### Issue #31 — FailureExperimentView page

**Title:** `[P8-Dashboard] FailureExperimentView — side-by-side baseline vs. failure comparison`

**Description & Context:**
The experiment page shows the core research result: metric degradation under failure injection. Two runs are selected and compared side-by-side with ΔM for each metric.

**Acceptance Criteria:**
- [ ] Run selector: two dropdowns (baseline run, failure run) populated from `GET /runs`
- [ ] Delta table: metric name | baseline score | failure score | ΔM (colored red if degraded, green if improved)
- [ ] Summary sentence auto-generated: "Task success dropped by 50 percentage points under tool failure."
- [ ] Side-by-side trace timelines showing the two runs' spans

**Technical Specs:**
- `frontend/src/pages/FailureExperimentView.jsx`
- `frontend/src/components/DeltaMetricTable.jsx`
- Delta color: `delta < -0.1` → red, `delta > 0.1` → green, else gray

---

## Phase 9 — Integration & Final Freeze

---

### Issue #32 — Export endpoint (CSV + JSON)

**Title:** `[P9-Integration] GET /export — full results dump in CSV and JSON`

**Description & Context:**
A single endpoint that exports all run + evaluation data for the final report. Supports both CSV and JSON formats. This is the data source for the project report's quantitative analysis.

**Acceptance Criteria:**
- [ ] `GET /export?format=json` returns all runs with their evaluations as a JSON array
- [ ] `GET /export?format=csv` returns a CSV with columns: `run_id, benchmark_item_id, category, condition, trial_index, status, task_success, retrieval_relevance, tool_correctness, groundedness, latency_ms, consistency`
- [ ] Empty DB → returns empty array/CSV (not a 500 error)
- [ ] `pytest backend/tests/integration/test_export.py` passes

**Technical Specs:**
- `backend/api/routers/export.py`
- CSV uses Python's `csv.DictWriter` — no pandas dependency
- JSON uses `jsonable_encoder` from FastAPI

---

### Issue #33 — Full benchmark end-to-end run and coverage hardening

**Title:** `[P9-Integration] Full P0 end-to-end run — benchmark × k=2 × failure injection`

**Description & Context:**
The final integration milestone. Run the complete benchmark (40–50 items × k=2 trials × normal + tool_failure + retrieval_failure conditions) and verify the entire pipeline. Fix any issues found. Fill test coverage gaps.

**Acceptance Criteria:**
- [ ] Full benchmark run completes without any unhandled exceptions
- [ ] `task_success` drops measurably under both failure conditions vs. normal baseline
- [ ] Consistency (pass^k) computed and shown for all 40–50 items
- [ ] `GET /export?format=csv` produces a parseable CSV covering all runs
- [ ] `pytest --cov=backend --cov-report=term-missing` shows ≥ 80% coverage
- [ ] Cold-start `docker-compose up` on a clean checkout works end to end
- [ ] `git tag p0-frozen` applied after all of the above pass

**Technical Specs:**
- `scripts/run_full_benchmark.py` — orchestration script for the full run
- Test coverage: any module below 60% coverage must have additional unit tests added in this issue

---

### Issue #34 — Demo rehearsal and scope freeze documentation

**Title:** `[P9-Integration] Final demo rehearsal + p0-frozen scope documentation`

**Description & Context:**
The full demo must be rehearsed end-to-end on a machine state that hasn't seen prior manual fixes. The final state of `agentcontext.md` must reflect all completed milestones, all dependency versions pinned, and all known limitations documented.

**Acceptance Criteria:**
- [ ] Full demo rehearsal script executed: seed → run benchmark → view dashboard → show failure comparison → export results
- [ ] `agentcontext.md` updated: all Phase checkboxes marked complete, all versions pinned, known limitations documented
- [ ] `workflow_diff.md` has an entry for every completed feature (append-only, no gaps)
- [ ] `ISSUES_BACKLOG.md` updated: all P0 issues marked with `[DONE]`
- [ ] `git tag p0-frozen` exists on the commit where all of the above are true

---

## P1 Stretch Issues (Only After `p0-frozen` Tag)

> **Start condition:** Gate 7 complete. P0 demo rehearsed successfully. Time budget > 2 weeks remaining.

---

### Issue #35 — Benchmark expansion to 80+ items
**Title:** `[P1-Stretch] Expand benchmark from 40–50 to 80+ items`
**Description:** Author additional benchmark items using the same labeling process. Focus on underrepresented categories (more `conflicting`, more complex `answerable`). Re-run the full evaluation and update the reliability report.

---

### Issue #36 — Third failure mode: corrupted tool result
**Title:** `[P1-Stretch] Add CORRUPTED_OUTPUT failure mode to FailureInjector`
**Description:** A third failure type where the tool returns a structurally valid but semantically wrong result (e.g., calculator returns `"NaN"`, lookup returns a completely wrong key). Add to the failure experiment comparison.

---

### Issue #37 — Gemini provider + cross-provider comparison
**Title:** `[P1-Stretch] Implement GeminiProvider and run 10-item cross-provider comparison`
**Description:** Replace the `GeminiProvider` stub with a real implementation. Run the same 10-item subset on Ollama and Gemini. Compare reliability profiles. Document model-name churn risk in `agentcontext.md`.

---

### Issue #38 — GitHub Actions CI pipeline
**Title:** `[P1-Stretch] GitHub Actions CI — smoke tests + benchmark subset on every push`
**Description:** A simple workflow that runs `pytest backend/tests/unit/` and a 5-item benchmark smoke subset on every push to `main`. Does not require GPU — mocks the Ollama provider.

---

### Issue #39 — Dashboard polish + composite score (optional)
**Title:** `[P1-Stretch] Dashboard polish and optional composite reliability score`
**Description:** Improve chart readability (latency waterfall, mobile layout). If composite score is added, it must be displayed below the Reliability Profile cards, clearly labeled as "project-defined summary (optional)", never as the headline result.

---

*End of Issues Backlog — Issues #1–#34 are P0 (required). Issues #35–#39 are P1 (stretch only).*
