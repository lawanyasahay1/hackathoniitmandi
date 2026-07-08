# TRIAVANI — Project Status vs. PRD

A gap analysis of the `triavani-main` codebase against the Hackathon PS #06 PRD, based on reading the actual source (not just module READMEs).

## Quick verdict

Only **Module A (ingestion)** is a real, working implementation. Everything downstream of it runs end-to-end, but on **rule-based heuristics and placeholders standing in for the "AI" parts** the PRD describes. The pipeline plumbing (contracts, orchestration, approval gate, reporting) is genuinely built — it's the *intelligence* inside each agent that's still a stub.

---

## Goal-by-goal (PRD Section 2)

| Goal | PRD target | Status | What's actually there |
|---|---|---|---|
| **G1** — Ingest repo/PR, build code context | ≥2 languages | ✅ **Done** | Real Tree-sitter parsing for Python + Java, real call graphs, real OSV.dev CVE lookups. Measured 3.4s on a real repo — beats the <2min target. |
| **G2** — Detect bugs/vulns/anti-patterns | ≥70% recall on seeded issues | ⚠️ **Partial** | Detection *runs*, but on regex/keyword heuristics, not the Code LLMs (Code Llama/DeepSeek-Coder/etc.) the PRD specifies. No recall number has actually been measured against the seeded datasets. |
| **G3** — Explainable reviews + fixes | Every finding has Why + How + reference | ⚠️ **Partial (shape only)** | The `Finding` object always has these *fields* populated — but the explanations come from canned templates tied to regex matches, not from an LLM reasoning about the specific code. |
| **G4** — Verifier scores confidence, gates at 90% | Confidence gate at 90% threshold | ⚠️ **Partial** | The gate itself is real and correctly implemented (`risk_gateway.py`, `confidence_scorer.py`). What's missing: real hallucination detection against an actual LLM's output — right now there's little for it to catch, since nothing upstream is LLM-generated yet. |
| **G5** — One-click report, UI + PDF, <5min | Full report in <5 min per PR | ✅ **Done (mechanically)** | Streamlit UI, FastAPI endpoints, and PDF export (via `reportlab`, with a raw-bytes fallback if it's not installed) all work. Speed target is trivially met since the "AI" steps are instant lookups, not real inference. |

---

## Non-goals — correctly respected

| Non-goal | Status |
|---|---|
| No auto-merge without approval | ✅ Enforced — `services/verification` never applies patches, only suggests them |
| No fine-tuning | ✅ N/A by design |
| ≤3 languages | ✅ Python + Java only, as scoped |
| No production CI/CD | ✅ Demo-only, as scoped |

---

## Agent-by-agent (6-agent architecture)

| # | Agent | PRD spec | Built | Real or mock? |
|---|---|---|---|---|
| 1 | Repository Understanding | AST, call graph, deps, CVEs | ✅ | **Real** — Tree-sitter, real dependency file parsing, real OSV CVE lookups |
| 2 | Security Analysis | Semgrep/CodeQL + LLM reasoning, CWE/OWASP, CVSS | ⚠️ | **Half-real** — Semgrep integration is genuine (runs the actual binary if installed), but silently returns nothing if Semgrep isn't present, with no LLM reasoning layer at all. CodeQL was never started — explicitly deferred in `docs/decisions.md`. |
| 3 | Code Review | Code LLM for bugs/anti-patterns/fixes | ❌ | **Mock** — `MockLLMProvider` is literally regex checks (`except:` with no type, `TODO`/`FIXME` strings) mapped to canned finding text. No Code Llama/DeepSeek/Qwen call exists anywhere in the code. |
| 4 | Knowledge Retrieval | RAG over OWASP/CWE/docs via FAISS | ❌ | **Placeholder** — `RagIndex` docstring literally says "Placeholder for a future FAISS-backed index; keyword retrieval is v1." It matches keywords against a handful of local markdown files, not a real retrieval pipeline. |
| 5 | Verification & Self-Reflection | Validate patches, detect hallucinations, confidence score | ⚠️ | **Real gate, thin substance** — the scoring/routing logic is properly implemented, but there's minimal hallucination surface to detect since nothing upstream is LLM-generated yet |
| 6 | Human Approval Gateway | Review high-risk changes, approve/reject | ✅ | **Real** — approve/reject API endpoints and routing logic work as designed |

---

## Also missing entirely

- **Graph Neural Networks / code graph representation** — explicitly dropped as out-of-scope in `docs/decisions.md`, despite being listed as a technique in the original PS.
- **Real LLM provider wiring** — `openai-compatible` and `ollama` provider options exist as config knobs (`LLM_PROVIDER` env var) but there's no evidence either path is actually implemented and exercised, only the mock path.
- **Measured recall against seeded datasets** — the PRD's core success metric (≥70% recall on Juliet/OWASP Benchmark) has no evaluation harness or logged result anywhere in the repo; only Module A has a measured benchmark (its 3.4s ingestion time).
- **DeepEval/Phoenix/MLflow eval logging** — listed as tooling in the PS, not present in `requirements.txt` or anywhere in code.

---

## Bottom line

Think of it as a fully-wired **skeleton with one real limb**. The "assembly line" — contracts, orchestration, gating, UI, reporting — is legitimately production-shaped and would accept real AI components without any interface changes (that was the whole point of freezing C1–C3 early). But the actual *reviewing intelligence* — the part a judge would care about most, i.e. "can it really catch bugs and vulnerabilities using AI" — is currently simulated by if/regex checks standing in for Code LLM calls and a real RAG index.

**Highest-leverage next step:** wire one real LLM provider into `services/analysis` and a real FAISS-backed retriever into `services/knowledge`.
