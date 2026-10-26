```markdown
# Story LLD: 01-agent — Accept FastAPI query without blocking; return agent_run_id immediately

**Ticket ID:** PER-43  
**Story ID:** 01-agent  
**Repository:** `agent`  
**Acceptance Criteria:** AC-1, AC-2  
**Story Type:** Backend Feature  

---

## Acceptance Criteria

### AC-1: Immediate accept

**Given** the agent is up and running  
**When** `POST /agent/query` receives `{"query": "<non-empty string>"}`  
**Then** the response is HTTP 200 `{"status": "ok", "agent_run_id": "<non-empty string>"}` before the LLM or MCP call finishes, and the handler does not wait for `process_question` to complete.

### AC-2: Reject without a run

**Given** `query` is missing or blank  
**When** `POST /agent/query` is called  
**Then** the response is HTTP 200 `{"status": "rejected", "agent_run_id": "<non-empty string>", "reason": "invalid query"}`, the LLM and MCP are not called, and `GET /agent/runs/{agent_run_id}/events` sends `failed` with that reason and does not send `completed`.

---

## File Plan

All paths resolve from the `agent` repository root. Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) are registered in `agent/main.py` only. No separate routes module.

### New Files

```
agent/
├── __init__.py
├── models.py
├── run_store.py
└── tests/
    └── test_agent_query_accept.py
```

**Key files:**

1. **`agent/models.py`** — Pydantic request/response models (`AgentQueryRequest`, `AgentQueryAccepted`, `AgentQueryRejected`).
2. **`agent/run_store.py`** — In-memory run registry; manages `agent_run_id` and event queues.
3. **`agent/main.py`** — Application startup; POST /agent/query route handler, run store initialization, and custom exception handler to convert HTTP 422 validation errors to HTTP 200 rejected responses.
4. **`agent/tests/test_agent_query_accept.py`** — Unit tests for AC-1 and AC-2 (8 tests total).

### Modified Files

```
agent/
├── main.py (add POST /agent/query handler and custom exception handler)
├── requirements.txt (verify FastAPI, uvicorn, pydantic, httpx present)
└── README.md (document new endpoint and agent_run_id concept)
```

---

## Interfaces

### Request Model (`agent/models.py`)

```python
from pydantic import BaseModel, Field

class AgentQueryRequest(BaseModel):
    """User query submitted to the agent."""
    query: str = Field(..., min_length=1, description="Non-empty user query string")
```

### Response Models (`agent/models.py`)

```python
from typing import Literal

class AgentQueryAccepted(BaseModel):
    """Successful acceptance of a query."""
    status: Literal["ok"]
    agent_run_id: str = Field(..., description="Unique run identifier for SSE stream")

class AgentQueryRejected(BaseModel):
    """Rejection due to invalid input."""
    status: Literal["rejected"]
    agent_run_id: str = Field(..., description="Unique run identifier for error stream")
    reason: str = Field(..., description="Reason for rejection (e.g., 'invalid query')")
```

### Run Store Interface (`agent/run_store.py`)

```python
from dataclasses import dataclass
from typing import Optional
from asyncio import Queue
import uuid
import time

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

class RunStore:
    """In-memory registry of active runs."""
    
    def __init__(self):
        self._runs: dict[str, Run] = {}
    
    async def create(self, query: str) -> Run:
        """Generate unique agent_run_id, create Run, enqueue in_progress event, return Run.
        
        Enqueues in_progress event synchronously before returning.
        Records in_progress_sent_at for 5-second hold calculation (Story 2).
        
        Returns: Run object with id, events queue pre-populated with in_progress event.
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
        
        Full implementation in Story 2; placeholder in Story 1.
        """
        run = self._runs.get(run_id)
        if run:
            await run.events.put({"event": event_type, "data": data})
    
    async def mark_failed(self, run_id: str, reason: str) -> None:
        """Mark rejected run as failed; enqueue failed event.
        
        Called during AC-2 rejection path (invalid input) to enqueue the failed event.
        The in_progress event was already enqueued by create().
        Both events are now queued in run.events for SSE delivery.
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
        
        Full implementation in Story 2; placeholder in Story 1.
        """
        pass
