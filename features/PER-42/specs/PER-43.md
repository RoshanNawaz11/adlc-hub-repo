```markdown
# Tech Stub: PER-43 — Accept a FastAPI query immediately and publish the PER-20 answer on SSE

**Ticket ID:** PER-43  
**Title:** Accept a FastAPI query immediately and publish the PER-20 answer on SSE  
**Repository:** agent  
**Dependencies:** [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server), [PER-28](https://linear.app/personalgs/issue/PER-28/fastapi-backend-integration-with-the-agent)  

---

## Stack (from repo-context)

- **Language:** Python 3.11+
- **Framework:** FastAPI 0.104+
- **Async Runtime:** asyncio (built-in)
- **Event Queue:** `asyncio.Queue` (in-memory, single-process)
- **HTTP Protocol:** Server-Sent Events (SSE) over HTTP/1.1
- **HTTP Client:** `httpx` (inherited from PER-20)
- **Dependency:** PER-20 `process_question(query: str) → dict` (answer or error, unchanged)

**No new external dependencies.** FastAPI's `StreamingResponse` handles SSE; `uuid.uuid4()` generates run IDs.

---

## Schema & API Contracts

### 1. POST /agent/query

**Request:**
```json
{
  "query": "<non-empty string>"
}
```

**Success Response (HTTP 200):**
```json
{
  "status": "ok",
  "agent_run_id": "<uuid v4 hex>"
}
```

**Rejection Response (HTTP 200):**
```json
{
  "status": "rejected",
  "agent_run_id": "<uuid v4 hex>",
  "reason": "invalid query"
}
```

**Behavior:**
- Input validation is synchronous (O(1) string check).
- Response is HTTP 200 **before** `process_question` is called.
- On valid input: spawn background job asynchronously; enqueue `in_progress` event.
- On invalid input (empty, missing, or whitespace-only `query`): do NOT spawn background job; enqueue `in_progress` and `failed` events before returning.
- Return `agent_run_id` in both cases (same id used in SSE stream).
- Pydantic validation errors (HTTP 422 for missing or non-string `query`) are caught by a custom exception handler and converted to HTTP 200 `{"status": "rejected", "agent_run_id": "<id>", "reason": "invalid query"}` before the handler body executes.

---

### 2. GET /agent/runs/{agent_run_id}/events

**Request:**
```http
GET /agent/runs/{agent_run_id}/events
Accept: text/event-stream
```

**Success Response (HTTP 200):**
```
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

event: in_progress
data: {"run_id": "<agent_run_id>"}

