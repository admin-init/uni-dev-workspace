# AGENTS.md — uni-dev

> Universal Backend Development Framework · Multi-Agent SDLC Automation
> 
> DDD → SDD → TDD across two repositories: `uni-kb` (Knowledge Base) + `uni-dev` (Agent System)

---

## Project Structure

```
uni-dev/                             # Monorepo root (this repo)
│
├── plan.html                        # Visual implementation plan (Catppuccin Mocha)
├── AGENTS.md                        # ← You are here
│
├── uni-kb/                          # Repository 1 — Knowledge Base Library
│   ├── README.md                    #   Architecture, phases, API docs
│   ├── pyproject.toml               #   Package config, deps, entry points
│   ├── src/uni_kb/
│   │   ├── __init__.py
│   │   ├── parsers/                 #   Plugin-based language parsers
│   │   │   ├── base.py              #     Abstract ParserPlugin interface
│   │   │   ├── registry.py          #     entry_points discovery & loading
│   │   │   ├── java/                #     Java/Spring Boot
│   │   │   │   ├── controller.py    #       @RestController → endpoints
│   │   │   │   ├── service.py       #       @Service → business logic
│   │   │   │   ├── entity.py        #       @Entity → models
│   │   │   │   └── mapper.py        #       MyBatis XML → SQL queries
│   │   │   └── nodejs/              #     Node.js/Express/NestJS
│   │   │       ├── route.py         #       router.get() / @Get() → endpoints
│   │   │       ├── service.py       #       Service classes → logic
│   │   │       ├── model.py         #       Sequelize/TypeORM → models
│   │   │       └── middleware.py    #       Auth middleware chains
│   │   ├── store/                   #   Data storage layer
│   │   │   ├── sqlite_store.py      #     8 tables (modules, classes, methods, ...)
│   │   │   ├── chroma_indexes.py    #     14 named collections (embedding + BM25)
│   │   │   └── code_graph.py        #     NetworkX graph (8 node types, 14 edge types)
│   │   ├── generators/              #   Spec generators (6)
│   │   │   ├── api_contract.py      #     Controller AST → OpenAPI 3.0 YAML
│   │   │   ├── business_logic.py    #     Service AST → Markdown pseudo-code
│   │   │   ├── data_model.py        #     Entity + DB → YAML model spec
│   │   │   ├── auth_matrix.py       #     Permissions → matrix YAML
│   │   │   ├── config_catalog.py    #     .env/YAML → config catalog
│   │   │   └── migration_checklist.py#    Dep graph → prioritized checklist MD
│   │   └── mcp_server.py            #   20 MCP tools, 5 categories
│   ├── specs/                       #   Per-component specifications (YAML)
│   ├── docs/                        #   Architecture decisions, domain model
│   └── tests/                       #   Mirroring src/uni_kb/ structure
│
└── uni-dev/                         # Repository 2 — Agent Automation System
    ├── README.md                    #   Pipeline flow, agent specs, CLI
    ├── pyproject.toml               #   Package config, deps (depends on uni-kb)
    ├── src/uni_dev/
    │   ├── __init__.py
    │   ├── main.py                  #   CLI entry point (click)
    │   ├── orchestrator.py          #   create_orchestrator() — clean create_deep_agent() call
    │   ├── core/                    #   Business-Semantic SOP Graph (v3)
    │   │   ├── graph.py             #     compile_pipeline() — StateGraph with conditional edges
    │   │   ├── state.py             #     PipelineState TypedDict (10 fields)
    │   │   ├── gates.py             #     Deterministic verification gates (DDD, SDD, Test Judge)
    │   │   ├── hitl.py              #     Human Approval node — LangGraph interrupt()
    │   │   ├── nodes/               #     Phase nodes (agent + shell)
    │   │   │   ├── classify.py      #       Issue classification node
    │   │   │   ├── domain_design.py #       Deepagent phase: task("domain-designer")
    │   │   │   ├── spec_write.py    #       Deepagent phase: task("spec-writer")
    │   │   │   ├── test_generate.py #       Deepagent phase: task("test-generator")
    │   │   │   ├── code_generate.py #       Deepagent phase: task("code-generator")
    │   │   │   ├── run_tests.py     #       Shell: execute("pytest")
    │   │   │   └── review.py        #       Deepagent phase: task("reviewer")
    │   │   ├── verification_gate.py #     DDD/SDD gate logic (~30 LOC each)
    │   │   ├── retry_controller.py  #     Fixed 3-attempt limit (~24 LOC)
    │   │   └── classification_router.py#  Immutable routing table (~30 LOC)
    │   ├── middleware/              #   Implementation glue (hidden from graph topology)
    │   │   ├── output_cleaner.py    #     LLM output format cleaning
    │   │   └── cost_tracking.py     #     Token/cost budget monitoring
    │   ├── agents/                  #   LLM sub-agents (deepagents task() targets)
    │   │   ├── domain_designer.py   #     DDD phase — entity/aggregate/bounded context
    │   │   ├── spec_writer.py       #     SDD phase — OpenAPI contracts
    │   │   ├── code_generator.py    #     TDD phase — implementation
    │   │   ├── test_generator.py    #     TDD phase — contract + unit tests
    │   │   └── reviewer.py          #     Post-phase — verify + document
    │   ├── webhooks/                #   Issue ingestion
    │   │   ├── server.py            #     FastAPI webhook receiver
    │   │   ├── github_handler.py    #     GitHub Issues/PR events
    │   │   └── codeberg_handler.py  #     Codeberg/Tea events
    │   └── security/                #   Sensitive data protection
    │       └── log_filter.py        #     Middleware: PII/secret redaction
    ├── config/
    │   └── default.yaml             #   Model, DB paths, pipeline settings, LangSmith config
    ├── skills/
    │   └── backend-dev/
    │       └── SKILL.md             #   Progressive disclosure skill for agents
    ├── specs/                       #   Per-component specifications (YAML)
    ├── docs/                        #   Architecture decisions, agent configs
    └── tests/                       #   Mirroring src/uni_dev/ structure
```

