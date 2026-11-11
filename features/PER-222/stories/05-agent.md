# Story 05-agent: Console notification integration — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 4: Everywhere a case is handled — console notification

**Given** a case has been withdrawn  
**When** the console's activity feed is notified  
**Then** the feed receives a "Case withdrawn" event, with the employee's name when the order has one

---

**Given** the console cannot be reached when a case is withdrawn  
**When** the withdrawal proceeds  
**Then** the withdrawal still completes, and the case shows as withdrawn

### AC 2: After withdrawing — activity feed event

**Given** a case has been withdrawn  
**When** the console backend is notified  
**Then** it receives an event of kind `case_withdrawn` with payload `{"employee_name": <name or null>, "reason": "<reason>", "withdrawn_at": "<timestamp>"}`

---

## File Plan

### Modified Files

#### `src/agents/runtime/backend.py`

**Changes:**

1. Add a new function `report_withdrawal` after the existing `report_progress` function (after line ~150):

```python
async def report_withdrawal(
    thread_id: str,
    reason: str,
    withdrawn_at: str,
    employee: str | None = None,
) -> None:
    """
    Report a case withdrawal to the console's activity feed.
    
    Best-effort: logs failure but does not raise an exception.
    
    Args:
        thread_id: The run's thread_id.
        reason: The trimmed withdrawal reason.
        withdrawn_at: ISO 8601 timestamp of when the case was withdrawn.
        employee: The employee's full name (from the service order), or None.
    """
    event = {
        "dedupe_key": f"lifecycle:withdrawn",
        "title": _clip(_display(employee, "case withdrawn") if employee else "Case withdrawn"),
        "action": "Case withdrawn",
        "actor": "Reviewer",
        "system": "Agents",
        "kind": "agent",
        "tone": "grey",
        "employee_name": employee,
        "feed_kind": "routine",
    }
    
    payload = {
        "events": [event],
        "values": None,
        "run_status": "withdrawn",
        "error": None,
    }
    
    url = f"{_backend_url()}/api/v1/processing-runs/{thread_id}/events"
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.post(url, json=payload)
        if response.status_code >= 400:
            logger.warning(
                "Console notification for withdrawal of %s failed: HTTP %s %s",
                thread_id,
                response.status_code,
                response.text[:300],
            )
    except httpx.HTTPError as exc:
        logger.warning(
            "Console notification for withdrawal of %s failed: %s",
            thread_id,
            exc,
        )
```

**Implementation notes:**

- **Best-effort semantics:** The function catches all HTTP errors and logs them at `WARNING` level without raising. This ensures the withdrawal operation (which calls this function) succeeds even if the console is unreachable.
- **Event payload:** Matches the existing `lifecycle_event` pattern from `backend.py` (lines ~127-149). The `dedupe_key` is `"lifecycle:withdrawn"` to match the pattern used for other lifecycle events (`"lifecycle:started"`, `"lifecycle:paused"`, etc.).
- **Title and display:** Uses the existing `_display` helper (line ~47) to prefix the employee name when available.
- **Tone:** Uses `"grey"` (matching the tone for `"payroll"` stage events, line ~28) to indicate an administrative action that is neither success nor failure.
- **Feed kind:** Uses `"routine"` (not `"attention"`) because a withdrawal is a deliberate reviewer action, not something requiring attention.
- **Actor:** `"Reviewer"` instead of `"AI Agents"` because the withdrawal is a human action, not an agent decision.
- **URL and timeout:** Matches the existing `report_progress` function (line ~151).

**No other changes to `backend.py`.** The existing functions (`employee_name`, `_clip`, `_display`, `_backend_url`, `events_from_update`, `lifecycle_event`, `report_progress`) are used as-is.

---

#### `src/agents/runtime/withdrawal.py`

**Changes:**

1. Add import for `report_withdrawal` (after the existing imports, around line ~10):

```python
from agents.runtime.backend import employee_name, report_withdrawal
```

2. Modify the `withdraw_case` function to call `report_withdrawal` after persisting the withdrawal (after the `checkpointer.aput` call, before closing the checkpointer connection, around line ~110):

