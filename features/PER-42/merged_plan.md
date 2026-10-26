# Merged plan — PER-42

Execution order (3 ticket(s)):

1. `PER-43`
2. `PER-44`
3. `PER-45`

---

## Ticket `PER-43`

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

## Ticket `PER-44`

# Tech Stub: PER-44 — Stream PER-20 answer via SSE with in_progress, completed, or failed events

**Ticket ID:** PER-44  
**Title:** Stream PER-20 answer via SSE with in_progress, completed, or failed events  
**Repository:** agent  
**Acceptance Criteria:** AC-3, AC-4, AC-5  

---

## Stack (from repo-context)

- **Language:** Python 3.11+
- **Framework:** FastAPI 0.104+
- **Async Runtime:** asyncio (built-in)
- **Event Queue:** `asyncio.Queue` (in-memory, single-process)
- **Protocol:** Server-Sent Events (SSE) over HTTP/1.1
- **Dependency:** PER-20 `process_question(query: str) → dict` (answer or error)

**No new external dependencies.** FastAPI's `StreamingResponse` handles SSE; `uuid.uuid4()` generates run IDs.

---

## Schema & API Contracts

### GET /agent/runs/{agent_run_id}/events

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

**Failure Response (HTTP 404):**
```http
HTTP 404
Content-Type: application/json

{"detail": "Run not found"}
```

**Event Types:**

- **in_progress:** Sent immediately when run is accepted, before background work begins.
  ```json
  {"run_id": "<agent_run_id>"}
  ```

- **completed:** Sent when `process_question` returns an answer. Timing enforced per AC-4:
  - If answer ready < 5s after `in_progress`: hold until 5s elapsed, then send.
  - If answer ready ≥ 5s after `in_progress`: send immediately.
  ```json
  {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
  ```

- **failed:** Sent immediately (no hold) when `process_question` returns error or LLM/MCP fails.
  ```json
  {"run_id": "<agent_run_id>", "error": "<message>"}
  ```

**Behavior:**
- First event is always `in_progress`.
- Exactly one of `completed` or `failed` follows.
- Stream closes after final event.
- Every event contains the same `run_id`.
- Multiple concurrent subscribers for the same `agent_run_id` are allowed; background job is not cancelled on subscriber disconnect.

---

## Acceptance Criteria

### AC-3: One event stream per run

**Given** an accepted run  
**When** the caller sends `GET /agent/runs/{agent_run_id}/events` with `Accept: text/event-stream`  
**Then** the first event is `in_progress` with `data: {"run_id": "<agent_run_id>"}`, every subsequent event includes the same `run_id`, and an unknown id returns HTTP 404.

### AC-4: Five-second hold on completed only

**Given** `process_question` returns an answer  
**When** the background job completes  
**Then** if the answer is ready less than 5 seconds after the `in_progress` event, the stream stays on `in_progress` until 5 seconds after that event, then sends `completed` with `data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}` and ends; if the answer is ready at 5 seconds or later, `completed` is sent as soon as the answer is ready and the stream ends. The hold does not delay the POST reply.

### AC-5: Failure is immediate

**Given** `process_question` returns `{"error": "..."}` or the LLM or MCP call fails  
**When** the error is detected  
**Then** the stream sends `failed` with `data: {"run_id": "<agent_run_id>", "error": "<that message>"}` immediately, without waiting for the 5-second hold and without sending `completed`.

---

## File Touch List

All paths resolve from the `agent` repository root.

### New Files

```
agent/
├── sse_handler.py
├── background_task.py
└── tests/
    └── test_agent_sse_stream.py
```

### Modified Files

```
agent/
├── models.py (add SSE event data models)
├── run_store.py (extend with cleanup method and per-run event queue management)
└── main.py (add GET /agent/runs/{agent_run_id}/events route; add cleanup task startup)
```

---

## Models & Interfaces

### Pydantic Event Data Models (`agent/models.py` — extend)

```python
from typing import Literal
from pydantic import BaseModel

class InProgressEventData(BaseModel):
    """Data for in_progress event."""
    run_id: str

class CompletedEventData(BaseModel):
    """Data for completed event."""
    run_id: str
    response: dict  # {"answer": "<text>"}

class FailedEventData(BaseModel):
    """Data for failed event."""
    run_id: str
    error: str
```

### Run Store Interface — Extensions (`agent/run_store.py`)

