# Story 06-agent: Persistence across restarts — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 2: After withdrawing — persistence across restarts

**Given** a case has been withdrawn  
**When** the service restarts  
**Then** the case still shows as `withdrawn`, with its reason

**Example 4 (from requirements):**  
After a restart, Ben opens a case withdrawn the day before. It still shows as withdrawn, with its reason.

---

## File Plan

### New Files

#### `tests/test_withdrawal_persistence.py`

**Purpose:** Integration test verifying that withdrawn state persists across checkpointer close/reopen cycles (simulating a service restart).

**Imports:**

```python
from __future__ import annotations

import asyncio
import tempfile
from pathlib import Path

import pytest

from agents.graph.states import State
from agents.runtime.checkpointer import get_checkpointer
from agents.runtime.withdrawal import withdraw_case
```

**Test implementation:**

```python
@pytest.mark.asyncio
async def test_withdrawn_case_persists_across_restart() -> None:
    """AC 7: A withdrawn case still shows as withdrawn after restart, with reason and timestamp."""
    thread_id = "test-persist-1"
    
    # Setup: create a temporary db for this test (avoids polluting logs/)
    test_dir = Path(tempfile.mkdtemp())
    db_path = test_dir / "checkpoint.db"
    
    # 1. Create an initial case in needs_review state
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
    
    # Close the first checkpointer
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    # 2. Withdraw the case
    result = await withdraw_case(
        thread_id=thread_id,
        reason="Duplicate of SO-1002",
        reviewer_id="reviewer-1",
        checkpointer_kind="sqlite",
        db_path=db_path,
    )
    
    # Verify withdrawal succeeded
    assert result["status"] == "withdrawn"
    assert result["reason"] == "Duplicate of SO-1002"
    withdrawn_at = result["withdrawn_at"]
    assert withdrawn_at is not None
    
    # 3. Simulate a restart: close and re-open the checkpointer
    # (withdraw_case closes it already, so just re-open)
    checkpointer2 = get_checkpointer("sqlite", db_path)
    await checkpointer2.setup()
    
    snapshot = await checkpointer2.aget(config)
    
    # Close second checkpointer
    conn2 = getattr(checkpointer2, "conn", None)
    if conn2 is not None:
        await conn2.close()
    
    # 4. Verify the withdrawn state persisted
    assert snapshot is not None
    assert snapshot.values is not None
    
    state_after_restart: State = snapshot.values  # type: ignore[assignment]
    
    assert state_after_restart["status"] == "withdrawn"
    assert state_after_restart.get("withdrawal_reason") == "Duplicate of SO-1002"
    assert state_after_restart.get("withdrawn_at") == withdrawn_at
    assert state_after_restart.get("withdrawn_by") == "reviewer-1"
    
    # 5. Verify the history entry persisted
    steps = state_after_restart.get("steps", [])
    assert len(steps) > 0
    
    # The last step should be the withdrawal history entry
    last_step = steps[-1]
    assert last_step.get("type") == "withdrawn"  # type: ignore[typeddict-item]
    assert last_step.get("by") == "reviewer-1"  # type: ignore[typeddict-item]
    assert last_step.get("reason") == "Duplicate of SO-1002"  # type: ignore[typeddict-item]
    assert last_step.get("at") == withdrawn_at  # type: ignore[typeddict-item]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

**Implementation notes:**

- **Test structure:** The test creates a case, withdraws it, closes and re-opens the checkpointer, then verifies all withdrawal fields persisted.
- **Checkpoint setup:** Uses a minimal checkpoint structure (same pattern as other test files). The exact shape may need adjustment based on LangGraph's internal schema, but the core pattern (write state, close checkpointer, re-open, read state) is valid.
- **Cleanup:** Removes the temporary test directory after the test completes.
- **Assertions:** Verifies all four withdrawal fields (`status`, `withdrawal_reason`, `withdrawn_at`, `withdrawn_by`) and the history entry.

---

### Modified Files

None. This story only creates a new test file; it imports and uses `withdraw_case` from Story 1 and the checkpointer utilities from `src/agents/runtime/checkpointer.py`, but does not modify them.

---

## Interfaces

### Test dependencies

**From Story 1:**

- `withdraw_case` function (`src/agents/runtime/withdrawal.py`)
- `State` TypedDict with withdrawal fields (`src/agents/graph/states.py`)

**From existing codebase:**

- `get_checkpointer` factory (`src/agents/runtime/checkpointer.py`)
- `AsyncSqliteSaver` (from `langgraph.checkpoint.sqlite.aio`, via `get_checkpointer`)
- `Checkpoint` TypedDict (from `langgraph.checkpoint.base`)

**Test pattern:**

1. Create a checkpointer with a temporary db file.
2. Write an initial state to the checkpointer.
3. Close the checkpointer connection.
4. Call `withdraw_case` (which opens, modifies, and closes its own checkpointer connection).
5. Re-open the checkpointer (simulating a restart).
6. Read the state back and verify it matches the withdrawn state.
7. Close the second checkpointer and clean up the temp directory.

---

## Algorithm Notes

### Simulating a restart

A real service restart would:

1. The process exits, closing all open file handles (including the sqlite connection).
2. The process restarts, loading the code but not the in-memory state.
3. When a request arrives, the code opens the checkpointer again and reads the latest state from disk.

The test simulates this by:

1. Closing the checkpointer connection after the initial state write.
2. Calling `withdraw_case` (which opens its own checkpointer connection, writes the withdrawal, and closes it).
3. Opening a new checkpointer connection (with the same db file path) and reading the state.

**Key invariant:** The sqlite checkpointer persists state to disk on `aput`, so closing and re-opening the connection reads the same state back (as long as the db file is not deleted).

---

### Checkpointer lifecycle in the test

```python
# 1. First checkpointer: write initial state
checkpointer = get_checkpointer("sqlite", db_path)
await checkpointer.setup()
await checkpointer.aput(config, checkpoint, state, {})
conn = getattr(checkpointer, "conn", None)
if conn is not None:
    await conn.close()

