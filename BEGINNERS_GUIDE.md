# TRIAVANI, Explained for Complete Beginners

> A plain-language walkthrough of what this project is, how it works, and how mature it actually is. Written to be understood with no prior context about the codebase.

---

## What is TRIAVANI, in one sentence?

It's a program that reads someone's code repository the way a senior engineer would during a code review — spotting bugs, security holes, and bad practices — and then explains *why* it's a problem, suggests a fix, double-checks its own suggestion, and only lets a human sign off when it's genuinely confident.

It was built for a hackathon (the problem statement is literally attached as a `.pptx` file in the project), so it's a prototype, not a finished product — but a surprisingly complete one.

---

## The core idea: an assembly line of specialists

Instead of one AI trying to do everything, TRIAVANI splits the job into **6 agents**, each handling one narrow task and passing its work to the next — like a factory assembly line.

```
Repo or PR input
      |
      v
Agent 1: ingest and understand
  (parses code, builds call graph)
      |
      v
Agents 2-4: analyze and review
  (security scan, code review, docs lookup)
      |
      v
Agent 5: verify and score
  (confidence, hallucination checks)
      |
      +-------------------+
      |                   |
      v                   v
Auto recommendation   Human approval gate
(confidence >= 90%)   (low confidence or high risk)
      |                   |
      +-------------------+
                |
                v
      Code review report
```

Here's what each stage actually does, explained simply:

### Agent 1 — "Read and understand the code" (Ingestion)

Before you can review code, you have to *understand* it. This agent:

- Downloads or reads the repository (a GitHub URL, a GitLab merge request, or just a folder on disk)
- Breaks the code into pieces — functions, classes, imports — using a tool called **Tree-sitter** (a parser that understands code structure, not just text)
- Builds a **call graph**: a map of which functions call which other functions (so the reviewer knows "if I change this function, what else might break?")
- Looks at the project's dependencies (like `requirements.txt` or `pom.xml`) and checks them against a public vulnerability database (**OSV.dev**) to catch "you're using a library with a known security bug"
- Stores everything in a searchable memory (using **FAISS**, a fast similarity-search library) so later agents can look things up

Think of this as a paralegal who reads the entire case file and builds an index before the lawyer even starts arguing.

**This is the most finished part of the whole project** — it's real, it's fast (about 3.4 seconds on a real open-source repo), and it works on both Python and Java code.

### Agents 2–4 — "Find the problems and explain them" (Analysis + Knowledge)

This group does three related jobs:

- **Security scanning**: runs a tool called **Semgrep** (an industry-standard static analysis scanner) plus some hand-written pattern checks, to catch things like SQL injection, hardcoded passwords, or unsafe `shell=True` calls
- **Code review**: looks for general bugs and bad practices (bare `except:` blocks, functions doing too much, duplicated logic) — this can be done by simple pattern-matching *or* handed off to a real AI language model
- **Knowledge retrieval**: for every problem it finds, it attaches a reference — "this is CWE-89: SQL Injection" or "this is OWASP Top 10, category A03" — so the explanation isn't just "this is bad," it's backed by an actual standard

### Agent 5 — "Don't trust yourself blindly" (Verification)

This is the safety-conscious part. Before showing a finding to a human, it asks:

- Does this file/line actually exist? (catches the AI "hallucinating" a problem that isn't real)
- Does the suggested patch make structural sense?
- How confident are we, really, on a 0–100% scale?

### Agent 6 — "When in doubt, ask a human" (Approval Gateway)

If confidence is **90% or higher**, TRIAVANI auto-recommends the fix. If it's lower, or the issue is high-risk, it doesn't just barrel ahead — it flags the finding for a human to explicitly approve or reject. This is the "safety valve" the project talks about: the AI never silently applies a fix.

### The output — a Code Review Report

Everything gets bundled into a report you can view in a web UI or export, with every finding showing: **what's wrong, why it matters, how to fix it, and a reference to a real security standard**.

---

## The tech stack, explained without jargon

| Piece | What it is | Why it's used here |
|---|---|---|
| **Python** | The programming language the whole backend is written in | Good ecosystem for AI/parsing tools |
| **FastAPI** | A framework for building web APIs | Powers the backend that the UI talks to |
| **Streamlit** | A framework for quickly building simple web UIs in Python | The demo dashboard people click through |
| **Tree-sitter** | A code parser | Turns raw source code into a structured tree (functions, classes) instead of just plain text |
| **Semgrep** | A static security scanner | Finds known-bad code patterns without running the code |
| **FAISS** | A vector similarity search library | Lets the system quickly "look up" similar code or documentation |
| **LangGraph** | A framework for chaining AI agent steps together | Defines the 6-stage pipeline as a graph/workflow |
| **Pydantic** | A data-validation library | Enforces that data passed between agents is structurally correct |
| **SQLite** | A lightweight embedded database | Stores run history and human approval decisions |
| **Docker Compose** | A tool to run multiple services together | Lets you spin up the API + UI with one command |

One important design choice: **the AI-review part defaults to a "mock" (fake but realistic) model** instead of a real paid AI, so anyone can run the whole demo without an API key or internet cost. Real AI providers (OpenAI-compatible endpoints, or a locally-run model via Ollama) are supported in the code, but you have to explicitly turn them on.

---

## The "contracts" — why this is safe for a team to build together

This is a clever bit of engineering discipline. Three teams were building three different pieces of this system in parallel (ingestion, analysis, orchestration), and to keep them from breaking each other's work, they agreed on **three fixed data shapes** that never change without everyone agreeing:

- **C1 — CodeContext**: what Agent 1 hands off (the parsed repo)
- **C2 — Finding**: what Agents 2–4 hand off (each individual bug/vulnerability found)
- **C3 — ReviewReport**: the final packaged report

Because these shapes are "frozen," each team could build and test their part against **fake sample data** (`mock_code_context.json`, etc.) without waiting for the other teams to finish. This is a very standard, very good real-world engineering practice — it's basically an API contract between teams.

---

## How mature is it, really?

Honestly assessed:

- **Ingestion (Agent 1)** — genuinely solid, not a toy. Real parsing, real CVE lookups, fast.
- **Analysis (Agents 2–4)** — real security scanning works out of the box; the "real AI reviewer" code exists but isn't turned on by default; the "knowledge lookup" is currently simple keyword matching, not true AI-powered search (that part is explicitly marked as a placeholder in the code).
- **Verification & reporting (Agents 5–6)** — the pipeline is fully wired end-to-end and runs, but the actual "how confident are we" logic is quite simple (a handful of rule-based checks, not a calibrated model), and PDF report export is a bare-bones placeholder.
- **No automated testing pipeline** — there are 21 test files, but nothing runs them automatically when code changes (no CI).
- **No login/security on the API or UI** — fine for a hackathon demo, not fine for anything public.

For the full, itemized feature-by-feature breakdown (what's done, what's mocked, what's missing), see `PROGRESS.md` in this repository.

---

## How you'd actually try it

```bash
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
make run-api     # starts the backend at localhost:8000
make run-ui       # starts the dashboard at localhost:8501
```

Or with Docker:

```bash
docker compose up --build
```

There's also a folder called `demo_repos/vulnerable_python_java_demo` — a deliberately broken sample project seeded with known bugs, so you can point TRIAVANI at it and watch it catch things, without needing a real-world repository.

- FastAPI docs: http://localhost:8000/docs
- Streamlit UI: http://localhost:8501
- Health check: http://localhost:8000/health

---

## Glossary (quick reference)

| Term | Plain-language meaning |
|---|---|
| **Agent** | One focused step in the pipeline, done by code (and sometimes an AI model) that has one job |
| **Static analysis** | Checking code for problems without actually running it |
| **CVE** | A publicly catalogued, known security vulnerability in a piece of software |
| **CWE** | A standard catalogue of *types* of software weaknesses (e.g. "CWE-89: SQL Injection") |
| **OWASP Top 10** | A well-known list of the ten most critical web-application security risks |
| **RAG (Retrieval-Augmented Generation)** | Looking up relevant reference material before answering, instead of relying purely on memorized knowledge |
| **Hallucination** | When an AI confidently states something false (e.g. references a file/line that doesn't exist) |
| **Confidence score** | A number representing how sure the system is that a finding is correct and safe to act on |
| **Mock** | A fake but realistic stand-in used for testing/demos, so the real (often paid or slow) version isn't required |
| **Pipeline / workflow** | The fixed sequence of steps data moves through, start to finish |

---

*This guide reflects a review of the repository contents as of 2026-07-08. For a more technical, itemized status table, see `PROGRESS.md` in the same repository.*