```python
from dataclasses import dataclass
from typing import Optional
import asyncio
import time

@dataclass
class Run:
    """Single agent run."""
    id: str                              # agent_run_id (UUID v4 as hex string)
    query: str
    created_at: float                    # Unix timestamp
    status: str                           # "pending", "completed", "failed"
    events: asyncio.Queue                # asyncio.Queue for event messages
    result: Optional[dict] = None        # {"answer": "…"} or {"error": "…"}
    started_at: Optional[float] = None
    ended_at: Optional[float] = None
    in_progress_sent_at: Optional[float] = None

class RunStore:
    """In-memory registry of active runs (from PER-43, extended here)."""
    
    async def enqueue_event(self, run_id: str, event_type: str, data: dict) -> None:
        """Append event to run's event queue for SSE delivery.
        
        Args:
            run_id: agent_run_id
            event_type: "in_progress", "completed", or "failed"
            data: JSON-serializable dict with run_id and result/error
        """
        run = self._runs.get(run_id)
        if run:
            await run.events.put({"event": event_type, "data": data})
    
    async def cleanup_expired(self, ttl_seconds: int = 3600) -> None:
        """Delete runs older than ttl_seconds (Unix timestamp).
        
        Called periodically by cleanup task on app startup.
        Default TTL: 1 hour.
        """
        now = time.time()
        expired_ids = [
            run_id for run_id, run in self._runs.items()
            if (now - run.created_at) > ttl_seconds
        ]
        for run_id in expired_ids:
            del self._runs[run_id]
```

### SSE Handler Interface (`agent/sse_handler.py`)

```python
from typing import AsyncGenerator
import json
from fastapi import HTTPException
from agent.run_store import run_store

async def stream_events(run_id: str) -> AsyncGenerator[str, None]:
    """
    Stream events for a given run in SSE format.
    
    Yields lines in SSE format:
      event: <type>
      data: <json>
      
    First event is in_progress.
    Second event is exactly one of (completed | failed).
    Stream closes after second event.
    
    Raises HTTPException(404) if run_id not found.
    """
    run = await run_store.get(run_id)
    if not run:
        raise HTTPException(status_code=404, detail="Run not found")
    
    async for event_payload in run.events:
        event_type = event_payload["event"]
        data_json = json.dumps(event_payload["data"])
        yield f"event: {event_type}\n"
        yield f"data: {data_json}\n\n"
        
        if event_type in ("completed", "failed"):
            break  # Close stream after final event
```

### Background Task Interface (`agent/background_task.py`)

```python
import asyncio
import time
from agent.run_store import run_store
from agent.orchestrator import process_question

async def run_background_job(agent_run_id: str, query: str) -> None:
    """
    Execute process_question in background; enqueue events to run store.
    
    Calls:
      1. run_store.get(agent_run_id)
      2. process_question(query) (from PER-20)
      3. Enforce 5-second hold for completed only
      4. run_store.enqueue_event() for result
    
    On error or LLM/MCP failure:
      - Enqueue failed event immediately (no hold)
      - Do not enqueue completed event
    """
    run = await run_store.get(agent_run_id)
    if not run:
        return  # Run already cleaned up; abort silently
    
    try:
        run.started_at = time.time()
        
        # Call existing PER-20 orchestrator
        result = await process_question(query)  # {"answer": "…"} or {"error": "…"}
        
        if "error" in result:
            # Failure path (AC-5): send immediately
            await run_store.enqueue_event(agent_run_id, "failed", {
                "run_id": agent_run_id,
                "error": result["error"]
            })
            run.status = "failed"
        else:
            # Success path (AC-4): apply 5-second hold
            elapsed = time.time() - run.in_progress_sent_at
            hold_remaining = 5.0 - elapsed
            
            if hold_remaining > 0:
                await asyncio.sleep(hold_remaining)
            
            await run_store.enqueue_event(agent_run_id, "completed", {
                "run_id": agent_run_id,
                "response": {"answer": result["answer"]}
            })
            run.status = "completed"
        
        run.result = result
        run.ended_at = time.time()
    
    except Exception as e:
        # Unhandled exception (AC-5): send failed immediately
        await run_store.enqueue_event(agent_run_id, "failed", {
            "run_id": agent_run_id,
            "error": str(e)
        })
        run.status = "failed"
        run.ended_at = time.time()
```

### Route Handler — Extension (`agent/main.py`)

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from agent.sse_handler import stream_events
from agent.run_store import run_store
import asyncio
import os

app = FastAPI()

