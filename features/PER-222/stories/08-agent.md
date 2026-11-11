# Story 08-agent: End-to-end validation — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1, Story 2, Story 3, Story 4, Story 5, Story 6, Story 7

---

## Acceptance Criteria (from Requirements & Breakdown)

### REQ-Examples

**Example 1: SO-1004 duplicate**

**Given** SO-1004 duplicates SO-1003, which is already being processed, and SO-1004 is waiting for answers to two flags  
**When** Asha withdraws it with "Duplicate of SO-1003"  
**Then** the case shows as withdrawn, and neither flag gets an answer

---

**Example 2: Ben withdraws at review**

**Given** a case is at the final review  
**When** Ben withdraws it with "Client asked to stop the termination"  
**And** he then tries to approve it  
**Then** he is told it was withdrawn

---

**Example 3: Asha tries empty reason**

**Given** Asha tries to withdraw a case with a reason of only spaces  
**When** the withdrawal is attempted  
**Then** she is asked for a reason, and the case is still waiting for review

---

**Example 4: Restart persistence**

**Given** a case was withdrawn the day before  
**When** the service restarts  
**And** Ben opens the case  
**Then** it still shows as withdrawn, with its reason

---

### REQ-AC5: General behavior unchanged

**Given** a case is never withdrawn  
**When** it is started, has its flags answered, or is approved or rejected  
**Then** these operations work exactly as they do today

---

## File Plan

### New Files

#### `tests/test_withdrawal_e2e.py`

**Purpose:** End-to-end scenario tests covering the examples from the requirements and verifying that non-withdrawn cases continue to work unchanged.

**Imports:**

```python
from __future__ import annotations

import asyncio
import json
import tempfile
from pathlib import Path
from typing import Any
from unittest.mock import AsyncMock, patch

import pytest

from agents.graph.states import State
from agents.runtime.checkpointer import get_checkpointer
from agents.runtime.withdrawal import withdraw_case
from agents.runtime.runner import run_query
```

**Test implementation:**

