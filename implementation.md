# AgentGuard — Master Execution Guide for Coding Agents

> This file is read by any coding agent (Claude Code or otherwise) **before touching this repository**. It governs *how* work gets done. `docs/blueprint.md` governs *what* gets built. `agentcontext.md` tracks *current state* and must be updated at the end of every feature.

---

## 1. Agent Operational Rules & Constraints

These rules are non-negotiable for every task, every session, no exceptions.

### 1.1 Minimal, Clean Code Only
- Write only the code a feature actually needs. No speculative abstractions, no "might need this later" scaffolding, no unused config flags.
- No boilerplate comments that restate the code (`# increment counter`). Comment only non-obvious decisions (*why*, not *what*).
- Prefer editing an existing small function over adding a new layer of indirection.
- If a feature can be done in 30 lines, 80 lines of "flexible" code is a failure, not a bonus.

### 1.2 Zero Hallucination, Zero Silent Overwriting
- **Never guess an API.** If unsure whether a library function exists or what its signature is, check the installed package (`pip show`, read the source, or check official docs) before writing the call.
- **Always inspect a file before editing it.** Run `view` (or `cat`) on the target file immediately before any edit — never edit from memory of what you think is there, even if you wrote it two turns ago.
- **Never overwrite a working module without explicit verification** that the change is needed and correct. If a file already implements the feature being requested, say so and ask — do not silently replace it.
- If a required file, table, or endpoint referenced in `docs/blueprint.md` doesn't exist yet, say so explicitly rather than assuming it and building on top of an assumption.

### 1.3 Concurrent Testing — Not a Follow-Up Step
- Every feature implementation and its tests are written in the **same work unit**, not "tests to be added later."
- Minimum bar per feature: one unit test for the core logic, one test for the documented failure/edge case (empty input, malformed data, tool error).
- A feature is not "done" until `pytest` passes locally for the new tests and the full existing suite still passes (no regressions).

### 1.4 Git Checkpoints
- Every completed feature ends with one commit — not several half-finished WIP commits, not a giant multi-feature commit.
- Commit message format: `<phase>: <what changed> — <why>`
  - Example: `phase-2-tracing: add TraceCollector span flush — persists run/span rows after each agent step`
- Never commit with failing tests. Never commit commented-out dead code.

### 1.5 When Blocked or Uncertain
- If a design decision isn't covered by `docs/blueprint.md` or an existing ADR in `agentcontext.md`, stop and ask — don't invent a new architectural pattern mid-feature.
- If something in `docs/blueprint.md` turns out to be wrong or infeasible once you're actually building it, flag it explicitly in `agentcontext.md`'s Known Blockers section.

---

## 2. Feature-by-Feature Implementation Plan

Ten sequential phases. Each phase assumes the previous one is complete and committed. Do not start a phase's self-review until every task in it is done; do not start the next phase until the current one's manual verification has passed.

---

### Phase 0 — Project Setup

**Tasks:**
1. Confirm repo structure exists: `backend/{agent,providers,tracing,evaluation,failure,api,db,tests}`, `frontend/`, `dataset/`, `scripts/`, `docs/`.
2. Verify `docker-compose.yml` with PostgreSQL 15 + pgvector service starts clean.
3. Implement FastAPI skeleton (`backend/api/main.py`) with a `/health` endpoint.
4. Implement `backend/db/session.py` and `backend/db/config.py` with settings from `.env`.
5. Verify Ollama responds to (a) a plain prompt and (b) a prompt requiring structured JSON output — both must work before proceeding.

**Self-Review Protocol:** confirm no hardcoded paths/secrets; confirm `.env.example` exists and `.env` is gitignored; confirm the Docker service starts clean from `docker-compose up` on a fresh clone.

**Manual Verification Steps:**
- Run `docker-compose up`, confirm Postgres is reachable.
- Hit `GET /health`, confirm `200 OK`.
- Run the two Ollama smoke prompts manually, confirm the JSON one actually parses.