---

## Repository Purpose

| Repo | Role | Consumed By |
|------|------|-------------|
| `uni-kb` | Reusable Knowledge Base library — parses code, indexes it, serves via MCP | `uni-dev` or standalone |
| `uni-dev` | Agent automation system — orchestrates DDD→SDD→TDD via deepagents + langgraph | Developers, CI/CD |

**Dependency flow:** `uni-dev` → imports → `uni-kb` (uni-kb has zero agent dependencies)

---

## Git Conventions

### Branch Strategy

```
main                    # Stable, tagged releases (only from develop)
develop                 # Integration branch — all work merges here first
feat/<scope>/<desc>     # Feature branches (from develop)
fix/<scope>/<desc>      # Bugfix branches (from develop)
docs/<scope>/<desc>     # Documentation-only changes (from develop)
```

**Workflow:**

```
feat/* ──► PR ──► develop ──► (runnable version) ──► PR ──► main
fix/*  ──► PR ──► develop ──► (runnable version) ──► PR ──► main
```

- Branch from `develop`. Merge feature/fix branches into `develop` via PR.
- `develop` accumulates changes until a complete, runnable version is ready.
- `main` only receives merges from `develop` when the version is stable and runnable.
- Merge via PR (squash or rebase preferred). No direct commits to `develop` or `main`.
- No merge commits on `main`.

### Commit Messages

**Format:** `<type>(<repo>/<scope>): <description>`

| Type | Use for |
|------|---------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes nor adds |
| `test` | Adding or updating tests |
| `docs` | Documentation only |
| `chore` | Build, CI, deps, tooling |
| `perf` | Performance improvement |

**Scopes by repo:**

| `uni-kb/` | `uni-dev/` |
|-----------|------------|
| `parsers` — parser plugins | `core` — deterministic nodes |
| `store` — SQLite/Chroma/Graph | `agents` — LLM sub-agents |
| `generators` — spec generation | `orchestrator` — main pipeline |
| `mcp` — MCP server | `middleware` — output/cost/monitor |
| `cli` — command-line | `webhooks` — issue ingestion |
| | `security` — log filter |
| | `cli` — command-line |

**Examples:**
```
feat(uni-kb/parsers): add Java controller parser with @RequestMapping extraction
fix(uni-kb/store): resolve ChromaDB collection name collision in code_java_controller
refactor(uni-dev/core): extract VerificationGate to pure function
test(uni-dev/core): add boundary tests for retry controller at attempt=3
docs(uni-kb): document parser plugin interface in README
chore(uni-dev): pin deepagents>=0.5.3 in pyproject.toml
```

---

## Issue Management

### Workflow

1. All work **must** be tied to a GitHub/Codeberg issue
2. Create issue first, then branch, then PR
3. Close issues via commit message: `Closes #<number>`
4. Each issue maps to **exactly one** milestone

### Labels