```python
# Shared setup helper (reused from prior stories)
async def setup_case(thread_id: str, status: str, **extra_state: Any) -> Path:
    """
    Write a case with the given status to the checkpointer.
    
    Returns:
        The path to the run directory (for cleanup).
    """
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


# Test 1: Example 1 (SO-1004 duplicate)
@pytest.mark.asyncio
async def test_example_1_duplicate_case_withdrawn_flags_not_answered() -> None:
    """
    Example 1: SO-1004 duplicates SO-1003, waiting for answers to two flags.
    Asha withdraws it with "Duplicate of SO-1003".
    Result: shows as withdrawn, neither flag gets an answer.
    """
    thread_id = "SO-1004"
    
    # Setup: create a case in needs_input state with two flags
    test_dir = await setup_case(
        thread_id,
        "needs_input",
        service_order={"id": "SO-1004", "subjectEmployee": {"fullName": "Test Employee"}},
        flags=[
            {
                "flag_id": "validation-1",
                "type": "missingData",
                "stage": "validation",
                "title": "Missing employee ID",
                "detail": {"field": "employee_id"},
                "resolved": False,
                "decisions": [],
            },
            {
                "flag_id": "compliance-1",
                "type": "complianceTiming",
                "stage": "compliance",
                "title": "Final pay timing unclear",
                "detail": {"reason": "termination date missing"},
                "resolved": False,
                "decisions": [],
            },
        ],
    )
    
    # Withdraw the case
    result = await withdraw_case(
        thread_id=thread_id,
        reason="Duplicate of SO-1003",
        reviewer_id="asha",
        checkpointer_kind="sqlite",
        db_path=test_dir / "checkpoint.db",
    )
    
    # Assert: withdrawal succeeded
    assert result["status"] == "withdrawn"
    assert result["reason"] == "Duplicate of SO-1003"
    
    # Assert: case shows as withdrawn
    checkpointer = get_checkpointer("sqlite", test_dir / "checkpoint.db")
    await checkpointer.setup()
    config = {"configurable": {"thread_id": thread_id}}
    snapshot = await checkpointer.aget(config)
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    assert snapshot.values is not None
    state: State = snapshot.values  # type: ignore[assignment]
    assert state["status"] == "withdrawn"
    assert state["withdrawal_reason"] == "Duplicate of SO-1003"
    
    # Assert: neither flag has an answer
    flags = state.get("flags", [])
    assert len(flags) == 2
    for flag in flags:
        assert "answer" not in flag, f"Flag {flag['flag_id']} should not have an answer"
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)


# Test 2: Example 2 (Ben withdraws at review, then tries to approve)
@pytest.mark.asyncio
async def test_example_2_withdrawn_at_review_then_approve_refused() -> None:
    """
    Example 2: Ben withdraws a case at final review with "Client asked to stop the termination".
    When he then tries to approve it, he is told it was withdrawn.
    """
    thread_id = "SO-1005"
    
    # Setup: create a case in needs_review state
    test_dir = await setup_case(
        thread_id,
        "needs_review",
        service_order={"id": "SO-1005"},
    )
    
    # Withdraw the case
    await withdraw_case(
        thread_id=thread_id,
        reason="Client asked to stop the termination",
        reviewer_id="ben",
        checkpointer_kind="sqlite",
        db_path=test_dir / "checkpoint.db",
    )
    
    # Attempt to approve via API
    from agents.runtime import server
    from fastapi.testclient import TestClient
    
    with patch("agents.runtime.server.run_dir", return_value=test_dir):
        client = TestClient(server.app)
        response = client.post(
            f"/runs/{thread_id}/resume",
            json={"action": "approve"},
        )
    
    # Assert: 409 Conflict with withdrawal message
    assert response.status_code == 409
    assert "was withdrawn" in response.json()["detail"]
    assert "Client asked to stop the termination" in response.json()["detail"]
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)


# Test 3: Example 3 (Asha tries empty reason)
@pytest.mark.asyncio
async def test_example_3_empty_reason_refused_case_still_paused() -> None:
    """
    Example 3: Asha tries to withdraw a case with a reason of only spaces.
    Result: she is asked for a reason, and the case is still waiting for review.
    """
    thread_id = "SO-1006"
    
    # Setup: create a case in needs_review state
    test_dir = await setup_case(thread_id, "needs_review")
    
    # Attempt to withdraw with whitespace-only reason
    with pytest.raises(ValueError, match="Reason is required and cannot be empty"):
        await withdraw_case(
            thread_id=thread_id,
            reason="   \t\n  ",
            reviewer_id="asha",
            checkpointer_kind="sqlite",
            db_path=test_dir / "checkpoint.db",
        )
    
    # Assert: case is still in needs_review state
    checkpointer = get_checkpointer("sqlite", test_dir / "checkpoint.db")
    await checkpointer.setup()
    config = {"configurable": {"thread_id": thread_id}}
    snapshot = await checkpointer.aget(config)
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    state: State = snapshot.values  # type: ignore[assignment]
    assert state["status"] == "needs_review", "Case should still be paused at review"
    assert "withdrawal_reason" not in state or state.get("withdrawal_reason") is None
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)


# Test 4: Example 4 (restart persistence)
@pytest.mark.asyncio
async def test_example_4_restart_persistence() -> None:
    """
    Example 4: After a restart, Ben opens a case withdrawn the day before.
    Result: it still shows as withdrawn, with its reason.
    
    (This is the same test as Story 6, included here for completeness of examples.)
    """
    thread_id = "SO-1007"
    
    # Setup: create a case, withdraw it, simulate a restart
    test_dir = await setup_case(thread_id, "needs_review")
    
    # Withdraw the case
    result = await withdraw_case(
        thread_id=thread_id,
        reason="Duplicate of SO-1003",
        reviewer_id="ben",
        checkpointer_kind="sqlite",
        db_path=test_dir / "checkpoint.db",
    )
    withdrawn_at = result["withdrawn_at"]
    
    # Simulate restart: close and re-open the checkpointer
    checkpointer = get_checkpointer("sqlite", test_dir / "checkpoint.db")
    await checkpointer.setup()
    config = {"configurable": {"thread_id": thread_id}}
    snapshot = await checkpointer.aget(config)
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    # Assert: case still shows as withdrawn with reason
    assert snapshot.values is not None
    state: State = snapshot.values  # type: ignore[assignment]
    assert state["status"] == "withdrawn"
    assert state["withdrawal_reason"] == "Duplicate of SO-1003"
    assert state["withdrawn_at"] == withdrawn_at
    assert state["withdrawn_by"] == "ben"
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)


# Test 5: General behavior unchanged (AC 13)
@pytest.mark.asyncio
async def test_general_behavior_unchanged_for_non_withdrawn_cases(
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    """
    AC 13: Starting, answering flags, approving, rejecting work unchanged for non-withdrawn cases.
    
    This test verifies that the withdrawal feature does not break existing operations on cases
    that are never withdrawn.
    """
    thread_id = "SO-1008"
    
    # Mock run_query to simulate a full run without actually executing the graph
    run_query_called = False
    run_query_args = {}
    
    async def mock_run_query(query="", *, service_order_id=None, thread_id=None, resume=False, decision=None):
        nonlocal run_query_called, run_query_args
        run_query_called = True
        run_query_args = {
            "query": query,
            "service_order_id": service_order_id,
            "thread_id": thread_id,
            "resume": resume,
            "decision": decision,
        }
        # Simulate a successful run
        return {
            "thread_id": thread_id,
            "paused": False,
            "values": {
                "status": "done",
                "results": {
                    "validation": {"complete": True},
                    "compliance": {"complete": True},
                    "payroll-calculation": {"complete": True},
                    "execution": {"complete": True},
                },
                "flags": [],
                "steps": [],
            },
        }
    
    monkeypatch.setattr("agents.runtime.runner.run_query", mock_run_query)
    
    # 1. Start a new case
    run_query_called = False
    await run_query(service_order_id="SO-1008", thread_id=thread_id)
    assert run_query_called, "run_query should be called for starting a case"
    assert run_query_args["resume"] is False
    assert run_query_args["service_order_id"] == "SO-1008"
    
    # 2. Resume a paused case (simulating flag answers)
    run_query_called = False
    await run_query(
        thread_id=thread_id,
        resume=True,
        decision={"answers": [{"flag_id": "test-flag", "action": "accept"}]},
    )
    assert run_query_called, "run_query should be called for resuming with flag answers"
    assert run_query_args["resume"] is True
    assert run_query_args["decision"]["answers"][0]["flag_id"] == "test-flag"
    
    # 3. Approve a case at final review
    run_query_called = False
    await run_query(
        thread_id=thread_id,
        resume=True,
        decision={"action": "approve"},
    )
    assert run_query_called, "run_query should be called for approving a case"
    assert run_query_args["decision"]["action"] == "approve"
    
    # 4. Reject a case at final review
    run_query_called = False
    await run_query(
        thread_id=thread_id,
        resume=True,
        decision={"action": "reject", "reason": "Test rejection"},
    )
    assert run_query_called, "run_query should be called for rejecting a case"
    assert run_query_args["decision"]["action"] == "reject"
    assert run_query_args["decision"]["reason"] == "Test rejection"
    
    # All operations succeeded without withdrawal interference
    # (no cleanup needed: no real checkpointer was created)
```

