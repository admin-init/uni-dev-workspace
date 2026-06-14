# Fix 2: `_noop` Gap — Option C Plan

> Replace 5 placeholder nodes in the pipeline graph with real LangGraph nodes
> that independently invoke LLM sub-agents using the same system prompts and
> model configurations already defined.

## Problem

The pipeline graph (`src/uni_dev/core/graph.py`) has 5 LLM-phase nodes implemented
as `_noop` placeholders that return `{}`:

```python
builder.add_node("DomainDesigner", _noop)   # Returns {}
builder.add_node("SpecWriter", _noop)        # Returns {}
builder.add_node("TestGenerator", _noop)     # Returns {}
builder.add_node("CodeGenerator", _noop)     # Returns {}
builder.add_node("Reviewer", _noop)          # Returns {}
```

### Root Cause

The architecture uses two separate LangGraph runtimes that cannot communicate:

- **Runtime A** (Orchestrator DeepAgent): has `task()` tool + SubAgentMiddleware
- **Runtime B** (Pipeline Graph CompiledSubAgent): plain LangGraph sub-invocation
  with no access to Runtime A's tools

When `task(subagent_type="pipeline-controller", ...)` invokes Runtime B, the graph
runs but its LLM-phase nodes produce nothing. The bridge between the pipeline graph
nodes and the LLM sub-agents was never built because deepagents doesn't expose a
"call parent's sub-agent from inside a sub-agent" API.

### Impact

- The pipeline graph cannot enforce DDD→SDD→TDD ordering with LLM agents
- The deterministic nodes (verification gate, retry controller) are unreachable
  through the graph path
- The orchestrator relies entirely on the LLM following its system prompt (Path A),
  with no programmatic enforcement

## Solution

### Approach: Self-contained agent nodes

Each `_noop` node is replaced with a real LangGraph node function that:
1. Creates its own `ChatOpenAI` model instance (with per-node model config)
2. Creates a LangChain agent with the sub-agent's system prompt and tools
3. Invokes the agent with context from the state
4. Returns structured output to the next node via state fields

### New UniDevState fields

```python
class UniDevState(TypedDict):
    # Existing fields (unchanged)
    messages: list[BaseMessage]
    classification: str
    attempt_count: int
    test_results: dict
    migration_idx: int
    migration_plan: list[str]
    current_phase: str
    kb_path: str

    # New phase-output fields (populated by graph nodes)
    domain_model: dict[str, Any]    # DDD output
    api_spec: str                   # OpenAPI YAML string
    test_files: list[str]           # Generated test file paths
    modified_files: list[str]      # Files modified by code generator
    review_report: dict[str, Any]  # Reviewer output
```

### Node implementations

#### `domain_designer_node(state) → dict`

```
1. Read issue from state["messages"]
2. Create ChatOpenAI(deepseek-v4-pro, reasoning_effort=high, thinking=enabled)
3. Invoke with DOMAIN_DESIGNER_SYSTEM_PROMPT + issue text
4. Parse LLM response → extract domain_model dict
5. Return {"domain_model": {...}, "current_phase": "sdd"}
```

Tools: `search_code`, `get_class_structure`, `get_dependency_graph` (MCP tools)

#### `spec_writer_node(state) → dict`

```
1. Read domain_model from state
2. Create ChatOpenAI(deepseek-v4-flash)
3. Invoke with SPEC_WRITER_SYSTEM_PROMPT + domain_model
4. Parse LLM response → extract OpenAPI YAML
5. Write spec to specs/ directory
6. Return {"api_spec": "<yaml>", "current_phase": "tdd"}
```

Tools: `get_api_contract`, `get_entity_spec`, filesystem

#### `test_generator_node(state) → dict`

```
1. Read api_spec from state
2. Create ChatOpenAI(deepseek-v4-flash)
3. Invoke with TEST_GENERATOR_SYSTEM_PROMPT + api_spec
4. Write test files to tests/ directory
5. Return {"test_files": ["tests/test_x.py", ...], "current_phase": "tdd"}
```

Tools: `get_api_contract`, `verify_contract`, shell, filesystem

#### `code_generator_node(state) → dict`

```
1. Read api_spec + test_files from state
2. Create ChatOpenAI(deepseek-v4-flash)
3. Invoke with CODE_GENERATOR_SYSTEM_PROMPT
4. Read existing source files, write implementation
5. Run tests via shell tool
6. If tests pass: set test_results.pass = True
7. Return {"modified_files": [...], "test_results": {...}}
```

Tools: `get_business_logic_doc`, `get_api_contract`, `get_entity_spec`,
       `get_db_schema`, shell, filesystem

#### `reviewer_node(state) → dict`

```
1. Read api_spec + modified_files + test_results from state
2. Create ChatOpenAI(deepseek-v4-pro, thinking=enabled)
3. Invoke with REVIEWER_SYSTEM_PROMPT
4. Compare implementation against spec
5. Return {"review_report": {approved: bool, issues: [...], ...}}
```

Tools: all MCP tools, `compare_api_responses`, `verify_contract`,
       `get_migration_checklist`, filesystem

### Files to change

| File | Change |
|------|--------|
| `core/graph.py` | Replace 5 `_noop` node registrations with real node functions |
| `core/graph.py` | Add 5 node function implementations |
| `core/graph.py` | Update `UniDevState` TypedDict with new optional fields |
| `core/graph.py` | Use `os.environ["DEEPSEEK_API_KEY"]` for API key (same as orchestrator) |
| `tests/test_graph.py` | Update tests — verify nodes are no longer noops |
| `tests/test_graph.py` | Add tests for node state transitions |

### What stays unchanged

- All 4 deterministic nodes: `verification_gate`, `retry_controller`,
  `classification_router`, `migration_stepper`
- `orchestrator.py` — the `CompiledSubAgent` wrapping continues to work
- Agent system prompts in `agents/*.py` — reused as constants
- Model config per agent type (v4-pro for domain_designer/reviewer,
  v4-flash for spec_writer/code_generator/test_generator)

### What this does NOT do

- Does NOT use deepagents `task()` / SubAgentMiddleware for graph nodes
- Does NOT connect back to the orchestrator's tools
- Each node is a self-contained LangChain agent invocation
- No shared LLM model instance between nodes

### Testing approach

Since these nodes call live LLMs, tests will:
1. Mock `ChatOpenAI` to return controlled responses
2. Verify each node produces correct state updates
3. Test full graph invocation with mocked LLM

### Relationship to M6 agents

The `agents/*.py` files remain as-is. They define `SubAgent` dicts with
system prompts used when the orchestrator calls `task()` (Path A).
The graph nodes use the same system prompts directly, creating their
own agent instances.

---

## Implementation Steps

1. Add new `UniDevState` fields in `core/graph.py`
2. Write `_domain_designer_node` function
3. Write `_spec_writer_node` function
4. Write `_test_generator_node` function
5. Write `_code_generator_node` function
6. Write `_reviewer_node` function
7. Replace `_noop` registrations in `build_graph()`
8. Update `tests/test_graph.py`
9. Run tests, lint, commit
