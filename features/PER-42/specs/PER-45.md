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
