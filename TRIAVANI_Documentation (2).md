# TRIAVANI — Project & Work Documentation

**AI Software Code Reviewer & Secure Development Agent**
Hackathon Problem Statement #06

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Repository Structure](#3-repository-structure)
4. [Core Concepts & Contracts](#4-core-concepts--contracts)
5. [Module-by-Module Explanation](#5-module-by-module-explanation)
6. [Glossary of Tools, Frameworks & Terms](#6-glossary-of-tools-frameworks--terms)
7. [How the Pipeline Runs, End to End](#7-how-the-pipeline-runs-end-to-end)
8. [Work Documentation — Fine-Tuned LLM Development for Agent Replacement](#8-work-documentation--fine-tuned-llm-development-for-agent-replacement)
9. [Current State Assessment — Real Gaps](#9-current-state-assessment--real-gaps)
10. [Summary](#10-summary)

---

## 1. Project Overview

**TRIAVANI** is an agentic AI system built for a hackathon that automatically **reviews software repositories end-to-end**. Given a repository (or a specific pull/merge request), it:

- Understands the structure of the codebase (files, functions, call graph, dependencies)
- Detects **bugs**, **security vulnerabilities**, and **coding anti-patterns**
- Retrieves relevant secure-coding knowledge (OWASP, CWE references, etc.)
- Generates **explainable** review comments and **suggested fixes**
- **Self-verifies** its own suggestions before presenting them
- Routes high-risk or low-confidence findings through a **human approval gateway**
- Produces a final **Code Review Report** (viewable in a web UI and exportable to PDF)

The project's own tagline describes it as *"a senior reviewer + security engineer in a box — with receipts (explainability) and a safety valve (human-in-the-loop)."*

### Hackathon Goals

| # | Goal | Success Metric |
|---|------|------|
| G1 | Ingest a GitHub/GitLab repo or PR and build a code context | Works on ≥2 languages (Python + Java) |
| G2 | Detect bugs, vulnerabilities & anti-patterns | Detect ≥70% of seeded issues in a demo repo |
| G3 | Generate explainable reviews + suggested fixes | Every finding has a "Why" + "How to fix" + reference |
| G4 | Verifier agent scores confidence; low-confidence/high-risk findings require human approval | Confidence gate set at a 90% threshold |
| G5 | One-click Code Review Report (UI + PDF export) | Full report generated in under 5 minutes per PR |

**Explicitly out of scope for the hackathon** (per the project's own README): auto-merging code without human approval, fine-tuning a model, supporting more than 3 programming languages, and production-grade CI/CD integration. (Section 8 of this document describes fine-tuning work done as a *proposed enhancement* on top of this baseline — see the discussion there for how it relates to this "non-goal.")

---

## 2. System Architecture

TRIAVANI is built as a **6-agent pipeline**, orchestrated as a directed graph, where each agent has a single, well-defined responsibility and agents pass data to each other only through **frozen JSON contracts** (schemas that are agreed upon up front and not allowed to change without team-wide agreement).

```
Streamlit UI  →  FastAPI  →  LangGraph workflow
                                    │
                                    ▼
        [1] Repository Understanding Agent
            (parses code, builds call graph, resolves dependencies/CVEs)
                                    │
                                    ▼
        [2] Security Analysis Agent
            (Semgrep static analysis + LLM findings, mapped to CWE/OWASP)
                                    │
                                    ▼
        [3] Code Review Agent
            (bugs, anti-patterns, style issues, suggested fixes)
                                    │
                    ⇄ [4] Knowledge Retrieval Agent
                        (RAG-style lookup over OWASP/CWE/secure-coding docs)
                                    │
                                    ▼
        [5] Verification & Self-Reflection Agent
            (validates findings, checks for hallucinations, scores confidence)
                                    │
              ┌─────── confidence ≥ 90% ───────┐
              ▼                                ▼
      Auto Recommendation          [6] Human Approval Gateway
              │                                │
              └───────────────┬────────────────┘
                               ▼
                Final Code Review Report (UI + PDF)
```

Throughout the pipeline, all agents read from and write to a **Shared Agent Memory** — a vector store (FAISS) with separate namespaces for code, dependencies, retrieved knowledge, findings, and decisions — so that later agents can "look back" at what earlier agents discovered.

### Why a graph, and why frozen contracts?

- **LangGraph** (see glossary) lets the team express the pipeline as a graph of nodes and edges instead of a single monolithic script, which makes it easy for three different people to build three different modules in parallel and have them slot together.
- **Frozen JSON contracts** (defined once as Pydantic models) mean any module can be developed and tested independently — as long as it consumes and produces the agreed-upon shape of data, it can be swapped out (e.g., swapping a mock LLM for a real fine-tuned model) without touching any other module. This is the architectural principle that makes the "Work Documentation" section of this file (fine-tuned model replacing two agents) possible without breaking anything downstream.

### Three Module Ownership Split

The hackathon team divided the system into three ownership modules:

| Module | Scope | Contents |
|--------|-------|----------|
| **Module A — Ingestion & Repository Understanding** | Agent [1] + Shared Agent Memory | `services/ingestion/` |
| **Module B — Analysis & Knowledge Agents** | Agents [2]–[4] | `services/analysis/`, `services/knowledge/` |
| **Module C — Orchestration, Verification & Experience** | Agents [5]–[6] + UI/report | `services/orchestration/`, `services/verification/`, `services/reporting/`, `apps/` |

---

## 3. Repository Structure

```
triavani-main/
├── apps/
│   ├── api/                    # FastAPI backend (HTTP API for running reviews)
│   └── ui/                     # Streamlit front-end (the web UI users interact with)
├── packages/
│   ├── contracts/               # Frozen Pydantic data contracts (C1, C2, C3) + JSON Schemas + mock data
│   └── shared/                  # Shared config, logging, security, and utility helpers
├── services/
│   ├── ingestion/                # Module A — repo cloning, parsing, dependency & CVE scanning
│   ├── analysis/                 # Module B — Semgrep + LLM-based security & review agents
│   ├── knowledge/                 # Module B — knowledge retrieval / RAG-style enrichment
│   ├── orchestration/             # Module C — LangGraph pipeline definition & nodes
│   ├── verification/               # Module C — patch validation, hallucination checks, confidence scoring
│   └── reporting/                  # Module C — report building, Markdown/PDF export, SQLite persistence
├── demo_repos/
│   └── vulnerable_python_java_demo/  # Deliberately vulnerable sample repo used for live demos
├── tests/
│   ├── unit/, integration/, contract/, fixtures/
├── docs/                          # architecture.md, decisions.md, api.md, team_handoff.md
├── data/                          # Local SQLite DB, FAISS vector index, generated PDF reports
├── docker-compose.yml, Dockerfile(s)
├── requirements.txt / pyproject.toml
└── Makefile
```

---

## 4. Core Concepts & Contracts

The entire system is glued together by **three frozen data contracts**, implemented as Pydantic models in `packages/contracts/models.py` and mirrored as JSON Schemas. Frozen means: no module is allowed to break these shapes without everyone agreeing, since every other module depends on them.

### C1 — `CodeContext` (Module A → B, C)

The structured "understanding" of a repository, produced by the ingestion module. Contains:

- `repo` — URL, branch, commit
- `files[]` — path, language, lines of code
- `functions[]` — id, file, name, start/end line, source code, resolved `callers`/`callees` (i.e. a call graph)
- `dependencies[]` — package name, version, and any `known_cves` (security vulnerabilities affecting that dependency)
- `diff[]` — the file/line hunks changed in a specific PR/MR, if reviewing a pull request rather than a whole repo
- `vector_store_path` — where the Shared Agent Memory for this run lives on disk

### C2 — `Finding` (Module B → C)

A single issue detected in the code — a bug, a vulnerability, a style problem, etc. Every finding carries:

- `type` (security / bug / quality / style), `severity` (critical/high/medium/low)
- `file`, `start_line`, `end_line` — exactly where the issue is
- `title`, `explanation_why`, `explanation_how_to_fix`, `suggested_patch` — the explainability requirement (Goal G3)
- `cwe_id`, `owasp_ref` — standardized vulnerability classification references
- `cvss_score` / `cvss_vector` — a severity score (0–10) and its formula components
- `taint_flow[]` — the path data takes from an untrusted source to a dangerous "sink" (for injection-style vulnerabilities)
- `references[]` — supporting documentation links/snippets from the knowledge base
- `source_agent` — which agent produced the finding (security or review)
- `raw_confidence` — a 0–1 score of how confident the system is
- `verification_result`, `approval_required`, `approval_status` — output of the Verification agent

### C3 — `ReviewReport` (Module C → user)

The final output shown to the user: aggregate `scores` (security score, quality score, overall confidence, risk level), the full list of verified `findings`, any `hallucination_flags`, the `approval` trail, and a natural-language `summary`.

Because every module only ever talks to these three contracts, the fine-tuned model described in Section 8 could be dropped in to replace two whole agents purely by making sure its output still matches the `Finding` schema — nothing else in the pipeline needed to change.

---

## 5. Module-by-Module Explanation

### Module A — Ingestion & Repository Understanding (`services/ingestion/`)

This is Agent [1]. Given a repo URL/local path (and optionally a PR/MR number), it:

- **Clones or loads the repository** (`fetcher.py`, `repository_loader.py`, `git_ingestion.py`, `local_path_ingestion.py`) — supports both local directories and shallow clones of public Git URLs, and pulls PR/MR diffs from the GitHub/GitLab APIs.
- **Parses source code** using **Tree-sitter** (`parser.py`) into functions, classes, and imports for **Python and Java**.
- **Builds a call graph** (`graphs.py`) — who calls whom — plus an "N-hop neighbor" capping mechanism so huge repos don't blow up processing time, and a module map for an architecture overview.
- **Parses dependency manifests** (`dependencies.py`) — `requirements.txt`, `pyproject.toml`, Maven `pom.xml`, Gradle files, Dockerfiles — and cross-references each dependency against **OSV.dev** (Open Source Vulnerabilities database) to find known CVEs, caching results locally so re-runs work offline.
- **Chunks and embeds code** (`embedder.py`) into the Shared Agent Memory, using either a real sentence-embedding model (MiniLM, via `sentence-transformers`) if installed, or a lightweight fallback "hashing embedder" if not — so the whole system still works even without heavy ML dependencies installed.
- **Assembles and validates** everything into the `CodeContext` contract (`code_context_builder.py`, `context_builder.py`), checked against a JSON Schema.
- Exposes a **CLI** (`cli.py`) for standalone use: `python -m services.ingestion.cli <repo_url_or_path> [--pr N] [--out FILE]`.

Per the project's own measurements, this module was reported as fully working, producing a `CodeContext` + populated vector store in ~3.4 seconds (cold) on a sample open-source repo.

### Module B — Analysis & Knowledge Agents (`services/analysis/`, `services/knowledge/`)

This covers Agents [2], [3], and [4].

**Security Analysis Agent** (`security_agent.py`, `semgrep_runner.py`, `analyzer.py`):
- Runs **Semgrep** (an open-source static analysis tool) using a custom rules file (`rules/semgrep.yml`) to find pattern-based security issues (e.g., SQL injection via string concatenation, unsafe `subprocess` calls, hardcoded secrets).
- Converts raw Semgrep JSON output — plus any deterministic scan results — into standardized `Finding` objects.
- `severity_mapper.py` and `risk_metadata.py` translate raw analysis results into severity levels and additional risk metadata (e.g., estimated CVSS score, taint-flow path).

**Code Review Agent** (`review_agent.py`, `llm_provider.py`):
- Uses an LLM (by default a `MockLLMProvider` that requires no API key, so the whole system is runnable offline/without credentials) to look for bugs, anti-patterns, and general code-quality issues that pattern-matching alone (Semgrep) would miss.
- The `llm_provider.py` module is designed as an adapter layer — real providers (OpenAI-compatible endpoints, Ollama-hosted local models) can be plugged in via environment variables without changing any other code. **This is the exact seam that Section 8's fine-tuned model integrates into.**

**Knowledge Retrieval Agent** (`services/knowledge/`):
- `knowledge_retriever.py` implements a keyword-matching enrichment step: it loads a handful of local Markdown "seed docs" (short notes on OWASP Top 10, CWE references, secure Python/Java practices) and attaches the most relevant reference strings (e.g., `"CWE-89: SQL Injection"`, `"OWASP Top 10: A03 Injection"`) to each finding based on keyword overlap.
- `rag_index.py` is explicitly labeled in the code as **a placeholder** for a future FAISS-backed semantic retriever — currently keyword matching is "v1."
- `finding_normalizer.py` ensures all findings, regardless of which agent produced them, conform to a single consistent shape before being handed to Module C.

### Module C — Orchestration, Verification & Experience (`services/orchestration/`, `services/verification/`, `services/reporting/`, `apps/`)

This covers Agents [5] and [6], plus the parts of the system a user actually sees.

**Orchestration** (`services/orchestration/`):
- `graph.py` wires the six pipeline stages together using **LangGraph**, a library for building agentic pipelines as explicit graphs of nodes and edges. If LangGraph isn't installed, `graph.py` automatically falls back to running the same nodes as a plain sequential Python loop, so tests and demos still work in constrained environments.
- `nodes/` contains one file per pipeline stage (`ingest_node.py`, `security_node.py`, `review_node.py`, `retrieval_node.py`, `verification_node.py`, `decision_node.py`, `report_node.py`) — each is a thin wrapper that calls into the relevant service module and updates the shared pipeline `state.py`.
- `run_pipeline.py` is the single stable integration entry point the whole team agreed to build against: `from services.orchestration.run_pipeline import run_pipeline`.

**Verification** (`services/verification/`) — Agent [5]:
- `verifier_agent.py` — for each finding, checks that the referenced file actually exists and that the line numbers are within the file's real bounds, then calls the confidence scorer and patch validator.
- `confidence_scorer.py` — nudges a finding's confidence score up slightly if the file exists, the line range is valid, and references were attached; also defines `requires_approval()`, which flags a finding as needing human sign-off if its severity is HIGH/CRITICAL or its confidence is below 90%.
- `patch_validator.py` — currently checks only that a suggested patch is non-empty, and for Python files, that the *existing* file still parses as valid Python via the `ast` module (Java files get a pass-through "structurally reviewable" message without any real check, and the suggested patch is never actually applied to test whether it *itself* is valid).
- `hallucination_checks.py` — currently a single check: does the file referenced in a finding actually exist in the repo?
- `risk_gateway.py` — applies the approval policy across all findings, tagging which ones must go through Agent [6] (human approval).

**Reporting** (`services/reporting/`):
- `report_builder.py` aggregates findings and scores into the final `ReviewReport` contract.
- `markdown_exporter.py` / `pdf_exporter.py` (using **ReportLab**) turn a report into a downloadable Markdown or PDF file.
- `repository.py` persists run metadata, reports, and approval decisions to a local **SQLite** database (kept intentionally simple/portable for a hackathon rather than using a hosted database).

**Applications** (`apps/`):
- `apps/api/` — a **FastAPI** backend exposing REST endpoints (`/health`, `/api/reviews/run`, `/api/reviews/{run_id}`, `/api/reviews/{run_id}/report`, `/api/reviews/{run_id}/report/pdf`, approve/reject endpoints, `/api/demo/status`).
- `apps/ui/` — a **Streamlit** web app that lets a user submit a repo, watch progress, review findings, approve/reject flagged ones, and download the final report — this is the primary way a human interacts with the system.

---

## 6. Glossary of Tools, Frameworks & Terms

This section explains, in plain language, every tool/technology/term used across the project (including in the fine-tuning work described in Section 8).

| Term | Explanation |
|---|---|
| **LLM (Large Language Model)** | A machine learning model trained on huge amounts of text/code that can generate and reason about natural language and source code. Used here to review code and explain issues in plain English. |
| **Agent** | In this project, a self-contained processing step (often powered by an LLM and/or static-analysis tool) with one job — e.g., the "Security Analysis Agent" only finds vulnerabilities; it doesn't write the final report. |
| **Agentic pipeline / multi-agent system** | An architecture where several specialized agents each do one job and pass structured data to the next, rather than one model doing everything in one shot. |
| **LangGraph** | A Python library (from the LangChain ecosystem) for building pipelines of LLM/agent steps as an explicit **graph** of nodes (processing steps) and edges (the order/conditions under which one step follows another). Lets you define branching logic (e.g., "if confidence < 90%, go to human approval instead of auto-recommend"). |
| **FastAPI** | A modern Python web framework used to build the project's REST API (`apps/api`). Automatically generates interactive API docs (Swagger UI) at `/docs`. |
| **Streamlit** | A Python framework for quickly building interactive data/web apps with plain Python (no HTML/CSS/JS needed). Used here to build the front-end UI (`apps/ui`). |
| **Pydantic** | A Python library for defining data models with type validation. Used here to define the three frozen contracts (`CodeContext`, `Finding`, `ReviewReport`) so that data passed between modules is always structurally correct. |
| **Tree-sitter** | A parsing library that builds a concrete syntax tree from source code in many languages (Python, Java, etc.), used here to extract functions, classes, and imports without writing a custom parser per language. |
| **Semgrep** | An open-source static analysis ("static application security testing"/SAST) tool that scans source code against pattern-based rules (e.g., "flag any `subprocess.run(..., shell=True)`") without executing the code. TRIAVANI uses a custom Semgrep rule file for its Security Analysis Agent. |
| **CodeQL** | GitHub's semantic code analysis engine (mentioned in the codebase as a **future**, not-yet-implemented adapter for deeper security analysis than Semgrep alone provides). |
| **CVE (Common Vulnerabilities and Exposures)** | A public, unique identifier for a known security vulnerability in a piece of software (e.g., in a specific version of a library). |
| **OSV.dev (Open Source Vulnerabilities)** | A public database/API of known vulnerabilities in open-source packages, queried by the ingestion module to check whether any of a repo's dependencies have known CVEs. |
| **CWE (Common Weakness Enumeration)** | A standardized catalog of *types* of software weaknesses (e.g., **CWE-89** = SQL Injection, **CWE-78** = OS Command Injection). Findings are tagged with the relevant CWE ID so a security-literate reviewer immediately recognizes the class of bug. |
| **OWASP Top 10** | A well-known, regularly updated list published by the Open Web Application Security Project ranking the ten most critical web-application security risks (e.g., "A03:2021 — Injection"). Findings reference the relevant OWASP category. |
| **MITRE (ATT&CK / CWE)** | MITRE Corporation maintains both the CWE catalog above and the ATT&CK framework (a knowledge base of real-world adversary tactics/techniques). Referenced in the project's requirements as a target knowledge source. |
| **CVSS (Common Vulnerability Scoring System)** | An industry-standard formula for scoring how severe a vulnerability is, from 0.0 (none) to 10.0 (critical), based on factors like attack complexity and impact. TRIAVANI currently *estimates* a CVSS-like score heuristically rather than computing it from a full CVSS vector calculator. |
| **Taint flow / taint analysis** | A technique for tracking how untrusted ("tainted") input data flows through a program from a "source" (e.g., a web request parameter) to a dangerous "sink" (e.g., a raw SQL query) — the classic way injection vulnerabilities are identified formally. TRIAVANI records a `taint_flow` field on findings, currently populated heuristically rather than via a real dataflow engine. |
| **RAG (Retrieval-Augmented Generation)** | A technique where, before (or while) an LLM generates an answer, the system first **retrieves** relevant reference text from a knowledge base and feeds it to the model as context — improving accuracy and letting the model cite real sources instead of relying purely on what it memorized during training. |
| **Vector store / vector database / embeddings** | To do retrieval, text (code, documents) is converted into **embeddings** — numeric vectors that capture semantic meaning — and stored in a **vector store**, which can then be searched for the vectors "closest" (most semantically similar) to a query. |
| **FAISS (Facebook AI Similarity Search)** | An open-source library for fast similarity search over vectors/embeddings. Used here as the backend for the project's Shared Agent Memory. |
| **MiniLM / sentence-transformers** | `sentence-transformers` is a Python library for turning text into embeddings; `MiniLM` is a small, efficient embedding model it can use. Used optionally in this project (falls back to a cheap "hashing embedder" if not installed, to keep the system lightweight by default). |
| **Hashing embedder** | A lightweight, non-neural fallback way of turning text into a numeric vector (e.g., via hashing tricks) — used automatically instead of MiniLM when the heavier `sentence-transformers`/`torch` dependency isn't installed, so the system still runs. |
| **Shared Agent Memory** | This project's term for its namespaced FAISS-backed store, with separate sections (`code`, `deps`, `secure_knowledge`, `findings`, `decisions`) so that different agents can write and later agents can search across everything discovered so far. |
| **Hallucination (in LLM context)** | When a language model confidently states something false or fabricated — e.g., claiming a vulnerability exists in a file/line that doesn't actually exist. The Verification agent's `hallucination_checks.py` module specifically checks for this failure mode. |
| **Confidence score** | A 0–1 number attached to each finding representing how sure the system is that the finding is correct/real, used to decide whether human approval is required (Goal G4). |
| **Human-in-the-loop / Human Approval Gateway** | A safety design pattern where, instead of the AI acting fully autonomously, certain decisions (here: low-confidence or high/critical-severity findings) are routed to a human for explicit approval or rejection before being finalized. |
| **Precision / Recall** | Standard evaluation metrics for a detection system. **Recall** = of all real issues that exist, what fraction did the system find? **Precision** = of everything the system flagged, what fraction was actually a real issue? The hackathon's Goal G2 target is ≥70% recall on a seeded demo repo. |
| **JSON Schema** | A specification format for describing the exact structure/types a piece of JSON data must have — used here to validate that `CodeContext`/`Finding`/`ReviewReport` outputs match the frozen contracts. |
| **SQLite** | A lightweight, file-based relational database (no separate server process needed) — used here to persist review runs, reports, and approval decisions locally, chosen for portability during the hackathon. |
| **ReportLab** | A Python library for programmatically generating PDF documents — used to export the final Code Review Report as a PDF. |
| **Docker / Docker Compose** | Docker packages an application and its dependencies into a portable "container"; Docker Compose lets you define and run multiple containers (e.g., the API, the UI) together with one command. Used here for easy local deployment. |
| **Mock provider / mock data (`MockLLMProvider`, `mock_findings.json`, etc.)** | Because the real LLM-calling agents cost money and require API keys, the project ships "mock" stand-ins that return realistic but hardcoded/deterministic responses — this lets every module and the full end-to-end demo run without any paid API key, and lets teammates develop against a module before the real version is finished. |
| **Ollama** | A tool for running open-source LLMs locally on your own machine, supported here as one of the optional pluggable providers for the Code Review Agent. |
| **OpenAI-compatible endpoint** | Many LLM providers (not just OpenAI) expose an API that matches OpenAI's request/response format, so a single "OpenAI-compatible" adapter in code can talk to many different backends by just changing a base URL/API key. |
| **Pytest / Ruff** | `pytest` is the testing framework used to run TRIAVANI's unit/integration/contract tests. `ruff` is a fast Python linter used to enforce code style/quality. |
| **Makefile** | A file defining shorthand commands (e.g., `make run-api`, `make run-ui`) to simplify running common project tasks. |

---

## 7. How the Pipeline Runs, End to End

1. A user (via the Streamlit UI, or directly via the FastAPI `/api/reviews/run` endpoint) submits a repo path/URL (and optionally a PR number).
2. **`run_pipeline()`** kicks off the LangGraph workflow (or its sequential fallback):
   `ingest_repository → security_analysis → code_review → knowledge_retrieval → verification → decision_gateway → build_report`
3. **`ingest_repository`** clones/loads the repo, parses it, builds the call graph, resolves dependency CVEs, and populates the Shared Agent Memory — producing a `CodeContext`.
4. **`security_analysis`** runs Semgrep (plus LLM-based analysis) over the `CodeContext` and produces a first batch of `Finding` objects.
5. **`code_review`** runs the (mock or real) LLM-based Code Review Agent to find additional bugs/anti-patterns, producing more `Finding` objects.
6. **`knowledge_retrieval`** enriches every finding with relevant OWASP/CWE reference strings pulled from the local knowledge base.
7. **`verification`** checks each finding's file/line validity, scores its confidence, and validates (very lightly) whether its suggested patch is structurally sound; also runs hallucination checks.
8. **`decision_gateway`** applies the approval policy: HIGH/CRITICAL severity or <90% confidence findings are flagged as `approval_required`.
9. **`build_report`** assembles the final `ReviewReport` (aggregate scores, all findings, hallucination flags, approval trail, summary) and persists it to SQLite.
10. The user views the report in the Streamlit UI (approving/rejecting flagged findings) and can export it as Markdown or PDF.

---

## 8. Work Documentation — Fine-Tuned LLM Development for Agent Replacement

> This section documents an individual contribution to the TRIAVANI project: replacing two of the LLM-powered agents above (the **Security Analysis Agent** and **Code Review Agent**, described in Module B, Section 5) with a single fine-tuned model.

### Project

AI Software Code Reviewer & Secure Development Agent

### Assigned Responsibility

The responsibility on this project was to replace the two LLM-powered analysis agents with a single domain-specific fine-tuned Large Language Model capable of performing both tasks:

- **Security Analysis Agent** (see `services/analysis/security_agent.py`)
- **Code Review Agent** (see `services/analysis/review_agent.py`)

Instead of relying purely on prompt engineering with an off-the-shelf model — which is what the pipeline's `MockLLMProvider`/pluggable `llm_provider.py` layer is designed to slot in by default — a fine-tuned version of **Qwen2.5-Coder-7B-Instruct** was developed to perform both responsibilities with higher consistency and lower inference cost.

### Objectives

The fine-tuned model was designed to:

- Detect software bugs
- Identify security vulnerabilities
- Detect coding anti-patterns
- Recommend fixes
- Produce structured JSON outputs compatible with the existing multi-agent pipeline (i.e., matching the `Finding` contract — see Section 4)
- Reduce hallucinations through supervised instruction tuning

### Work Completed

#### 1. Environment Setup

Configured a complete, reproducible training environment including:

- Hugging Face ecosystem
- **Transformers** — Hugging Face's library for loading and running pretrained/fine-tuned transformer models
- **TRL (Transformer Reinforcement Learning)** — Hugging Face's library for supervised fine-tuning and RLHF-style training loops
- **PEFT (Parameter-Efficient Fine-Tuning)** — Hugging Face's library implementing techniques like LoRA (below) so you don't have to retrain an entire multi-billion-parameter model
- **BitsAndBytes** — a library enabling 4-bit/8-bit quantization, i.e. compressing model weights so training/inference fits in less GPU memory
- **Accelerate** — Hugging Face's library for running training across different hardware setups (single GPU, multi-GPU, CPU) with minimal code changes
- Dataset management tooling
- GPU/CPU compatibility handling
- Persistent storage configuration (so checkpoints and datasets survive across sessions)

#### 2. Base Model Selection

Selected **Qwen2.5-Coder-7B-Instruct** — an open-source, 7-billion-parameter, instruction-tuned code-focused language model from Alibaba's Qwen series.

Reasons for this choice:

- Strong code understanding
- Instruction-following capability (it's already tuned to follow structured commands, which suits generating structured JSON findings)
- Open-source (no licensing cost/API dependency)
- Efficient to fine-tune with LoRA at its size (7B parameters is small enough to fine-tune on modest hardware)
- Well-suited for software review tasks specifically (as opposed to a general-purpose chat model)

The training notebook initializes this model before training begins.

#### 3. Dataset Preparation

Prepared datasets for supervised fine-tuning by:

- Loading multiple raw datasets
- Cleaning samples
- Standardizing outputs (so every training example follows the same target format)
- Removing duplicates
- Creating **leakage-safe** train/test splits (ensuring no example appears in both sets, which would make evaluation numbers artificially inflated)
- Using **stratified sampling** for balanced learning (making sure, e.g., different bug/vulnerability types are proportionally represented in training rather than the model only ever seeing one dominant category)

#### 4. Unified Instruction Schema

Designed a unified instruction format allowing one model to perform multiple software engineering tasks (security analysis *and* code review) instead of needing two separate specialized models.

The model learns to output structured findings including:

- Issue presence (is there actually a problem here or not?)
- Issue type
- Severity
- Title
- Explanation
- Suggested fix
- CWE references
- OWASP references
- Confidence

This schema was designed to be directly compatible with the project's `Finding` JSON contract (Section 4, C2) — meaning the fine-tuned model's raw output maps cleanly onto what the rest of the pipeline (`finding_normalizer.py`, verification, reporting) already expects, requiring no changes downstream.

#### 5. Teacher Distillation

Implemented a **teacher-distillation** pipeline, where a stronger "teacher" model generates high-quality responses that are then used as supervision (training targets) for the smaller model being fine-tuned. This is a common technique for producing a compact, efficient model that mimics the reasoning quality of a much larger/more expensive one.

This improves:

- Reasoning quality
- Explanation quality
- Structured-output reliability
- Consistency across similar inputs

#### 6. LoRA Fine-Tuning

Applied **Parameter-Efficient Fine-Tuning (PEFT)** using **LoRA (Low-Rank Adaptation)** — a technique that freezes the original model's weights and instead trains small, additional "adapter" matrices inserted into the model's layers.

Benefits:

- Lower GPU memory usage than full fine-tuning
- Faster training
- Small adapter checkpoint files (megabytes, not the tens of gigabytes a full fine-tuned model would take)
- Easy deployment — the adapter can simply be loaded on top of the unmodified base model at inference time

Only the LoRA adapter weights are trained; the base Qwen2.5-Coder model itself remains frozen throughout.

#### 7. Training Pipeline

Implemented a complete supervised fine-tuning pipeline including:

- Prompt formatting (turning raw examples into the model's expected chat/instruction format)
- Tokenizer setup
- Dataset conversion (into the tensor format the trainer expects)
- Trainer configuration (learning rate, batch size, epochs, etc.)
- Checkpoint saving
- Logging
- Evaluation hooks

#### 8. Model Evaluation

Evaluated the fine-tuned model on unseen (held-out) samples to verify:

- Prediction quality
- JSON validity (does the model reliably produce well-formed, schema-conformant JSON?)
- Reasoning quality
- Output consistency across similar/repeated inputs

#### 9. Adapter Packaging

Packaged the trained LoRA adapter for integration into the agentic workflow. The adapter can be loaded on top of the base Qwen2.5-Coder model during inference — meaning deployment just requires distributing a small adapter file alongside a reference to the public base model, rather than a full duplicated multi-billion-parameter checkpoint.

### Impact on Project Architecture

**Original Design**

```
Repository Context
      ↓
Security Analysis Agent (LLM)
      ↓
Code Review Agent (LLM)
      ↓
Verification Agent
```

**Updated Design**

```
Repository Context
      ↓
Fine-Tuned Qwen2.5-Coder Model
(Performs Security Analysis + Code Review)
      ↓
Verification Agent
```

The replacement preserves downstream interfaces because the fine-tuned model generates outputs matching the standardized `Finding` schema expected by the orchestration module (Section 4, C2) — this is only possible *because* the original architecture strictly separated agents behind frozen contracts (Section 2), so one combined model could stand in for two separate agent slots without any other module needing to change.

### Technologies Used

- **Python** — primary programming language
- **PyTorch** — the deep learning framework underlying the training and inference
- **Hugging Face Transformers** — model loading, architecture, and generation utilities
- **TRL** — supervised fine-tuning trainer utilities
- **PEFT** — parameter-efficient fine-tuning framework
- **LoRA** — the specific PEFT technique used
- **BitsAndBytes (4-bit quantization)** — memory-efficient model loading/training
- **Accelerate** — hardware-agnostic training orchestration
- **Datasets** (Hugging Face `datasets` library) — dataset loading/processing
- **Qwen2.5-Coder-7B-Instruct** — the base model fine-tuned

### Deliverables

- Fine-tuned Qwen2.5-Coder LoRA adapter
- Training notebook
- Dataset preprocessing pipeline
- Teacher distillation pipeline
- Evaluation pipeline
- Packaged model, ready for integration into the multi-agent system (i.e., ready to be wired in behind `services/analysis/llm_provider.py` in place of `MockLLMProvider`/an off-the-shelf API model)

### Summary

Successfully replaced the two original LLM-based analysis agents (Security Analysis Agent and Code Review Agent) with a single fine-tuned Qwen2.5-Coder-7B-Instruct model, trained using LoRA-based supervised fine-tuning and teacher distillation. The resulting model performs both secure code analysis and general code review while producing standardized JSON outputs compatible with the existing agentic architecture, reducing reliance on prompt engineering and improving consistency.

---

## 9. Current State Assessment — Real Gaps

An honest look at the current repository state (as of this documentation) shows several gaps between what the Problem Statement/PRD asks for and what is currently implemented. These are worth tracking explicitly, both to prioritize remaining work and to be ready for questions from hackathon judges.

### 1. No eval harness / recall proof (blocks Success Metric G2: ≥70% recall)

There is a demo repo (`demo_repos/vulnerable_python_java_demo`) with only **5 seeded issues across 2 files**, but no script that actually runs the pipeline against it and reports **precision/recall** against that seeded ground truth. If a judge asks *"how do you know it's ≥70%?"*, the honest answer right now is "we haven't measured it."

This is the **single highest-leverage gap to close**. A small evaluation script — in the spirit of the project's own "Module D" pattern: run the pipeline, diff the resulting findings against a `seeded_issues.json` ground-truth file, and print a recall percentage — would be roughly 30 lines of code and would let the team cite a real, defensible number in the demo instead of an assertion.

### 2. The knowledge base is a stub, not RAG in any meaningful sense

The `services/knowledge/seed_docs/*.md` files are each only **147–178 bytes** — one-liners, not a real OWASP/CWE/MITRE corpus (see Section 6 exact byte counts observed: `cwe_reference.md` 147B, `owasp_top_10.md` 178B, `secure_java.md` 166B, `secure_python.md` 170B). The project's own PRD explicitly calls for retrieving *"OWASP, CWE, MITRE data... project docs, wiki, APIs, best practices."* Right now findings do cite references, but there is no real retrieval happening over substantive content — and `rag_index.py` is literally commented in the code as **a placeholder** for a future FAISS-backed retriever, with keyword matching (`knowledge_retriever.py`) standing in as "v1."

Pulling in even a few real OWASP Top 10 write-ups and CWE descriptions (a few KB each, rather than one-liners) would make the "RAG" claim defensible.

### 3. Verification is shallower than what the PRD asks for

Looking at `services/verification/`:

- `verify_findings()` in `verifier_agent.py` checks only that the referenced file exists and that the line range is plausible.
- `validate_patch_shape()` in `patch_validator.py` only AST-parses **existing** Python files to confirm they currently parse — Java patches get a no-op **"structurally reviewable"** pass-through message, and in neither language is the *suggested patch itself* actually applied to test whether the patched result is valid.
- `hallucination_checks.py` implements exactly **one** check: does the referenced file exist.

The PRD's stated Definition of Done for verification is effectively *"does the fix apply? does the code still parse/compile?"* — but the pipeline is not actually applying patches or compiling/parsing anything post-patch. This is a reasonable hackathon-scope shortcut, but it is the weakest link if a judge probes it. At minimum, applying the suggested patch to a **temporary copy of the file** and re-parsing it (for both Python and Java) would substantially strengthen this module.

### 4. No inline PR integration

The PRD's developer-facing user story is *"I open a PR and get inline review comments I can accept/reject."* Today, everything lives in the internal Streamlit UI (`apps/ui/streamlit_app.py`) — there is no code posting comments back to a GitHub/GitLab pull-request review-comments API. This isn't required for the demo flow that has already been built around the UI, but if a judge maps the current implementation literally against that specific user story, it's a visible gap. It's worth explicitly deciding — and stating on a slide — whether to build this or scope it out.

### 5. CVSS/taint-flow are heuristic, not computed

The `Finding.cvss_score`/`cvss_vector` and `taint_flow` fields exist in the contract (Section 4), but nothing in `services/analysis/risk_metadata.py` or elsewhere runs a real CVSS vector calculator or a real dataflow/taint-tracking engine — these values are derived heuristically. That's a legitimate hackathon shortcut, but the demo and any written materials should say **"estimated"** rather than implying a real CVSS calculation or dataflow engine — a security-literate evaluator will notice a fake-precise CVSS score immediately.

### 6. No DeepEval/MLflow/Phoenix observability, despite appearing in the project's technology list

The project references an eval/observability stack (tools like DeepEval, MLflow, or Phoenix) as part of its intended toolset, but nothing in the current codebase logs runs through any of them. This is explicitly optional per the PRD ("participants may choose alternatives"), but it is directly connected to gap #1 above: if an eval harness is built to get a real recall number, logging those evaluation runs through a tool like **DeepEval** (an open-source LLM evaluation framework) would provide a free, credible talking point — *"we use industry-standard eval tooling"* — for judges.

### Summary of Gaps

| # | Gap | Severity | Effort to Close |
|---|-----|----------|------------------|
| 1 | No eval harness / recall proof | 🔴 Highest priority — blocks the core success metric | Low (~30-line script) |
| 2 | Knowledge base is a stub, not real RAG | 🟠 High — undermines a headline architectural claim | Low–Medium (source real OWASP/CWE text) |
| 3 | Verification doesn't apply/compile patches | 🟠 High — weakest link if probed by judges | Medium |
| 4 | No inline PR integration | 🟡 Medium — gap vs. a specific user story, not the demo flow | Medium–High |
| 5 | CVSS/taint-flow are heuristic | 🟡 Medium — messaging risk, not a build risk | Low (fix wording) |
| 6 | No eval/observability tooling in use | 🟢 Low — optional, but pairs well with fixing #1 | Low–Medium |

---

## 10. Summary

TRIAVANI is a six-agent, contract-driven pipeline for automated, explainable, human-supervised code review and security analysis, built around Python (FastAPI, Streamlit, LangGraph, Semgrep, Tree-sitter, FAISS) with a strong emphasis on modular independence via frozen JSON/Pydantic contracts. On top of this baseline architecture, a fine-tuned **Qwen2.5-Coder-7B-Instruct** model (via LoRA and teacher distillation) was developed to consolidate the Security Analysis and Code Review agents into a single, more consistent, lower-cost model that still speaks the same `Finding` contract the rest of the system expects. The project currently demonstrates the full pipeline end-to-end on a seeded demo repository, but has clearly identified gaps around quantitative recall proof, the depth of its knowledge-retrieval and patch-verification steps, PR-native integration, and the honesty of its risk-scoring language — all of which are tracked above as prioritized next steps.