```

### Route Handler (`agent/main.py` — POST /agent/query endpoint)

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from agent.models import AgentQueryRequest, AgentQueryAccepted, AgentQueryRejected
from agent.run_store import RunStore
import asyncio

app = FastAPI()

# Global singleton run store
run_store = RunStore()

# Custom exception handler: convert HTTP 422 (Pydantic validation error) to HTTP 200 rejected response
# Register BEFORE routes to intercept validation errors before handler body executes
@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    """Convert Pydantic validation errors to HTTP 200 rejected response.
    
    FastAPI will raise RequestValidationError (HTTP 422) when query field is missing
    or is not a string. This handler intercepts it and returns HTTP 200 with
    rejected status, matching AC-1 and AC-2 contract.
    
    Creates a run with empty query and immediately marks it failed, ensuring both
    in_progress and failed events are queued before returning to caller.
    """
    run = await run_store.create("")
    await run_store.mark_failed(run.id, "invalid query")
    return JSONResponse(
        status_code=200,
        content={
            "status": "rejected",
            "agent_run_id": run.id,
            "reason": "invalid query"
        }
    )

@app.post("/agent/query", response_model=AgentQueryAccepted | AgentQueryRejected)
async def post_agent_query(req: AgentQueryRequest) -> AgentQueryAccepted | AgentQueryRejected:
    """Accept a user query immediately and return agent_run_id.
    
    AC-1: Return HTTP 200 immediately without waiting for process_question.
    AC-2: Reject if query is missing or empty/whitespace-only; do not spawn background job.
    
    Spawns background orchestration via asyncio.create_task() without awaiting,
    ensuring immediate return to caller before any LLM or MCP work begins.
    """
    query = req.query.strip()
    
    if not query:
        # Rejection path (AC-2): empty or whitespace-only query
        # Handler-level validation (after Pydantic schema validation passes)
        run = await run_store.create("")
        await run_store.mark_failed(run.id, "invalid query")
        return AgentQueryRejected(
            status="rejected",
            agent_run_id=run.id,
            reason="invalid query"
        )
    
    # Acceptance path (AC-1)
    # Create run and enqueue in_progress event synchronously
    run = await run_store.create(query)
    
    # Spawn background job asynchronously without awaiting (returns immediately)
    # Background task will call process_question, enforce 5-second hold (Story 2),
    # and enqueue completed or failed event
    asyncio.create_task(run_background_job(run.id, query))
    
    return AgentQueryAccepted(status="ok", agent_run_id=run.id)

async def run_background_job(agent_run_id: str, query: str) -> None:
    """Placeholder for background job orchestration.
    
    This function is spawned via asyncio.create_task() and does not block the POST handler.
    Full implementation (process_question call, 5-second hold, event enqueue) is provided
    in Story 2 (02-agent).
    
    Args:
        agent_run_id: unique run identifier (UUID v4 hex)
        query: user query string (already validated and stripped from AC-1/AC-2)
    
    In Story 1, this function is spawned but executes no work (pass placeholder).
    Story 2 (02-agent) implements:
      - Call process_question(query) from PER-20 orchestrator
      - Apply 5-second hold for completed events only (AC-4)
      - Enqueue result (completed or failed) to run_store via run_store.enqueue_event()
      - Handle exceptions and emit failed events immediately (AC-5)
      - Record run.started_at and run.ended_at timestamps
    """
    pass

# Existing endpoints (unchanged)
@app.post("/query")
async def post_query(question: str) -> dict:
    """Existing blocking query endpoint (PER-20).
    
    No changes; verified in Story 3 (03-agent) for backward compatibility (AC-6).
    """
    pass

@app.get("/health")
async def get_health() -> dict:
    """Existing health check endpoint (unchanged).
    
    No changes; verified in Story 3 (03-agent) for backward compatibility (AC-6).
    """
    return {"status": "ok"}
```

---

## Algorithm Notes

### Input Validation & Error Handling

1. **Pydantic validation (automatic, before handler):**
   - FastAPI receives request body and validates against `AgentQueryRequest` schema.
   - If `query` field is missing: FastAPI raises `RequestValidationError` → HTTP 422.
   - If `query` is not a string (e.g., integer, null): FastAPI raises `RequestValidationError` → HTTP 422.
   - Custom exception handler (registered **before routes** at app startup) catches `RequestValidationError` and converts to HTTP 200 rejected response **before POST handler body executes**.

