```markdown
# Story LLD: 03-agent — Preserve existing POST /query and GET /health endpoints

**Ticket ID:** PER-45  
**Story ID:** 03-agent  
**Repository:** `agent`  
**Acceptance Criteria:** AC-6  
**Story Type:** Backend Verification & Testing  

---

## Acceptance Criteria

### AC-6: Existing /query path unchanged

**Given** the existing `POST /query` endpoint  
**When** it receives `{"question": "..."}` and `GET /health` is called  
**Then** `POST /query` still returns the blocking `{"answer": "..."}` or `{"error": "..."}`, `GET /health` is unchanged, and MCP is still called only from the agent process.

---

## File Plan

All paths resolve from the `agent` repository root.

### New Files

```
agent/
└── tests/
    └── test_backward_compat.py
```

### Modified Files

```
agent/
└── main.py                         (audit existing POST /query and GET /health; no code changes)
```

### Configuration Files (Audit Only)

```
agent/
└── .env or config file                 (verify MCP_URL unchanged — audit only, no modifications)
```

---

## Interfaces

### POST /query (existing — no changes)

**Request:**
```json
POST /query
Content-Type: application/json

{
  "question": "<non-empty string>"
}
```

**Response (success):**
```json
HTTP 200
Content-Type: application/json

{
  "answer": "<text>"
}
```

**Response (error):**
```json
HTTP 200
Content-Type: application/json

{
  "error": "<error message>"
}
```

**Behavior Contract:**
- Blocking: handler calls `process_question(question)` directly and awaits completion.
- Returns HTTP 200 immediately upon `process_question` completion.
- No changes to signature, latency profile, error handling, or tool names.
- MCP is called only from the main agent process (no child processes or threads).

### GET /health (existing — no changes)

**Request:**
```http
GET /health
Accept: application/json
```

**Response:**
```json
HTTP 200
Content-Type: application/json

