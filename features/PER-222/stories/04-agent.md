# Story 04-agent: Queue handler integration for resume no-op — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 4: Everywhere a case is handled — queue resume handling

**Given** a case has been withdrawn  
**When** a queued request to resume the case arrives  
**Then** nothing resumes, and the case's queue status shows it as withdrawn

### AC 2: After withdrawing — no resume

**Given** a case has been withdrawn  
**When** a resume job is dequeued  
**Then** the graph is not invoked, queue status is written with `status: "withdrawn"`, the job is marked successful, and the no-op is logged

### AC 13: General behavior unchanged

**Given** a case is never withdrawn  
**When** a resume job is dequeued  
**Then** the graph is invoked as usual (existing behavior)

---

## File Plan

### Modified Files

#### `src/agents/queue/orders.py`

**Changes:**

1. Add imports at the top (after existing imports, around line 18):

```python
import json
from agents.runtime.checkpointer import get_checkpointer
from agents.runtime.run_dir import run_dir
```

2. Modify the `resume_service_order` function (starting around line 93):

**Current implementation (lines 93-98):**

```python
async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]:
    """Continue a paused run. ``job`` is ``{"thread_id", "decision"}``: the
    analyst's answers to every flag (``{"answers": [...]}``), or at the final
    review the approval or rejection (runner.run_query)."""
    logger.info("[orders] resume %s", job.get("thread_id"))
    return await _run(ctx, job, resume=True)
```

**New implementation:**

```python
async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]:
    """Continue a paused run. ``job`` is ``{"thread_id", "decision"}``: the
    analyst's answers to every flag (``{"answers": [...]}``), or at the final
    review the approval or rejection (runner.run_query).
    
    If the case was withdrawn, skips graph invocation and writes queue status."""
    thread_id = job.get("thread_id")
    logger.info("[orders] resume %s", thread_id)
    
    # Check if the case was withdrawn (before running the graph)
    try:
        db_path = run_dir(thread_id) / "checkpoint.db"
        checkpointer = get_checkpointer("sqlite", db_path)
        await checkpointer.setup()
        
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await checkpointer.aget(config)
        
        # Close checkpointer connection immediately (avoid hanging)
        conn = getattr(checkpointer, "conn", None)
        if conn is not None:
            await conn.close()
        
        if snapshot and snapshot.values:
            state = snapshot.values
            if state.get("status") == "withdrawn":
                # Case was withdrawn: skip graph invocation, write queue status
                reason = state.get("withdrawal_reason", "")
                withdrawn_at = state.get("withdrawn_at", "")
                logger.info(
                    "Case %s already withdrawn, resume ignored (reason: %s)",
                    thread_id,
                    reason,
                )
                
                # Write queue status with withdrawal details
                payload = {
                    "status": "withdrawn",
                    "reason": reason,
                    "withdrawn_at": withdrawn_at,
                }
                await ctx["redis"].set(
                    status_key(thread_id),
                    json.dumps(payload),
                    ex=STATUS_TTL_SECONDS,
                )
                
                return {
                    "thread_id": thread_id,
                    "status": "withdrawn",
                    "reason": reason,
                    "withdrawn_at": withdrawn_at,
                }
    except Exception as exc:
        # If we can't check the state (e.g. checkpointer doesn't exist yet),
        # fall through to the normal resume path (which will fail appropriately)
        logger.warning("Could not check withdrawal status for %s: %s", thread_id, exc)
    
    # Case is not withdrawn (or we couldn't check): proceed with normal resume
    return await _run(ctx, job, resume=True)
```

**Implementation notes:**

- **Status check placement:** The check happens before `_run` is called, so the graph is never invoked for a withdrawn case.
- **Checkpointer interaction:** The resume handler doesn't have direct access to the graph (it's built inside `_run`), so it must open the checkpointer directly to read the state. This mirrors the pattern in `server.py._snapshot` but with sqlite-specific setup.
- **Queue status write:** Inlines the Redis write instead of calling `write_status` to include `reason` and `withdrawn_at` fields. The existing `write_status` signature only accepts `status` and `error`, and extending it would require updating all call sites and the contract with FastAPI backend's `app/clients/order_queue.py`.
- **Error vs. status:** A withdrawn case is not a failure — it's a valid terminal state. The queue status payload reflects this: `"status": "withdrawn"` (not `"status": "failed"`).

---

