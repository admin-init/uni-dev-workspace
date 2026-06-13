# M1 Retrospective — Parser System

**Repo:** [admin-init/uni-kb](https://github.com/admin-init/uni-kb) — independent Git repository under the uni-dev workspace.
**Commit:** `d57689e` | **Issue:** [#1](https://github.com/admin-init/uni-kb/issues/1)

---

## What Was Done

| Component | File(s) | Tests |
|-----------|---------|-------|
| **Abstract interface** | `parsers/base.py` — 6 dataclass models, `ParserPlugin` ABC | 15 |
| **Plugin registry** | `parsers/registry.py` — entry-point discovery, load/find/reset | 10 |
| **Java Controller** | `parsers/java/controller.py` — @RestController, auth extraction | 11 |
| **Java Service** | `parsers/java/service.py` — @Service, method body hashing | 9 |
| **Java Entity** | `parsers/java/entity.py` — @Entity, constraints, indexes | 11 |
| **Java Mapper** | `parsers/java/mapper.py` — MyBatis XML + interface annotations | 13 |
| **Package init** | `__init__.py` at `uni_kb/`, `parsers/`, `nodejs/` (stub) | — |
| **Spec (SDD)** | `specs/parser-system.yaml` | — |
| | **Total** | **79** |

---

## Problems & Solutions

| # | Problem | Root Cause | Solution |
|---|---------|------------|----------|
| 1 | Controller class-level `@RequestMapping` matched as endpoint | Regex `.*?` backtracks past `class` keyword, anchoring at first method `public` | Replaced `.*?` with tempered greedy token `(?:(?!\bclass\b|interface\b|enum\b).)*?` |
| 2 | `consumes` field leaked path string (`/{id}`) | `_extract_annotation_value()` bare-string fallback returned positional arg for named-key lookups | Added `allow_positional` flag — only enabled for `value`/`path` keys |
| 3 | Service method names were `None`, params empty | Regex capture group indices off by one (group 3 vs 4 for method name) | Corrected group(3)→group(4) for method name, group(4)→group(5) for params, group(5)→group(6) for throws |
| 4 | Entity `@Transient` field not skipped; picked up preceding field's annotations | Fixed 200-char window included previous field declarations; `@Transient` was missing from annotation regex | Replaced window with `rfind(";")` bounding to previous field end; added `Transient` to annotation alternation |
| 5 | `camel_to_snake("ID")` produced `i_d` instead of `id` | Regex `([A-Z])` inserted underscores between consecutive uppercase letters | Two-phase conversion: first handle acronyms (`([A-Z]+)([A-Z][a-z])`), then regular camelCase |
| 6 | Mapper `@Insert("INSERT INTO users (email) VALUES (#{e})")` — SQL parens broke regex | `[^)]*` stopped at first `)` inside SQL string, breaking annotation and param parsing | Split into two-phase: regex finds method declarations, then extracts preceding `@Annotation` names from bounded text window |
| 7 | Registry crashed on `nodejs` entry point with no `register()` function | `ep.load()` in `discover()` raised `AttributeError`, not caught | Wrapped `ep.load()` in try/except; added stub `nodejs/__init__.py` with empty `register()` |
| 8 | Registry tests failed because `get()` auto-loaded real parsers, mixing with test fakes | `get()`, `all_plugins()`, `supported_languages()` all called `load_all()` | Removed auto-load from query methods; only `find()` triggers discovery |
| 9 | CI failed on Python 3.11 — `ModuleNotFoundError: importlib_metadata` | `importlib_metadata` backport not declared as conditional dependency | Added `"importlib-metadata>=5.0; python_version < '3.12'"` to `pyproject.toml` |
| 10 | CI lint job installed all project deps (chromadb, onnxruntime) just to run ruff | `uv run --with ruff` discovers `pyproject.toml` and installs full project | Switched to `uvx ruff check src/` — standalone runner, zero project deps |

---

## Branch & CI/CD Configuration

```
# Branch strategy (per AGENTS.md)
main                       ← stable, fast-forward from develop
develop                    ← integration, PR target
feat/parsers/m1-*         ← feature work (merged to develop via squash PR)
fix/ci/*                   ← bugfix work

# CI/CD (.github/workflows/ci.yml)
# Triggers: push + pull_request on main, develop
lint:   uvx ruff check src/                     # <5s
test:   uv pip install -e ".[dev]" && pytest -v  # 3.11, 3.12, 3.13
```

**SSH remote:** `git@github.com:admin-init/uni-kb.git` — bypasses OAuth token scope issues for workflow file pushes.

---

## Key Takeaways for Future Milestones

- **Regex-based parsing is fragile.** Nested parens, cross-method matching, and class vs. method ambiguity caused most bugs. Consider tree-sitter AST for complex parsing tasks.
- **Test fixtures must be realistic.** SQL strings with `()`, annotations with `hasRole('ADMIN')` — these edge cases only surfaced through thoughtful test data.
- **Registry auto-load is a side effect.** Query methods should not trigger discovery — `find()` is the only operation that needs full scan.
- **Token scope matters.** HTTPS pushes of workflow files require `workflow` OAuth scope. SSH bypasses this entirely.
- **uvx for lint.** `uv run --with <tool>` installs the whole project; `uvx` runs the tool in isolation.
- **Conditional deps are real.** `importlib_metadata` is available on Python >= 3.12 as stdlib, but 3.11 needs the backport explicitly.