2. **Custom validation in handler (after Pydantic passes):**
   - Strip whitespace from `query`.
   - If result is empty string, treat as invalid (rejection path AC-2).

3. **Response contract:** HTTP 200 for both acceptance and rejection (per AC-1, AC-2); never 422 from this endpoint.

### Acceptance Path (AC-1)

1. Input passes validation (non-empty after strip).
2. Call `run_store.create(query.strip())`:
   - Generate `agent_run_id` as UUID v4 hex string (32 chars, globally unique).
   - Create `Run` object with status `"pending"`.
   - Create `asyncio.Queue` for events.
   - Record `in_progress_sent_at = time.time()` (timestamp for AC-4 hold calculation in Story 2).
   - Enqueue `in_progress` event with data `{"run_id": agent_run_id}` **synchronously** to `run.events` before method returns.
   - Store `Run` in `_runs` dict keyed by `agent_run_id`.
   - Return `Run` object.
3. Spawn background job: `asyncio.create_task(run_background_job(run.id, query.strip()))`.
   - **Critical:** Do not `await` this call; return immediately.
   - Background task will execute asynchronously in same process (no fork, no thread).
   - Background task will call `process_question(query)` from PER-20 (implementation deferred to Story 2).
4. Return HTTP 200 `AgentQueryAccepted(status="ok", agent_run_id=run.id)` immediately.
5. **Do NOT wait for background job or process_question to complete.**

### Rejection Path (AC-2)

**Path A: Pydantic validation error (HTTP 422 → 200 via custom handler)**

1. Request body missing `query` field or `query` is not a string.
2. FastAPI auto-validation fails, raises `RequestValidationError`.
3. Custom `@app.exception_handler(RequestValidationError)` intercepts:
   - Calls `run_store.create("")` (empty query for error run).
   - Calls `run_store.mark_failed(run.id, "invalid query")`.
   - Returns HTTP 200 `{"status": "rejected", "agent_run_id": run.id, "reason": "invalid query"}`.
4. **POST handler body is never executed.**
5. **`asyncio.create_task()` is never called; background job never spawned.**

**Path B: Handler-level validation (empty or whitespace-only query)**

1. `AgentQueryRequest` validation passes (query is a non-empty string per Pydantic schema).
2. POST handler body executes.
3. Handler strips whitespace; if result is empty, rejection path:
   - Call `run_store.create("")` (empty query for error run).
   - Call `run_store.mark_failed(run.id, "invalid query")`.
   - Return HTTP 200 `AgentQueryRejected(status="rejected", agent_run_id=run.id, reason="invalid query")`.
4. **`asyncio.create_task()` is never called; background job never spawned.**

**Both rejection paths result in:**
- Run created with `id` (agent_run_id).
- Both `in_progress` and `failed` events queued in `run.events` **synchronously before HTTP response returns**.
- No background job spawned; `process_question` never called.
- HTTP 200 response returned immediately.
- When SSE stream opens later (Story 2), subscriber reads both events from `run.events` and closes stream after `failed`.

### ID Generation

- Use `uuid.uuid4().hex` → 32-character lowercase hex string (e.g., `"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"`).
- Collision probability: negligible (< 2^-120 for < 1 billion IDs).
- No locking required; UUID v4 generation is thread-safe.
- Same `agent_run_id` returned in both acceptance and rejection paths.

### Error Handling in Story 1

- **Pydantic validation error (missing or non-string `query`):** Caught by custom exception handler **before handler body executes**. Returns HTTP 200 rejected response.
- **Handler-level validation error (empty/whitespace query):** Caught by post_agent_query after Pydantic passes. Calls `mark_failed()` to enqueue `failed` event synchronously before returning HTTP 200.
- **Unhandled exception in POST handler:** Bubbles up to FastAPI; returns HTTP 500 by default. Story 2 wraps background job in try/except to emit `failed` events on exceptions in background orchestration.

---

## Tests

### Unit Test File: `agent/tests/test_agent_query_accept.py`

#### Test 1: AC-1 — Accept valid query returns ok with agent_run_id