@app.get("/agent/runs/{agent_run_id}/events")
async def get_agent_events(agent_run_id: str) -> StreamingResponse:
    """
    Stream SSE events for a given run.
    
    - AC-3: Returns 200 text/event-stream if run exists; 404 if unknown.
    - AC-4, AC-5: Emits in_progress, then (completed | failed), with timing rules enforced.
    """
    run = await run_store.get(agent_run_id)
    if not run:
        raise HTTPException(status_code=404, detail="Run not found")
    
    return StreamingResponse(
        stream_events(agent_run_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )

# Cleanup task (TTL-based run expiration)
async def cleanup_task():
    """Periodically delete expired runs."""
    ttl_seconds = int(os.getenv("RUN_CLEANUP_TTL_SECONDS", 3600))
    interval_seconds = int(os.getenv("RUN_CLEANUP_INTERVAL_SECONDS", 60))
    while True:
        await asyncio.sleep(interval_seconds)
        await run_store.cleanup_expired(ttl_seconds=ttl_seconds)

@app.on_event("startup")
async def startup():
    """Start cleanup task on app startup."""
    asyncio.create_task(cleanup_task())
```

---

## Implementation Notes

### 1. Event Queue per Run

- Each `Run` object has an `asyncio.Queue` for events.
- Background task enqueues `in_progress` event (via `run_store.create()` in PER-43) before POST returns.
- Background task enqueues `completed` or `failed` event after orchestration completes.
- SSE handler drains queue and yields events to subscriber(s).

### 2. 5-Second Hold Calculation (AC-4)

```
Timeline for fast answer (< 5 sec):
  t=0.0:  in_progress event enqueued to run.events; in_progress_sent_at = 0.0
  t=0.5:  process_question returns answer
  t=0.5:  elapsed = 0.5 - 0.0 = 0.5s
  t=0.5:  hold_remaining = 5.0 - 0.5 = 4.5s
  t=0.5:  await asyncio.sleep(4.5) starts
  t=5.0:  sleep completes; completed event enqueued
  
Timeline for slow answer (>= 5 sec):
  t=0.0:  in_progress event enqueued; in_progress_sent_at = 0.0
  t=6.0:  process_question returns answer
  t=6.0:  elapsed = 6.0 - 0.0 = 6.0s
  t=6.0:  hold_remaining = 5.0 - 6.0 = -1.0s (negative, so skip sleep)
  t=6.0:  completed event enqueued immediately
```

### 3. Failure Timing (AC-5)

- No calculation of hold_remaining.
- No `asyncio.sleep()`.
- `failed` event enqueued immediately to `run.events`.
- SSE stream reader picks up `failed` event and closes stream.

### 4. Cleanup Task (Run Expiration)

- Runs every 60 seconds (configurable via `RUN_CLEANUP_INTERVAL_SECONDS`).
- Deletes runs older than 1 hour (configurable via `RUN_CLEANUP_TTL_SECONDS`).
- Old runs become unretrievable; 404 on subsequent `GET /agent/runs/{id}/events`.

---

## Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| **Event loss if process crashes before SSE subscribes** | Medium | Events queued in `Run.events` before subscriber connects; replay on stream open. TTL-based cleanup ensures old runs are deleted. |
| **Unbounded run accumulation** | Medium | Implement async cleanup task; delete runs 1 hour after completion/failure. Log cleanup policy in startup. |
| **5-second hold blocks other runs** | Low | Hold enforced per-run via `asyncio.sleep()` in background task; does not block event queue or other jobs. |
| **Multiple subscribers miss events queued before they connect** | Low | Events are queued synchronously in `Run.events`; SSE reader drains the queue on first connect. Acceptable for PoC; upgrade to persistent store if multi-process. |
| **Unknown run_id returns 404 immediately; caller unsure if job running** | Low | Clarify in API docs: 404 means run never created or TTL expired. Keep runs for at least 1 hour. |
| **PER-20 contract changes** | High | Design depends on current `process_question` return type. Any future change must preserve `{"answer": "…"} \| {"error": "…"}`. Document as hard dependency. |

---

## Acceptance Criteria Verification

| AC | File(s) | Implementation | Test Coverage |
|----|---------|----------------|----------------|
| **AC-3: One event stream** | `sse_handler.py`, `run_store.py`, `main.py` (GET route) | `stream_events()` generator yields SSE-formatted events; 404 on unknown run_id; multiple subscribers supported via shared queue | 7 tests (HTTP 200, text/event-stream, first event in_progress, all events same run_id, 404 on unknown, multi-subscriber, late subscriber replay) |
| **AC-4: 5-second hold on completed** | `background_task.py`, `run_store.py` | Hold calculated as `max(0, 5.0 - elapsed)` where `elapsed = now - in_progress_sent_at`; fast answers held via `asyncio.sleep()`, slow answers sent immediately | 4 tests (fast answer held to 5s, slow answer immediate, edge case at exactly 5s, stream closes after completed) |
| **AC-5: Failure immediate** | `background_task.py`, `sse_handler.py` | `failed` event enqueued immediately on error detection (no hold logic); stream closes after failed | 4 tests (LLM error immediate, MCP error immediate, invalid query rejection, stream closes after failed) |

**Total Test Count:** 15

---

## Verify

**Test Count:** 15 tests across AC-3, AC-4, AC-5 (see `stories/02-agent.md`).

**Story 2 (PER-44: 02-agent):** 15 tests verify SSE stream delivery with correct run_id, 5-second hold timing for fast/slow answers, immediate failure events, and stream closure.

Tests are listed in `stories/02-agent.md`.

---

## Dependencies & Constraints

- **On PER-20:** Depends on stable `process_question(query: str) → dict` signature returning `{"answer": "…"}` or `{"error": "…"}`.
- **On PER-43:** Depends on `run_store.create()`, `run_store.get()`, and background task spawn via `asyncio.create_task()` in `agent/main.py`.
- **On existing FastAPI app:** Registers GET route in same app instance as POST /query (AC-6); no interference with existing endpoints.

---

## Configuration & Environment

- **RUN_CLEANUP_TTL_SECONDS:** Default 3600 (1 hour); configurable via env var.
- **RUN_CLEANUP_INTERVAL_SECONDS:** Default 60 seconds; configurable via env var.
- **Event queue depth:** Unconstrained in-memory `asyncio.Queue`; upgrade to Redis if > 100 concurrent runs or multi-process deployment.

---

## Notes

- **Event Format:** SSE spec compliance: `event: <type>\ndata: <json>\n\n` (two newlines after data).
- **Process Isolation:** MCP calls remain in main agent process; no child processes forked by background task or SSE stream.
- **Exception Handling:** Any unhandled exception in background task caught and converted to `failed` event with exception message as string.
- **Backward Compatibility:** This ticket does not modify POST /query or GET /health; see PER-45 for verification and audit.
- **Timestamp Precision:** `in_progress_sent_at` recorded by PER-43; used for hold calculation. Millisecond precision sufficient for 5-second hold.
- **Routes Location:** Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query and GET /health) are registered in `agent/main.py` only. No separate routes module.

---

## Open Questions

1. **Run ID format:** UUID v4 (recommended) or sequential integer? → Recommend UUID v4; deterministic, collision-free, opaque.
2. **Event queue backing:** In-memory Python `asyncio.Queue` or Redis? → In-memory for PoC; upgrade if > 100 concurrent runs or multi-process deployment required.
3. **5-second hold configurable?** → Yes; default to 5 seconds, expose as env var `AGENT_COMPLETE_HOLD_SECONDS` in config or `.env`.
4. **Cleanup interval:** How often to delete expired runs? → Every 60 seconds (tunable); configurable via `RUN_CLEANUP_INTERVAL_SECONDS`.
5. **Subscription limit per run:** Should we cap concurrent SSE subscribers? → No limit for PoC; add if resource contention surfaces.

## Ticket `PER-45`

```markdown
# Tech Stub: Ticket PER-45 — Accept a FastAPI query immediately and publish the PER-20 answer on SSE

**Ticket ID:** PER-45  
**Title:** Accept a FastAPI query immediately and publish the PER-20 answer on SSE  
**Repository:** `agent`  
**Acceptance Criteria:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6  

---

## Stack (from repo-context)

- **Language:** Python 3.11+
- **Framework:** FastAPI 0.104+
- **Async Runtime:** asyncio (built-in)
- **Event Queue:** `asyncio.Queue` (in-memory, single-process)
- **HTTP Client:** `httpx` (inherited from PER-20)
- **Protocol:** Server-Sent Events (SSE) over HTTP/1.1
- **Dependency:** PER-20 `process_question(query: str) → dict` (answer or error)

**No new external dependencies.** FastAPI's `StreamingResponse` handles SSE; `uuid.uuid4()` generates run IDs. Existing stack: FastAPI, asyncio, httpx.

---

## Schema & API Contracts

### 1. POST /agent/query

**Request:**
```json
POST /agent/query
Content-Type: application/json

{
  "query": "<non-empty string>"
}
```

**Response — Acceptance (AC-1):**
```json
HTTP 200
Content-Type: application/json

{
  "status": "ok",
  "agent_run_id": "<uuid v4 hex>"
}
```

**Response — Rejection (AC-2):**
```json
HTTP 200
Content-Type: application/json

{
  "status": "rejected",
  "agent_run_id": "<uuid v4 hex>",
  "reason": "invalid query"
}
```

**Behavior:**
- Input validation: reject if `query` is missing, `None`, or empty/whitespace-only.
- Return HTTP 200 immediately; do not wait for `process_question` or any LLM/MCP call.
- Generate and return `agent_run_id` in both success and rejection cases.
- On rejection: do not launch background job; do not call LLM or MCP. Enqueue `in_progress` and `failed` events synchronously before returning.
- On acceptance: launch background job asynchronously via `asyncio.create_task()` without awaiting; enqueue `in_progress` event synchronously before returning.
- Pydantic validation errors (HTTP 422 for missing or non-string `query`) are caught by a custom exception handler and converted to HTTP 200 `{"status": "rejected", "agent_run_id": "<id>", "reason": "invalid query"}` before the handler body executes.

### 2. GET /agent/runs/{agent_run_id}/events

**Request:**
```http
GET /agent/runs/{agent_run_id}/events
Accept: text/event-stream
```

**Response Headers (AC-3):**
```http
HTTP 200
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

**Event Stream Format (SSE):**

Each event is a JSON object sent as `data: <json>`.

**Event 1: in_progress (AC-3)**
```
event: in_progress
data: {"run_id": "<agent_run_id>"}
```
Sent immediately when the run is accepted, before any background work begins.

**Event 2a: completed (AC-4)**
```
event: completed
data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
```
Sent when `process_question` returns a successful answer. Timing:
- If answer is ready **less than 5 seconds** after `in_progress` is sent: hold until 5 seconds have elapsed, then send `completed`.
- If answer is ready **5 seconds or later** after `in_progress`: send `completed` immediately.

**Event 2b: failed (AC-5)**
```
event: failed
data: {"run_id": "<agent_run_id>", "error": "<error message>"}
```
Sent immediately (no 5-second hold) when `process_question` returns `{"error": "…"}` or when LLM/MCP call fails, or when the query is rejected in the POST handler. Stream ends after this event.

**Error Response (AC-3):**
```json
HTTP 404
Content-Type: application/json

{
  "detail": "Run not found"
}
```
Sent if `agent_run_id` does not exist or has been cleaned up.

**Behavior:**
- First event is always `in_progress`.
- Exactly one of `completed` or `failed` follows.
- Stream ends after `completed` or `failed`.
- Every event contains the same `run_id` (the `agent_run_id` from POST).
- Multiple subscribers for the same `agent_run_id` are allowed; the background job is not cancelled when a subscriber disconnects.

### 3. POST /query (unchanged — AC-6)

**Request:**
```json
POST /query
Content-Type: application/json

{
  "question": "<non-empty string>"
}
```

**Response:**
```json
HTTP 200
Content-Type: application/json

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

**Behavior:** Blocking; calls `process_question` directly and waits for result. No changes.

### 4. GET /health (unchanged — AC-6)

**Response:**
```json
HTTP 200
Content-Type: application/json

{"status": "ok"}
```

**Behavior:** No changes.

---

## File Touch List

All paths resolve from the `agent` repository root. Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{agent_run_id}/events, existing POST /query, GET /health) are registered in `agent/main.py` only.