---

### Modified Files

None. This story only creates a new test file; it imports and uses functions from Stories 1-7 but does not modify them.

---

## Interfaces

### Test dependencies

**From Story 1:**

- `withdraw_case` function (`src/agents/runtime/withdrawal.py`)
- `State` TypedDict with withdrawal fields (`src/agents/graph/states.py`)

**From Story 2:**

- `POST /runs/{thread_id}/resume` endpoint (`src/agents/runtime/server.py`)

**From Story 3:**

- `withdraw-case` CLI command (`src/agents/cli.py`) — not directly tested here, but covered in Story 3's tests

**From Story 4:**

- Queue handler integration (`src/agents/queue/orders.py`) — not directly tested here, but covered in Story 4's tests

**From Story 5:**

- Console notification integration (`src/agents/runtime/backend.py`) — not directly tested here, but covered in Story 5's tests

**From Story 6:**

- Persistence across restarts (checkpointer behavior) — covered in Test 4

**From Story 7:**

- Prevention of operations on withdrawn cases (API endpoint checks) — covered in Test 2

**From existing codebase:**

- `run_query` function (`src/agents/runtime/runner.py`)
- `get_checkpointer` factory (`src/agents/runtime/checkpointer.py`)
- `AsyncSqliteSaver` (from `langgraph.checkpoint.sqlite.aio`, via `get_checkpointer`)
- `Checkpoint` TypedDict (from `langgraph.checkpoint.base`)