| Label | Color | Purpose |
|-------|-------|---------|
| `bug` | `#f38ba8` | Something is broken |
| `enhancement` | `#a6e3a1` | New feature or improvement |
| `documentation` | `#89b4fa` | Docs, specs, README |
| `good first issue` | `#cba6f7` | Accessible for newcomers |
| `phase-1` … `phase-8` | `#f9e2af` | Implementation phase tag |

### Milestones

| Milestone | Repo | Phase | Content |
|-----------|------|-------|---------|
| `M1-parser-system` | uni-kb | 1 | Abstract parser + registry + Java parser |
| `M2-storage-layer` | uni-kb | 2 | SQLite (8 tables) + ChromaDB (14 idx) + NetworkX |
| `M3-nodejs-parser` | uni-kb | 3 | Node.js/Express/NestJS parser plugins |
| `M4-generators-mcp` | uni-kb | 4 | 6 generators + 20 MCP tools |
| `M5-sop-graph` | uni-dev | 5 | Business-semantic SOP StateGraph + verification gates + HITL checkpoint |
| `M6-orchestrator-v3` | uni-dev | 6 | Clean orchestrator + 5 sub-agents + phase nodes |
| `M7-middleware` | uni-dev | 7 | OutputCleaner + CostTracking + Monitor + SecurityFilter middleware |
| `M8-observability` | uni-dev | 8 | LangSmith tracing + CLI resume/approve/reject + webhook server |

---

## Development Methodology: DDD → SDD → TDD

```
Issue Received
  │
  ▼
[DDD] Domain-Driven Design
  ├─ Identify bounded contexts
  ├─ Define entities, aggregates, value objects
  ├─ Document in /docs/domain-model.md
  │
  ▼
[VERIFY] DDD Gate (deterministic, zero LLM)
  ├─ Entities defined? Relationships mapped? Bounded contexts identified?
  ├─ Pass → continue to SDD
  ├─ Fail → Human Approval checkpoint
  │
  ▼
[SDD] Specification-Driven Development
  ├─ Write spec YAML before any code
  ├─ API contracts (OpenAPI), interfaces, agent prompts
  ├─ Spec file: /specs/<component>.yaml
  │
  ▼
[VERIFY] SDD Gate (deterministic, zero LLM)
  ├─ YAML parseable? Endpoints defined? Schemas valid?
  ├─ Pass → continue to TDD
  ├─ Fail → Human Approval checkpoint
  │
  ▼
[TDD] Test-Driven Development
  ├─ Write test → Red (fail) → Green (implement) → Refactor
  ├─ Test location: tests/ mirroring src/
  ├─ Run pytest before every commit
  │
  ▼
[VERIFY] Test Result Judge (deterministic, zero LLM)
  ├─ pytest exit code == 0?
  ├─ Pass → continue to Review
  ├─ Fail → Retry Controller (max 3 attempts)
  │   ├─ attempt ≤ 3 → back to Code Generator with failure details
  │   └─ attempt > 3 → Human Approval checkpoint
  │
  ▼
[Review] Final Review
  ├─ Spec compliance check
  ├─ Security review (no hardcoded secrets)
  ├─ Documentation gaps
  │
  ▼
[END] Pipeline Complete
```

### DDD Phase (Before Code)

1. Map the domain:
   - Identify bounded contexts
   - List entities, aggregates, value objects
   - Define relationships and invariants
2. Output: `docs/domain-model.md` or `specs/domain.yaml`
3. Used by sub-agents to understand the problem space

### SDD Phase (Spec First)

1. Write the specification before touching implementation:
   - **uni-kb:** Public API signatures, MCP tool contracts, parser output schema
   - **uni-dev:** Deterministic node I/O, agent system prompts, pipeline flow
2. Output: `specs/<component>.yaml`
3. Spec is the source of truth — code must conform

### TDD Phase (Test First)

1. Write test → Expect fail → Implement → Expect pass → Refactor
2. Tests mirror source structure:
   ```
   src/uni_kb/parsers/java/controller.py  →  tests/test_java_controller.py
   src/uni_dev/core/verification_gate.py  →  tests/test_verification_gate.py
   ```
3. Run before commit: `pytest`

---

## Code Conventions

### Python Style

- Type hints on all public functions
- Docstrings on all public functions (Google style)
- Zero `print()` — use `logging` module
- Zero comments unless explaining a non-obvious decision
- Max line length: 100 characters

### Imports

```python
# Standard library
import json
from pathlib import Path
from typing import Protocol

# Third-party
from pydantic import BaseModel

# First-party
from uni_kb.parsers.base import ParserPlugin
```

### Testing