### New Files

```
agent/
├── __init__.py
├── models.py                   (Pydantic models)
├── run_store.py                (in-memory run registry)
├── sse_handler.py              (SSE event stream generator)
├── background_task.py          (background orchestrator)
├── main.py                     (add /agent/query and /agent/runs/{id}/events; keep POST /query and GET /health)
├── orchestrator.py             (existing PER-20 process_question)
├── health.py                   (existing PER-20 health)
├── mcp_client.py               (existing PER-20 MCP client)
└── tests/
    ├── test_agent_query_accept.py      (Story 1: AC-1, AC-2 — 8 tests)
    ├── test_agent_sse_stream.py        (Story 2: AC-3, AC-4, AC-5 — 15 tests)
    └── test_backward_compat.py         (Story 3: AC-6 — 9 tests)
```

### Modified Files

```
agent/
├── main.py        (add POST /agent/query, GET /agent/runs/{id}/events handlers; startup cleanup task)
├── models.py      (add SSE event data models)
└── run_store.py   (create with all methods)
```

### Configuration

```
agent/
├── requirements.txt                    (verify FastAPI, uvicorn, pydantic, httpx)
```

---

## Pydantic Models

### Request/Response Models (agent/models.py)

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
    """Data for in_progress event."""
    run_id: str

