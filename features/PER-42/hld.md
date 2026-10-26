```markdown
# HLD: Accept a FastAPI query immediately and publish the PER-20 answer on SSE

## Overview

The agent system adds a non-blocking query endpoint (`POST /agent/query`) that accepts user questions immediately, returns a unique `agent_run_id`, and publishes status updates via Server-Sent Events (SSE). The background orchestrator executes the existing [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) work (LLM + MCP integration) without blocking the HTTP response. A 5-second hold applies only to `completed` events; failures are sent immediately.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ FastAPI Application (agent/main.py)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  POST /agent/query              GET /agent/runs/{id}/events
│     │                                    │
│     ├─ Input validation                  ├─ Stream setup
│     ├─ Generate agent_run_id             ├─ in_progress (immediate)
│     ├─ Return 200 immediately            ├─ Publish from queue
│     └─ Launch background job             └─ (completed | failed)
│                                                       ▲
│  ┌─────────────────────────────────────────────────┐ │
│  │ Background Task (agent/background_task.py)  │ │
│  │                                                 │ │
│  │  • agent_run_id ← from POST                    │ │
│  │  • Execute PER-20 process_question(query)     │ │
│  │  • on answer: enforce 5-sec hold              │ │
│  │  • enqueue completed or failed event          │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │ Run Store (agent/run_store.py)              │ │
│  │                                                 │ │
│  │  agent_run_id → Run (asyncio.Queue events)     │─┘
│  │  In-memory dict; TTL-based cleanup             │
│  └─────────────────────────────────────────────────┘
│                                                             │
│  GET /health                POST /query (unchanged)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

          ▼
┌─────────────────────────────────────────────────────────────┐
│ PER-20 Agent (agent/orchestrator.py)                        │
│                                                             │
│  process_question(query: str) → {"answer": str}            │
│                              or {"error": str}             │
│  • Calls MCP server via HTTP                              │
│  • No changes to signature or behavior                     │
└─────────────────────────────────────────────────────────────┘
```

## Contracts

### 1. POST /agent/query

**Request:**
```json
POST /agent/query
Content-Type: application/json

{
  "query": "<non-empty string>"
}
```

**Success Response (HTTP 200, AC-1):**
```json
{
  "status": "ok",
  "agent_run_id": "<uuid v4 hex>"
}
```

**Rejection Response (HTTP 200, AC-2):**
```json
{
  "status": "rejected",
  "agent_run_id": "<uuid v4 hex>",
  "reason": "invalid query"
}
```

**Behavior:**
- Input validation: reject if `query` is missing, `None`, empty string, or whitespace-only.
- Return HTTP 200 immediately without waiting for `process_question` or any LLM/MCP call.
- Generate and return `agent_run_id` (UUID v4 hex) in both success and rejection cases.
- **Acceptance path (AC-1):** Create run, enqueue `in_progress` event synchronously, spawn background job via `asyncio.create_task()` without awaiting, return HTTP 200 ok immediately.
- **Rejection path (AC-2):** Create run, enqueue both `in_progress` and `failed` events synchronously with reason `"invalid query"`, do not spawn background job, return HTTP 200 rejected immediately.
- Pydantic validation errors (HTTP 422 for missing/non-string `query`) caught by custom exception handler and converted to HTTP 200 rejected response before POST handler body executes.

### 2. GET /agent/runs/{agent_run_id}/events

**Request:**
```http
GET /agent/runs/{agent_run_id}/events
Accept: text/event-stream
```