**Rollback Plan:** `docker-compose down -v` to wipe the DB volume; delete and re-init the repo scaffold — nothing from this phase carries risk to anything else since no data exists yet.

---

### Phase 1 — Reference Agent

**Tasks:**
1. Implement `LLMProvider` abstract interface + `OllamaProvider` (`backend/providers/`).
2. Implement `AgentContext` dataclass (per `docs/blueprint.md` Section 2.3 schema).
3. Implement the three tools: `SearchTool` (stub retrieval; real pgvector search comes in Phase 4), `CalculatorTool`, `LookupTool` (`backend/agent/tools/`).
4. Implement `AgentRuntime`'s ReAct loop (max 4–6 steps, configurable via env) (`backend/agent/runtime.py`).
5. Run 5–10 sample tasks manually end-to-end (no persistence yet) via a CLI smoke script.

**Self-Review Protocol:** re-read `AgentRuntime` for an unbounded loop risk (step limit actually enforced?); confirm tool errors don't crash the loop silently; confirm `AgentContext` isn't accumulating full document text (only IDs/scores per ADR-002 context rule).

**Manual Verification Steps:**
- Run 3 manually-written sample questions through the agent via a CLI script; confirm each terminates with either a final answer or `max_steps_exceeded`, never an unhandled exception.

---

### Phase 2 — Tracing Infrastructure

**Tasks:**
1. Create `runs`/`spans` tables (DDL per `docs/blueprint.md` Section 4.1) via Alembic migration.
2. Implement `TraceCollector` (`backend/tracing/collector.py`): buffers span events, flushes on run close.
3. Wire `AgentRuntime` to call `TraceCollector` at each step (llm_call, retrieval, tool_call spans).
4. Expose `POST /runs` and `GET /runs/{id}` (`backend/api/routers/runs.py`).

**Self-Review Protocol:** confirm every span has correct `parent_span_id`; confirm a run that errors mid-execution still gets `status=error` persisted; confirm no raw provider SDK objects are being serialized into `attributes`.

**Manual Verification Steps:**
- Trigger a run via `POST /runs`, then `GET /runs/{id}` and manually inspect the span tree — does it match what actually happened?
- Query the DB directly and confirm row counts match expected span count for a 3-step run.

---

### Phase 3 — Deterministic Evaluators

**Tasks:**
1. Create `evaluations` table (migration).
2. Implement `Evaluator` base class with uniform `.evaluate(run, benchmark_item) -> EvaluationResult` (`backend/evaluation/base.py`).
3. Implement `TaskEvaluator`, `RetrievalEvaluator`, `ToolEvaluator` (selection + argument correctness).
4. Wire evaluator execution into `POST /runs/{id}/evaluate` (or auto-trigger on run completion).

**Self-Review Protocol:** confirm every evaluator is read-only against the trace (never mutates run/span rows); confirm evaluators handle a missing/empty trace gracefully instead of throwing.

**Manual Verification Steps:**
- Build 3 fixture traces with known expected scores (one clean pass, one tool failure, one wrong-tool case); confirm each evaluator returns the expected score.

---

### Phase 4 — Benchmark & Real Retrieval

**Tasks:**
1. Create `documents`/`benchmark_items` tables + pgvector index via migration.
2. Build `scripts/seed_db.py` to load the authored corpus (80–150 docs) and compute embeddings.
3. Replace `SearchTool`'s stub with real pgvector similarity search.
4. Author and label the first 40–50 benchmark items per the documented labeling process (calibration round first, two annotators label the same 10–15 items independently, resolve disagreements, document in `dataset/benchmark/labeling_notes.md`).

**Self-Review Protocol:** confirm the embedding model is pinned to a specific version; confirm `relevant_doc_ids` on each benchmark item actually correspond to real seeded documents.

---

### Phase 5 — Groundedness Judge + Calibration