class CompletedEventData(BaseModel):
    """Data for completed event."""
    run_id: str
    response: dict  # {"answer": "<text>"}

class FailedEventData(BaseModel):
    """Data for failed event."""
    run_id: str
    error: str
```

---

## Data Structures

### Run Store (agent/run_store.py)

```python
from dataclasses import dataclass
from typing import Optional
from asyncio import Queue
import time
import uuid

@dataclass
class Run:
    """Single agent run."""
    id: str                                    # agent_run_id (UUID v4 hex)
    query: str
    created_at: float                          # Unix timestamp
    status: str                                # "pending", "completed", "failed"
    events: Queue                              # asyncio.Queue for SSE delivery
    result: Optional[dict] = None              # {"answer": "…"} or {"error": "…"}
    started_at: Optional[float] = None
    ended_at: Optional[float] = None
    in_progress_sent_at: Optional[float] = None

class RunStore:
    """In-memory registry of active runs."""
    
    def __init__(self):
        self._runs: dict[str, Run] = {}
    
    async def create(self, query: str) -> Run:
        """Generate unique agent_run_id, create Run, enqueue in_progress event.
        
        Enqueues in_progress event synchronously before returning.
        """
        run_id = uuid.uuid4().hex
        now = time.time()
        
        run = Run(
            id=run_id,
            query=query,
            created_at=now,
            status="pending",
            events=Queue(),
            in_progress_sent_at=now
        )
        
        self._runs[run_id] = run
        
        # Enqueue in_progress event immediately (synchronous)
        await run.events.put({
            "event": "in_progress",
            "data": {"run_id": run_id}
        })
        
        return run
    
    async def get(self, run_id: str) -> Optional[Run]:
        """Retrieve Run by agent_run_id; return None if not found or expired."""
        return self._runs.get(run_id)
    
    async def enqueue_event(self, run_id: str, event_type: str, data: dict) -> None:
        """Append event to run's queue for SSE delivery.
        
        Args:
            run_id: agent_run_id
            event_type: "in_progress", "completed", or "failed"
            data: JSON-serializable dict with run_id and result/error
        """
        run = self._runs.get(run_id)
        if run:
            await run.events.put({"event": event_type, "data": data})
    
    async def mark_failed(self, run_id: str, reason: str) -> None:
        """Mark rejected run as failed; enqueue failed event.
        
        Called during AC-2 rejection path (invalid input).
        Enqueues: failed event with reason.
        The in_progress event was already enqueued by create().
        """
        run = self._runs.get(run_id)
        if run:
            run.status = "failed"
            await run.events.put({
                "event": "failed",
                "data": {"run_id": run_id, "error": reason}
            })
    
    async def cleanup_expired(self, ttl_seconds: int = 3600) -> None:
        """Delete runs older than ttl_seconds.
        
        Called periodically by cleanup task on app startup.
        Default TTL: 1 hour.
        """
        now = time.time()
        expired_ids = [
            run_id for run_id, run in self._runs.items()
            if (now - run.created_at) > ttl_seconds
        ]
        for run_id in expired_ids:
            del self._runs[run_id]