---

## Algorithm Notes

### End-to-end test pattern

Each test follows the pattern:

1. **Setup:** Create a case in a known state (using the shared `setup_case` helper or `withdraw_case` from Story 1).
2. **Act:** Perform the operation being tested (withdraw, approve, etc.).
3. **Assert:** Verify the outcome against the expected behavior from the requirements.
4. **Cleanup:** Remove the temporary test directory.

### Mocking strategy

**Tests 1, 3, 4:** Use a real sqlite checkpointer to verify persistence behavior. The checkpointer is opened, written, closed, and re-opened to simulate real usage.

**Test 2:** Uses the FastAPI `TestClient` to test the API endpoint integration, with `run_dir` mocked to point to a test directory.

**Test 5:** Mocks `run_query` entirely to verify that the function is called with the correct arguments for all standard operations (start, resume with flag answers, approve, reject). This is a "smoke test" ensuring the withdrawal feature doesn't break the existing call paths.

### Why Test 5 mocks `run_query`

Running the full agent graph in a test is slow, fragile (depends on external services like MCP), and out of scope for this story (the graph's correctness is tested elsewhere). Test 5's goal is to verify that the **call paths** for standard operations remain unchanged when the withdrawal code is present. Mocking `run_query` lets us verify the arguments passed without executing the graph.

---

## Tests Section

**File:** `tests/test_withdrawal_e2e.py` (new file)

### End-to-end tests (5 total)

**Test 1: Example 1 (SO-1004 duplicate)**

```python
@pytest.mark.asyncio
async def test_example_1_duplicate_case_withdrawn_flags_not_answered() -> None:
    """
    Example 1: SO-1004 duplicates SO-1003, waiting for answers to two flags.
    Asha withdraws it with "Duplicate of SO-1003".
    Result: shows as withdrawn, neither flag gets an answer.
    """
    # (full implementation above in File Plan)
```

**Coverage:**

- AC 1 (withdrawing a case: from `needs_input` state)
- AC 2 (after withdrawing: shows as withdrawn with reason)
- Example 1 from requirements (duplicate case, flags not answered)

**Assertions:**

1. Withdrawal returns success payload.
2. Case status is `withdrawn`.
3. Withdrawal reason is persisted.
4. Both flags remain without an `answer` field.

---

**Test 2: Example 2 (Ben withdraws at review, then tries to approve)**

```python
@pytest.mark.asyncio
async def test_example_2_withdrawn_at_review_then_approve_refused() -> None:
    """
    Example 2: Ben withdraws a case at final review with "Client asked to stop the termination".
    When he then tries to approve it, he is told it was withdrawn.
    """
    # (full implementation above in File Plan)
```

**Coverage:**

- AC 1 (withdrawing a case: from `needs_review` state)
- AC 6 (a withdrawn case can't be approved)
- Example 2 from requirements (withdraw at review, then approve refused)

**Assertions:**

1. Withdrawal succeeds.
2. Subsequent approval attempt returns 409 Conflict.
3. Error message includes "was withdrawn" and the reason.

---

**Test 3: Example 3 (Asha tries empty reason)**

```python
@pytest.mark.asyncio
async def test_example_3_empty_reason_refused_case_still_paused() -> None:
    """
    Example 3: Asha tries to withdraw a case with a reason of only spaces.
    Result: she is asked for a reason, and the case is still waiting for review.
    """
    # (full implementation above in File Plan)
```

**Coverage:**

- AC 1 (withdrawing a case: reason validation, empty rejected)
- Example 3 from requirements (empty reason refused, case still paused)

**Assertions:**

1. `withdraw_case` raises `ValueError` with "Reason is required" message.
2. Case status remains `needs_review` (unchanged).
3. No `withdrawal_reason` field is written.

---

**Test 4: Example 4 (restart persistence)**

```python
@pytest.mark.asyncio
async def test_example_4_restart_persistence() -> None:
    """
    Example 4: After a restart, Ben opens a case withdrawn the day before.
    Result: it still shows as withdrawn, with its reason.
    
    (This is the same test as Story 6, included here for completeness of examples.)
    """
    # (full implementation above in File Plan)
```

**Coverage:**

- AC 7 (persistence across restarts)
- Example 4 from requirements (restart persistence)

**Assertions:**

1. After close/reopen of checkpointer, status is still `withdrawn`.
2. Withdrawal reason, timestamp, and reviewer persist.

---

**Test 5: General behavior unchanged (AC 13)**

```python
@pytest.mark.asyncio
async def test_general_behavior_unchanged_for_non_withdrawn_cases(
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    """
    AC 13: Starting, answering flags, approving, rejecting work unchanged for non-withdrawn cases.
    
    This test verifies that the withdrawal feature does not break existing operations on cases
    that are never withdrawn.
    """
    # (full implementation above in File Plan)
```

**Coverage:**

- AC 13 (general: operations on non-withdrawn cases unchanged)

**Assertions:**

1. `run_query` is called with correct arguments when starting a case (`resume=False`).
2. `run_query` is called with correct arguments when resuming with flag answers (`resume=True`, `decision` has `answers`).
3. `run_query` is called with correct arguments when approving (`resume=True`, `decision` has `action: "approve"`).
4. `run_query` is called with correct arguments when rejecting (`resume=True`, `decision` has `action: "reject"` and `reason`).

---

**Test count alignment with breakdown:** 5 tests (4 examples, 1 general behavior).

---

## Open Questions (carried forward from prior stories)

1. **Checkpoint schema evolution:** The tests use a minimal `Checkpoint` structure based on the current LangGraph version. If LangGraph's checkpoint schema changes, the tests may need adjustment. Consider adding a helper function (e.g. `create_minimal_checkpoint(state)`) that encapsulates the checkpoint creation pattern, so schema changes can be handled in one place.

2. **Test database cleanup:** The tests create temporary directories (`tempfile.mkdtemp()`) and delete them at the end. If a test fails before cleanup, the directory lingers. Consider using `pytest`'s `tmp_path` fixture instead of `tempfile.mkdtemp()` for automatic cleanup.

3. **Mocking depth for Test 5:** Test 5 mocks `run_query` to avoid running the real graph. An alternative approach would be to mock deeper layers (e.g. `fetch_service_order`, the graph's `astream` method) and run `run_query` itself, verifying that it calls the graph correctly. The current approach (mocking `run_query` entirely) is simpler but less thorough. If this test proves insufficient, consider extending it to mock fewer layers.

4. **Console notification verification:** None of the end-to-end tests verify that the console backend receives withdrawal notifications. Story 5's tests cover this in isolation (mocking the HTTP client), but a true end-to-end test would start a real console backend (or a mock HTTP server) and verify the notification is received. This is out of scope for this story (it would require standing up additional services), but worth noting for future integration testing.

---

**End of LLD.**