**Tasks:**
1. Design the judge prompt and structured output schema.
2. Implement `GroundednessEvaluator` (LLM-judge based, routed through `LLMProvider`) (`backend/evaluation/groundedness.py`).
3. Run the judge against 30–40 completed runs; have two independent human annotators label the same 30–40 outputs.
4. Compute and store the judge-vs-human agreement rate in `agentcontext.md`.

**Self-Review Protocol:** confirm the judge's `rationale` field is actually populated and non-generic; confirm agreement-rate calculation is on a held-out basis (labels done independently, not after seeing the judge's output).

---

### Phase 6 — Failure Injection

**Tasks:**
1. Implement `FailureInjector` as a single wrapper around `ToolExecutor`/retrieval calls (`backend/failure/injector.py`).
2. Implement the two P0 failure types: `tool_failure`, `retrieval_failure`.
3. Add `failure_injected` tracking to the `attributes` JSONB on the affected span.
4. Run a baseline vs. failure-injected comparison on a benchmark subset (~10 items).

**Self-Review Protocol:** confirm failure injection logic exists in exactly one place — grep the codebase for any other conditional that simulates a failure; confirm injected failures are clearly distinguishable from real bugs in the trace.

---

### Phase 7 — Repeated Execution / Reliability Analyzer

**Tasks:**
1. Implement `ReliabilityAnalyzer` (`backend/evaluation/reliability.py`): given k runs of the same benchmark item/provider/condition, compute pass^k consistency score.
2. Add `GET /reliability?benchmark_item_id=` endpoint.
3. Run k=2 trials on a benchmark subset (~15–20 items).

**Self-Review Protocol:** confirm the analyzer consumes existing runs only (doesn't trigger new runs itself — single responsibility); confirm it correctly handles k=1 (no comparison possible) without erroring.

---

### Phase 8 — Minimal Dashboard

**Tasks:**
1. Build `RunsView` (table + filters) consuming `GET /runs`.
2. Build `TraceDetailView` (span list/tree) consuming `GET /runs/{id}`.
3. Build `MetricsDashboard` (aggregate charts) consuming `GET /runs/{id}/evaluations` across a batch.
4. Build `FailureExperimentView` (before/after comparison).

**Self-Review Protocol:** confirm no metric computation happens client-side (frontend only renders what the backend already computed); confirm empty/loading/error states are handled for every view, not just the happy path.

---

### Phase 9 — Integration, Hardening, Final Freeze

**Tasks:**
1. Run the full benchmark (40–50 items) × Ollama × k=2, plus the failure-injection subset, end to end.
2. Fill any test-coverage gaps identified during the above.
3. Add the export endpoint (`GET /export`).
4. Freeze scope; tag `p0-frozen`; no new features past this phase.

**Self-Review Protocol:** run the full `pytest` suite; confirm zero skipped/xfail tests hiding a real problem; confirm a cold-start `docker-compose up` on a clean checkout works end to end.

---

## 3. Workflow Diff Tracking

After **every completed feature** (i.e., at the end of each phase task or sub-task that gets its own commit), the agent appends an entry to `workflow_diff.md` in the repo root. Create the file if it doesn't exist. **Never overwrite previous entries — always append.**

### Entry Template

```markdown
## [YYYY-MM-DD HH:MM] <phase-id>: <feature name>

**What changed:**
- <bullet list of concrete changes — files touched, endpoints added, schema changes>

**Why:**
- <one or two sentences tying this back to docs/blueprint.md / agentcontext.md>

**Tests:**
- Added: <list of new test files/cases>
- Status: <all passing | N failing — see below>

**Self-review notes:**
- <anything non-obvious the review protocol surfaced>

**Commit:** `<git commit hash>` — `<commit message>`
```

### Rules for This File
- One entry per feature, appended in chronological order — never edited after the fact except to fix a typo.
- If a feature required touching `agentcontext.md` (new ADR, new blocker, dependency version change), say so explicitly in the entry and confirm it was actually done.
- If tests are failing at the time of the entry, the entry must say so honestly — do not write "all passing" to make an entry look cleaner than reality.
