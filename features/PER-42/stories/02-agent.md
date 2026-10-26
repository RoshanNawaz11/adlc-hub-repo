```markdown
# Story LLD: 02-agent — Stream PER-20 answer via SSE with in_progress, completed, or failed events

**Ticket ID:** PER-44  
**Story ID:** 02-agent  
**Repository:** `agent`  
**Acceptance Criteria:** AC-3, AC-4, AC-5  
**Story Type:** Backend Feature  

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

## File Plan

All paths resolve from the `agent` repository root.

### New Files

```
agent/
├── sse_handler.py
├── background_task.py
└── tests/
    └── test_agent_sse_stream.py
```

**Key files:**

1. **`agent/sse_handler.py`** — SSE stream reader and event formatter; yields `text/event-stream` format.
2. **`agent/background_task.py`** — Background orchestrator; calls `process_question`, enforces 5-second hold, enqueues result events.
3. **`agent/tests/test_agent_sse_stream.py`** — Unit and integration tests for AC-3, AC-4, AC-5 (15 tests).

### Modified Files

```
agent/
├── run_store.py (extend with cleanup_expired and enqueue_event methods)
├── models.py (add SSE event data models)
└── main.py (add GET /agent/runs/{agent_run_id}/events handler and cleanup task startup)
```

---

## Interfaces

### Event Data Models (`agent/models.py` — extend from Story 1)

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

### SSE Handler Interface (`agent/sse_handler.py`)

```python
from typing import AsyncGenerator
import json
from fastapi import HTTPException

async def stream_events(run_id: str) -> AsyncGenerator[str, None]:
    """
    Stream events for a given run in SSE format.
    
    Yields lines in SSE format:
      event: <type>
      data: <json>
      
    First event is in_progress.
    Second event is exactly one of (completed | failed).
    Stream closes after second event.
    
    Args:
        run_id: agent_run_id to stream events for
    
    Yields:
        SSE-formatted lines (event: ..., data: ..., blank line)
    
    Raises:
        HTTPException(404) if run_id not found or expired
    """
    from agent.run_store import run_store
    
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

async def run_background_job(agent_run_id: str, query: str) -> None:
    """
    Execute process_question in background; enqueue events to run store.
    
    Calls:
      1. run_store.get(agent_run_id) to retrieve run
      2. process_question(query) from PER-20 orchestrator
      3. Enforces 5-second hold for completed events only (AC-4)
      4. Enqueues result (completed or failed) to run_store (AC-5)
    
    On success: Enqueues completed event after enforcing 5-second hold.
    On error: Enqueues failed event immediately (no hold).
    On exception: Catches and enqueues failed event with error message.
    
    Args:
        agent_run_id: unique run identifier (UUID v4 hex)
        query: user query string (already validated and stripped from AC-1, AC-2)
    
    Returns:
        None (background execution; no return value to caller)
    """
    from agent.run_store import run_store
    from agent.orchestrator import process_question
    
    run = await run_store.get(agent_run_id)
    if not run:
        return  # Run already cleaned up; abort silently
    
    try:
        run.started_at = time.time()
        
        # Call existing PER-20 orchestrator
        # Expected return: {"answer": "<text>"} or {"error": "<message>"}
        result = await process_question(query)
        
        if "error" in result:
            # Failure path (AC-5): enqueue failed immediately, no hold
            await run_store.enqueue_event(agent_run_id, "failed", {
                "run_id": agent_run_id,
                "error": result["error"]
            })
            run.status = "failed"
        else:
            # Success path (AC-4): apply 5-second hold before enqueuing completed
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
        # Unhandled exception (AC-5): enqueue failed immediately
        await run_store.enqueue_event(agent_run_id, "failed", {
            "run_id": agent_run_id,
            "error": str(e)
        })
        run.status = "failed"
    
    finally:
        run.ended_at = time.time()
```

### Run Store Interface — Extensions (`agent/run_store.py` — extend from Story 1)

```python
import time