```

---

## Key Interfaces

### Background Task (agent/background_task.py)

```python
import asyncio
import time
from agent.run_store import run_store
from agent.orchestrator import process_question

async def run_background_job(agent_run_id: str, query: str) -> None:
    """Execute process_question in background; enqueue events.
    
    Calls process_question from PER-20.
    Enforces 5-second hold for completed events only.
    Enqueues result events to run_store.
    """
    run = await run_store.get(agent_run_id)
    if not run:
        return  # Run already cleaned up; abort silently
    
    try:
        run.started_at = time.time()
        
        # Call existing PER-20 orchestrator
        result = await process_question(query)
        
        if "error" in result:
            # Failure path (AC-5): send immediately
            await run_store.enqueue_event(agent_run_id, "failed", {
                "run_id": agent_run_id,
                "error": result["error"]
            })
            run.status = "failed"
        else:
            # Success path (AC-4): apply 5-second hold
            elapsed = time.time() - run.in_progress_sent_at
            hold_remaining = 5.0 - elapsed
            
            if hold_remaining > 0:
                await asyncio.sleep(hold_remaining)
            
            await run_store.enqueue_event(agent_run_id, "completed", {
                "run_id": agent_run_id,
                "response": {"answer": result["answer"]}
            })
            run.status = "completed"
        
        run.result = result
    
    except Exception as e:
        # Unhandled exception (AC-5): send failed immediately
        await run_store.enqueue_event(agent_run_id, "failed", {
            "run_id": agent_run_id,
            "error": str(e)
        })
        run.status = "failed"
    
    finally:
        run.ended_at = time.time()
```

### SSE Handler (agent/sse_handler.py)

```python
from typing import AsyncGenerator
import json
from fastapi import HTTPException
from agent.run_store import run_store

async def stream_events(run_id: str) -> AsyncGenerator[str, None]:
    """Stream SSE events for a given run.
    
    Yields events in SSE format: event: <type>\ndata: <json>\n\n
    First event is in_progress.
    Second event is exactly one of (completed | failed).
    Stream closes after final event.
    """
    run = await run_store.get(run_id)
    if not run:
        raise HTTPException(status_code=404, detail="Run not found")
    
    async for event_payload in run.events:
        event_type = event_payload["event"]
        data_json = json.dumps(event_payload["data"])
        yield f"event: {event_type}\n"
        yield f"data: {data_json}\n\n"
        
        if event_type in ("completed", "failed"):
            break  # Close stream after final event
