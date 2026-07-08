# TRIAVANI — Progress Tracker

> Living status doc for the hackathon build. Update this file at every checkpoint instead of relying on tribal knowledge. Last reviewed against the codebase on **2026-07-08**.

## Legend

| Symbol | Meaning |
|:---:|---|
| ✅ | Implemented and working (real logic, not a mock) |
| 🟡 | Partially implemented — works, but mock/heuristic-backed or thin |
| 🚧 | Scaffolded only — interface exists, logic is minimal or a placeholder |
| ❌ | Not started / missing |

---

## 1. Snapshot

| Area | Status | One-line summary |
|---|:---:|---|
| Module A — Ingestion & Repository Understanding | ✅ | Real, working: clone/local ingest, Tree-sitter parsing, call graphs, dependency + CVE lookup, FAISS memory |
| Module B — Analysis & Knowledge | 🟡 | Real Semgrep + deterministic engine; LLM providers wired but default to mock; RAG is keyword search, not vector |
| Module C — Orchestration, Verification & Reporting | 🟡 | Full pipeline wired end-to-end; verification logic is rule-based/shallow; PDF export is a stub |
| API (FastAPI) | 🟡 | Core endpoints work; no auth, no persistence beyond SQLite, no CI |
| UI (Streamlit) | 🟡 | Functional demo UI; not production-grade (no auth, limited error states) |
| Tests | 🟡 | 21 test files across unit/integration/contract; no coverage reporting, no CI to run them automatically |
| DevOps / CI | ❌ | No `.github/workflows`, no lint/test automation, no deployment pipeline |
| Docs | ✅ | README, architecture, decisions, API list, team handoff all present |

---

## 2. Feature-by-feature Progress

### Module A — Ingestion & Repository Understanding (`services/ingestion/`)

| Feature | Status | Notes |
|---|:---:|---|
| Local path ingestion | ✅ | `local_path_ingestion.py` |
| Public Git URL shallow clone | ✅ | `git_ingestion.py`, `fetcher.py` |
| GitHub PR / GitLab MR diff extraction | ✅ | Uses `GITHUB_TOKEN` / `GITLAB_TOKEN` for API + private repos |
| File filtering (binaries, vendored dirs, etc.) | ✅ | `file_filter.py` |
| Tree-sitter parsing (Python) | ✅ | `parser.py` |
| Tree-sitter parsing (Java) | ✅ | `parser.py` |
| Parsing for other languages (JS/TS/Go/…) | ❌ | Explicit non-goal for hackathon scope (>3 languages) |
| Call graph construction + N-hop capping | ✅ | `graphs.py` |
| Dependency parsing (requirements/pyproject/pom/gradle/Dockerfile) | ✅ | `dependencies.py` |
| Known-CVE lookup via OSV.dev (with offline cache) | ✅ | `dependencies.py`, cached under `data/osv_cache` |
| Code chunking + embeddings into Shared Agent Memory (FAISS) | ✅ | `embedder.py`, `memory.py` — MiniLM if `sentence-transformers` installed, else hashing fallback |
| `CodeContext` (C1) build + JSON-schema validation | ✅ | `code_context_builder.py`, `schemas/code_context.schema.json` |
| CLI entry point | ✅ | `services/ingestion/cli.py` |
| Performance target (<2 min ingestion) | ✅ | Measured ~3.4s cold on `flask-realworld-example-app` (per README) |

**Module A is the most mature part of the codebase.**

---

### Module B — Analysis & Knowledge (`services/analysis/`, `services/knowledge/`)