# 2. withdraw_case opens and closes its own checkpointer internally
await withdraw_case(..., checkpointer_kind="sqlite", db_path=db_path)

# 3. Second checkpointer: read withdrawn state
checkpointer2 = get_checkpointer("sqlite", db_path)
await checkpointer2.setup()
snapshot = await checkpointer2.aget(config)
conn2 = getattr(checkpointer2, "conn", None)
if conn2 is not None:
    await conn2.close()
```

**Why `setup()` twice:** `AsyncSqliteSaver.setup()` is idempotent — it creates the sqlite tables if they don't exist, or does nothing if they already exist. Calling it before the first `aput` and before the second `aget` ensures the db is ready, and mirrors the pattern in `withdraw_case` and other parts of the codebase.

---

### Minimal checkpoint structure

The test creates a minimal `Checkpoint` object to write the initial state:

```python
checkpoint = Checkpoint(
    v=1,
    id=str(uuid.uuid4()),
    ts="2026-10-15T14:00:00Z",
    channel_values={"state": state},
    channel_versions={},
    versions_seen={},
    pending_sends=[],
)
```

**Note:** The exact shape of a valid checkpoint depends on LangGraph's internal schema (which may evolve). If the test fails with a schema error, adjust the checkpoint structure to match the current LangGraph version. The core pattern (write state, close, re-open, read state) remains valid regardless of the checkpoint shape.

---

## Tests Section

**File:** `tests/test_withdrawal_persistence.py` (new file)

### Integration test (1 total)

**Test: Withdrawn case persists across restart**

```python
@pytest.mark.asyncio
async def test_withdrawn_case_persists_across_restart() -> None:
    """AC 7: A withdrawn case still shows as withdrawn after restart, with reason and timestamp."""
    thread_id = "test-persist-1"
    
    # Setup: create a temporary db for this test (avoids polluting logs/)
    test_dir = Path(tempfile.mkdtemp())
    db_path = test_dir / "checkpoint.db"
    
    # 1. Create an initial case in needs_review state
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
    
    # Close the first checkpointer
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    # 2. Withdraw the case
    result = await withdraw_case(
        thread_id=thread_id,
        reason="Duplicate of SO-1002",
        reviewer_id="reviewer-1",
        checkpointer_kind="sqlite",
        db_path=db_path,
    )
    
    # Verify withdrawal succeeded
    assert result["status"] == "withdrawn"
    assert result["reason"] == "Duplicate of SO-1002"
    withdrawn_at = result["withdrawn_at"]
    assert withdrawn_at is not None
    
    # 3. Simulate a restart: close and re-open the checkpointer
    # (withdraw_case closes it already, so just re-open)
    checkpointer2 = get_checkpointer("sqlite", db_path)
    await checkpointer2.setup()
    
    snapshot = await checkpointer2.aget(config)
    
    # Close second checkpointer
    conn2 = getattr(checkpointer2, "conn", None)
    if conn2 is not None:
        await conn2.close()
    
    # 4. Verify the withdrawn state persisted
    assert snapshot is not None
    assert snapshot.values is not None
    
    state_after_restart: State = snapshot.values  # type: ignore[assignment]
    
    assert state_after_restart["status"] == "withdrawn"
    assert state_after_restart.get("withdrawal_reason") == "Duplicate of SO-1002"
    assert state_after_restart.get("withdrawn_at") == withdrawn_at
    assert state_after_restart.get("withdrawn_by") == "reviewer-1"
    
    # 5. Verify the history entry persisted
    steps = state_after_restart.get("steps", [])
    assert len(steps) > 0
    
    # The last step should be the withdrawal history entry
    last_step = steps[-1]
    assert last_step.get("type") == "withdrawn"  # type: ignore[typeddict-item]
    assert last_step.get("by") == "reviewer-1"  # type: ignore[typeddict-item]
    assert last_step.get("reason") == "Duplicate of SO-1002"  # type: ignore[typeddict-item]
    assert last_step.get("at") == withdrawn_at  # type: ignore[typeddict-item]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