**Response Headers (HTTP 200, AC-3):**
```http
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

**Event Stream (SSE format):**

**Event 1: in_progress (AC-3)**
```
event: in_progress
data: {"run_id": "<agent_run_id>"}
```
Sent immediately when run is accepted, before any background work begins.

**Event 2a: completed (AC-4)**
```
event: completed
data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
```
Sent when `process_question` returns successful answer. Timing rule:
- If answer ready **< 5 seconds** after `in_progress` enqueued: hold until 5 seconds elapsed, then send `completed`.
- If answer ready **≥ 5 seconds** after `in_progress`: send `completed` immediately.

**Event 2b: failed (AC-5)**
```
event: failed
data: {"run_id": "<agent_run_id>", "error": "<error message>"}
```
Sent immediately (no 5-second hold) when `process_question` returns `{"error": "…"}`, LLM/MCP call fails, or query is rejected in POST handler (AC-2 rejection path). Stream ends after this event.

**Error Response (HTTP 404, AC-3):**
```json
{
  "detail": "Run not found"
}
```
Sent if `agent_run_id` does not exist or has been cleaned up by TTL expiration.

**Behavior:**
- First event is always `in_progress`.
- Exactly one of `completed` or `failed` follows.
- Stream ends after `completed` or `failed`.
- Every event contains same `run_id` (the `agent_run_id` from POST).
- Multiple concurrent subscribers allowed for same `agent_run_id`; background job not cancelled on subscriber disconnect.

### 3. POST /query (unchanged)

**Request:**
```json
POST /query
Content-Type: application/json