event: completed
data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
```

**Error Response (HTTP 404):**
```json
{
  "detail": "Run not found"
}
```

**Event Types:**

- **in_progress:** Sent immediately after run creation, before background work begins.
  ```json
  {"run_id": "<agent_run_id>"}
  ```

- **completed:** Sent when `process_question` returns an answer. Timing per AC-4:
  - If answer ready < 5s after `in_progress`: hold until 5s elapsed, then send.
  - If answer ready ≥ 5s after `in_progress`: send immediately.
  ```json
  {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
  ```

- **failed:** Sent immediately (no hold) on error or LLM/MCP failure.
  ```json
  {"run_id": "<agent_run_id>", "error": "<message>"}
  ```

**Behavior:**
- First event is always `in_progress`.
- Exactly one of `completed` or `failed` follows.
- Stream closes after final event.
- Every event contains the same `run_id`.
- Multiple concurrent subscribers for the same `agent_run_id` are allowed; background job is not cancelled on subscriber disconnect.
- If run is unknown or has been cleaned up (TTL expired): return HTTP 404.

---

### 3. POST /query (unchanged)

**Request:**
```json
{
  "question": "<non-empty string>"
}
```

**Response (success):**
```json
{
  "answer": "<text>"
}
```

**Response (error):**
```json
{
  "error": "<error message>"
}
```

**Behavior:** Blocking call to `process_question(question)`. No changes from PER-20.

---

### 4. GET /health (unchanged)

**Response:**
```json
{
  "status": "ok"
}
```

**Behavior:** Simple health check. No changes.

---

## File Touch List

All paths resolve from the `agent` repository root.

### New Files

```
agent/
├── __init__.py
├── models.py                # Pydantic request/response models
├── run_store.py             # In-memory run registry + event queue management
├── sse_handler.py           # SSE event generator
├── background_task.py       # Background orchestrator; calls process_question
├── main.py                  # register POST /agent/query and GET /agent/runs/{id}/events
├── orchestrator.py          # existing PER-20 process_question
├── health.py                # existing PER-20 health
├── mcp_client.py            # existing PER-20 MCP client
└── tests/
    ├── test_agent_query_accept.py   # Tests for AC-1, AC-2 (8 tests)
    ├── test_agent_sse_stream.py     # Tests for AC-3, AC-4, AC-5 (15 tests)
    └── test_backward_compat.py      # Tests for AC-6 (9 tests)
```

### Modified Files

```
agent/
├── main.py            # Add POST /agent/query and GET /agent/runs/{id}/events handlers; add cleanup task startup
├── models.py          # Add SSE event data models
├── run_store.py       # Extend with cleanup method and event queue management
└── requirements.txt   # Verify FastAPI, uvicorn, pydantic, httpx present
```

---

## Pydantic Models

### Request/Response Models (`agent/models.py`)

```python
from pydantic import BaseModel, Field
from typing import Literal

class AgentQueryRequest(BaseModel):
    """User query submitted to the agent."""
    query: str = Field(..., min_length=1, description="Non-empty user query string")

class AgentQueryAccepted(BaseModel):
    """Successful acceptance of a query."""
    status: Literal["ok"]
    agent_run_id: str = Field(..., description="Unique run identifier for SSE stream")

class AgentQueryRejected(BaseModel):
    """Rejection due to invalid input."""
    status: Literal["rejected"]
    agent_run_id: str = Field(..., description="Unique run identifier for error stream")
    reason: str = Field(..., description="Reason for rejection (e.g., 'invalid query')")

class InProgressEventData(BaseModel):
    """Data for in_progress SSE event."""
    run_id: str

class CompletedEventData(BaseModel):
    """Data for completed SSE event."""
    run_id: str
    response: dict  # {"answer": "<text>"}

class FailedEventData(BaseModel):
    """Data for failed SSE event."""
    run_id: str
    error: str
```

---

## Run Store Data Structure

### Run Object (`agent/run_store.py`)

```python
from dataclasses import dataclass
from typing import Optional
from asyncio import Queue

@dataclass
class Run:
    """Single agent run."""
    id: str                              # agent_run_id (UUID v4 as hex string)
    query: str
    created_at: float                    # Unix timestamp
    status: str                           # "pending", "completed", "failed"
    events: Queue                        # asyncio.Queue for event messages
    result: Optional[dict] = None        # {"answer": "…"} or {"error": "…"}
    started_at: Optional[float] = None
    ended_at: Optional[float] = None
    in_progress_sent_at: Optional[float] = None
```

### RunStore Interface (`agent/run_store.py`)

```python
class RunStore:
    """In-memory registry of active runs."""
    
    async def create(query: str) -> Run:
        """
        Generate unique agent_run_id, create Run, enqueue in_progress event, return Run.
        
        Returns: Run object with id, events queue pre-populated with in_progress event.
        Timestamp in_progress_sent_at is recorded for 5-second hold calculation.
        """
    
    async def get(run_id: str) -> Optional[Run]:
        """
        Retrieve Run by agent_run_id.
        
        Returns: Run object if found and not expired, None otherwise.
        """
    
    async def enqueue_event(run_id: str, event_type: str, data: dict) -> None:
        """
        Append event to run's queue for SSE delivery.
        
        Args:
            run_id: agent_run_id
            event_type: "in_progress", "completed", or "failed"
            data: JSON-serializable dict with run_id and result/error
        """
    
    async def mark_failed(run_id: str, reason: str) -> None:
        """
        Mark rejected run as failed; enqueue failed event.
        
        Called on AC-2 rejection path (invalid input).
        Enqueues: in_progress (already queued by create()), failed event with reason.
        """
    
    async def cleanup_expired(ttl_seconds: int = 3600) -> None:
        """
        Delete runs older than ttl_seconds (Unix timestamp).
        
        Called periodically by background cleanup task on app startup.
        Default TTL: 1 hour.
        Configurable via RUN_CLEANUP_TTL_SECONDS env var.
        """
```

---

## Acceptance Criteria Mapping

| AC | File(s) | Methods/Functions | Tests (count) |
|----|---------|---|---|
| **AC-1: Immediate accept** | `agent/main.py`, `agent/models.py`, `agent/run_store.py` | `POST /agent/query` handler, `run_store.create()`, `asyncio.create_task()` | test_agent_query_accept.py (2 tests) |
| **AC-2: Reject without a run** | `agent/main.py`, `agent/run_store.py` | Input validation, `run_store.mark_failed()` | test_agent_query_accept.py (6 tests) |
| **AC-3: One event stream** | `agent/sse_handler.py`, `agent/main.py` | `GET /agent/runs/{id}/events`, `stream_events()` | test_agent_sse_stream.py (7 tests) |
| **AC-4: Five-second hold** | `agent/background_task.py`, `agent/run_store.py` | `run_background_job()`, 5-second hold calculation | test_agent_sse_stream.py (4 tests) |
| **AC-5: Failure is immediate** | `agent/background_task.py` | Exception handling in background task, `enqueue_event("failed", ...)` | test_agent_sse_stream.py (4 tests) |
| **AC-6: Existing paths unchanged** | `agent/main.py` (audit only) | `POST /query`, `GET /health` | test_backward_compat.py (9 tests) |

**Total Test Count: 32** (8 + 15 + 9)

---

## Implementation Algorithms

### AC-1: Input Validation & Acceptance Path

```
1. Receive POST /agent/query with AgentQueryRequest
2. FastAPI auto-validates: query must be string, non-None
3. Custom exception handler converts HTTP 422 validation errors to HTTP 200 rejected response
4. Custom validation in handler: strip(query); if empty, mark as invalid
5. If valid:
     a. run_store.create(query.strip()) → Run object with:
        - id = UUID v4 hex
        - status = "pending"
        - events = asyncio.Queue with [in_progress event]
        - in_progress_sent_at = time.time()
     b. asyncio.create_task(run_background_job(run.id, query.strip()))
        (do NOT await; return immediately)
     c. Return HTTP 200 AgentQueryAccepted(status="ok", agent_run_id=run.id)
6. Response sent BEFORE process_question is called
```

### AC-2: Rejection Path (Invalid Input)

```
1. Input validation fails (empty, missing, or whitespace-only query)
   OR Pydantic validation error caught by custom exception handler (HTTP 422 → 200)
2. run_store.create("") → Run object with same structure as AC-1
3. run_store.mark_failed(run.id, "invalid query")
     - Updates run.status = "failed"
     - Enqueues failed event with error: "invalid query"
4. Do NOT call run_background_job()
5. Return HTTP 200 AgentQueryRejected(status="rejected", agent_run_id=run.id, reason="invalid query")
6. When SSE stream opens, it reads queued events:
     - in_progress (enqueued by create())
     - failed (enqueued by mark_failed())
```

### AC-3: SSE Stream Delivery

```
1. GET /agent/runs/{agent_run_id}/events
2. run_store.get(agent_run_id)
   - If None: raise HTTPException(404, "Run not found")
   - Else: proceed with stream
3. Return StreamingResponse(stream_events(agent_run_id), media_type="text/event-stream", headers={...})
4. stream_events() generator:
   - Reads events from run.events (asyncio.Queue)
   - For each event:
     a. Extract event_type ("in_progress", "completed", "failed")
     b. Extract data (dict with run_id and result/error)
     c. Serialize data to JSON
     d. Yield SSE format: "event: {type}\ndata: {json}\n\n"
   - If event_type in ("completed", "failed"): break (close stream)
5. Multiple subscribers read from same queue; all get same events in order
6. Late subscribers drain queue after background job completes (event replay)
```

### AC-4: Five-Second Hold on Completed

```
Timeline for fast answer (< 5 sec):
  t=0.0:  in_progress event enqueued by run_store.create()
  t=0.0:  in_progress_sent_at = time.time()
  t=0.1:  process_question returns answer
  t=0.1:  elapsed = 0.1 - 0.0 = 0.1s
  t=0.1:  hold_remaining = 5.0 - 0.1 = 4.9s
  t=0.1:  await asyncio.sleep(4.9) starts
  t=5.0:  sleep completes
  t=5.0:  completed event enqueued

Timeline for slow answer (>= 5 sec):
  t=0.0:  in_progress event enqueued
  t=0.0:  in_progress_sent_at = time.time()
  t=6.0:  process_question returns answer
  t=6.0:  elapsed = 6.0 - 0.0 = 6.0s
  t=6.0:  hold_remaining = 5.0 - 6.0 = -1.0s
  t=6.0:  hold_remaining <= 0: skip sleep
  t=6.0:  completed event enqueued immediately

Edge case (exactly 5 sec):
  t=5.0:  process_question returns answer
  t=5.0:  elapsed = 5.0s
  t=5.0:  hold_remaining = 0.0s
  t=5.0:  hold_remaining <= 0: skip sleep
  t=5.0:  completed event enqueued immediately
```

### AC-5: Failure Immediate (No Hold)

```
When process_question returns {"error": "…"} or raises exception:
  1. No calculation of hold_remaining
  2. No asyncio.sleep()
  3. Failed event enqueued immediately
  4. Elapsed time from in_progress to failed: < 1 second typically
  5. No completed event sent

Examples:
  - LLM timeout: process_question returns {"error": "LLM timeout"}
  - MCP connection error: process_question raises exception
  - Invalid query (AC-2): mark_failed() called in POST handler
  
All cases enqueue failed event synchronously, without 5-second hold.
```

### AC-6: Backward Compatibility (Unchanged Behavior)

```
POST /query (existing, blocking):
  1. Receive {"question": "..."}
  2. Call process_question(question) directly (synchronous or await)
  3. Wait for result
  4. Return {"answer": "…"} or {"error": "…"}
  5. No changes to latency, signature, or error handling

GET /health (existing, unchanged):
  1. Return {"status": "ok"}

MCP isolation (unchanged):
  1. MCP calls originate only from main agent process
  2. No child processes or threads forked
  3. Tool names, id normalization, MCP_URL unchanged
```

---

## Key Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| **Event loss if process crashes before SSE connects** | Medium | Events queued in `Run.events` before subscriber connects; replay on stream open; TTL-based cleanup prevents stale runs. |
| **Unbounded run accumulation** | Medium | Implement TTL-based cleanup; delete runs 1 hour after creation. Cleanup task runs every 60 seconds. Configurable via `RUN_CLEANUP_TTL_SECONDS` env var. |
| **5-second hold blocks other work** | Low | Hold enforced per-run via `asyncio.sleep()` in background task; does not block event queue or other jobs. Non-blocking. |
| **Multiple subscribers cause desync** | Low | All subscribers read from same queue in order; identical timing and event content guaranteed. |
| **Unknown run_id returns 404 immediately** | Low | 404 means run never created or TTL expired. Keep runs for ≥ 1 hour. API docs clarify semantics. |
| **PER-20 contract changes** | High | Design depends on `process_question(query: str) → dict` signature. Return type must be `{"answer": "…"} \| {"error": "…"}`. Document as hard dependency. Audit on any PER-20 changes. |
| **Validation at 422 instead of 200** | Medium | Pydantic will reject missing/non-string `query` at 422. Custom exception handler converts 422 to 200 + rejected response before handler body executes. |

---

## Test Coverage Summary

### Story 1: AC-1, AC-2 (agent/tests/test_agent_query_accept.py)

| Test # | Condition | Pass Criteria |
|--------|-----------|---------------|
| 1 | Valid query → ok response | HTTP 200, `status: "ok"`, `agent_run_id` non-empty, returned < 100ms |
| 2 | Immediate return | Response < 100ms; background job state "pending" immediately after |
| 3 | Empty query → rejected | HTTP 200, `status: "rejected"`, `reason: "invalid query"`, `agent_run_id` |
| 4 | Missing query field → rejected | HTTP 200, `status: "rejected"`, `reason: "invalid query"` |
| 5 | Whitespace-only query → rejected | HTTP 200, `status: "rejected"` |
| 6 | Rejection does not call LLM/MCP | `process_question` call count = 0; background job not spawned |
| 7 | Both paths return agent_run_id | Acceptance and rejection both have non-empty `agent_run_id`; different on separate calls |
| 8 | Run store contains created run with events | Post response `agent_run_id` matches run.id; rejection path enqueues `in_progress` and `failed` events synchronously |

**Total: 8 tests**

### Story 2: AC-3, AC-4, AC-5 (agent/tests/test_agent_sse_stream.py)

| Test # | Condition | Pass Criteria |
|--------|-----------|---------------|
| 1 | Known run_id → 200 text/event-stream | HTTP 200, `Content-Type: text/event-stream`, cache/connection headers |
| 2 | First event in_progress | Event type "in_progress", data `{"run_id": "<agent_run_id>"}` |
| 3 | All events same run_id | Every event data contains same `run_id` |
| 4 | Unknown run_id → 404 | HTTP 404, `{"detail": "Run not found"}` |
| 5 | Multiple subscribers | Both subscribers receive identical events in same order |
| 6 | Fast answer (< 5s): hold until 5s | Elapsed time (in_progress to completed) ≥ 4.9s |
| 7 | Slow answer (≥ 5s): no extra hold | Elapsed time (in_progress to completed) ≈ 6.0s (no extra hold) |
| 8 | Edge case at 5s exact | Elapsed time ≈ 5.0s, no extra hold |
| 9 | LLM error → failed immediately | in_progress, then failed; elapsed < 1s; no completed |
| 10 | MCP error → failed immediately | in_progress, then failed; elapsed < 1s; no completed |
| 11 | Invalid query rejection → failed | in_progress, then failed; error: "invalid query" |
| 12 | Stream closes after completed | Last event is completed; exactly 2 events (in_progress, completed) |
| 13 | Stream closes after failed | Last event is failed; exactly 2 events (in_progress, failed) |
| 14 | Subscriber disconnect ≠ cancel job | Background job continues after subscriber disconnects; new subscriber gets events |
| 15 | Late subscriber gets all events | Events queued before subscriber connects; no replay loss |

**Total: 15 tests**

### Story 3: AC-6 (agent/tests/test_backward_compat.py)

| Test # | Condition | Pass Criteria |
|--------|-----------|---------------|
| 1 | POST /query valid question → answer | HTTP 200, `{"answer": "…"}`, blocking (latency ≥ process_question time) |
| 2 | POST /query error → error response | HTTP 200, `{"error": "…"}`, blocking |
| 3 | POST /query latency unchanged | Mean latency post ≤ mean baseline * 1.1 (10% tolerance) |
| 4 | GET /health → ok | HTTP 200, `{"status": "ok"}` |
| 5 | Both endpoints coexist | Agent POST (immediate), Query POST (blocking), SSE stream all work together; no corruption |
| 6 | MCP from main process only | All MCP HTTP calls from main process PID; no child processes |
| 7 | Tool names unchanged | Tool names match pre-Stories baseline; no additions/removals |
| 8 | ID normalization unchanged | Normalization behavior (if any) matches baseline |
| 9 | MCP_URL unchanged | Configuration value matches baseline |

**Total: 9 tests**

---

## Configuration & Environment

### Environment Variables

```bash
# SSE/Run store configuration
RUN_CLEANUP_TTL_SECONDS=3600          # TTL for runs before cleanup (default: 1 hour)
RUN_CLEANUP_INTERVAL_SECONDS=60       # Cleanup task interval (default: 60 seconds)
AGENT_COMPLETE_HOLD_SECONDS=5         # 5-second hold on completed events (default: 5)
```

### Startup Behavior

All routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) are registered in `agent/main.py` with cleanup task startup:

```python
# In agent/main.py:

# 1. Register routes (no separate routes module per change_requests.md)
@app.post("/agent/query", response_model=AgentQueryAccepted | AgentQueryRejected)
async def post_agent_query(req: AgentQueryRequest) -> ...: ...

@app.get("/agent/runs/{agent_run_id}/events")
async def get_agent_events(agent_run_id: str) -> StreamingResponse: ...

@app.post("/query")
async def post_query(question: str) -> dict: ...

@app.get("/health")
async def get_health() -> dict: ...

# 2. Start cleanup task on app startup
async def cleanup_task():
    while True:
        await asyncio.sleep(int(os.getenv("RUN_CLEANUP_INTERVAL_SECONDS", 60)))
        await run_store.cleanup_expired(ttl_seconds=int(os.getenv("RUN_CLEANUP_TTL_SECONDS", 3600)))

@app.on_event("startup")
async def startup():
    asyncio.create_task(cleanup_task())
```

---

## Deferred / Out of Scope

- **Auth/authorization:** No authentication on new endpoints; same as PER-20.
- **Rate limiting:** Assume upstream FastAPI middleware; not implemented in this ticket.
- **Metrics/logging:** No instrumentation of run duration, latency, or error rates; basic startup logging only.
- **Run history API:** No `GET /agent/runs` or `GET /agent/runs/{id}` metadata endpoint; only event stream.
- **Persistent event store:** In-memory queue (PoC); Redis upgrade if multi-process deployment required.
- **Per-user subscription limits:** No cap on concurrent SSE subscribers; add if resource contention surfaces.

---

## Integration Points

### PER-20 Contract (Fixed)

```python
# Existing, unchanged:
async def process_question(query: str) -> dict:
    """
    Orchestrate LLM + MCP integration for a query.
    
    Returns:
      {"answer": "<text>"}  # On success
      {"error": "<message>"}  # On error
    
    Raises:
      Any exception indicates failure (wrapped in failed event).
    """
```

### PER-28 Consumer (No Changes)

PER-28 consumes SSE events as published by this ticket's `GET /agent/runs/{agent_run_id}/events` endpoint. No changes to PER-28; it forwards events as sent.

---

## Acceptance Criteria Verification

All 6 acceptance criteria are fully covered by the 32 tests across 3 stories:

✓ **AC-1:** Immediate accept (Tests 1–2 in Story 1)  
✓ **AC-2:** Reject without a run (Tests 3–8 in Story 1)  
✓ **AC-3:** One event stream (Tests 1–5, 14–15 in Story 2)  
✓ **AC-4:** Five-second hold (Tests 6–8, 12 in Story 2)  
✓ **AC-5:** Failure is immediate (Tests 9–11, 13 in Story 2)  
✓ **AC-6:** Existing paths unchanged (Tests 1–9 in Story 3)  

**Total: 32 tests, all ACs covered, all edge cases included.**
```