```python
async def withdraw_case(
    thread_id: str,
    reason: str,
    reviewer_id: str,
    *,
    checkpointer_kind: str = "memory",
    db_path: Path | None = None,
) -> dict[str, Any]:
    """
    Withdraw a paused case with a reason.
    
    ... (existing docstring)
    """
    # ... (existing validation and state transition logic)
    
    # 7. Write checkpoint
    await checkpointer.aput(config, snapshot.checkpoint, state, snapshot.metadata)
    
    # 8. Dispatch console notification (best-effort)
    employee = employee_name(state.get("service_order"))
    await report_withdrawal(
        thread_id=thread_id,
        reason=trimmed_reason,
        withdrawn_at=withdrawn_at,
        employee=employee,
    )
    
    # 9. Close sqlite connection if needed (otherwise process hangs)
    if checkpointer_kind == "sqlite":
        conn = getattr(checkpointer, "conn", None)
        if conn is not None:
            await conn.close()
    
    logger.info(
        "Case %s withdrawn by %s: %s",
        thread_id,
        reviewer_id,
        trimmed_reason,
    )
    
    return {
        "status": "withdrawn",
        "reason": trimmed_reason,
        "withdrawn_at": withdrawn_at,
    }
```

**Implementation notes:**

- **Employee name extraction:** Uses the existing `employee_name` helper from `backend.py` to extract the employee's full name from the `service_order` field in the state (if present). This matches the pattern in `runner.py` (line ~182) and `backend.py` (line ~35).
- **Call placement:** The notification is dispatched after the checkpoint write succeeds, ensuring the withdrawal is durable before notifying the console. If the notification fails, the withdrawal still completes (the `report_withdrawal` function logs the failure but does not raise).
- **Async call:** `report_withdrawal` is async, so it must be awaited. The existing code in `withdraw_case` is already async, so no restructuring is needed.

**No other changes to `withdrawal.py`.** The existing validation, state transition, and checkpoint write logic remain unchanged.

---

### New Files

None. This story only modifies existing files created in Story 1 (`withdrawal.py`) and an existing runtime file (`backend.py`).

---

## Interfaces

### `report_withdrawal` function

**Location:** `src/agents/runtime/backend.py`

**Signature:**

```python
async def report_withdrawal(
    thread_id: str,
    reason: str,
    withdrawn_at: str,
    employee: str | None = None,
) -> None
```

**Parameters:**

- `thread_id` (str): The run's unique identifier (matches `State["run_id"]` and the checkpointer's `thread_id`).
- `reason` (str): The trimmed withdrawal reason (as persisted in `State["withdrawal_reason"]`).
- `withdrawn_at` (str): ISO 8601 timestamp of when the case was withdrawn (as persisted in `State["withdrawn_at"]`).
- `employee` (str | None, optional): The employee's full name (from `service_order.subjectEmployee.fullName` or `service_order.subject_employee.full_name`), or `None` if not available.

**Returns:** `None`

**Side effects:**

- POSTs to the console backend's activity feed endpoint (`{BACKEND_URL}/api/v1/processing-runs/{thread_id}/events`) with the withdrawal event payload.
- Logs a warning at `WARNING` level if the console backend is unreachable or returns an HTTP error (4xx or 5xx).

**Error handling:**

- All `httpx.HTTPError` exceptions are caught and logged; the function does not raise.
- Network errors, timeouts, and HTTP errors are all treated the same way: log and continue.

---

### Console backend event payload

**URL:** `{BACKEND_URL}/api/v1/processing-runs/{thread_id}/events`

**Method:** `POST`

**Request body:**

```json
{
  "events": [
    {
      "dedupe_key": "lifecycle:withdrawn",
      "title": "John Doe — case withdrawn",
      "action": "Case withdrawn",
      "actor": "Reviewer",
      "system": "Agents",
      "kind": "agent",
      "tone": "grey",
      "employee_name": "John Doe",
      "feed_kind": "routine"
    }
  ],
  "values": null,
  "run_status": "withdrawn",
  "error": null
}
```

**Field descriptions:**

- `dedupe_key`: Unique key to prevent duplicate events (matches the pattern `"lifecycle:<kind>"` used for other lifecycle events).
- `title`: Human-readable summary for the activity feed, prefixed with the employee's name when available.
- `action`: Short description of the action (always `"Case withdrawn"`).
- `actor`: Who performed the action (always `"Reviewer"` for withdrawals).
- `system`: The system that recorded the action (always `"Agents"`).
- `kind`: Event category (always `"agent"` for agent-related lifecycle events).
- `tone`: Visual tone for the UI (always `"grey"` for withdrawals, indicating an administrative action).
- `employee_name`: The employee's full name, or `null` if not available.
- `feed_kind`: How the event is filed in the Fleet Activity Feed (always `"routine"` for withdrawals).
- `values`: The full state snapshot (not included for withdrawal events; always `null`).
- `run_status`: The run's new status (always `"withdrawn"` for withdrawal events).
- `error`: Error message (not applicable for withdrawals; always `null`).

**Response (success):**