{
  "status": "ok"
}
```

**Behavior Contract:**
- Immediate response (no blocking).
- No changes to output format or status code.
- No dependency on run store or new agent routes.

---

## Algorithm Notes

### Verification Approach

This story performs an **audit** of existing endpoints and **regression testing** to confirm Stories 1–2 did not introduce side effects or performance regressions.

#### Audit Checklist

1. **POST /query handler location:**
   - Located in `main.py` (per `change_requests.md`).
   - Verify handler calls `process_question(question)` directly (blocking).
   - Confirm no async/await wrapper that alters behavior from pre-Story-1 state.

2. **GET /health handler location:**
   - Located in `main.py` (per `change_requests.md`).
   - Verify handler returns `{"status": "ok"}`.
   - Confirm no dependency on run store, SSE streams, or new agent routes.

3. **Tool names and MCP integration:**
   - Search codebase for tool name definitions (in `agent/orchestrator.py` or config).
   - Search for `MCP_URL` configuration (environment variable or config file).
   - Confirm these are unchanged from pre-Story-1 state.

4. **ID normalization:**
   - If any query/question ID normalization exists in `process_question` or prior handlers, verify it is unchanged.
   - No new normalization logic introduced by Stories 1–2.

5. **Process isolation:**
   - Verify `process_question` is called only from the main agent process.
   - No child processes or thread pools spawned for MCP calls by Stories 1–2 background tasks.
   - Confirm Stories 1–2 use only `asyncio.create_task()` (same process, no fork/threads).

#### Regression Test Categories

**Category A: Blocking Behavior**
- POST /query with valid question blocks until `process_question` completes.
- Measure latency; confirm no regression compared to baseline.

**Category B: Response Format**
- POST /query returns `{"answer": "..."}` or `{"error": "..."}` as JSON.
- GET /health returns `{"status": "ok"}`.

**Category C: Coexistence**
- Both POST /agent/query (new, non-blocking from Stories 1–2) and POST /query (existing, blocking) work in same session.
- No global state corruption, route collision, or event queue interference.

**Category D: MCP Isolation**
- MCP is called only from main process (no child processes spawned).
- Tool names and configuration unchanged.
- No threading or multiprocessing overhead introduced.

---

## Tests

### Test File: `agent/tests/test_backward_compat.py`

**Total Test Count: 9**

#### Test 1: AC-6 — POST /query with valid question returns blocking answer

**Condition:** Valid question input  
**Verify:**
- Response status is HTTP 200.
- Response body contains `"answer": "<text>"` (exact match with mock return).
- Response elapsed time ≥ mock `process_question` latency (confirms blocking behavior).
- `process_question` was invoked and completed before response returned.

#### Test 2: AC-6 — POST /query with error condition returns error blocking

**Condition:** Question that causes `process_question` to return error  
**Verify:**
- Response status is HTTP 200.
- Response body contains `"error": "<message>"` (exact match with mock error).
- Response elapsed time ≥ mock `process_question` latency (confirms blocking behavior).
- `process_question` was invoked and error returned before response sent.

#### Test 3: AC-6 — POST /query latency unchanged (no regression from Stories 1–2)

**Condition:** POST /query called 10 times after Stories 1–2 implementation  
**Verify:**
- Measure wall-clock elapsed time for each call (from request to response receipt).
- Calculate mean latency post-implementation.
- Load baseline latency from test fixture (established pre-Stories-1–2).
- Assert `mean_post ≤ mean_baseline × 1.1` (10% tolerance for JIT, system variance).
- Individual call latencies do not exceed baseline + 20% (outlier check).

#### Test 4: AC-6 — GET /health returns ok with no side effects

**Condition:** GET /health called after Stories 1–2 implementation  
**Verify:**
- Response status is HTTP 200.
- Response body is exactly `{"status": "ok"}` (no extra fields, case-sensitive match).
- No exceptions or errors during call.
- No impact on run store, event queues, or background tasks.

#### Test 5: AC-6 — POST /query and POST /agent/query coexist without interference

**Condition:** Both endpoints active in same session; execute both in rapid succession  
**Verify:**
- POST /agent/query with valid query returns HTTP 200 immediately (< 100ms) with `agent_run_id`.
- POST /query with valid question (in same session) returns HTTP 200 with blocking latency and correct answer.
- SSE stream for returned `agent_run_id` delivers `in_progress` event as first event with correct `run_id`.
- No global state corruption (run store, event queue, or handler state).
- No route collision or mutual interference.

#### Test 6: AC-6 — MCP called only from main process (no child processes)

**Condition:** POST /query invoked after Stories 1–2 implementation  
**Verify:**
- Instrument `process_question` or MCP HTTP client to capture `os.getpid()` on each call.
- Verify all captured PIDs match the main FastAPI process PID (no spawned children).
- Confirm no child processes forked by background tasks (query via `psutil.Process().children()`).
- Confirm no threads spawned for MCP calls (no `threading.Thread()` or `ThreadPoolExecutor`).

#### Test 7: AC-6 — Tool names used in POST /query remain unchanged

**Condition:** Tool registry in current codebase  
**Verify:**
- Extract tool names from `agent/orchestrator.py` or configuration (search for tool definitions).
- Load baseline tool names from test fixture (established pre-Stories-1–2).
- Assert current set equals baseline set (exact match; no additions, removals, or renames).
- Verify tool signatures and parameters unchanged.

#### Test 8: AC-6 — ID normalization logic unchanged

**Condition:** Any ID normalization in agent (if present)  
**Verify:**
- Search codebase for ID normalization logic (e.g., slugify, lowercase, special char removal).
- If found: apply to baseline test inputs and confirm outputs match baseline.
- If not found: pass trivially (no normalization to regress).
- No new normalization introduced by Stories 1–2.

#### Test 9: AC-6 — MCP_URL configuration unchanged

**Condition:** MCP_URL read from environment or config file  
**Verify:**
- Read `MCP_URL` from `os.getenv("MCP_URL")` or config file.
- Load baseline MCP_URL from test fixture (established pre-Stories-1–2).
- Assert current value equals baseline (exact string match, case-sensitive).
- No config file modifications by Stories 1–2.

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|--------|-----------|---------------|
| AC-6 | 1, 2 | POST /query blocking behavior preserved | Returns `{"answer": "…"}` or `{"error": "…"}` with blocking latency ≥ process_question time; response received after completion |
| AC-6 | 3, 5, 9 | POST /query latency and configuration unchanged | Mean latency ≤ baseline + 10% tolerance; MCP_URL exact match to baseline; coexists with new endpoints without interference |
| AC-6 | 4 | GET /health unchanged | Returns exactly `{"status": "ok"}`; no side effects from Stories 1–2 |
| AC-6 | 6, 7, 8 | Process isolation and tool configuration unchanged | All MCP calls originate from main process PID only; no child processes; tool names and ID normalization exact match to baseline |

**All 9 tests cover AC-6 acceptance criteria from the ticket and requirements.md**

---

## Notes

- **No code changes to existing endpoints.** This story is audit and testing only. All assertions confirm pre-Stories-1–2 behavior is preserved.
- **Baseline establishment:** Tests 3, 7, 8, 9 require baseline values captured from pre-Stories-1–2 state. Document baselines in test fixtures (e.g., `conftest.py` or module-level constants). Baseline values:
  - Test 3 baseline: Mean latency of 10 POST /query calls before Stories 1–2 implementation.
  - Test 7 baseline: Tool name set extracted from pre-implementation codebase.
  - Test 8 baseline: ID normalization test cases and expected outputs from pre-implementation.
  - Test 9 baseline: MCP_URL environment variable value from pre-implementation state.
- **Coexistence test (Test 5):** Critical verification that Stories 1–2 do not corrupt global state or interfere with legacy endpoint when both are active in the same session. Executes both endpoints in rapid succession (within same app lifecycle).
- **MCP process isolation (Test 6):** Verifies no threading or multiprocessing overhead introduced by background tasks. Stories 1–2 must use only `asyncio.create_task()`, not `Thread()`, `Process()`, or `ThreadPoolExecutor`. Inspect runtime PID and process tree.
- **Latency regression (Test 3):** 10% tolerance allows for normal variance (JIT compilation, system load); only fails if Stories 1–2 add significant overhead (e.g., global lock, synchronous work in request path). Outlier check (20% threshold) catches individual slow requests.
- **Out of scope:** Auth (same as AC-6 and PER-20), rate limiting, metrics/logging beyond test assertions.
- **Routes location:** Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) are registered in `agent/main.py` only. No separate routes module. Verify by code inspection: no `agent/routes/`, `agent/routers/`, or `agent/src/` directories created.
- **Preserved modules:** `agent/orchestrator.py`, `agent/health.py`, `agent/mcp_client.py` are not modified by Stories 1–2; audit verifies this constraint by file modification check (git diff or filesystem timestamp).
- **PER-20 contract stability:** Tests 1, 2, 6, 7, 8, 9 depend on the PER-20 `process_question` contract remaining stable. Expected return type: `{"answer": "<text>"}` (on success) or `{"error": "<message>"}` (on error). Any change to this contract breaks AC-6. Recommend locking contract in PER-20 ticket or creating explicit test for return type validation.
- **Baseline fixtures example:** In `conftest.py` or test module constants:
  ```python
  BASELINE_QUERY_LATENCY_MS = 100.0  # Mean from pre-implementation runs
  BASELINE_TOOL_NAMES = {"weather", "search", "calculator"}  # Tool set
  BASELINE_MCP_URL = "http://localhost:3001"  # MCP server URL
  ```

---

## Dependencies

- **On Stories 1–2:** This story verifies they do not break existing behavior. Execute after Stories 1–2 are complete and all their tests pass.
- **On PER-20:** Existing `process_question` contract is stable (no changes required). If PER-20 modifies return type or behavior, AC-6 tests will fail; coordinate with PER-20 owner to ensure backward compatibility.
- **On baseline fixtures:** Tests 3, 7, 8, 9 require baseline values established before Stories 1–2 implementation. Document in test fixtures or test constants. Without baselines, these tests cannot run (will raise `KeyError` or `AssertionError` on missing fixture).

---

## Implementation Constraints

- All routes registered in `agent/main.py` only (per `change_requests.md`).
- No new files created in this story except `agent/tests/test_backward_compat.py`.
- No modifications to `agent/orchestrator.py`, `agent/health.py`, or `agent/mcp_client.py`.
- No modifications to `agent/main.py` route handlers; audit only (no code changes).
- Baseline fixtures must be established and documented before test execution.
- Test execution order: After Stories 1–2 complete; before epic marked done.
```
