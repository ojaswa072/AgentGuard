# AgentGuard — Agent Context (Persistent Tracking File)

> Read this file at the **start of every session**, before touching code. Update it at the end of every completed feature, per `implementation.md` Section 3. This file is the single source of truth for "where are we right now" — do not trust memory of a prior session over what's written here.

---

## 1. Project Current State

**Version:** 0.1.0 (pre-implementation)
**Active phase:** Phase 0 — Project Setup
**Last updated:** 2026-10-26
**Last updated by:** Initial scaffold (Antigravity agent)

### Milestone Checklist

- [ ] Phase 0 — Project setup (repo scaffold, Docker Compose, FastAPI skeleton, Ollama smoke test)
- [ ] Phase 1 — Reference agent (ReAct loop, 3 tools, provider interface)
- [ ] Phase 2 — Tracing infrastructure (runs/spans persisted, `GET /runs/{id}`)
- [ ] Phase 3 — Deterministic evaluators (Task Success, Retrieval Relevance, Tool Correctness)
- [ ] Phase 4 — Benchmark + real retrieval (corpus seeded, 40–50 items labeled, pgvector search live)
- [ ] Phase 5 — Groundedness judge + human calibration (agreement rate computed and recorded)
- [ ] Phase 6 — Failure injection (tool failure, retrieval failure, baseline-vs-failure comparison)
- [ ] Phase 7 — Repeated execution / Reliability Analyzer (k=2 consistency scoring)
- [ ] Phase 8 — Minimal dashboard (Runs, Trace Detail, Metrics, Failure Experiment views)
- [ ] Phase 9 — Integration, hardening, final freeze (`p0-frozen` tag)

### Gate Status

| Gate | Target | Status |
|---|---|---|
| Gate 1 — Agent runs locally | End of Phase 1 | Not started |
| Gate 2 — Tracing works | End of Phase 2 | Not started |
| Gate 3 — Deterministic evaluation works | End of Phase 3 | Not started |
| Gate 4 — Groundedness judge works (calibrated) | End of Phase 5 | Not started |
| Gate 5 — Failure injection works | End of Phase 6 | Not started |
| Gate 6 — Dashboard works | End of Phase 8 | Not started |
| Gate 7 — P0 complete | End of Phase 9 | Not started |

---

## 2. Active Architecture Decisions (ADRs)

These are locked. Do not silently deviate from them mid-implementation — if one turns out to be wrong, log it under Known Blockers (Section 3) and raise it explicitly; don't route around it in code.

| ID | Decision | Rationale (short) |
|---|---|---|
| ADR-001 | Ollama is the default/primary LLM provider. Gemini is optional, config-driven, never load-bearing for P0. | No quota ceiling; guarantees the benchmark run completes regardless of external service availability or model-name churn. |
| ADR-002 | The agent loop is hand-written (custom ReAct loop), not built on LangGraph or any agent framework. | Every line of agent decision logic must be something the team wrote and can explain/defend. |
| ADR-003 | PostgreSQL + pgvector is the only data store. No separate vector database. | One service to run/secure/demo at this corpus scale (80–150 docs); unjustified complexity otherwise. |
| ADR-004 | No `users` table, no `agents` table in the schema. | Single-user local tool (no auth in scope); one fully-owned reference agent (not a pluggable multi-agent platform). |
| ADR-005 | Repeated-execution baseline is k=2. k=3 is a P1 upgrade only, not default. | Conservative baseline given local-model runtime cost; k=2 produces a meaningful consistency signal without multiplying benchmark run time. |
| ADR-006 | All failure injection happens through exactly one wrapper point: `FailureInjector` around `ToolExecutor`/`LLMProvider`. No scattered failure-simulation conditionals elsewhere. | Keeps failure injection auditable, testable, and reversible via a single config flag. |
| ADR-007 | A composite reliability score, if built at all, is optional, explicitly weighted, and documented as a project-defined summary — it never replaces the per-metric Reliability Profile. | Prevents the single-score-as-truth failure mode; methodologically transparent and defensible. |
| ADR-008 | Benchmark floor is 40–50 hand-labeled items, single authored domain (not scraped). Growth beyond this is P1. | Labeling time is the most commonly underestimated part of this kind of project. |
| ADR-009 | Trace spans use `gen_ai.*`-style attribute naming (OTel-GenAI-inspired) stored in our own Postgres schema. No real OpenTelemetry Collector/Grafana stack deployed. | Naming-convention portability without operational overhead. |
| ADR-010 | No message broker or task queue. Batch benchmark runs execute as in-process `asyncio` background tasks, polled by the frontend. | Unjustified infrastructure for a single-user tool at this run volume. |

**To add a new ADR:** append a new row with the next ID, one-line decision, one-line rationale. Never delete or renumber existing ADRs, even if later superseded — mark superseded ADRs' rationale with `[SUPERSEDED BY ADR-0XX]` instead of removing.

---

## 3. Known Blockers / Technical Debt

Running log. Add an entry the moment something is discovered — don't wait until end of session.

**Format:**
```
### [OPEN|RESOLVED] <short title>
- Discovered: <date>, during <phase>
- Impact: <what it blocks or degrades>
- Notes: <detail>
- Resolution (if RESOLVED): <what fixed it, commit hash>
```

*(No real entries yet — project has not started implementation. Delete the example row below on first real entry.)*

### [EXAMPLE — delete this entry once real ones exist]
- Discovered: —, during Phase 0
- Impact: none, this is a format example
- Notes: shows the expected shape of a real entry.

---

## 4. Environment & Dependency State

Pin exact versions here as soon as they're installed in Phase 0 — an agent picking up a later session must reproduce the exact environment.

| Component | Target version (pin exactly in Phase 0) |
|---|---|
| Python | 3.11.x |
| FastAPI | 0.115.6 |
| SQLAlchemy | 2.0.36 (async) |
| asyncpg | 0.30.0 |
| Alembic | 1.14.0 |
| PostgreSQL | 15.x |
| pgvector extension | 0.3.6 |
| sentence-transformers | 3.3.1 |
| torch | 2.5.1+cpu |
| httpx | 0.28.1 |
| Ollama | pin local model name + quantization after Week 1 smoke test |
| google-generativeai | **do not pin in advance** — check live Gemini docs at time of P1 start |
| Node.js | 20.x LTS |
| React | 18.x |
| Vite | pin exact once installed |
| Tailwind CSS | pin exact once installed |
| Recharts | pin exact once installed |
| pytest | 8.3.4 |
| pytest-asyncio | 0.24.0 |

**Rule:** once a version is pinned, do not silently upgrade mid-project. If an upgrade is genuinely needed, log it as a new ADR (Section 2) and update this table in the same commit.

---

## 5. Session Handoff Note

*(Fill this in at the end of every session so the next session — human or agent — can resume without re-reading the whole history.)*

**Last session ended at:** Initial scaffold — Phase 0 not yet started
**Next task to pick up:** Issue #1 — Phase 0: Project Setup (FastAPI skeleton + Docker Compose + Ollama smoke test)
**Anything mid-flight / uncommitted:** none — clean state, scaffold committed