**Given** valid query `{"query": "What is the weather?"}`  
**When** `POST /agent/query` is called  
**Then** response status 200, body contains `{"status": "ok", "agent_run_id": "<non-empty>"}`, response returned before `process_question` completes.

**Verification:**
- Response HTTP status is 200.
- Response body contains `"status": "ok"`.
- Response body contains `"agent_run_id"` as non-empty string (UUID v4 hex format).
- Response JSON is valid and parseable.

#### Test 2: AC-1 — Immediate return; does not wait for background job

**Given** valid query and POST /agent/query handler spawns background job  
**When** `POST /agent/query` returns  
**Then** response elapsed time < 100ms (no waiting for LLM/MCP). Run state is "pending" immediately after POST returns.

**Verification:**
- Record wall-clock time before POST call: `t_start`.
- Call `POST /agent/query` with valid query.
- Record wall-clock time of response receipt: `t_response`.
- Assert `(t_response - t_start) < 0.1` seconds (100ms).
- Query run store for returned `agent_run_id`; verify `run.status == "pending"` (not completed/failed).
- Background task is spawned asynchronously; run store is accessible before task completion.

#### Test 3: AC-2 — Empty query returns rejected with reason

**Given** empty query `{"query": ""}`  
**When** `POST /agent/query` is called  
**Then** response status 200 (not 422), body contains `{"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}`.

**Verification:**
- Response HTTP status is 200 (not 422).
- Response body contains `"status": "rejected"`.
- Response body contains `"reason": "invalid query"` (exact match).
- Response body contains `"agent_run_id"` as non-empty string.

#### Test 4: AC-2 — Missing query field returns rejected (HTTP 422 → 200)

**Given** request body missing `query` field: `{}`  
**When** `POST /agent/query` is called with `{}`  
**Then** response status 200 (not 422), body contains `{"status": "rejected", "reason": "invalid query"}`.

**Verification:**
- Response HTTP status is 200 (not 422; custom exception handler converts validation error).
- Response body contains `"status": "rejected"`.
- Response body contains `"reason": "invalid query"` (exact match).
- Response body contains `"agent_run_id"` as non-empty string.
- FastAPI Pydantic validation error is intercepted by `@app.exception_handler(RequestValidationError)`.

#### Test 5: AC-2 — Whitespace-only query returns rejected

**Given** whitespace-only query `{"query": "   \t\n  "}`  
**When** `POST /agent/query` is called  
**Then** response status 200, body contains `{"status": "rejected", "reason": "invalid query"}`.

**Verification:**
- Response HTTP status is 200.
- Response body contains `"status": "rejected"`.
- Response body contains `"reason": "invalid query"` (exact match).
- Response body contains `"agent_run_id"` as non-empty string.
- Whitespace stripping in handler correctly detects empty string after strip.

#### Test 6: AC-2 — Rejection does not spawn background job

**Given** rejected query (empty, missing, or non-string)  
**When** `POST /agent/query` returns  
**Then** `asyncio.create_task()` is not called; `run_background_job()` function is never invoked; `process_question` never called.

**Verification:**
- Mock `asyncio.create_task` to track call count.
- Mock `run_background_job` to track invocations.
- Call `POST /agent/query` with rejected input (e.g., missing `query`, empty string, whitespace).
- Assert `asyncio.create_task.call_count == 0`.
- Assert `run_background_job.call_count == 0` (or inspection confirms task never scheduled).
- Acceptance path (valid query) increments call count to 1; rejection path leaves it at 0.

#### Test 7: AC-1, AC-2 — Both paths return non-empty, unique agent_run_id

**Given** multiple calls (acceptance and rejection scenarios)  
**When** `POST /agent/query` is called multiple times with different inputs  
**Then** all responses have non-empty `agent_run_id`, and separate calls return different values (UUID v4 collision-free).

**Verification:**
- Call `POST /agent/query` with valid query → response A with `agent_run_id_A`.
- Call `POST /agent/query` with empty query → response B with `agent_run_id_B`.
- Call `POST /agent/query` with missing `query` field → response C with `agent_run_id_C`.
- Assert `agent_run_id_A`, `agent_run_id_B`, `agent_run_id_C` all non-empty strings.
- Assert `agent_run_id_A != agent_run_id_B != agent_run_id_C` (no collisions, all unique).
- Each is a valid UUID v4 hex string (32 lowercase hex characters).

