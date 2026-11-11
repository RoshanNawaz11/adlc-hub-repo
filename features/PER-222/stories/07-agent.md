# Story 07-agent: Prevent operations on withdrawn cases — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1, Story 2, Story 4

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 2: After withdrawing — operations refused

**Given** a case has been withdrawn  
**When** a reviewer attempts to resume it via the API  
**Then** the API returns 409 Conflict with "Case was withdrawn"

---

**Given** a case has been withdrawn  
**When** a reviewer attempts to approve it via the API  
**Then** the API returns 409 Conflict with "Case was withdrawn"

---

**Given** a case has been withdrawn  
**When** a reviewer attempts to reject it via the API  
**Then** the API returns 409 Conflict with "Case was withdrawn"

---

**Given** a case has been withdrawn  
**When** a queued request to start the case again arrives  
**Then** the job fails with "Case was withdrawn"

---

**Given** a case has been withdrawn  
**When** a reviewer attempts to withdraw it a second time via the API  
**Then** the API returns 409 Conflict with "Case was withdrawn on <timestamp>: <reason>"

*Note: This AC is already covered by Story 1 (domain logic) and Story 2 (API endpoint), but is included here for completeness.*

---

## File Plan

### Modified Files

#### `src/agents/runtime/server.py`

**Changes:**

1. Modify the `resume_run` endpoint (starting around line 138) to check the case's status before resuming:

```python
@app.post("/runs/{thread_id}/resume", status_code=202)
async def resume_run(thread_id: str, decision: FlagAnswers | ReviewDecision) -> dict[str, Any]:
    if _runs.get(thread_id, {}).get("status") == "running":
        raise HTTPException(409, f"Run {thread_id} is running")
    
    # Check if the case was withdrawn (before checking if it's paused)
    snapshot = await _snapshot(thread_id)
    if not snapshot.values:
        raise HTTPException(404, f"Run {thread_id} not found")
    
    state = snapshot.values
    if isinstance(state, dict) and state.get("status") == "withdrawn":
        reason = state.get("withdrawal_reason", "")
        withdrawn_at = state.get("withdrawn_at", "")
        raise HTTPException(
            409,
            f"Case was withdrawn on {withdrawn_at}: {reason}" if reason else "Case was withdrawn"
        )
    
    # (existing pause check continues unchanged)
    if not snapshot.next:
        raise HTTPException(409, f"Run {thread_id} is not paused")
    # ... rest of existing code
```

**Implementation notes:**

- The withdrawn check happens after loading the snapshot (which we need anyway for the pause check), but before checking `snapshot.next`.
- The error message includes the withdrawal timestamp and reason when available, matching the pattern from Story 1's `InvalidStateError`.
- The 404 check for `snapshot.values` is added explicitly (it was implicit before — if `snapshot.next` is empty and values exist, the run is done; if values don't exist, the run doesn't exist).

---

**No changes to other API endpoints:**

- `create_run` (line 130): already checks if the thread_id is "running" or "paused" in `_runs`. A withdrawn case is neither, so it will proceed to call `_start`, which calls `run_query` with `resume=False`. The `run_query` function (in `runner.py`) guards against restarting a paused thread (line 225-230), but does not currently check for withdrawn status. We add that check below.

---

#### `src/agents/runtime/runner.py`

**Changes:**

1. Modify the `run_query` function to check for withdrawn status when `resume=False` (around line 225, after the idempotency check and before the mid-run guard):

```python
async def run_query(
    query: str = "",
    *,
    service_order_id: str | None = None,
    thread_id: str | None = None,
    resume: bool = False,
    decision: dict[str, Any] | None = None,
) -> dict[str, Any]:
    # ... (existing setup: thread_id, service_order fetch, session_id, config, key)
    
    async with open_graph(thread_id) as graph:
        if not resume and already_processed(key):
            # ... (existing idempotent replay logic)
        
        employee = None
        if resume:
            # ... (existing resume logic)
        else:
            # Guard against restarting a thread that's mid-run instead of
            # resuming it. A queue redelivery calling this without --resume
            # on a paused thread_id would otherwise silently re-invoke with
            # a fresh initial_state on top of existing checkpoint history —
            # and Worker's dispatch step calls non-idempotent live actions
            # (notify_on_call, create_ticket), so a silent restart risks
            # firing those twice.
            snapshot = await graph.aget_state(config)
            
            # Check if the case was withdrawn
            if snapshot.values and isinstance(snapshot.values, dict):
                if snapshot.values.get("status") == "withdrawn":
                    reason = snapshot.values.get("withdrawal_reason", "")
                    withdrawn_at = snapshot.values.get("withdrawn_at", "")
                    raise RuntimeError(
                        f"Thread {thread_id} was withdrawn on {withdrawn_at}: {reason}" if reason
                        else f"Thread {thread_id} was withdrawn — cannot start it again."
                    )
            
            # (existing mid-run check continues unchanged)
            if snapshot.next:
                raise RuntimeError(
                    f"Thread {thread_id} is paused mid-run (next={list(snapshot.next)}) — "
                    "use --resume, not a fresh query, or you will restart instead of continue."
                )
            
            # ... (rest of existing code: initial_state, report_progress, etc.)
```