{
  "question": "<non-empty string>"
}
```

**Response (AC-6):**
```json
HTTP 200
{
  "answer": "<text>"
}
```
or
```json
{
  "error": "<error message>"
}
```

**Behavior:** Blocking call to `process_question(question)` directly; waits for result. No changes from PER-20.

### 4. GET /health (unchanged)

**Response (AC-6):**
```json
HTTP 200
{
  "status": "ok"
}
```

**Behavior:** No changes.

---

## Data Flow

### Acceptance Path (AC-1)

1. Receive `POST /agent/query` with valid `{"query": "…"}`.
2. Handler validates input (non-empty after strip).
3. Call `run_store.create(query.strip())`:
   - Generate `agent_run_id` (UUID v4 hex).
   - Create `Run` object with status `"pending"`, `asyncio.Queue` for events.
   - Record `in_progress_sent_at = time.time()` (timestamp for AC-4 hold calculation).
   - Enqueue `in_progress` event synchronously to `run.events`.
4. Spawn background task: `asyncio.create_task(run_background_job(agent_run_id, query))` (do not await).
5. Return HTTP 200 `{"status": "ok", "agent_run_id": "…"}` immediately.
6. Subscriber opens `GET /agent/runs/{agent_run_id}/events`; SSE stream drains `run.events` queue.
7. Background task (async) calls `process_question(query)` from PER-20.
8. On success: Apply AC-4 hold, enqueue `completed` event.
9. On error: Enqueue `failed` event immediately.
10. Stream reader delivers events and closes after `completed` or `failed`.

### Rejection Path (AC-2)

1. Receive `POST /agent/query` with `{"query": ""}`, missing field, or whitespace-only query.
2. Pydantic validation error caught by custom exception handler (HTTP 422 → 200 rejected response), OR handler-level validation (strip and empty check) triggers rejection.
3. Call `run_store.create("")`:
   - Generate `agent_run_id`.
   - Create `Run` object; record `in_progress_sent_at`.
   - Enqueue `in_progress` event synchronously.
4. Call `run_store.mark_failed(agent_run_id, "invalid query")`:
   - Enqueue `failed` event synchronously to `run.events`.
5. Do not spawn background job; do not call `process_question`.
6. Return HTTP 200 `{"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}` immediately.
7. Subscriber opens SSE stream; receives `in_progress` then `failed` events (both queued before POST returned).
8. No `completed` event sent; stream closes after `failed`.

### Five-Second Hold on Completed (AC-4)

```
Fast answer (< 5 sec):
  t=0.0:  in_progress enqueued; in_progress_sent_at = 0.0
  t=0.5:  process_question returns answer
  t=0.5:  elapsed = 0.5s; hold_remaining = 5.0 - 0.5 = 4.5s
  t=0.5:  await asyncio.sleep(4.5)
  t=5.0:  sleep completes; completed event enqueued

Slow answer (>= 5 sec):
  t=0.0:  in_progress enqueued; in_progress_sent_at = 0.0
  t=6.0:  process_question returns answer
  t=6.0:  elapsed = 6.0s; hold_remaining = 5.0 - 6.0 = -1.0s
  t=6.0:  hold_remaining <= 0: skip sleep; completed event enqueued immediately
```

---

## Integration with PER-20

**Dependency:** This feature wraps the existing [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) `process_question(query: str)` function, which:
- Takes a query string and returns `{"answer": "<text>"}` or `{"error": "<message>"}`.
- Calls MCP server via HTTP tool.
- Handles LLM synthesis and error cases.
- Runs in main agent process only.

**Changes to PER-20:** None. Signature and behavior unchanged. This feature wraps it in a non-blocking async background task via `run_background_job` in `agent/background_task.py`.

**Consumer:** [PER-28](https://linear.app/personalgs/issue/PER-28/fastapi-backend-integration-with-the-agent) consumes SSE events as published; no changes needed.

---

## Key Invariants

1. **No wait on POST:** HTTP 200 returned before `process_question` called; response sent immediately (≤ 1ms after event enqueue).
2. **One stream per run:** Single SSE stream publishes all events for a run; multiple subscribers allowed and see identical events in order.
3. **Event sequence:** `in_progress` → exactly one of (`completed` | `failed`) → stream closes.
4. **5-second hold local to background task:** Hold enforced via `asyncio.sleep()` in background task, not blocking event stream reader, other runs, or POST response.
5. **Rejection never spawns background job:** Invalid input does not call LLM or MCP. Both `in_progress` and `failed` events enqueued synchronously before POST returns; no background task.
6. **Backward compatibility:** `POST /query`, `GET /health`, tool names, MCP URL all unchanged.

---

## Implementation Notes

### File Locations (per change_requests.md)

All routes (`POST /agent/query`, `GET /agent/runs/{id}/events`, existing `POST /query`, existing `GET /health`) registered in **`agent/main.py`** only. No separate routes module or `agent/src/` tree.

### Run Store Data Structure

Each run keyed by `agent_run_id` (UUID v4 hex string):

```python
@dataclass
class Run:
    id: str                         # agent_run_id
    query: str                      # user query (stripped)
    created_at: float               # Unix timestamp
    status: str                     # "pending" | "completed" | "failed"
    events: asyncio.Queue           # Enqueued events: [in_progress, (completed|failed)]
    result: dict | None             # {"answer": "…"} or {"error": "…"}
    started_at: float | None        # When background task started
    ended_at: float | None          # When background task ended
    in_progress_sent_at: float      # Timestamp of in_progress enqueue (for AC-4 hold calc)
```

### Pydantic Models

```python
# Request
class AgentQueryRequest(BaseModel):
    query: str = Field(..., min_length=1)

# Responses
class AgentQueryAccepted(BaseModel):
    status: Literal["ok"]
    agent_run_id: str

class AgentQueryRejected(BaseModel):
    status: Literal["rejected"]
    agent_run_id: str
    reason: str

# SSE event data
class InProgressEventData(BaseModel):
    run_id: str

class CompletedEventData(BaseModel):
    run_id: str
    response: dict  # {"answer": str}

class FailedEventData(BaseModel):
    run_id: str
    error: str
```

### Input Validation Handling

FastAPI Pydantic validation rejects missing/non-string `query` at HTTP 422. Custom `@app.exception_handler(RequestValidationError)` catches these and converts to HTTP 200 `{"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}` before POST handler body executes. This ensures AC-1/AC-2 HTTP 200 contract.

### Background Task Spawn

POST handler calls `asyncio.create_task(run_background_job(agent_run_id, query))` without awaiting. Task runs concurrently with no blocking on response.

### RunStore Interface

```python
class RunStore:
    async def create(query: str) -> Run
        # Create run, record in_progress_sent_at, enqueue in_progress event
    
    async def get(run_id: str) -> Run | None
        # Retrieve run by agent_run_id
    
    async def enqueue_event(run_id: str, event_type: str, data: dict) -> None
        # Append event to run.events (asyncio.Queue)
    
    async def mark_failed(run_id: str, reason: str) -> None
        # Enqueue failed event (for AC-2 rejection path)
    
    async def cleanup_expired(ttl_seconds: int = 3600) -> None
        # Delete runs older than ttl_seconds (called periodically)
```

### Background Task Function Signature

```python
async def run_background_job(agent_run_id: str, query: str) -> None
    # 1. Retrieve run via run_store.get()
    # 2. Call process_question(query) from PER-20
    # 3. On success: apply AC-4 hold, enqueue completed event
    # 4. On error: enqueue failed event immediately
    # 5. On exception: enqueue failed event with error message
```

### SSE Handler

```python
async def stream_events(run_id: str) -> AsyncGenerator[str, None]
    # 1. Retrieve run; raise HTTPException(404) if not found
    # 2. Drain run.events queue, yield SSE-formatted lines
    # 3. Close stream after completed or failed event
```

### Cleanup Task

```python
async def cleanup_task()
    # Runs indefinitely in background (started via @app.on_event("startup"))
    # Sleep RUN_CLEANUP_INTERVAL_SECONDS (default 60)
    # Call run_store.cleanup_expired(RUN_CLEANUP_TTL_SECONDS) (default 3600)
```

---

## Implementation Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| **Event loss if process crashes before SSE connects** | Medium | Queue events in `asyncio.Queue` before subscriber opens stream; replay on stream open; TTL-based cleanup prevents stale runs |
| **Unbounded run accumulation** | High | TTL-based cleanup; default 1 hour; configurable via `RUN_CLEANUP_TTL_SECONDS` env var; cleanup task runs every 60s |
| **5-second hold blocks other work** | Low | Hold enforced per-run via `asyncio.sleep()` in background task; non-blocking; does not block event queue or other jobs |
| **Multiple subscribers desync** | Low | All subscribers read from same queue in order; identical timing and content guaranteed |
| **Unknown run_id returns 404 immediately** | Low | 404 indicates run never created or TTL expired; keep runs ≥ 1 hour; clarify in API docs |
| **PER-20 contract changes** | High | Depends on stable `process_question(query: str) → {"answer": "…"} \| {"error": "…"}` return type; lock contract in AC-6 tests |
| **Validation error not converted to 200** | Medium | Custom exception handler must be registered before routes; verify registration order in app startup |

---

## Story Breakdown

This epic is decomposed into three stories, executed in strict order (each blocks the next):

**Story 1 (PER-43: 01-agent) — Accept FastAPI query without blocking; return agent_run_id immediately**
- Acceptance Criteria: AC-1, AC-2
- Test Count: 8
- Deliverables: POST /agent/query endpoint, run store initialization, input validation with 422→200 handler, background job spawn via `asyncio.create_task()`, synchronous event enqueue for rejection path
- Files: `agent/models.py`, `agent/run_store.py`, `agent/background_task.py` (spawn placeholder), `agent/main.py` (POST route, exception handler), `agent/tests/test_agent_query_accept.py`
- Blocks: Story 2 (requires `run_store.create()`, `run_store.get()`, `run_store.enqueue_event()`, `run_store.mark_failed()` implemented)

**Story 2 (PER-44: 02-agent) — Stream PER-20 answer via SSE with in_progress, completed, or failed events**
- Acceptance Criteria: AC-3, AC-4, AC-5
- Test Count: 15
- Deliverables: GET /agent/runs/{agent_run_id}/events endpoint, SSE stream handler, full background orchestrator with 5-second hold, cleanup task with TTL-based expiration
- Files: `agent/sse_handler.py`, `agent/background_task.py` (full impl), `agent/run_store.py` (extend with cleanup), `agent/main.py` (add GET route and cleanup task), `agent/tests/test_agent_sse_stream.py`
- Blocks: Story 3 (requires full background execution and SSE delivery complete)

**Story 3 (PER-45: 03-agent) — Preserve existing POST /query and GET /health endpoints**
- Acceptance Criteria: AC-6
- Test Count: 9
- Deliverables: Audit and regression tests; no code changes (verification only)
- Files: `agent/tests/test_backward_compat.py`
- No blockers (final validation step)

**Total: 3 stories, 32 tests, all 6 ACs covered**

---

## Acceptance Criteria Coverage

| AC | Story | Test Count | Coverage |
|----|-------|------------|----------|
| AC-1: Immediate accept | 01-agent | 2 | Valid query returns ok immediately (< 100ms); response before process_question called |
| AC-2: Reject without a run | 01-agent | 6 | Empty/missing/whitespace query returns rejected; no LLM/MCP call; failed event enqueued synchronously |
| AC-3: One event stream | 02-agent | 7 | HTTP 200 text/event-stream; first event in_progress; all events same run_id; 404 on unknown; multi-subscriber support |
| AC-4: Five-second hold | 02-agent | 4 | Fast answer (< 5s) held until ≥ 5s; slow answer (≥ 5s) sent immediately; stream closes after completed |
| AC-5: Failure immediate | 02-agent | 4 | LLM/MCP/invalid query errors send failed immediately; no 5-sec hold; stream closes after failed |
| AC-6: Existing paths unchanged | 03-agent | 9 | POST /query blocking behavior and latency unchanged; GET /health unchanged; coexistence; MCP isolation; tool names and config unchanged |
| **Total** | **3 stories** | **32 tests** | **All 6 ACs covered, all 6 test categories from requirements.md Verify** |

---

## Testing Summary

All 32 tests documented in respective story LLDs:
- `stories/01-agent.md` — 8 tests for AC-1, AC-2 (immediate accept, rejection, no background job spawn on invalid input)
- `stories/02-agent.md` — 15 tests for AC-3, AC-4, AC-5 (SSE stream, 5-sec hold timing, failure handling, multi-subscriber, event replay)
- `stories/03-agent.md` — 9 tests for AC-6 (blocking behavior, latency regression, health check, coexistence, MCP isolation, tool/config unchanged)

Test categories from requirements.md Verify aligned:
✓ Immediate return (Story 1, Tests 1–2)
✓ Rejection (Story 1, Tests 3–8)
✓ Fast and slow completed cases (Story 2, Tests 6–7)
✓ Tool/LLM failure (Story 2, Tests 9–10)
✓ Unknown run id (Story 2, Test 4)
✓ Existing POST /query answer (Story 3, Tests 1–3, 9)

---

## Decisions & Deferred Items

### Adopted

- **Event queue:** In-memory `asyncio.Queue` per run (PoC; upgrade to Redis if multi-process required)
- **5-second hold:** Enforced via `asyncio.sleep()` in background task (non-blocking per-run)
- **Run cleanup:** TTL-based; default 1 hour; configurable via `RUN_CLEANUP_TTL_SECONDS` env var
- **Run ID format:** UUID v4 hex (32-char, collision-free)
- **Input validation:** O(1) string check (strip, empty test); synchronous before task spawn
- **Exception handling:** Custom `@app.exception_handler(RequestValidationError)` converts HTTP 422 to HTTP 200 rejected response before handler body executes
- **Routes location:** All routes in `agent/main.py` only (per change_requests.md)

### Deferred

- **Auth/authorization:** Out of scope per requirements; same as PER-20
- **Rate limiting:** Assume upstream FastAPI middleware
- **Metrics/logging:** No instrumentation beyond startup logs
- **Run history API:** No `GET /agent/runs` metadata endpoint; query-only via SSE
- **Persistent event store:** In-memory queue is PoC; upgrade to Redis if > 100 concurrent runs

---

## Risks & Open Questions

| Item | Answer |
|------|--------|
| **Max queue depth?** | Unconstrained; upgrade to Redis if > 100 concurrent runs |
| **PER-20 changes?** | Must preserve `{"answer": "…"} \| {"error": "…"}` return type |
| **Run history?** | Deferred; SSE query-only |
| **Subscriber disconnect?** | Background job continues; new subscriber retrieves all queued events |
| **Hold duration configurable?** | Yes; expose `AGENT_COMPLETE_HOLD_SECONDS` env var (default 5) |
| **Cleanup interval configurable?** | Yes; expose `RUN_CLEANUP_INTERVAL_SECONDS` env var (default 60) |
| **Multi-process support?** | In-memory queue is PoC; design requires Redis upgrade for multi-process |
| **Exception handler registration order?** | Critical; must be registered before routes to intercept validation errors |
| **TTL cleanup boundary condition?** | Strict inequality: delete if `(now - created_at) > ttl_seconds`; run at exactly ttl boundary not deleted |
| **Concurrent subscriber queue ownership?** | All subscribers read from same `run.events` queue; queue ownership is per-run (shared), not per-subscriber |
```