class RunStore:
    """In-memory registry of active runs (from Story 1, extended here)."""
    
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
        
        Args:
            ttl_seconds: time-to-live in seconds (default 3600 = 1 hour)
        """
        now = time.time()
        expired_ids = [
            run_id for run_id, run in self._runs.items()
            if (now - run.created_at) > ttl_seconds
        ]
        for run_id in expired_ids:
            del self._runs[run_id]
```

### Route Handler — Extension (`agent/main.py` — add to existing app)

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
import asyncio
import os

# Assuming app, run_store initialized in Story 1

@app.get("/agent/runs/{agent_run_id}/events")
async def get_agent_events(agent_run_id: str) -> StreamingResponse:
    """
    Stream SSE events for a given run.
    
    - AC-3: Returns 200 text/event-stream if run exists; 404 if unknown.
    - AC-4, AC-5: Emits in_progress, then (completed | failed), with timing rules enforced.
    
    Args:
        agent_run_id: unique run identifier (from POST /agent/query response)
    
    Returns:
        StreamingResponse with SSE events
    
    Raises:
        HTTPException(404) if run_id not found or expired
    """
    from agent.run_store import run_store
    from agent.sse_handler import stream_events
    
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
    from agent.run_store import run_store
    
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

## Algorithm Notes

### AC-3: SSE Stream Delivery

**Request flow:**

1. Receive `GET /agent/runs/{agent_run_id}/events`.
2. Call `run_store.get(agent_run_id)`:
   - If `None`: raise `HTTPException(status_code=404, detail="Run not found")`.
   - Else: proceed to stream.
3. Create `StreamingResponse` with `stream_events(agent_run_id)` generator.
4. Return response with headers:
   - `Content-Type: text/event-stream`
   - `Cache-Control: no-cache`
   - `Connection: keep-alive`

**Event generator (`stream_events()`):**

1. Retrieve run from store; if missing, raise 404.
2. Loop: `async for event_payload in run.events` (drains `asyncio.Queue`):
   - Extract `event_type` from payload (string: "in_progress", "completed", "failed").
   - Extract `data` dict from payload.
   - Serialize data to JSON.
   - Yield SSE line format:
     ```
     event: {event_type}\ndata: {json_data}\n\n
     ```
   - If `event_type` in (`"completed"`, `"failed"`): break (close stream).
3. On queue exhaustion or exception: close stream.

**Multi-subscriber behavior:**

- Each subscriber opens a new SSE connection and reads from `run.events` queue.
- `run.events` is an `asyncio.Queue` shared across all subscribers for the same run.
- All subscribers reading from the same queue see events in identical order (FIFO).
- All subscribers see identical content; timing differences are only due to network I/O.
- Late subscribers (connecting after background job completes) drain the queue and receive all previously enqueued events with no loss (full replay guarantee).

### AC-4: Five-Second Hold on Completed Events

**Timeline for fast answer (< 5 sec):**

```
t=0.0:  in_progress event enqueued by run_store.create() [Story 1]
        in_progress_sent_at = time.time() recorded
t=0.1:  process_question() returns answer
t=0.1:  elapsed = time.time() - in_progress_sent_at = 0.1s
t=0.1:  hold_remaining = 5.0 - 0.1 = 4.9s
t=0.1:  await asyncio.sleep(4.9) starts
t=5.0:  sleep completes
t=5.0:  completed event enqueued
        SSE subscriber receives event at t ≈ 5.0s
```

**Timeline for slow answer (>= 5 sec):**

```
t=0.0:  in_progress event enqueued; in_progress_sent_at = 0.0
t=6.0:  process_question() returns answer
t=6.0:  elapsed = 6.0s
t=6.0:  hold_remaining = 5.0 - 6.0 = -1.0s (negative)
t=6.0:  hold_remaining <= 0: skip sleep, send immediately
t=6.0:  completed event enqueued
        SSE subscriber receives event at t ≈ 6.0s (no extra hold)
```

**Algorithm:**

1. Background task calls `process_question(query)`.
2. If result contains `"answer"` key:
   - Calculate `elapsed = time.time() - run.in_progress_sent_at`.
   - Calculate `hold_remaining = 5.0 - elapsed`.
   - If `hold_remaining > 0`: `await asyncio.sleep(hold_remaining)`.
   - Enqueue `completed` event with answer.
3. The hold is enforced per-run via `asyncio.sleep()` and does not block:
   - Event stream reader (reads from queue; no blocking I/O).
   - Other background jobs (each has its own asyncio task).
   - POST /agent/query response (spawned asynchronously; no await).

### AC-5: Failure Immediate (No Hold)

**On error from `process_question`:**

1. Result contains `"error"` key.
2. No hold calculation; no `asyncio.sleep()`.
3. Enqueue `failed` event immediately with error message.
4. Elapsed time from error detection to enqueue: < 1 millisecond.

**On exception in background task:**

1. Exception caught by try/except in `run_background_job()`.
2. No hold.
3. Enqueue `failed` event with error message = `str(exception)`.
4. Log (optional): stderr or application logger for debugging.

**Event behavior:**

- `failed` event sent immediately (no 5-second hold).
- No `completed` event sent (either-or, never both).
- Stream closes after `failed` (same as after `completed`).
- SSE subscriber receives event typically < 1 second after failure detection.

### Run Cleanup & TTL

**Startup:**

1. On app startup, `cleanup_task()` created via `asyncio.create_task()`.
2. Cleanup task runs indefinitely in background.

**Cleanup loop:**

1. Sleep for `RUN_CLEANUP_INTERVAL_SECONDS` (default 60 seconds).
2. Call `run_store.cleanup_expired(ttl_seconds=RUN_CLEANUP_TTL_SECONDS)`.
3. Cleanup logic: delete all runs where `(now - created_at) > ttl_seconds`.

**Effect:**

- Runs deleted approximately `ttl_seconds` after creation (default 1 hour).
- If SSE subscriber requests deleted run: `get()` returns `None`, handler raises 404.
- Configurable via environment variables:
  - `RUN_CLEANUP_TTL_SECONDS` (default 3600 = 1 hour)
  - `RUN_CLEANUP_INTERVAL_SECONDS` (default 60 seconds)

---

## Tests

### Unit Test File: `agent/tests/test_agent_sse_stream.py`

#### Test 1: AC-3 — GET /agent/runs/{known_id}/events returns HTTP 200 with text/event-stream

**Condition:** Known run_id  
**Verification:**
- Response status is 200.
- Response header `Content-Type` is `text/event-stream`.
- Response header `Cache-Control` is `no-cache`.
- Response header `Connection` is `keep-alive`.

#### Test 2: AC-3 — First event is in_progress with correct run_id

**Condition:** SSE stream for known run  
**Verification:**
- First event line is `event: in_progress`.
- First data line is `data: {"run_id": "<agent_run_id>"}` (exact match).
- `run_id` value matches POST response `agent_run_id`.

#### Test 3: AC-3 — All events include same run_id

**Condition:** Read all events from stream until closure  
**Verification:**
- Parse all events.
- Extract `run_id` from each event's data dict.
- Confirm all values match POST response `agent_run_id` (no changes across events).

#### Test 4: AC-3 — Unknown run_id returns HTTP 404

**Condition:** GET with non-existent or expired run_id  
**Verification:**
- Response status is 404.
- Response body contains `"detail": "Run not found"`.

#### Test 5: AC-3 — Multiple concurrent subscribers receive identical events in same order

**Condition:** Two SSE streams open simultaneously for same run_id  
**Verification:**
- Both streams open at same time (concurrent).
- Both streams read events from shared `run.events` queue.
- Event order is identical across both streams (first event is in_progress for both).
- Event content is identical (same `run_id`, same message text).
- No desync or ordering differences.
- Verify events drained from queue are removed (FIFO consumption per queue semantics).

#### Test 6: AC-4 — Fast answer (< 5s): hold until 5s, send completed at t ≥ 5s

**Condition:** Answer ready < 5s after in_progress  
**Verification:**
- Record timestamp of `in_progress` event receipt (`t_in_progress`).
- Record timestamp of `completed` event receipt (`t_completed`).
- Assert `(t_completed - t_in_progress) >= 4.9` (allow 100ms jitter).
- Assert `completed` event data contains `response.answer` field.

#### Test 7: AC-4 — Slow answer (≥ 5s): send completed immediately (no extra hold)

**Condition:** Answer ready ≥ 5s after in_progress  
**Verification:**
- Record timestamps `t_in_progress` and `t_completed`.
- Assert `(t_completed - t_in_progress) >= 5.9` and `<= 6.1` (tight range, no extra hold).

#### Test 8: AC-4 — Edge case: answer ready exactly at 5s

**Condition:** Answer ready exactly at 5.0s after in_progress  
**Verification:**
- Record timestamps.
- Assert `(t_completed - t_in_progress) >= 4.95` and `<= 5.05` (tight tolerance).

#### Test 9: AC-5 — LLM error: in_progress, then failed immediately (no hold, no completed)

**Condition:** `process_question` returns `{"error": "LLM timeout"}`  
**Verification:**
- First event is `in_progress`.
- Second event is `failed`.
- No `completed` event present.
- Elapsed time from `in_progress` to `failed` is < 1.0 second.
- `failed` event data contains `"error": "LLM timeout"`.

#### Test 10: AC-5 — MCP error: in_progress, then failed immediately (no hold, no completed)

**Condition:** MCP call raises exception  
**Verification:**
- First event is `in_progress`.
- Second event is `failed`.
- No `completed` event.
- Elapsed time < 1.0 second.
- `failed` event data contains error message (exception string).

#### Test 11: AC-5 — Invalid query rejection: SSE stream sends failed immediately (no completed)

**Condition:** Query rejected in POST /agent/query [AC-2]  
**Verification:**
- First event is `in_progress`.
- Second event is `failed`.
- No `completed` event.
- `failed` event data contains `"error": "invalid query"`.

#### Test 12: AC-4 — Stream closes after completed event

**Condition:** SSE stream reads until closure  
**Verification:**
- Read events until stream closes (no more data).
- Last event received is `completed`.
- Event count is exactly 2 (in_progress, completed).

#### Test 13: AC-5 — Stream closes after failed event

**Condition:** SSE stream reads until closure  
**Verification:**
- Read events until stream closes.
- Last event received is `failed`.
- Event count is exactly 2 (in_progress, failed).

#### Test 14: AC-3 — Subscriber disconnect does not cancel background job

**Condition:** Subscriber closes connection while background job running  
**Verification:**
- Open SSE stream, read first event (`in_progress`), close connection immediately.
- Wait 3 seconds (allow background job to complete).
- Open new SSE stream for same run_id.
- New subscriber receives all queued events (`in_progress`, then `completed` or `failed`).
- No events lost; background job continued despite subscriber disconnect.

#### Test 15: AC-3 — Late subscriber retrieves all events (no replay loss)

**Condition:** Background job completes before subscriber connects  
**Verification:**
- Submit POST /agent/query with valid query.
- Wait for background job to complete (receive `completed` or `failed` event timing + 1 second).
- Open SSE stream with GET /agent/runs/{agent_run_id}/events.
- Verify stream delivers all previously queued events in order (`in_progress`, `completed` or `failed`).
- No events dropped or lost; queue replay is complete.

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|--------|-----------|---------------|
| AC-3 | 1, 2, 3, 4, 5, 14, 15 | Stream setup, event delivery, run lookup, multi-subscriber | HTTP 200 with text/event-stream headers; first event in_progress with correct run_id; all events same run_id; 404 on unknown; multi-subscriber sync; no event loss on disconnect or late subscribe |
| AC-4 | 6, 7, 8, 12 | 5-second hold timing for completed | Fast answer held to ≥ 4.9s; slow answer immediate (no extra hold); edge case at 5s handled correctly; stream closes after completed |
| AC-5 | 9, 10, 11, 13 | Failure immediate, no hold, no completed | LLM error, MCP error, invalid query all send failed immediately (< 1s); no completed event sent; stream closes after failed |

**Total Test Count: 15**

---

## Notes

- **Event Queue (Run.events):** `asyncio.Queue` per run (in-memory, single-process). Events enqueued synchronously by background task before subscribers open; replay available to late subscribers. Queue is FIFO; all subscribers reading from same queue see events in identical order.
- **5-Second Hold:** Enforced in background task via `asyncio.sleep()` (non-blocking per-run); does not block event stream reader, other jobs, or POST response.
- **Timestamp Recording:** `in_progress_sent_at` recorded by Story 1 `run_store.create()` at UTC/epoch time; used by background task for hold calculation (Algorithm: `elapsed = now - in_progress_sent_at`; `hold_remaining = 5.0 - elapsed`).
- **Cleanup Policy:** Default 1-hour TTL; configurable via `RUN_CLEANUP_TTL_SECONDS` and `RUN_CLEANUP_INTERVAL_SECONDS` environment variables. Cleanup condition: `(now - created_at) > ttl_seconds` (strict inequality; run at exactly ttl_seconds is not deleted).
- **Event Format:** SSE spec compliant: `event: <type>\ndata: <json>\n\n` (two newlines after data).
- **Exception Handling:** Any unhandled exception in background task caught by try/except; converted to `failed` event with `error: str(exception)`.
- **PER-20 Dependency:** `process_question(query: str) → dict` signature is stable (AC-6). Expected return: `{"answer": "<text>"}` (string value) or `{"error": "<message>"}` (string value). Return type must be `dict`; data values must be JSON-serializable strings.
- **Process Isolation:** MCP calls remain in main agent process; no child processes forked by background task or SSE stream (only `asyncio.create_task()` used; no `Thread()`, `Process()`, or `ThreadPoolExecutor`).
- **Backward Compatibility:** Story 2 does not modify POST /query or GET /health; Story 3 verifies no regression.
- **Routes Location:** Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) in `agent/main.py` only. No separate routes module.
- **Global Run Store:** `run_store` is a singleton initialized in Story 1 `main.py` and imported by `background_task.py` and `sse_handler.py`. No per-call passing required; all modules share the same instance.
```