**Implementation notes:**

- The withdrawn check happens after loading the snapshot (which we need anyway for the mid-run guard), but before the mid-run check.
- The error message matches the pattern from `InvalidStateError` in Story 1.
- Raises `RuntimeError` (same as the mid-run guard) because this is a usage error, not a domain validation failure.

---

#### `src/agents/queue/orders.py`

**Changes:**

1. Modify the `process_service_order` function to check for withdrawn status before calling `_run` (around line 88, before the existing `_run` call):

```python
async def process_service_order(ctx: dict, job: dict) -> dict[str, Any]:
    """Start a run. ``job`` is ``{"thread_id", "query"}``."""
    thread_id = job.get("thread_id")
    logger.info("[orders] start %s", thread_id)
    
    # Check if the case was already withdrawn (before starting)
    try:
        from agents.runtime.checkpointer import get_checkpointer
        from agents.runtime.run_dir import run_dir
        
        db_path = run_dir(thread_id) / "checkpoint.db"
        if not db_path.exists():
            # No checkpoint yet — not withdrawn, proceed normally
            return await _run(ctx, job, resume=False)
        
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
                # Case was withdrawn: refuse to start again
                reason = state.get("withdrawal_reason", "")
                withdrawn_at = state.get("withdrawn_at", "")
                error_msg = (
                    f"Case was withdrawn on {withdrawn_at}: {reason}" if reason
                    else "Case was withdrawn"
                )
                logger.warning(
                    "Attempt to start withdrawn case %s — refusing (%s)",
                    thread_id,
                    error_msg,
                )
                await write_status(ctx["redis"], thread_id, "failed", error=error_msg)
                return {"thread_id": thread_id, "status": "failed", "error": error_msg}
    except Exception as exc:
        # If we can't check the state (e.g. permission error), fall through
        # to the normal start path (which will fail appropriately in run_query)
        logger.warning("Could not check withdrawal status for %s: %s", thread_id, exc)
    
    # Case is not withdrawn (or we couldn't check): proceed with normal start
    return await _run(ctx, job, resume=False)
```

**Implementation notes:**

- The check mirrors the pattern from Story 4 (queue resume handler): load the checkpointer, check the status, close the connection, fall through on error.
- If the case is withdrawn, the job is marked `failed` (not `paused` or `done`) with an error message, and no graph invocation happens.
- The db_path existence check prevents creating an empty checkpoint directory for a thread_id that was never started (the withdrawn check only applies to cases that have a checkpoint).
- The try-except is defensive: if the checkpointer can't be opened (e.g. corrupted db, permission error), the function falls through to `_run`, which will call `run_query`, which will perform the same withdrawn check (added above) and raise `RuntimeError`.

---

**No changes to `resume_service_order`:**

Story 4 already added the withdrawn check to `resume_service_order` (around line 93). That check reads the state from the checkpointer before calling `_run`, and if `status == "withdrawn"`, it skips graph invocation and writes the queue status as `"withdrawn"` (not `"failed"`). The existing implementation from Story 4 handles resume operations on withdrawn cases; this story (Story 7) does not modify `resume_service_order`.

**Assumption:** The division of labor is: Story 4 handles resume operations (queue and API both delegate to the check Story 4 added to `resume_service_order` and the API endpoint), and Story 7 handles start/approve/reject operations. This is reflected in the file plan: Story 7 modifies `process_service_order` (start), `resume_run` (approve/reject via API), and `run_query` (start guard), but does not modify `resume_service_order` (already covered by Story 4).

---

### New Files

None. This story only modifies existing files.

## Interfaces

