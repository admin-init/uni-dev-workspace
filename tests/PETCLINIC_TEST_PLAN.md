# Test Plan: Spring PetClinic REST Integration

> End-to-end integration test for uni-dev against the Spring PetClinic REST API codebase.

## Test Environment

| Item | Value |
|------|-------|
| Test project | [spring-petclinic/spring-petclinic-rest](https://github.com/spring-petclinic/spring-petclinic-rest) |
| Test project path | `uni-dev/petclinic-rest/` (cloned into repo, gitignored) |
| Platform | Python 3.11+, DeepSeek API |
| uni-dev branch | `feat/runner/m10-continuous` |

## Pre-Flight Checks

Run before each test session:

```bash
# Verify branch
cd uni-dev && git branch --show-current
# Expected: feat/runner/m10-continuous

# Verify install
.venv/bin/uni-dev --version
# Expected: uni-dev, version 0.1.0

# Verify API key
echo $DEEPSEEK_API_KEY | head -c 5
# Should show: "sk-..."
```

---

## Test 1: KB Initialization

### Purpose
Verify `uni-dev init` can parse a real Spring Boot codebase and populate the knowledge base.

### Steps

```bash
# Clean slate — remove any previous KB
rm -rf /tmp/petclinic/.uni-kb

# Clone PetClinic if not present
git clone https://github.com/spring-petclinic/spring-petclinic-rest.git /tmp/petclinic

# Initialize KB
.venv/bin/uni-dev init /tmp/petclinic
```

### Expected Output

```
Initializing uni-dev for /tmp/petclinic...
[uni-kb output with file count, classes, methods, endpoints, entities]
Done.
```

### Verification

```bash
# KB directory exists
ls /tmp/petclinic/.uni-kb/
# Expected: store.db, chroma/, graph.gml

# SQLite store has data
.venv/bin/python -c "
import sqlite3
conn = sqlite3.connect('/tmp/petclinic/.uni-kb/store.db')
tables = conn.execute(\"SELECT name FROM sqlite_master WHERE type='table'\").fetchall()
print('Tables:', [t[0] for t in tables])
for t in ['classes', 'methods', 'api_endpoints']:
    count = conn.execute(f'SELECT COUNT(*) FROM {t}').fetchone()[0]
    print(f'  {t}: {count} rows')
"
# Expected: 8 tables, classes>0, methods>0, api_endpoints>0
```

### Pass/Fail Criteria

- [ ] Command exits with code 0
- [ ] `.uni-kb/` directory exists with `store.db`, `chroma/`, `graph.gml`
- [ ] `classes` table has rows (should find OwnerController, PetController, etc.)
- [ ] `api_endpoints` table has rows (should find GET /api/owners, POST /api/pets, etc.)
- [ ] `entities` table has rows (should find Owner, Pet, Visit, Vet, etc.)

---

## Test 2: One-Shot Pipeline (Simple Feature)

### Purpose
Verify the full DDD→SDD→TDD pipeline processes a well-scoped feature request against a real codebase.

### Pre-condition
Test 1 must pass (KB initialized).

### Test Issue

```
Add a 'description' field to the PetType entity:

- Add a 'description' String field to PetType with @Column annotation (max 255 chars)
- Update PetTypeController to accept and return description in DTOs
- No service layer changes needed — field is just metadata
- Add contract test verifying GET /api/pettypes returns description field
```

Why this issue:
- Minimal scope: touches 1 entity, 1 controller, no service logic
- File changes: PetType.java, PetTypeController.java (2 files)
- Tests: 1 contract test
- Low API token cost — should complete in ~5 turns

### Steps

```bash
cd uni-dev
export DEEPSEEK_API_KEY=sk-your-key

.venv/bin/uni-dev run "Add a 'description' field to the PetType entity:
- Add a 'description' String field to PetType with @Column annotation (max 255 chars)
- Update PetTypeController to accept and return description in DTOs
- No service layer changes needed — field is just metadata
- Add contract test verifying GET /api/pettypes returns description field"
```

### Expected Flow (per agent turn)

| Turn | Agent | What it should do | Key output to watch for |
|------|-------|-------------------|------------------------|
| 1 | **domain-designer** (v4-pro) | Analyze the PetType entity and PetTypeController, identify that only 2 files need changes | `bounded_contexts: [Pet Management]`, entity analysis |
| 2 | **spec-writer** (v4-flash) | Write/update OpenAPI for GET /api/pettypes to include description | OpenAPI YAML with `description: {type: string, maxLength: 255}` |
| 3 | **test-generator** (v4-flash) | Write contract test verifying the API response schema | Test file with assertion checking for `description` field |
| 4 | **code-generator** (v4-flash) | Add `private String description` to PetType.java, add `@Column(length=255)`, update PetTypeController DTO mapping | Modified PetType.java and PetTypeController.java |
| 5 | **reviewer** (v4-pro) | Review the changes, verify against spec | `approved: true/false`, review report |

### Monitor Verification

```bash
# Check what happened (run after pipeline completes)
.venv/bin/uni-dev status --db /tmp/petclinic/.uni-kb/monitor.db
```

Expected: 4-5 task runs with status=success, showing domain-designer, spec-writer, test-generator, code-generator, reviewer.

### Checking Generated Code

```bash
# See what files were modified
git -C /tmp/petclinic diff --stat
# Or check specific files
cat /tmp/petclinic/src/main/java/org/springframework/samples/petclinic/model/PetType.java | grep -A2 description
```

### Pass/Fail Criteria

- [ ] Orchestrator runs without crashing (no API key error, no import error)
- [ ] At least 4 `task()` calls are recorded in monitor
- [ ] All recorded task runs have status=success
- [ ] PetType.java has a `description` field added
- [ ] PetTypeController.java handles the description field in request/response

### Failure Modes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `DEEPSEEK_API_KEY is not set` | Env var not exported | `export DEEPSEEK_API_KEY=sk-...` |
| Orchestrator stops after 1-2 turns | LLM didn't follow system prompt | Try a more specific issue description |
| `code-generator` doesn't write files | deepagents filesystem tools error | Check permissions on /tmp/petclinic |
| Pipeline completes but no files changed | LLM described changes but didn't apply | Check code-generator's system prompt instructs file writes |

---

## Test 3: Watch Mode (TUI + Runner)

### Purpose
Verify the continuous runner and TUI dashboard work correctly.

### Pre-condition
DEEPSEEK_API_KEY set. No other uni-dev processes running.

### Steps

#### 3a: Start watch mode

```bash
cd uni-dev
.venv/bin/uni-dev watch
```

Expected: Textual TUI opens with:
- Header: "uni-dev — Pipeline Runner"
- Left panel: Issue Queue (empty, shows column headers: ID, St, Source, Title)
- Right panel: "Select an issue to see details"
- Bottom: "Runner started"
- Footer: n=New Issue, r=Refresh, q=Quit
- Status bar: P:0 R:0 H:0 C:0 F:0

#### 3b: Create issue via chat modal

1. Press `n` — chat modal opens
2. Type: *"Add a health check endpoint to PetClinic that returns service status and an uptime counter"*
3. Press Enter
4. LLM responds with `[TITLE]: ...` and `[BODY]: ...`
5. Optional: iterate (type more, press Enter)
6. Click **Submit** or press Tab to focus Submit, then Enter

Expected:
- Modal closes
- Log shows: "Issue abc123: created via chat"
- Issue appears in queue with: abc123, ⏳ pending, chat, title

#### 3c: Watch runner process the issue

The runner polls every 5 seconds. Watch:
- Issue status changes: ⏳ pending → ▶ running
- Log shows: "Processing issue abc123: ..."
- After completion: status → ✓ completed
- Log shows: "Issue abc123: completed (N task runs)"

#### 3d: Human-in-the-loop (manual trigger)

```bash
# Insert an issue marked as needs_human
.venv/bin/python -c "
from uni_dev.store.issue_store import IssueStore
s = IssueStore('.uni-kb/issues.db')
iid = s.insert_issue({
    'title': 'Test human approval',
    'body': 'This issue needs human review',
    'source': 'cli',
})
s.update_status(iid, 'needs_human')
print(f'Created {iid}')
"
```

Then in the TUI:
1. Select the needs_human issue in the queue
2. Verify action buttons appear: [Approve] [Reject] [Retry]
3. Click **Approve** → status changes to ✓ completed
4. Log shows: "Issue xxx: approved"

#### 3e: Keyboard shortcuts

| Key | Action | Expected |
|-----|--------|----------|
| `n` | New Issue chat | Opens modal |
| `r` | Refresh | Queue refreshes |
| `q` | Quit | Clean exit |

### Pass/Fail Criteria

- [ ] TUI renders all 3 panels correctly
- [ ] Chat modal opens with `n`, LLM responds
- [ ] Issue created via chat appears in queue
- [ ] Issue transitions through pending→running→completed
- [ ] Log panel shows processing messages
- [ ] Action buttons appear for needs_human issues
- [ ] Approve/Reject/Retry buttons work
- [ ] `q` exits cleanly

---

## Test 4: Webhook → Queue Integration

### Purpose
Verify webhook events are inserted into the IssueStore where the runner picks them up.

### Pre-condition
Watch mode running in one terminal (or run headless: `uni-dev watch --no-tui`).

### Steps

```bash
# In another terminal
curl -X POST http://localhost:8080/webhook/github \
  -H "Content-Type: application/json" \
  -d '{
    "action": "opened",
    "issue": {
      "title": "Add rate limiting to all API endpoints",
      "body": "Implement rate limiting with configurable limits per endpoint. Default: 100 requests/minute per IP. Return 429 when exceeded.",
      "number": 101,
      "html_url": "http://github.com/owner/repo/issues/101"
    },
    "sender": {"login": "ops-team"},
    "repository": {"full_name": "spring-petclinic/spring-petclinic-rest"}
  }'
```

### Expected Response

```json
{
  "status": "accepted",
  "event_type": "issue",
  "issue": {
    "title": "Add rate limiting to all API endpoints",
    "body": "Implement rate limiting...",
    "number": 101,
    "action": "opened",
    "sender": "ops-team",
    "repo": "spring-petclinic/spring-petclinic-rest"
  },
  "issue_id": "abc123def456"
}
```

### Verification

```bash
# Check issue was queued
.venv/bin/python -c "
from uni_dev.store.issue_store import IssueStore
s = IssueStore('.uni-kb/issues.db')
issues = s.list_issues(source='github')
for i in issues:
    print(f\"  {i['issue_id']}: {i['title']} [{i['status']}]\")
"
```

Expected: Issue appears with source=github, status=pending or running.

### Pass/Fail Criteria

- [ ] HTTP 200 response from webhook endpoint
- [ ] Response contains `issue_id` field
- [ ] Issue appears in IssueStore with source=github
- [ ] Runner picks up the issue (status changes from pending if watch is running)

---

## Failure Diagnosis

### If `uni-dev init` fails

```bash
# Check if uni-kb is installed
.venv/bin/python -c "import uni_kb; print(uni_kb.__version__)"

# Run uni-kb directly
.venv/bin/python -m uni_kb.cli init --project /tmp/petclinic
```

### If `uni-dev run` fails

```bash
# Check API key
.venv/bin/python -c "
import os
key = os.environ.get('DEEPSEEK_API_KEY','')
print('Key found:', bool(key), 'Starts:', key[:5] if key else 'N/A')
"

# Test DeepSeek connectivity
.venv/bin/python -c "
from langchain_openai import ChatOpenAI
m = ChatOpenAI(model='deepseek-v4-flash', base_url='https://api.deepseek.com', api_key='$DEEPSEEK_API_KEY')
r = m.invoke('Say hello in one word')
print(r.content)
"
```

### If `uni-dev watch` fails

```bash
# Check textual installation
.venv/bin/python -c "import textual; print(textual.__version__)"
# Expected: 8.2.7 or similar

# Check if port 8080 is in use
ss -tlnp | grep 8080
```