```

### Routes (agent/main.py — add to existing)

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from fastapi.exceptions import RequestValidationError
from agent.models import AgentQueryRequest, AgentQueryAccepted, AgentQueryRejected
from agent.sse_handler import stream_events
from agent.run_store import RunStore
from agent.background_task import run_background_job
import asyncio

# Initialize run store (singleton)
run_store = RunStore()

app = FastAPI()

# Custom exception handler: convert HTTP 422 (Pydantic validation error) to HTTP 200 rejected response
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    """Convert Pydantic validation errors to HTTP 200 rejected response."""
    run = await run_store.create("")
    await run_store.mark_failed(run.id, "invalid query")
    return {
        "status": "rejected",
        "agent_run_id": run.id,
        "reason": "invalid query"
    }

# Cleanup task
async def cleanup_task():
    """Periodically delete expired runs."""
    while True:
        await asyncio.sleep(60)
        await run_store.cleanup_expired(ttl_seconds=3600)

@app.on_event("startup")
async def startup():
    """Start cleanup task on app startup."""
    asyncio.create_task(cleanup_task())

# New endpoints
@app.post("/agent/query", response_model=AgentQueryAccepted | AgentQueryRejected)
async def post_agent_query(req: AgentQueryRequest) -> AgentQueryAccepted | AgentQueryRejected:
    """Accept a user query immediately and return agent_run_id."""
    query = req.query.strip()
    
    if not query:
        # Rejection path (AC-2)
        run = await run_store.create("")
        await run_store.mark_failed(run.id, "invalid query")
        return AgentQueryRejected(
            status="rejected",
            agent_run_id=run.id,
            reason="invalid query"
        )
    
    # Acceptance path (AC-1)
    run = await run_store.create(query)
    asyncio.create_task(run_background_job(run.id, query))
    return AgentQueryAccepted(status="ok", agent_run_id=run.id)

@app.get("/agent/runs/{agent_run_id}/events")
async def get_agent_events(agent_run_id: str) -> StreamingResponse:
    """Stream SSE events for a given run."""
    run = await run_store.get(agent_run_id)
    if not run:
        raise HTTPException(status_code=404, detail="Run not found")
    
    return StreamingResponse(
        stream_events(agent_run_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )

# Existing endpoints (unchanged)
@app.post("/query")
async def post_query(question: str) -> dict:
    """Existing blocking query endpoint (PER-20)."""
    # Implementation from PER-20; no changes

@app.get("/health")
async def get_health() -> dict:
    """Existing health check endpoint (unchanged)."""
    return {"status": "ok"}
```

---

## Risks

| Risk | Impact | Severity | Mitigation |
|------|--------|----------|-----------|
| **Event loss if process crashes before SSE connects** | Subscriber misses `in_progress` or answers. | Medium | Queue events in `asyncio.Queue` per run before SSE stream reads them. Replay on stream open. TTL-based cleanup ensures old runs are deleted. |
| **Unbounded run accumulation** | Memory leak; OOM crash. | High | Implement TTL-based cleanup (delete runs after 1 hour). Log cleanup policy in startup. Configurable via `RUN_CLEANUP_TTL_SECONDS` env var. |
| **Multiple subscribers desync** | One subscriber slow; others see different timings. | Low | All events read from same queue in order; no per-subscriber logic. All subscribers see identical timing. |
| **5-second hold blocks other work** | System throughput reduced. | Low | Hold enforced via `asyncio.sleep()` in background task, not blocking event queue or other jobs. Non-blocking per-run. |
| **Unknown run_id returns 404 immediately** | Caller assumes run failed, but job may still be running. | Low | Clarify in API docs: 404 means run was never created or has been cleaned up. Return 404 only for truly unknown ids; keep runs for at least 1 hour (configurable). |
| **PER-20 signature changes** | Breaking change to new endpoint. | High | This feature depends on the current PER-20 contract (`process_question` return type). Any future change must preserve `{"answer": "…"} \| {"error": "…"}`. Document as hard dependency. Lock contract in Story 3 tests (AC-6). |
| **Synchronous validation blocks request thread** | High-latency requests pile up. | Low | Validation is O(1) string check; negligible. Keep validation synchronous. |
| **In-memory run store grows unbounded** | Memory leak. | Medium | Implement TTL-based cleanup (delete runs 1 hour after completion). Log policy in startup. |
| **Background task never spawned** | Job runs but no result is ever published. | Medium | Test that spawn completes before HTTP return; verify in unit tests. |
| **Exception in background task silently swallowed** | Job fails without emitting error event; stream never closes. | Medium | Wrap background task in try/except; emit `failed` event on any exception. Log to stderr. |

---

## Test Coverage

### Story 1 (PER-43): AC-1, AC-2 (8 tests)
- Test 1: Valid query → HTTP 200 `{"status": "ok", "agent_run_id": "…"}` immediately
- Test 2: Valid query → response < 100ms (does not wait for `process_question`)
- Test 3: Empty query → HTTP 200 `{"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}`
- Test 4: Missing query field → HTTP 200 rejected response
- Test 5: Whitespace-only query → HTTP 200 rejected response
- Test 6: Rejection does not call `process_question` (no background job spawn)
- Test 7: Both paths return non-empty `agent_run_id` (different on separate calls)
- Test 8: Run store contains created run with matching `id`; rejection path enqueues `in_progress` and `failed` events synchronously