### Modified API endpoint: `POST /runs/{thread_id}/resume`

**Location:** `src/agents/runtime/server.py`

**Signature (unchanged):**

```python
@app.post("/runs/{thread_id}/resume", status_code=202)
async def resume_run(thread_id: str, decision: FlagAnswers | ReviewDecision) -> dict[str, Any]
```

**New error response (409 Conflict for withdrawn cases):**

```json
{
  "detail": "Case was withdrawn on 2026-10-14T10:00:00Z: Duplicate of SO-1003"
}
```

or (when reason/timestamp are missing):

```json
{
  "detail": "Case was withdrawn"
}
```

**Behavior change:**

- Before calling `snapshot.next` (to check if the case is paused), the endpoint checks `state.get("status")`.
- If `status == "withdrawn"`, raises `HTTPException(409, "Case was withdrawn on <timestamp>: <reason>")`.
- The withdrawn check happens before the pause check, so a withdrawn case returns 409 even if it has `snapshot.next` entries (which it shouldn't, but defensive).

---

### Modified function: `run_query` in `src/agents/runtime/runner.py`

**Location:** `src/agents/runtime/runner.py`

**Signature (unchanged):**

```python
async def run_query(
    query: str = "",
    *,
    service_order_id: str | None = None,
    thread_id: str | None = None,
    resume: bool = False,
    decision: dict[str, Any] | None = None,
) -> dict[str, Any]
```

**New exception (RuntimeError for withdrawn cases when `resume=False`):**

```python
raise RuntimeError(
    f"Thread {thread_id} was withdrawn on {withdrawn_at}: {reason}"
)
```

or (when reason/timestamp are missing):

```python
raise RuntimeError(f"Thread {thread_id} was withdrawn — cannot start it again.")
```

**Behavior change:**

- When `resume=False` (starting a new run), after loading the snapshot and before the mid-run guard, checks `snapshot.values.get("status")`.
- If `status == "withdrawn"`, raises `RuntimeError` with the withdrawal details.
- The withdrawn check happens before the mid-run check, so a withdrawn case raises even if it has `snapshot.next` entries.

---

### Modified function: `process_service_order` in `src/agents/queue/orders.py`

**Location:** `src/agents/queue/orders.py`

**Signature (unchanged):**

```python
async def process_service_order(ctx: dict, job: dict) -> dict[str, Any]
```

**New return value (for withdrawn cases):**

```python
{
    "thread_id": "<thread_id>",
    "status": "failed",
    "error": "Case was withdrawn on <timestamp>: <reason>"
}
```

**Side effect (for withdrawn cases):**

- Writes to Redis key `order-run:{thread_id}`: `{"status": "failed", "error": "Case was withdrawn on <timestamp>: <reason>"}` with TTL `STATUS_TTL_SECONDS`.

**Behavior change:**

- Before calling `_run`, checks if a checkpoint exists and loads the state.
- If `status == "withdrawn"`, skips `_run`, writes queue status as `failed`, and returns the error payload.
- The db_path existence check (`if not db_path.exists()`) prevents creating an empty checkpoint directory for a thread_id that was never started.

---

## Algorithm Notes

### Withdrawn check pattern (shared across all endpoints)

```
1. Load case state from checkpointer (snapshot = checkpointer.aget(config))
2. If snapshot.values is None or not a dict:
   - (For API/runner: raise 404 or proceed to normal error path)
   - (For queue: fall through to normal start path)
3. Extract status = state.get("status")
4. If status == "withdrawn":
   a. Extract reason = state.get("withdrawal_reason", "")
   b. Extract withdrawn_at = state.get("withdrawn_at", "")
   c. Construct error message:
      - If reason and withdrawn_at: "Case was withdrawn on {withdrawn_at}: {reason}"
      - Else: "Case was withdrawn"
   d. (For API: raise HTTPException(409, error_msg))
   e. (For runner: raise RuntimeError(error_msg))
   f. (For queue: write status("failed", error_msg), return error payload)
5. Else (status != "withdrawn"): proceed with normal operation
```

### Order of checks in `resume_run` (API)

```
1. Check if thread_id in _runs and status == "running" → 409 "is running"
2. Load snapshot from checkpointer
3. If snapshot.values is None → 404 "not found"
4. Check if status == "withdrawn" → 409 "was withdrawn on <timestamp>: <reason>"
5. Check if snapshot.next is empty → 409 "is not paused"
6. Check if paused on flags vs. review, validate decision type → 422 "send <expected>"
7. Check if all flags have answers (when paused on flags) → 422 "Answer every flag"
8. Proceed with resume
```

### Order of checks in `run_query` (when `resume=False`)

```
1. Check idempotency key → return cached result if already processed
2. Load snapshot from checkpointer
3. Check if status == "withdrawn" → RuntimeError "was withdrawn on <timestamp>: <reason>"
4. Check if snapshot.next is not empty → RuntimeError "is paused mid-run"
5. Proceed with normal start (initial_state, graph.astream, etc.)
```

### Order of checks in `process_service_order` (queue)

```
1. Check if db_path exists → if not, proceed with normal start (no checkpoint = not withdrawn)
2. Load snapshot from checkpointer
3. Check if status == "withdrawn" → write_status("failed", error), return error payload
4. Proceed with normal start (_run)
```

---

## Tests Section

**File:** `tests/test_withdrawal_operations.py` (new file)

### Integration tests (4 total)

All tests create a withdrawn case (via `withdraw_case` from Story 1 or by directly writing to the checkpointer), then attempt a mutating operation and verify the error response or failure.

**Setup helper (shared across tests):**

```python
from __future__ import annotations

import asyncio
import json
import tempfile
from pathlib import Path
from typing import Any
from unittest.mock import AsyncMock

import pytest

from agents.graph.states import State
from agents.runtime.checkpointer import get_checkpointer
from agents.runtime.withdrawal import withdraw_case

async def setup_withdrawn_case(thread_id: str, **extra_state: Any) -> Path:
    """
    Create a case in needs_review state, withdraw it, and return the test directory.
    
    Returns:
        The path to the run directory (for cleanup).
    """
    import tempfile
    test_dir = Path(tempfile.mkdtemp())
    db_path = test_dir / "checkpoint.db"
    
    # 1. Create initial case in needs_review state
    checkpointer = get_checkpointer("sqlite", db_path)
    await checkpointer.setup()
    
    state: State = {
        "schema_version": 1,
        "run_id": thread_id,
        "status": "needs_review",
        "flagged": False,
        "results": {},
        "steps": [],
        "flags": [],
        "needs": [],
        "runs": {},
        **extra_state,
    }
    
    config = {"configurable": {"thread_id": thread_id}}
    
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
    
    # 2. Withdraw the case
    await withdraw_case(
        thread_id=thread_id,
        reason="Test withdrawal",
        reviewer_id="test-reviewer",
        checkpointer_kind="sqlite",
        db_path=db_path,
    )
    
    return test_dir
```

---

**Test 1: Attempt to resume withdrawn case via API**

```python
@pytest.mark.asyncio
async def test_resume_withdrawn_case_via_api_returns_409() -> None:
    """AC 6: Resuming a withdrawn case via API returns 409 Conflict."""
    thread_id = "test-ops-1"
    
    # Setup: create and withdraw a case
    test_dir = await setup_withdrawn_case(thread_id)
    
    # Mock run_dir to return the test directory
    from agents.runtime import server
    import pytest
    from unittest.mock import patch
    
    with patch("agents.runtime.server.run_dir", return_value=test_dir):
        from fastapi.testclient import TestClient
        client = TestClient(server.app)
        
        # Attempt to resume with a review decision
        response = client.post(
            f"/runs/{thread_id}/resume",
            json={"action": "approve"}
        )
    
    # Assert: 409 Conflict with withdrawal details
    assert response.status_code == 409
    assert "was withdrawn" in response.json()["detail"]
    assert "Test withdrawal" in response.json()["detail"]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 2: Attempt to approve withdrawn case via API**

```python
@pytest.mark.asyncio
async def test_approve_withdrawn_case_via_api_returns_409() -> None:
    """AC 6: Approving a withdrawn case via API returns 409 Conflict."""
    thread_id = "test-ops-2"
    
    # Setup: create and withdraw a case
    test_dir = await setup_withdrawn_case(thread_id)
    
    from agents.runtime import server
    from unittest.mock import patch
    
    with patch("agents.runtime.server.run_dir", return_value=test_dir):
        from fastapi.testclient import TestClient
        client = TestClient(server.app)
        
        # Attempt to approve
        response = client.post(
            f"/runs/{thread_id}/resume",
            json={"action": "approve"}
        )
    
    # Assert: 409 Conflict
    assert response.status_code == 409
    assert "was withdrawn" in response.json()["detail"]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 3: Attempt to reject withdrawn case via API**

```python
@pytest.mark.asyncio
async def test_reject_withdrawn_case_via_api_returns_409() -> None:
    """AC 6: Rejecting a withdrawn case via API returns 409 Conflict."""
    thread_id = "test-ops-3"
    
    # Setup: create and withdraw a case
    test_dir = await setup_withdrawn_case(thread_id)
    
    from agents.runtime import server
    from unittest.mock import patch
    
    with patch("agents.runtime.server.run_dir", return_value=test_dir):
        from fastapi.testclient import TestClient
        client = TestClient(server.app)
        
        # Attempt to reject
        response = client.post(
            f"/runs/{thread_id}/resume",
            json={"action": "reject", "reason": "Test reject"}
        )
    
    # Assert: 409 Conflict
    assert response.status_code == 409
    assert "was withdrawn" in response.json()["detail"]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 4: Attempt to start withdrawn case again via queue**

```python
@pytest.mark.asyncio
async def test_start_withdrawn_case_via_queue_fails() -> None:
    """AC 6: Starting a withdrawn case again via queue fails with error."""
    thread_id = "test-ops-4"
    
    # Setup: create and withdraw a case
    test_dir = await setup_withdrawn_case(thread_id)
    
    # Mock run_dir to return the test directory
    from agents.queue import orders
    from unittest.mock import patch
    
    # Mock Redis
    class FakeRedis:
        def __init__(self) -> None:
            self.values: dict[str, str] = {}
        
        async def set(self, key: str, value: str, ex: int | None = None) -> None:
            self.values[key] = value
    
    redis = FakeRedis()
    ctx = {"redis": redis}
    
    with patch("agents.queue.orders.run_dir", return_value=test_dir):
        # Attempt to start the case again
        result = await orders.process_service_order(
            ctx,
            {"thread_id": thread_id, "query": "retry"}
        )
    
    # Assert: job failed with withdrawal error
    assert result["status"] == "failed"
    assert "was withdrawn" in result["error"]
    
    # Assert: queue status reflects failure
    status_value = redis.values.get(f"order-run:{thread_id}")
    assert status_value is not None
    status_payload = json.loads(status_value)
    assert status_payload["status"] == "failed"
    assert "was withdrawn" in status_payload["error"]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test count alignment with breakdown:** 4 tests (1 per mutating operation: resume, approve, reject, start again).

**Coverage:**

- AC 2 (after withdrawing: operations refused): All 4 tests
- AC 6 (a withdrawn case can't be resumed, approved, rejected, or started again): All 4 tests

---

## Open Questions (carried forward from prior stories)

1. **Withdrawn check in `create_run` API endpoint:** The current implementation does not add a withdrawn check to `create_run` because it delegates to `run_query`, which performs the check when `resume=False`. However, `create_run` checks if the thread_id is "running" or "paused" in the in-memory `_runs` dict before calling `run_query`. A withdrawn case that has never been started by this server instance will not be in `_runs`, so it will proceed to `run_query`, which will raise `RuntimeError`. Should `create_run` also load the snapshot and check for withdrawn status explicitly (for a clearer 409 response instead of a 500 from the `RuntimeError`)? The current approach is acceptable (the withdrawn check happens in `run_query` and surfaces as a 500 or logged error), but a 409 is more semantically correct.

2. **Queue handler error semantics:** The queue handler (`process_service_order`) marks a withdrawn case as `status: "failed"` in Redis. An alternative is to introduce a new queue status (e.g. `"withdrawn"`) instead of reusing `"failed"`. This would align with the domain model (withdrawn is not a failure, it's a deliberate stop) and match the API/CLI semantics. However, it requires coordination with the FastAPI backend's `order_queue.py` client (which polls the Redis status). The current approach (`"failed"`) is acceptable and consistent with other usage errors (e.g. resuming a non-paused case), but the alternative is worth noting for the consistency check.

3. **Mid-run guard vs. withdrawn guard order:** In `run_query`, the withdrawn check happens before the mid-run guard (paused case check). This means a case that is both withdrawn and paused (which shouldn't happen in practice, but is possible if the withdrawal logic didn't clear `snapshot.next`) will raise the withdrawn error first. This is the correct priority (withdrawn is more terminal than paused), but worth documenting.

---

**End of LLD.**
