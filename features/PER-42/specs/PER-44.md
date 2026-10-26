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