#### Test 8: AC-2 — Rejection path enqueues failed event synchronously before POST returns

**Given** rejected query (empty)  
**When** `POST /agent/query` returns  
**Then** run store contains run with matching id, and `run.events` queue contains exactly 2 events (in_progress, then failed) queued synchronously before POST response is sent.

**Verification:**
- Call `POST /agent/query` with empty query.
- Get response with `agent_run_id`.
- Query run store: `run = await run_store.get(agent_run_id)`.
- Assert run is not None.
- Assert `run.id == agent_run_id` (exact match).
- Assert `run.status == "failed"` (marked failed by rejection path).
- Drain `run.events` queue without awaiting (peek or get_nowait):
  - Event 1: type `"in_progress"`, data `{"run_id": agent_run_id}` (enqueued by `create()`).
  - Event 2: type `"failed"`, data `{"run_id": agent_run_id, "error": "invalid query"}` (enqueued by `mark_failed()`).
- Assert exactly 2 events in queue (no more, no less).
- Assert both events enqueued before POST returned (confirmed by run store state immediately after POST).

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|--------|-----------|---------------|
| AC-1 | 1–2, 7 | Valid query → immediate ok response | Response contains `status: "ok"`, `agent_run_id`; returned < 100ms; no wait for `process_question`; unique agent_run_id on each call |
| AC-2 | 3–8 | Empty/missing/whitespace query → immediate rejected response, no background job spawn | Response contains `status: "rejected"`, `reason: "invalid query"`, `agent_run_id`; HTTP 200 (not 422); `asyncio.create_task()` not called; in_progress and failed events enqueued synchronously before POST returns |
| AC-1, AC-2 | 7 | agent_run_id uniqueness and return | Both paths return non-empty, unique agent_run_id on separate calls; UUID v4 collision-free |

**Total Test Count: 8**

---

## Dependencies & Blockers

- **Blocks Story 2 (02-agent):** Story 2 requires `run_store.create()`, `run_store.get()`, `run_store.enqueue_event()`, `run_store.mark_failed()` implemented and tested.
- **Depends on PER-20:** Existing `process_question(query: str) → dict` function (not modified). Story 1 does not call it; Story 2 will.
- **Assumes FastAPI 0.104+** with `@app.exception_handler()` decorator support.

---

## Notes

- **No auth required.** Out of scope per AC-6 and requirements.
- **No rate limiting.** Assume upstream FastAPI middleware.
- **UUID v4 collision safety:** Safe for up to billions of runs; no ID reuse or collision risk.
- **Synchronous validation:** O(1) string check (strip and empty test); no async overhead; validation happens before task spawn.
- **Background job spawn:** Via `asyncio.create_task()` without await; does not block request handler. Returns immediately to caller.
- **Event enqueue on rejection:** `in_progress` and `failed` events enqueued to `run.events` (an `asyncio.Queue`) **synchronously** before POST returns. Story 2 SSE stream handler reads and delivers these queued events when `GET /agent/runs/{agent_run_id}/events` is called.
- **Pydantic validation handling:** Custom exception handler catches HTTP 422 from Pydantic validation (missing or non-string `query`) and converts to HTTP 200 rejected response before handler body executes. This ensures AC-1 and AC-2 contract for HTTP 200 responses is honored.
- **PER-20 dependency:** `process_question(query: str) → dict` signature is fixed (AC-6 Story 3). Any future change must preserve return type: `{"answer": "<text>"}` or `{"error": "<message>"}`.
- **Run store lifecycle:** Runs are created and stored in memory. Cleanup (TTL-based expiration) deferred to Story 2. No persistence layer; in-memory queue is sufficient for PoC.
- **Routes location:** Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) are registered in `agent/main.py` only. No separate routes module or file.
- **Exception handler registration order:** Custom `@app.exception_handler(RequestValidationError)` must be registered **before** route handlers to intercept validation errors before they reach the route handler. In FastAPI, decorator order matters; register exception handlers at app startup before route definitions are encountered.
- **Background task placeholder:** `run_background_job()` is a no-op (pass) in Story 1. Story 2 implements the full orchestration (call `process_question`, apply 5-second hold, enqueue events).
```