| Feature | Status | Notes |
|---|:---:|---|
| Semgrep integration | ✅ | `semgrep_runner.py` runs `semgrep scan --json` against `rules/semgrep.yml` when the binary is available |
| Deterministic/heuristic security & bug detection | ✅ | `security_agent.py`, pattern checks (SQLi, `shell=True`, hardcoded secrets, bare `except`, etc.) |
| Severity / CWE / OWASP mapping | ✅ | `severity_mapper.py`, `finding_normalizer.py` |
| Real LLM-backed review (OpenAI-compatible endpoint) | ✅ (code) / ❌ (default) | `llm_provider.py` implements `OpenAICompatibleProvider`; **not exercised by default**, requires `LLM_PROVIDER=openai-compatible` + API key |
| Real LLM-backed review (Ollama) | ✅ (code) / ❌ (default) | `OllamaProvider` implemented, same caveat — needs a running Ollama instance |
| Mock LLM reviewer (default path) | ✅ | `MockLLMProvider` — deterministic pattern-based findings, no API key needed |
| CodeQL adapter | ❌ | Explicitly deferred ("future adapter") per `services/analysis/README.md` |
| Knowledge retrieval (keyword match over seed docs) | ✅ | `knowledge_retriever.py` matches finding title/CWE/OWASP against local markdown |
| Knowledge retrieval (vector/RAG via FAISS) | 🚧 | `rag_index.py` is an explicit placeholder — "keyword retrieval is v1" |
| Seed knowledge base content | 🟡 | 4 markdown docs (OWASP Top 10, CWE reference, secure Python, secure Java) — not a full corpus |
| Fine-tuned or specialized security model | ❌ | Out of scope by design |

**Bottom line:** the analysis pipeline runs and produces real findings out of the box (Semgrep + heuristics + mock LLM), but the "real LLM" and "real RAG" stories are code-complete, unwired-by-default, and unproven end-to-end.

---

### Module C — Orchestration, Verification & Experience

| Feature | Status | Notes |
|---|:---:|---|
| LangGraph workflow definition | ✅ | `services/orchestration/graph.py` |
| Sequential fallback (no LangGraph installed) | ✅ | Keeps local tests runnable per README |
| `run_pipeline()` stable integration entry point | ✅ | `services/orchestration/run_pipeline.py` |
| Pipeline nodes (ingest → security → review → retrieval → verify → decision → report) | ✅ | All 7 nodes present in `services/orchestration/nodes/` — each is intentionally thin (8–35 lines), delegating to the module logic |
| Confidence scoring | 🟡 | `confidence_scorer.py` — 17 lines, simple heuristic, not a calibrated model |
| Hallucination checks | 🟡 | `hallucination_checks.py` — 12 lines, basic sanity checks (e.g., file/line existence), not deep verification |
| Patch validation | 🟡 | `patch_validator.py` — 17 lines, structural checks only, no sandboxed execution |
| Risk gateway / human-approval routing | 🟡 | `risk_gateway.py` — 8 lines; routes on confidence threshold (90%) and severity, logic is minimal |
| Verifier agent (orchestrates the above) | 🟡 | `verifier_agent.py` — 21 lines |
| Human approval workflow (approve/reject API) | ✅ | `POST /api/reviews/{run_id}/findings/{finding_id}/approve|reject` implemented |
| Review report builder | ✅ | `report_builder.py` |
| Markdown export | ✅ | `markdown_exporter.py` |
| PDF export | 🚧 | `pdf_exporter.py` writes a **static placeholder PDF** unless `reportlab` renders it; not a real formatted report generator yet |
| SQLite persistence (runs, approvals) | ✅ | `services/reporting/repository.py` |
| FastAPI app + routes | ✅ | `apps/api/app/` — `/health`, `/api/reviews/run`, `/api/reviews/{id}`, `/api/reviews/{id}/report(.pdf)`, approve/reject, `/api/demo/status` |
| Streamlit UI | ✅ | `apps/ui/streamlit_app.py` (154 lines) — repo/PR input, run trigger, findings view, approval actions |
| Auth / access control on API or UI | ❌ | None — anyone with network access can trigger runs / approve findings |
| Rate limiting / abuse protection | ❌ | Not implemented |

---

### Contracts, Shared Packages & Data (`packages/`, `data/`)

| Feature | Status | Notes |
|---|:---:|---|
| C1 `CodeContext` contract (pydantic + JSON schema) | ✅ | `packages/contracts/models.py`, `schemas/code_context.schema.json` |
| C2 `Finding[]` contract | ✅ | `schemas/finding.schema.json` |
| C3 `ReviewReport` contract | ✅ | `schemas/review_report.schema.json` |
| Mock fixtures for parallel dev | ✅ | `mock_code_context.json`, `mock_findings.json`, `mock_review_report.json` |
| Shared config / logging / security helpers | ✅ | `packages/shared/` |
| Vector store (FAISS) on disk | ✅ | `data/faiss/` (populated by Module A) |
| Reports directory | ✅ | `data/reports/` |
| SQLite DB directory | ✅ | `data/sqlite/` |

---

### Tests