**No other files modified.** Story 1 created `src/agents/runtime/withdrawal.py` and modified `src/agents/graph/states.py`; this story only reads the state fields they introduced.

---

## Interfaces

### `resume_service_order` function

**Location:** `src/agents/queue/orders.py`

**Signature (unchanged):**

```python
async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]
```

**Parameters:**

- `ctx` (dict): The arq worker context, containing `{"redis": <aioredis client>}`.
- `job` (dict): The job payload, containing `{"thread_id": str, "decision": dict | None}`.

**Returns:**

```python
{
    "thread_id": str,
    "status": "withdrawn" | "paused" | "done" | "failed",
    "reason": str | None,           # present only when status == "withdrawn"
    "withdrawn_at": str | None,     # present only when status == "withdrawn"
    "error": str | None,            # present only when status == "failed"
    # ... other fields from _run when status != "withdrawn"
}
```

**Side effects (new behavior):**

- When the case's `status == "withdrawn"`:
  1. Logs: `"Case <thread_id> already withdrawn, resume ignored (reason: <reason>)"` at `INFO` level.
  2. Writes to Redis key `order-run:{thread_id}`: `{"status": "withdrawn", "reason": "<reason>", "withdrawn_at": "<timestamp>"}` with TTL `STATUS_TTL_SECONDS` (7 days).
  3. Returns immediately without calling `_run` (graph is not invoked).
- When the case's `status != "withdrawn"` (or the state check fails):
  - Calls `_run(ctx, job, resume=True)` as before (existing behavior unchanged).

**Error handling:**

- If opening the checkpointer raises an exception (e.g. `db_path` doesn't exist, permission error), the exception is logged at `WARNING` level and the function falls through to the normal resume path. This means a queued resume for a non-existent thread_id will still fail in `_run` as it does today (with appropriate error), rather than silently succeeding.

---

### Queue status payload contract

**Redis key:** `order-run:{thread_id}` (unchanged)

**Payload shape (extended):**

**Existing statuses:**

```json
{"status": "queued"}                          // written by FastAPI when enqueuing
{"status": "running"}                         // written by process_service_order and resume_service_order (_run)
{"status": "paused"}                          // written by _run (IncompleteAnswers caught)
{"status": "done"}                            // written by _run (success)
{"status": "failed", "error": "<message>"}    // written by _run (exception caught)
```

**New status (this story):**

```json
{"status": "withdrawn", "reason": "<trimmed reason>", "withdrawn_at": "<ISO 8601 timestamp>"}
```

**Consumed by:** FastAPI backend's `app/clients/order_queue.py` (per the module docstring), which polls this key to track job progress. The new `withdrawn` status must be handled there (show as a terminal status, don't retry, display reason and timestamp).

---

## Algorithm Notes

### Resume job flow (with withdrawal check)

```
1. Dequeue resume_service_order job from Redis queue
2. Extract thread_id from job payload
3. Log "[orders] resume <thread_id>"
4. Try to load case state from checkpointer:
   a. Resolve db_path = run_dir(thread_id) / "checkpoint.db"
   b. Get checkpointer = get_checkpointer("sqlite", db_path)
   c. Setup checkpointer (idempotent, creates tables if missing)
   d. Load snapshot = checkpointer.aget({"configurable": {"thread_id": thread_id}})
   e. Close checkpointer connection (avoid process hang)
   f. If snapshot exists and snapshot.values is not None:
      i. Extract state = snapshot.values
      ii. If state["status"] == "withdrawn":
          - Log "Case <thread_id> already withdrawn, resume ignored (reason: <reason>)"
          - Write queue status to Redis: {"status": "withdrawn", "reason": "<reason>", "withdrawn_at": "<timestamp>"}
          - Return {"thread_id": thread_id, "status": "withdrawn", "reason": <reason>, "withdrawn_at": <timestamp>}
   g. If any exception during checkpointer load:
      - Log warning "Could not check withdrawal status for <thread_id>: <exception>"
      - Fall through to step 5 (normal resume)
5. Call _run(ctx, job, resume=True) (existing resume logic)
6. Return _run's result
```

### Why the checkpointer check is try-except

The resume handler is called by arq for every job dequeued from the `service_orders` queue. A job can be enqueued for a thread_id that:

- Exists and is paused (normal resume).
- Exists and was withdrawn (the case this story handles).
- Doesn't exist yet (the job was enqueued before the initial run started, or the run_dir was deleted). In this case, `run_dir(thread_id)` will create the directory, but `db_path / checkpoint.db` won't exist, and `checkpointer.aget` will return `None` (or raise an exception, depending on the sqlite driver).
- Exists but the checkpointer is corrupted or locked.

The try-except ensures that the withdrawal check is best-effort: if we can't load the state, we fall through to the normal resume path, which will handle the error appropriately (e.g. `run_query` will raise an error, which `_run` catches and writes to queue status as `"failed"`).

---

### Checkpointer lifecycle in the resume handler

The existing `_run` function (lines 40-77) calls `run_query`, which opens the graph (including the checkpointer) inside `open_graph` (an async context manager). The checkpointer is closed when the context exits.

For the withdrawal check, we need to read the state before calling `_run`, so we must open the checkpointer directly. The pattern:

```python
checkpointer = get_checkpointer("sqlite", db_path)
await checkpointer.setup()  # idempotent: creates tables if missing
snapshot = await checkpointer.aget(config)
# Use snapshot.values
conn = getattr(checkpointer, "conn", None)
if conn is not None:
    await conn.close()  # required: otherwise the process hangs
```

This mirrors the cleanup in `checkpointer.close_checkpointer` (which `open_graph` calls on exit) and the pattern in Story 1's `withdraw_case` function.

---

## Tests Section

**File:** `tests/test_withdrawal_queue.py` (new file)

### Integration tests (2 total)