- HTTP 200 OK or 201 Created (the console backend's ingestion endpoint returns one of these on success).

**Response (failure):**

- HTTP 4xx or 5xx: logged at `WARNING` level in the agents service; the withdrawal still succeeds.

---

### Integration with `withdraw_case`

The `withdraw_case` function (from Story 1) is modified to call `report_withdrawal` after persisting the withdrawal state:

```python
# (inside withdraw_case, after checkpointer.aput)

employee = employee_name(state.get("service_order"))
await report_withdrawal(
    thread_id=thread_id,
    reason=trimmed_reason,
    withdrawn_at=withdrawn_at,
    employee=employee,
)
```

**Call order:**

1. Validate reason.
2. Load state from checkpointer.
3. Validate state transition.
4. Update state fields (`status`, `withdrawn_at`, `withdrawn_by`, `withdrawal_reason`).
5. Append history entry.
6. Write checkpoint.
7. **Dispatch console notification** (this story).
8. Close checkpointer connection.
9. Log and return success.

**Error handling:**

- If `report_withdrawal` fails (network error, timeout, HTTP error), it logs the failure at `WARNING` level but does not raise an exception.
- The withdrawal operation continues: the checkpointer connection is closed, the success message is logged, and the function returns the success payload.
- From the caller's perspective (API endpoint, CLI command), the withdrawal always succeeds if the state transition and checkpoint write succeed; console notification failure is invisible.

---

## Algorithm Notes

### Console notification dispatch sequence

```
1. Extract employee name from state["service_order"] (if present):
   - Try service_order["subjectEmployee"]["fullName"]
   - Fallback to service_order["subject_employee"]["full_name"]
   - Default to None if missing or not a dict
2. Construct event payload:
   - dedupe_key = "lifecycle:withdrawn"
   - title = "<employee name> — case withdrawn" or "Case withdrawn"
   - actor = "Reviewer"
   - system = "Agents"
   - kind = "agent"
   - tone = "grey"
   - feed_kind = "routine"
3. Construct request body:
   - events = [event]
   - values = null
   - run_status = "withdrawn"
   - error = null
4. Resolve console backend URL from BACKEND_URL env var (default http://127.0.0.1:8000)
5. POST to {backend_url}/api/v1/processing-runs/{thread_id}/events with:
   - timeout = 5.0 seconds
   - json = request body
6. On HTTP 4xx or 5xx:
   - Log warning: "Console notification for withdrawal of <thread_id> failed: HTTP <status> <response text>"
7. On httpx.HTTPError (network error, timeout):
   - Log warning: "Console notification for withdrawal of <thread_id> failed: <exception>"
8. Return (no exception raised)
```

### Best-effort semantics

The notification is fire-and-forget:

- **No retries:** If the console backend is down or slow, the notification is lost. A follow-up enhancement (mentioned in the HLD's *Deferred Decisions*) could add a retry queue or outbox table for failed notifications, but that is out of scope for this story.
- **No rollback:** If the notification fails, the withdrawal is not rolled back. The case remains `withdrawn` in the checkpointer, and the withdrawal operation returns success.
- **Logged for ops:** All failures are logged at `WARNING` level with enough detail for operations to reconcile (thread_id, failure reason). Monitoring/alerting on these warnings is an ops concern, not a code change.

---

## Tests Section

**File:** `tests/test_withdrawal_console.py` (new file)

### Integration tests (2 total)

Both tests use `pytest-httpx` (or monkeypatching `httpx.AsyncClient`) to mock the console backend's HTTP responses. The tests create a withdrawn case (via `withdraw_case` from Story 1), then verify that `report_withdrawal` was called with the correct payload and that the withdrawal succeeded regardless of the console's response.

**Setup helper (shared across tests):**

```python
import json
from pathlib import Path
from typing import Any
from unittest.mock import AsyncMock, patch

import pytest

from agents.runtime.backend import report_withdrawal
from agents.runtime.withdrawal import withdraw_case
from agents.graph.states import State
from agents.runtime.checkpointer import get_checkpointer

async def setup_case(thread_id: str, status: str, **extra_state: Any) -> Path:
    """
    Write a case with the given status to the checkpointer.
    
    Returns:
        The path to the run directory (for cleanup).
    """
    import tempfile
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
```

---

**Test 1: Successful withdrawal with console notification**

```python
@pytest.mark.asyncio
async def test_withdrawal_notifies_console_on_success() -> None:
    """AC 4, AC 2: A withdrawn case triggers a console notification with the correct payload."""
    thread_id = "test-console-1"
    
    # Setup: create a case in needs_input state with a service order
    test_dir = await setup_case(
        thread_id,
        "needs_input",
        service_order={
            "id": "SO-1003",
            "subjectEmployee": {"fullName": "John Doe"},
        },
    )
    
    # Mock httpx.AsyncClient to capture the POST request
    posted_payload = None
    posted_url = None
    
    async def mock_post(url: str, *, json: dict = None, **kwargs):
        nonlocal posted_payload, posted_url
        posted_url = url
        posted_payload = json
        # Return a mock response
        from unittest.mock import AsyncMock
        response = AsyncMock()
        response.status_code = 200
        response.text = "OK"
        return response
    
    with patch("httpx.AsyncClient.post", mock_post):
        # Call withdraw_case (which calls report_withdrawal internally)
        result = await withdraw_case(
            thread_id=thread_id,
            reason="Duplicate of SO-1002",
            reviewer_id="reviewer-1",
            checkpointer_kind="sqlite",
            db_path=test_dir / "checkpoint.db",
        )
    
    # Assert: withdrawal succeeded
    assert result["status"] == "withdrawn"
    assert result["reason"] == "Duplicate of SO-1002"
    
    # Assert: console was notified
    assert posted_url is not None
    assert f"/processing-runs/{thread_id}/events" in posted_url
    
    # Assert: payload is correct
    assert posted_payload is not None
    events = posted_payload.get("events", [])
    assert len(events) == 1
    event = events[0]
    assert event["dedupe_key"] == "lifecycle:withdrawn"
    assert event["title"] == "John Doe — case withdrawn"
    assert event["action"] == "Case withdrawn"
    assert event["actor"] == "Reviewer"
    assert event["employee_name"] == "John Doe"
    assert posted_payload["run_status"] == "withdrawn"
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 2: Withdrawal succeeds when console is unreachable**

```python
@pytest.mark.asyncio
async def test_withdrawal_succeeds_when_console_fails(caplog: pytest.LogCaptureFixture) -> None:
    """AC 4: Withdrawal still succeeds when the console backend is unreachable."""
    thread_id = "test-console-2"
    
    # Setup: create a case in needs_review state (no service order)
    test_dir = await setup_case(thread_id, "needs_review")
    
    # Mock httpx.AsyncClient to raise a network error
    async def mock_post(url: str, *, json: dict = None, **kwargs):
        import httpx
        raise httpx.ConnectError("Connection refused")
    
    with patch("httpx.AsyncClient.post", mock_post):
        # Call withdraw_case (which calls report_withdrawal internally)
        result = await withdraw_case(
            thread_id=thread_id,
            reason="Client requested to stop",
            reviewer_id="reviewer-2",
            checkpointer_kind="sqlite",
            db_path=test_dir / "checkpoint.db",
        )
    
    # Assert: withdrawal succeeded despite console failure
    assert result["status"] == "withdrawn"
    assert result["reason"] == "Client requested to stop"
    
    # Assert: failure was logged at WARNING level
    assert any(
        record.levelname == "WARNING"
        and "Console notification for withdrawal" in record.message
        and thread_id in record.message
        for record in caplog.records
    )
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test count alignment with breakdown:** 2 tests (1 success, 1 failure handling).

**Coverage:**

- AC 4 (everywhere a case is handled: console notification): Both tests
- AC 2 (after withdrawing: activity feed event): Test 1 (verifies the event payload)
- Best-effort semantics: Test 2 (verifies withdrawal succeeds when console fails)

---

## Open Questions (carried forward from Story 1 / HLD)

1. **Console backend endpoint:** The LLD assumes the console backend's activity feed endpoint is `{BACKEND_URL}/api/v1/processing-runs/{thread_id}/events`, matching the existing `report_progress` function in `backend.py` (line ~151). If the endpoint or payload shape differs, the `report_withdrawal` function will need adjustment.

2. **Event dedupe_key collision:** The `dedupe_key` for withdrawal events is `"lifecycle:withdrawn"`. If multiple withdrawals of the same case are allowed (e.g. a case is resumed and then withdrawn again), this key will not deduplicate them correctly. The current design assumes a case can only be withdrawn once (per AC 6 from the requirements), so this is acceptable. If the requirement changes, the dedupe_key should include the withdrawal timestamp or a unique identifier.

3. **Employee name extraction:** The `employee_name` function (from `backend.py`) tries two paths: `service_order.subjectEmployee.fullName` and `service_order.subject_employee.full_name`. If the console's service order schema uses a different field name, the employee name will be `None` in the notification. This is acceptable (the notification still succeeds), but the console team should be aware.

---

**End of LLD.**