| Suite | Status | File count | Notes |
|---|:---:|:---:|---|
| Unit tests | 🟡 | 13 files | Covers ingestion (fetcher, graphs, parser, dependencies, file filter, git), analysis (semgrep mapping, security, LLM provider, finding normalizer), verification (confidence routing), knowledge retriever |
| Integration tests | 🟡 | 4 files | Ingestion pipeline, API smoke test, Module B on mock context, full pipeline demo |
| Contract tests | 🟡 | 1 file | Validates contract objects/schemas |
| Coverage measurement | ❌ | — | No `pytest-cov` / coverage report configured |
| CI-run tests | ❌ | — | Tests exist but nothing runs them automatically on push/PR |

> ⚠️ Test suite **could not be executed in this environment** (no network access to install dependencies), so pass/fail status is based on static code review, not a live run. Run `pytest` locally to confirm current pass rate before relying on this table for a demo.

---

## 3. What's Actually Missing / Not Done

These are gaps, not just "not verified":

1. **CI/CD** — no `.github/workflows`, no automated lint/test/build on push or PR.
2. **Real LLM review path is unproven** — code exists for OpenAI-compatible and Ollama providers, but the default demo path never exercises them; needs an end-to-end run with real keys to validate.
3. **RAG/vector knowledge retrieval** — `rag_index.py` is a stub; current retrieval is keyword matching over 4 seed docs, not FAISS-backed semantic search despite FAISS already being used for code memory.
4. **PDF export is a placeholder** — produces a minimal/static PDF unless `reportlab` is present and wired to real content; not yet a polished report.
5. **Verification agents are shallow** — confidence scoring, hallucination checks, and patch validation are all under 20 lines each; no calibration, no LLM-based self-critique, no sandboxed patch testing.
6. **No auth/access control** anywhere in the API or UI — not safe to expose publicly as-is.
7. **CodeQL adapter** — explicitly deferred, Semgrep is the only static analysis engine today.
8. **Language coverage** — Python + Java only (by design/non-goal), no JS/TS/Go/etc.
9. **No coverage reporting** — can't currently quantify how much of the codebase the 21 test files actually exercise.
10. **No deployment target** — Docker Compose exists for local dev; no staging/prod deployment config (K8s, cloud infra, secrets management).
11. **Knowledge base is thin** — 4 markdown files; no ingestion pipeline to grow it from external sources (OWASP site, CWE DB, etc.) beyond the static seed docs.
12. **No rate limiting / cost controls** — relevant once real LLM providers are turned on (token/cost budget per run).

---

## 4. Quick Reference — Where Things Live

| Concern | Path |
|---|---|
| Ingestion (Module A) | `services/ingestion/` |
| Analysis + Knowledge (Module B) | `services/analysis/`, `services/knowledge/` |
| Orchestration / Verification / Reporting (Module C) | `services/orchestration/`, `services/verification/`, `services/reporting/` |
| API | `apps/api/` |
| UI | `apps/ui/` |
| Contracts (C1/C2/C3) | `packages/contracts/` |
| Shared utilities | `packages/shared/` |
| Tests | `tests/unit/`, `tests/integration/`, `tests/contract/` |
| Demo/seeded-vulnerability repo | `demo_repos/vulnerable_python_java_demo/` |
| Docs | `docs/architecture.md`, `docs/decisions.md`, `docs/api.md`, `docs/team_handoff.md` |

---

## 5. Suggested Next Steps (priority order)

1. Wire and validate a real LLM provider end-to-end (pick one: OpenAI-compatible or Ollama) against the demo repo; compare findings to the mock provider baseline.
2. Add a minimal `.github/workflows/ci.yml` — `ruff check .` + `pytest` on every push/PR.
3. Replace the RAG placeholder with an actual FAISS-backed retriever over the seed docs (the FAISS plumbing already exists for code memory — reuse it).
4. Flesh out `pdf_exporter.py` into a real templated report (reportlab is already a dependency).
5. Strengthen verification: add LLM-based hallucination/self-critique pass, or at least widen the rule-based checks beyond the current ~15-line implementations.
6. Add basic auth (even a shared API key) before any public demo deployment.
7. Add `pytest-cov` and publish a coverage number so "tests exist" becomes "tests cover X%."

---

*This file reflects a static review of the repository contents on 2026-07-08. It was not validated by running the test suite (no network/dependency access in the review environment) — re-run `pytest` and update the Tests table before treating this as a certified status report.*