```python
def test_verification_gate_pass():
    """Given passing tests and valid contract, expect 'pass'."""
    state = {"test_results": {"pass": True, "failures": []}}
    result = verification_gate(state)
    assert result == {"status": "pass"}

def test_verification_gate_fail():
    """Given failing tests, expect 'fail' — LLM cannot override."""
    state = {"test_results": {"pass": False, "failures": ["test_auth"]}}
    result = verification_gate(state)
    assert result == {"status": "fail"}
```

---

## Review Checklist (per PR)

```
[ ] Commit follows conventional commit format
[ ] Linked to an issue (Closes #...)
[ ] Correct milestone assigned
[ ] Spec exists before implementation (SDD)
[ ] Tests exist and pass (TDD): pytest
[ ] Lint passes: ruff check src/
[ ] Type checker passes (once configured): mypy src/
[ ] For uni-dev/core/*: zero LLM calls, ~50 LOC
[ ] Domain model updated if new concepts introduced
[ ] Docs updated if public API changed
[ ] No secrets, tokens, or passwords in code
```

---

## Post-Milestone Conclusion

After each milestone completes, write a retrospective inline in `retrospectives.html` under the `M<N> Retrospective` section,
formatted with the Catppuccin Mocha theme (cards, tables, badges, <code>&lt;pre&gt;</code> blocks).
The workspace commit documents what was done, what broke, and what was learned.

### Structure (retrospectives.html HTML)

```html
<h2 id="m<N>-retro">M<N> Retrospective — <Milestone Name></h2>

<div class="card">
  <p>
    <strong>Repo:</strong> <a href="..."><repo url></a>
    &nbsp;|&nbsp; <strong>Commit:</strong> <code>&lt;hash&gt;</code>
    &nbsp;|&nbsp; <strong>Issue:</strong> <a href="...">#&lt;number&gt;</a>
  </p>
</div>

<div class="card">
  <h4>What Was Done</h4>
  <table><!-- per-component file, test count, commit hash, issue link --></table>
</div>

<div class="card">
  <h4>Problems &amp; Solutions</h4>
  <table><!-- # → problem → root cause → solution --></table>
</div>

<div class="card">
  <h4>Branch &amp; CI/CD Configuration</h4>
  <pre>...</pre>
</div>

<div class="card">
  <h4>Key Takeaways for Future Milestones</h4>
  <ul><!-- lessons learned, patterns to avoid, tooling insights --></ul>
</div>
```

### plan.html Summary Card

Each milestone gets a short summary card in `plan.html` under Implementation Phases linking to the full retrospective:

```html
<div class="card">
  <p>
    <span class="badge badge-green">M<N> Complete</span>
    All deliverables done. &rarr; <a href="retrospectives.html#m<N>-retro">Full retrospective</a>
  </p>
</div>
```

### Rules

- **One file for all milestones.** Append each retrospective to `retrospectives.html`; don't create separate files.
- **Update the phase progress bar** in `plan.html` → `Implementation Phases` (width percentage, green checkmarks).
- **Add summary card to plan.html** linking to `retrospectives.html#m<N>-retro`.
- **Commit to workspace** via `docs/retrospective/m<N>-<slug>` branch, squash-merge PR.
- Include a **Problems & Solutions table** — every bug fixed during the milestone must be documented with root cause and solution.
- Link the GitHub issue and commit hash.
- Use the existing CSS — no new styles needed.


## CI/CD Setup

Each repository (uni-kb, uni-dev) must have a GitHub Actions CI workflow before any feature work begins.

### Workflow File

Location: `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uvx ruff check src/

  test:
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12", "3.13"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
        with:
          python-version: ${{ matrix.python-version }}
      - run: uv pip install -e ".[dev]"
        timeout-minutes: 10
      - run: uv run pytest -v
```

### Rules

- **Use `uvx` for lint** — `uvx ruff check src/` runs ruff in isolation without installing project dependencies.
- **Use SSH remote** — `git remote set-url origin git@github.com:&lt;user&gt;/&lt;repo&gt;.git`. HTTPS pushes of workflow files require OAuth `workflow` scope; SSH bypasses this.
- **Python 3.11 minimum** — per `requires-python = ">=3.11"`. Add `importlib-metadata>=5.0; python_version < '3.12'` for `entry_points` backport.
- **Test timeout** — set `timeout-minutes: 10` on `uv pip install` for large dependency chains (chromadb, onnxruntime).
- **Branch protection** — after initial setup, protect `main` and `develop` to require CI pass before merge.

### GitHub Labels & Milestones

After repo creation:

```bash
# Labels (per AGENTS.md label spec)
gh label create bug --color f38ba8
gh label create enhancement --color a6e3a1
gh label create documentation --color 89b4fa
gh label create "good first issue" --color cba6f7
for i in $(seq 1 8); do gh label create "phase-$i" --color f9e2af; done

# Milestones (repo-specific)
# uni-kb: M1-M4
# uni-dev: M5-M8
gh api repos/:owner/:repo/milestones -f title="M<N>-<name>" -f description="..."
```

---

## Agent-Specific Rules (uni-dev only)

### Business-Semantic SOP Graph (`uni-dev/src/uni_dev/core/`)

> **CRITICAL:** The graph topology is a "map of business semantics," not a "stream of code execution."

**Architecture (v3):**
- `compile_pipeline()` builds a LangGraph `StateGraph` with conditional edges
- **Agent nodes** invoke deepagents via `task()` — LLM has full autonomy within each phase
- **Gate nodes** are deterministic, zero-LLM — they validate business requirements between phases
- **HITL node** uses LangGraph `interrupt()` for human approval checkpoints
- **Checkpointing** via `SqliteSaver` enables pause/resume per thread

**Gate Nodes (explicit in topology, zero LLM):**

| Gate | File | Purpose | LOC |
|------|------|---------|-----|
| DDD Gate | `gates.py` | Validates domain model: entities, relationships, bounded contexts | ~30 |
| SDD Gate | `gates.py` | Validates OpenAPI spec: YAML parseable, endpoints defined, schemas valid | ~30 |
| Test Judge | `gates.py` | Reads pytest exit code, routes pass/fail | ~25 |
| Retry Controller | `retry_controller.py` | Fixed 3-attempt limit, routes retry/escalate | ~24 |
| Classification Router | `classification_router.py` | Immutable routing table for issue types | ~30 |

**Decision Heuristic: Node vs Middleware:**
- Does its result affect the next step in the graph? → **Explicit Node**
- Does it need manual review or independent visibility in Trace? → **Explicit Node**
- Is it merely "cleaning/enhancement" of current node's output? → **Middleware**
- Is it a global guard (budget, logging) unrelated to SOP phases? → **Middleware**

### Middleware (`uni-dev/src/uni_dev/middleware/`)

Implementation glue hidden from the graph topology:

| Middleware | Hook | Purpose |
|-----------|------|---------|
| `OutputCleanerMiddleware` | `wrap_model_call` | LLM output format cleaning (markdown markers, JSON escapes) |
| `CostTrackingMiddleware` | `wrap_model_call` | Token/cost budget monitoring, per-phase tracking |
| `MonitorMiddleware` | `wrap_tool_call` | Observability: task() delegation timing, tool calls |
| `SecurityFilterMiddleware` | `wrap_tool_call` | PII/secret redaction (7 regex patterns) |

### LLM Sub-Agents (`uni-dev/src/uni_dev/agents/`)

- System prompts are specifications — draft in YAML before code
- Agents query uni-kb; they never trust memory
- Tools must be from uni-kb MCP or whitelisted filesystem operations
- Each agent returns structured output; sub-agents return `{"status": "BLOCKED", ...}` when stuck
- Phase nodes invoke sub-agents via `create_deep_agent()` with the sub-agent's system prompt

### Human-in-the-Loop

- Pipeline runs autonomously by default
- Human intervention triggered only on: verification gate fail, retry exhaustion (3 attempts), or BLOCKED with `HUMAN_REQUIRED`
- `HumanApprovalNode` uses LangGraph `interrupt()` to pause execution
- Checkpoint via `SqliteSaver` enables `uni-dev resume <thread_id>`
- CLI: `approve`, `reject`, `retry` provide human decisions to resume

---

## Security

- PII/secret redaction via `uni-dev/src/uni_dev/security/log_filter.py`
- Applies to: logs, intermediate sub-agent outputs, KB inserts
- Zero hardcoded credentials — everything via env vars or config
- Webhook endpoints validate payload signatures

---

## Getting Started

```bash
# Clone and set up uni-kb
cd uni-kb
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"

# Run tests
pytest

# Parse a project
uni-kb init --project /path/to/backend
```

```bash
# Clone and set up uni-dev
cd uni-dev
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"

# Run tests
pytest

# Run the pipeline on an issue
uni-dev run "Add user avatar upload endpoint"

# Resume a paused pipeline
uni-dev resume <thread_id>

# Human-in-the-loop actions
uni-dev approve <thread_id>
uni-dev reject <thread_id>
uni-dev retry <thread_id>
```