Both tests mock the queue worker context and job payload. The tests create a case in a known state by directly writing to a sqlite checkpointer, invoke `resume_service_order`, and assert on the return value, queue status, and that `run_query` was (or wasn't) called.

**Setup helper (shared across tests):**

```python
import asyncio
import json
import tempfile
from pathlib import Path
from typing import Any

import pytest

from agents.graph.states import State
from agents.queue.orders import resume_service_order, status_key, STATUS_TTL_SECONDS
from agents.runtime.checkpointer import get_checkpointer

# Mock Redis client (arq uses aioredis or redis-py; mock the interface we need)
class FakeRedis:
    def __init__(self) -> None:
        self.data: dict[str, tuple[str, int | None]] = {}  # key -> (value, expiry)
    
    async def set(self, key: str, value: str, ex: int | None = None) -> None:
        self.data[key] = (value, ex)
    
    async def get(self, key: str) -> str | None:
        entry = self.data.get(key)
        return entry[0] if entry else None


async def setup_case(thread_id: str, status: str, **extra_state: Any) -> Path:
    """
    Write a case with the given status to the checkpointer.
    
    Returns:
        The path to the run directory (for cleanup).
    """
    # Use a temp directory for the run (avoids polluting logs/ during tests)
    test_dir = Path(tempfile.mkdtemp())
    db_path = test_dir / "checkpoint.db"
    
    checkpointer = get_checkpointer("sqlite", db_path)
    await checkpointer.setup()
    
    state: State = {
        "schema_version": 1,
        "run_id": thread_id,
        "status": status,
        "flagged": False,
        "results": {},
        "steps": [],
        "flags": [],
        "needs": [],
        "runs": {},
        **extra_state,
    }
    
    config = {"configurable": {"thread_id": thread_id}}
    
    # Create a minimal checkpoint (sketch; may need adjustment for real LangGraph schema)
    from langgraph.checkpoint.base import Checkpoint
    import uuid
    checkpoint = Checkpoint(
        v=1,
        id=str(uuid.uuid4()),
        ts="2026-10-15T14:00:00Z",
        channel_values={"state": state},
        channel_versions={},
        versions_seen={},
        pending_sends=[],
    )
    await checkpointer.aput(config, checkpoint, state, {})
    
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    return test_dir
```

---

**Test 1: Resume job for withdrawn case (no-op)**

```python
@pytest.mark.asyncio
async def test_resume_withdrawn_case_is_noop(monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 11, AC 2: A queued resume for a withdrawn case skips graph invocation and writes queue status."""
    thread_id = "test-queue-1"
    
    # Setup: create a withdrawn case
    test_dir = await setup_case(
        thread_id,
        "withdrawn",
        withdrawn_at="2026-10-14T10:00:00Z",
        withdrawn_by="reviewer-1",
        withdrawal_reason="Duplicate of SO-1003",
    )
    
    # Mock run_dir to return the test directory
    monkeypatch.setattr("agents.queue.orders.run_dir", lambda tid, so=None: test_dir)
    
    # Mock Redis
    redis = FakeRedis()
    ctx = {"redis": redis}
    
    # Job payload (resume with no decision, as if from a queue redelivery)
    job = {"thread_id": thread_id, "decision": None}
    
    # Mock run_query to detect if it's called (it shouldn't be)
    run_query_called = False
    
    async def mock_run_query(*args, **kwargs):
        nonlocal run_query_called
        run_query_called = True
        return {"thread_id": thread_id, "status": "done"}
    
    monkeypatch.setattr("agents.queue.orders.run_query", mock_run_query)
    
    # Invoke resume_service_order
    result = await resume_service_order(ctx, job)
    
    # Assert: run_query was NOT called (graph not invoked)
    assert not run_query_called, "run_query should not be called for a withdrawn case"
    
    # Assert: result shows withdrawn
    assert result["status"] == "withdrawn"
    assert result["reason"] == "Duplicate of SO-1003"
    assert result["withdrawn_at"] == "2026-10-14T10:00:00Z"
    
    # Assert: queue status was written
    status_value = await redis.get(status_key(thread_id))
    assert status_value is not None
    status_payload = json.loads(status_value)
    assert status_payload["status"] == "withdrawn"
    assert status_payload["reason"] == "Duplicate of SO-1003"
    assert status_payload["withdrawn_at"] == "2026-10-14T10:00:00Z"
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 2: Resume job for non-withdrawn case (control)**

```python
@pytest.mark.asyncio
async def test_resume_non_withdrawn_case_proceeds_normally(monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 13: A queued resume for a non-withdrawn case invokes the graph as usual."""
    thread_id = "test-queue-2"
    
    # Setup: create a paused case (waiting for review)
    test_dir = await setup_case(
        thread_id,
        "needs_review",
    )
    
    # Mock run_dir to return the test directory
    monkeypatch.setattr("agents.queue.orders.run_dir", lambda tid, so=None: test_dir)
    
    # Mock Redis
    redis = FakeRedis()
    ctx = {"redis": redis}
    
    # Job payload (resume with approval decision)
    job = {"thread_id": thread_id, "decision": {"action": "approve"}}
    
    # Mock run_query to return a success result
    run_query_called = False
    
    async def mock_run_query(*args, **kwargs):
        nonlocal run_query_called
        run_query_called = True
        return {"thread_id": thread_id, "status": "done", "paused": False}
    
    monkeypatch.setattr("agents.queue.orders.run_query", mock_run_query)
    
    # Invoke resume_service_order
    result = await resume_service_order(ctx, job)
    
    # Assert: run_query WAS called (graph invoked)
    assert run_query_called, "run_query should be called for a non-withdrawn case"
    
    # Assert: result shows done (from mock_run_query)
    assert result["status"] == "done"
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

**Note:** The existing `tests/test_order_queue.py` already has tests for `resume_service_order` (e.g. `test_resume_records_done`). This test verifies that the withdrawal check doesn't break the normal resume path.

---

**Test count alignment with breakdown:** 2 tests (1 withdrawn case, 1 control case).

---

## Open Questions (carried forward from Story 1 / HLD)

1. **Queue status payload contract:** The `write_status` function currently accepts `(redis, thread_id, status, error=None)`. This story inlines the Redis write for the withdrawn case to include `reason` and `withdrawn_at` fields. Should `write_status` be extended to accept arbitrary payload fields (`**extra`), or is the inline approach preferred? The consistency check (Story 7) should verify that the FastAPI backend's `order_queue.py` can parse the new `withdrawn` status payload.

2. **Checkpointer error handling:** If the checkpointer can't be opened (e.g. permission error, corrupted db), the try-except falls through to the normal resume path. Should this log at `ERROR` level instead of `WARNING`, or raise a specific exception? The current approach (log warning, fall through) matches the "best-effort" principle from Story 5 (console notification).

3. **run_dir behavior:** The `run_dir` function (from `agents.runtime.run_dir`) creates the directory if it doesn't exist. For a resume job on a thread_id that was never started, this means an empty directory is created during the withdrawal check, then `checkpointer.aget` returns `None`, and the try-except falls through. Is this acceptable, or should the withdrawal check skip the `run_dir` call and handle `FileNotFoundError` on `db_path` explicitly?

---

**End of LLD.**