**Coverage:**

- AC 2 (after withdrawing: persistence across restarts)
- Example 4 from requirements (case withdrawn the day before still shows as withdrawn after restart)

**Assertions:**

1. `state["status"] == "withdrawn"` (status field persisted)
2. `state["withdrawal_reason"] == "Duplicate of SO-1002"` (reason field persisted)
3. `state["withdrawn_at"] == <original timestamp>` (timestamp field persisted)
4. `state["withdrawn_by"] == "reviewer-1"` (reviewer field persisted)
5. Last history entry is a `withdrawn` entry with correct fields (history persisted)

**Test count alignment with breakdown:** 1 test.

---

## Open Questions (carried forward from Story 1 / HLD)

1. **Checkpoint schema evolution:** The test uses a minimal `Checkpoint` structure based on the current LangGraph version. If LangGraph's checkpoint schema changes, the test may need adjustment. Consider adding a helper function (e.g. `create_minimal_checkpoint(state)`) that encapsulates the checkpoint creation pattern, so schema changes can be handled in one place.

2. **Test database cleanup:** The test creates a temporary directory (`tempfile.mkdtemp()`) and deletes it at the end. If the test fails before cleanup, the directory lingers. Consider using `pytest`'s `tmp_path` fixture instead of `tempfile.mkdtemp()` for automatic cleanup.

3. **Checkpointer connection leaks:** The test manually closes checkpointer connections with `conn.close()`. If an assertion fails before the close, the connection leaks (and may prevent the temp directory from being deleted on Windows). Consider using an async context manager pattern (`async with` over a helper that yields the checkpointer and closes it on exit) for safer cleanup.

---

**End of LLD.**