### Story 2 (PER-44): AC-3, AC-4, AC-5 (15 tests)
- Test 1: GET /agent/runs/{known_id}/events returns HTTP 200 `text/event-stream`
- Test 2: First event is `in_progress` with `data: {"run_id": "<agent_run_id>"}`
- Test 3: All events include same `run_id`
- Test 4: Unknown run_id returns HTTP 404
- Test 5: Multiple subscribers receive identical events in order
- Test 6: Fast answer (< 5s) held until 5s, then `completed` sent
- Test 7: Slow answer (≥ 5s) `completed` sent immediately when ready (no extra hold)
- Test 8: Edge case: answer at exactly 5s receives `completed` immediately
- Test 9: LLM error → `in_progress`, then `failed` (no hold, no `completed`)
- Test 10: MCP error → `in_progress`, then `failed` immediately
- Test 11: Invalid query rejection → `failed` event (no `completed`)
- Test 12: Stream closes after `completed` event
- Test 13: Stream closes after `failed` event
- Test 14: Subscriber disconnect does not cancel background job
- Test 15: Late subscriber retrieves all events (no replay loss)

### Story 3 (PER-45): AC-6 (9 tests)
- Test 1: POST /query with valid question returns `{"answer": "…"}` blocking
- Test 2: POST /query with error returns `{"error": "…"}` blocking
- Test 3: POST /query latency unchanged (no regression from Stories 1–2)
- Test 4: GET /health returns `{"status": "ok"}`
- Test 5: POST /query and POST /agent/query coexist without interference
- Test 6: MCP called only from main process (no child processes)
- Test 7: Tool names unchanged
- Test 8: ID normalization logic unchanged
- Test 9: MCP_URL configuration unchanged

**Total Test Count: 32** (covering all 6 ACs)

---

## Acceptance Criteria Mapping

| AC | File(s) | Story | Tests |
|----|---------|-------|-------|
| **AC-1: Immediate accept** | `agent/main.py`, `agent/models.py`, `agent/run_store.py` | Story 1 (PER-43) | 1, 2, 7 |
| **AC-2: Reject without a run** | `agent/main.py`, `agent/run_store.py` | Story 1 (PER-43) | 3, 4, 5, 6, 7, 8 |
| **AC-3: One event stream** | `agent/sse_handler.py`, `agent/main.py`, `agent/run_store.py` | Story 2 (PER-44) | 1, 2, 3, 4, 5, 14, 15 |
| **AC-4: Five-second hold** | `agent/background_task.py`, `agent/run_store.py` | Story 2 (PER-44) | 6, 7, 8, 12 |
| **AC-5: Failure immediate** | `agent/background_task.py`, `agent/run_store.py` | Story 2 (PER-44) | 9, 10, 11, 13 |
| **AC-6: Existing paths unchanged** | `agent/main.py` (audit), existing code | Story 3 (PER-45) | 1–9 |

---

## Key Decisions & Assumptions

**Adopted:**
- Event queue: In-memory `asyncio.Queue` per run (PoC; upgrade to Redis if multi-process required).
- 5-second hold: Enforced via `asyncio.sleep()` in background task; non-blocking per-run.
- Run cleanup: TTL-based; runs deleted 1 hour after completion/failure (configurable via `RUN_CLEANUP_TTL_SECONDS` env var).
- Run ID format: UUID v4 hex (32-char string, collision-free).
- Input validation: O(1) string check; synchronous before task spawn.
- Validation error handling: Custom exception handler converts HTTP 422 (Pydantic validation) to HTTP 200 rejected response before handler body executes.
- No auth: Out of scope per AC-6 requirements.
- Routes location: All routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) in `agent/main.py` per change_requests.md.

**Deferred:**
- Run history API (`GET /agent/runs`) — not in scope.
- Rate limiting — assume upstream FastAPI middleware.
- Metrics/logging beyond startup — out of scope.

---

## Open Questions

1. **Should hold duration be configurable?** → Yes; default 5s via `AGENT_COMPLETE_HOLD_SECONDS` env var.
2. **Should cleanup interval be tunable?** → Yes; default 60s via `RUN_CLEANUP_INTERVAL_SECONDS` env var.
3. **Multi-process deployment support?** → In-memory queue is PoC; upgrade to Redis if > 100 concurrent runs or multi-process required.
4. **Should we cap concurrent SSE subscribers per run?** → No limit for PoC; add if resource contention surfaces.
```

---

# Story summaries

- `01-agent`: ```markdown
- `02-agent`: ```markdown
- `03-agent`: ```markdown

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
