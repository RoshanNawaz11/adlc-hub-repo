# Story 01-agent: Domain logic and state transitions — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** _(none — foundational story)_

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 1: Withdrawing a case — state transitions and reason validation

**Given** a case is paused waiting for answers to its flags (`status == "needs_input"`)  
**When** a reviewer withdraws it with a valid reason  
**Then** the case transitions to `status == "withdrawn"` without running any agent or carrying out the payroll change, and the answers already given to its flags remain as they were

---

**Given** a case is paused waiting for the final review (`status == "needs_review"`)  
**When** a reviewer withdraws it with a valid reason  
**Then** the case transitions to `status == "withdrawn"` without running any agent or carrying out the payroll change

---

**Given** a reviewer is withdrawing a case  
**When** the reason is empty, or contains only spaces, tabs or line breaks  
**Then** the withdrawal is refused (raises `ValueError`), and the case remains paused

---

**Given** a reviewer is withdrawing a case  
**When** the reason is 500 characters or fewer (after trimming)  
**Then** the withdrawal proceeds

---

**Given** a reviewer is withdrawing a case  
**When** the reason exceeds 500 characters (after trimming)  
**Then** the withdrawal is refused (raises `ValueError`), and the case remains paused

### AC 2: After withdrawing — state persistence and history entry

**Given** a case has been withdrawn  
**When** the state is read from the checkpointer  
**Then** `status == "withdrawn"`, `withdrawn_at` is an ISO 8601 timestamp, `withdrawn_by` is the reviewer identifier, and `withdrawal_reason` is the trimmed reason

---

**Given** a case has been withdrawn  
**When** the case's history (`State["steps"]`) is read  
**Then** the final entry is of type `withdrawn`, with fields `by` (reviewer), `at` (timestamp), `reason` (trimmed reason)

---

**Given** a case has been withdrawn  
**When** a reviewer attempts to withdraw it a second time  
**Then** the operation is refused (raises `InvalidStateError`), and the reviewer is told it was already withdrawn (with the existing reason and timestamp)

### AC 3: Cases that can't be withdrawn — state validation

**Given** a case is still running (`status == "running"`)  
**When** a reviewer attempts to withdraw it  
**Then** the withdrawal is refused (raises `InvalidStateError`), and the reviewer is told to wait until it pauses

---

**Given** a case is done, rejected, or failed (`status in ("done", "rejected", "failed")`)  
**When** a reviewer attempts to withdraw it  
**Then** the withdrawal is refused (raises `InvalidStateError`), and the reviewer is told the case's status

---

**Given** a case does not exist (checkpointer returns `None` for the thread_id)  
**When** a reviewer attempts to withdraw it  
**Then** a `CaseNotFoundError` is raised

---

## File Plan

### Modified Files

#### `src/agents/graph/states.py`

**Changes:**

1. Add `"withdrawn"` to the `status` Literal type (line ~96):
   ```python
   status: Literal['running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed', 'withdrawn']
   ```

2. Add three new `NotRequired` fields to `State` TypedDict (after line ~97, before `plan`):
   ```python
   withdrawn_at: NotRequired[str | None]  # ISO 8601 timestamp, None if not withdrawn
   withdrawn_by: NotRequired[str | None]  # reviewer identifier, None if not withdrawn
   withdrawal_reason: NotRequired[str | None]  # trimmed reason, None if not withdrawn
   ```

**No other changes.** The existing `steps` field (line ~105) already holds the history; the withdrawal entry is appended there.

---

### New Files

#### `src/agents/runtime/withdrawal.py`

**Purpose:** Shared withdrawal service function implementing state transition validation, reason validation, checkpointer persistence, and history entry appending.

**Imports:**

```python
from __future__ import annotations

import logging
from datetime import UTC, datetime
from pathlib import Path
from typing import Any

from agents.graph.states import State, StepRecord, step_record
from agents.runtime.checkpointer import get_checkpointer

logger = logging.getLogger(__name__)
```

**Custom exceptions:**

```python
class CaseNotFoundError(LookupError):
    """The case does not exist in the checkpointer."""


class InvalidStateError(ValueError):
    """The case's current state does not allow withdrawal."""
```

**Core function signature and implementation:**

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

    Args:
        thread_id: The run's thread_id (State["run_id"]).
        reason: The withdrawal reason, trimmed and validated.
        reviewer_id: The authenticated reviewer's identifier.
        checkpointer_kind: "memory" or "sqlite" (default "memory" for tests).
        db_path: Path to the sqlite checkpoint.db file (required if checkpointer_kind == "sqlite").

    Returns:
        {"status": "withdrawn", "reason": "<trimmed reason>", "withdrawn_at": "<ISO 8601 timestamp>"}

    Raises:
        ValueError: If reason is invalid (empty after trimming or exceeds 500 characters).
        InvalidStateError: If the case is not in a withdrawable state (running, done, rejected, failed, withdrawn).
        CaseNotFoundError: If the case does not exist.
    """
    # 1. Validate reason
    trimmed_reason = reason.strip()
    if not trimmed_reason:
        raise ValueError("Reason is required and cannot be empty")
    if len(trimmed_reason) > 500:
        raise ValueError("Reason exceeds 500 characters")

    # 2. Load case state from checkpointer
    checkpointer = get_checkpointer(checkpointer_kind, db_path)
    config = {"configurable": {"thread_id": thread_id}}
    
    # AsyncSqliteSaver requires setup before use (creates tables)
    if checkpointer_kind == "sqlite":
        await checkpointer.setup()
    
    snapshot = await checkpointer.aget(config)
    
    # 3. Check case exists
    if snapshot is None or snapshot.values is None:
        raise CaseNotFoundError(f"Case {thread_id} not found")
    
    state: State = snapshot.values  # type: ignore[assignment]
    current_status = state.get("status")
    
    # 4. Check current status (state transition rules)
    if current_status == "running":
        raise InvalidStateError("Case is running, wait until it pauses")
    
    if current_status in ("done", "rejected", "failed"):
        raise InvalidStateError(f"Case is {current_status}, cannot withdraw")
    
    if current_status == "withdrawn":
        existing_reason = state.get("withdrawal_reason", "")
        existing_timestamp = state.get("withdrawn_at", "")
        raise InvalidStateError(
            f"Case was withdrawn on {existing_timestamp}: {existing_reason}"
        )
    
    if current_status not in ("needs_input", "needs_review"):
        raise InvalidStateError(
            f"Case status {current_status} is not withdrawable"
        )
    
    # 5. Transition to withdrawn
    withdrawn_at = datetime.now(UTC).isoformat()
    
    # Update state fields
    state["status"] = "withdrawn"
    state["withdrawn_at"] = withdrawn_at
    state["withdrawn_by"] = reviewer_id
    state["withdrawal_reason"] = trimmed_reason
    
    # 6. Append history entry
    history_entry: StepRecord = step_record(
        stage="planning",  # withdrawal is an orchestrator-level action
        agent="Orchestrator",
        title=f"Case withdrawn by {reviewer_id}",
        source="Withdrawal API",
        result="withdrawn",
        at=withdrawn_at,
    )
    # Add withdrawal-specific fields (not in step_record signature)
    history_entry["type"] = "withdrawn"  # type: ignore[typeddict-unknown-key]
    history_entry["by"] = reviewer_id  # type: ignore[typeddict-unknown-key]
    history_entry["reason"] = trimmed_reason  # type: ignore[typeddict-unknown-key]
    
    state.setdefault("steps", []).append(history_entry)
    
    # 7. Write checkpoint
    await checkpointer.aput(config, snapshot.checkpoint, state, snapshot.metadata)
    
    # Close sqlite connection if needed (otherwise process hangs)
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

- **Reason validation:** `.strip()` removes leading/trailing whitespace (spaces, tabs, newlines per Python's definition); the empty check comes first, then the length check on the trimmed value.
- **Checkpointer interaction:** Uses the existing `get_checkpointer` factory from `checkpointer.py`. `AsyncSqliteSaver` requires `.setup()` before first use (creates tables if missing); call it unconditionally (idempotent). `.aget(config)` returns `None` if no checkpoint exists for the thread_id. `.aput(config, checkpoint, state, metadata)` writes the updated state; `checkpoint` and `metadata` are carried from the snapshot.
- **State transition validation:** The order of checks matters for error message clarity: `running` → "wait until it pauses", `done/rejected/failed` → "cannot withdraw", `withdrawn` → "was withdrawn on <timestamp>: <reason>" (idempotency), any other status → "not withdrawable" (defensive, should never hit given the current status Literal).
- **History entry schema:** The breakdown and HLD specify `{"type": "withdrawn", "by": "<reviewer_id>", "at": "<timestamp>", "reason": "<reason>"}`. `step_record` produces a `StepRecord` TypedDict with fields `stage`, `agent`, `title`, `source`, `at`, `result`. We add the withdrawal-specific fields (`type`, `by`, `reason`) as extra keys — TypedDict allows this at runtime (though mypy will complain; suppress with `# type: ignore[typeddict-unknown-key]` or cast to `dict[str, Any]`). The `at` field from `step_record` duplicates the `withdrawn_at` timestamp for consistency with other steps.
- **Cleanup:** `AsyncSqliteSaver.conn` must be closed after writing to avoid hanging the process (the aiosqlite connection's thread keeps it alive). The `close_checkpointer` utility in `checkpointer.py` does this; inline it here since we're already holding the checkpointer reference.

---

## Interfaces

### `withdraw_case` function

**Location:** `src/agents/runtime/withdrawal.py`

**Signature:**

```python
async def withdraw_case(
    thread_id: str,
    reason: str,
    reviewer_id: str,
    *,
    checkpointer_kind: str = "memory",
    db_path: Path | None = None,
) -> dict[str, Any]
```

**Parameters:**

- `thread_id` (str): The run's unique identifier (matches `State["run_id"]` and the checkpointer's `thread_id`).
- `reason` (str): The withdrawal reason, as entered by the reviewer. Will be trimmed (leading/trailing whitespace removed) and validated (non-empty, ≤500 characters).
- `reviewer_id` (str): The authenticated reviewer's identifier (e.g. user ID, email). Persisted in `State["withdrawn_by"]` and the history entry.
- `checkpointer_kind` (str, optional): `"memory"` (for tests) or `"sqlite"` (default for production). Passed to `get_checkpointer`.
- `db_path` (Path | None, optional): Path to the sqlite `checkpoint.db` file, required when `checkpointer_kind == "sqlite"`. The caller (API endpoint, CLI command) resolves this from `run_dir(thread_id) / "checkpoint.db"`.

**Returns:**

```python
{
    "status": "withdrawn",
    "reason": "<trimmed reason>",
    "withdrawn_at": "<ISO 8601 timestamp in UTC>"
}
```

**Raises:**

- `ValueError`: Reason is empty (after trimming) or exceeds 500 characters.
- `InvalidStateError`: Case is not in a withdrawable state (running, done, rejected, failed, or already withdrawn). The error message specifies the reason (e.g. "Case is running, wait until it pauses").
- `CaseNotFoundError`: No checkpoint exists for the `thread_id`.

**Side effects:**

- Writes to the checkpointer: updates `State["status"]`, `State["withdrawn_at"]`, `State["withdrawn_by"]`, `State["withdrawal_reason"]`, and appends a history entry to `State["steps"]`.
- Logs the withdrawal at `INFO` level: `"Case <thread_id> withdrawn by <reviewer_id>: <reason>"`.

---

### `State` TypedDict changes

**Location:** `src/agents/graph/states.py`

**New fields:**

```python
withdrawn_at: NotRequired[str | None]  # ISO 8601 timestamp, None if not withdrawn
withdrawn_by: NotRequired[str | None]  # reviewer identifier, None if not withdrawn
withdrawal_reason: NotRequired[str | None]  # trimmed reason, None if not withdrawn
type: NotRequired[str]  # for history entries: "withdrawn" when entry records a withdrawal
by: NotRequired[str]  # for history entries: reviewer identifier
reason: NotRequired[str]  # for history entries: withdrawal reason
```

**Updated field:**

```python
status: Literal['running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed', 'withdrawn']
```

**History entry schema (appended to `State["steps"]`):**

The existing `StepRecord` TypedDict is extended with three optional fields (`type`, `by`, `reason`) so that withdrawal entries can carry their distinguishing data without casting or type suppression. The withdrawal entry sets:

```python
{
    "type": "withdrawn",
    "stage": "planning",
    "agent": "Orchestrator",
    "title": "Case withdrawn by <reviewer_id>",
    "source": "Withdrawal API",
    "at": "<ISO 8601 timestamp>",
    "result": "withdrawn",
    "by": "<reviewer_id>",
    "reason": "<trimmed reason>"
}
```

Assumption: The `StepRecord` TypedDict definition in `states.py` is modified to include `type: NotRequired[str]`, `by: NotRequired[str]`, and `reason: NotRequired[str]` fields. This allows withdrawal entries to use the same type as other steps, with extra fields present only when needed. The alternative—defining a separate `WithdrawalHistoryEntry` TypedDict—would require runtime type narrowing everywhere the history is read, complicating the console's display logic. The modification to `StepRecord` is simpler and follows the pattern of other `NotRequired` fields already present (e.g. `result`, `evidence`, `rule`).

## Algorithm Notes

### State transition logic

```
Current status → Withdrawal allowed?
-------------------------------------
"running"       → No (raise "wait until it pauses")
"needs_input"   → Yes (valid transition)
"needs_review"  → Yes (valid transition)
"done"          → No (raise "case is done, cannot withdraw")
"rejected"      → No (raise "case is rejected, cannot withdraw")
"failed"        → No (raise "case is failed, cannot withdraw")
"withdrawn"     → No (raise "was withdrawn on <timestamp>: <reason>", idempotency)
<other>         → No (raise "status <other> is not withdrawable", defensive)
```

### Reason validation sequence

1. `trimmed_reason = reason.strip()` — removes leading/trailing spaces, tabs, newlines.
2. If `not trimmed_reason` (empty after trimming) → raise `ValueError("Reason is required and cannot be empty")`.
3. If `len(trimmed_reason) > 500` → raise `ValueError("Reason exceeds 500 characters")`.

### Checkpointer read/write pattern

```python
# 1. Get checkpointer instance
checkpointer = get_checkpointer(checkpointer_kind, db_path)
config = {"configurable": {"thread_id": thread_id}}

# 2. Setup (sqlite only, idempotent)
if checkpointer_kind == "sqlite":
    await checkpointer.setup()

# 3. Read current state
snapshot = await checkpointer.aget(config)
if snapshot is None or snapshot.values is None:
    raise CaseNotFoundError(...)

state: State = snapshot.values

# 4. Mutate state (in memory)
state["status"] = "withdrawn"
state["withdrawn_at"] = withdrawn_at
state["withdrawn_by"] = reviewer_id
state["withdrawal_reason"] = trimmed_reason
state.setdefault("steps", []).append(history_entry)

# 5. Write updated state
await checkpointer.aput(config, snapshot.checkpoint, state, snapshot.metadata)

# 6. Close connection (sqlite only)
if checkpointer_kind == "sqlite":
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
```

**Open question from HLD:** Does `AsyncSqliteSaver` support optimistic locking or transactions to prevent race conditions between withdrawal and a concurrent resume? The current implementation does not use compare-and-swap or a version field. If a resume job loads the state, then the withdrawal completes, then the resume writes back, the withdrawal will be overwritten. This is the **accepted risk** per the HLD; mitigating it (adding a `version: int` field to `State` and using compare-and-swap in both `withdrawal.py` and `resume_service_order`) is deferred to a future enhancement.

---

## Tests Section

**File:** `tests/test_withdrawal.py` (new file)

### Unit tests (10 total)

**Test 1: Valid transition from `needs_input` to `withdrawn`**

```python
async def test_withdraw_from_needs_input():
    """AC 1: A case waiting for flag answers can be withdrawn."""
    thread_id = "test-thread-1"
    # Setup: create a case in needs_input state
    # Call withdraw_case(thread_id, "Test reason", "reviewer-1", checkpointer_kind="memory")
    # Assert: state["status"] == "withdrawn"
    # Assert: state["withdrawn_at"] is not None (ISO 8601 format)
    # Assert: state["withdrawn_by"] == "reviewer-1"
    # Assert: state["withdrawal_reason"] == "Test reason"
```

**Test 2: Valid transition from `needs_review` to `withdrawn`**

```python
async def test_withdraw_from_needs_review():
    """AC 1: A case waiting for final review can be withdrawn."""
    thread_id = "test-thread-2"
    # Setup: create a case in needs_review state
    # Call withdraw_case(thread_id, "Test reason", "reviewer-2", checkpointer_kind="memory")
    # Assert: state["status"] == "withdrawn"
```

**Test 3: Reject withdrawal from `running`**

```python
async def test_withdraw_from_running_rejected():
    """AC 3, AC 8: A running case cannot be withdrawn."""
    thread_id = "test-thread-3"
    # Setup: create a case in running state
    # Assert: withdraw_case raises InvalidStateError("Case is running, wait until it pauses")
    # Assert: state["status"] == "running" (unchanged)
```

**Test 4: Reject withdrawal from `done`**

```python
async def test_withdraw_from_done_rejected():
    """AC 3, AC 9: A done case cannot be withdrawn."""
    thread_id = "test-thread-4"
    # Setup: create a case in done state
    # Assert: withdraw_case raises InvalidStateError("Case is done, cannot withdraw")
```

**Test 5: Reject withdrawal from `withdrawn` (idempotency)**

```python
async def test_withdraw_already_withdrawn():
    """AC 2, AC 6: Withdrawing an already-withdrawn case is refused with existing details."""
    thread_id = "test-thread-5"
    # Setup: withdraw a case once
    # Assert: second withdraw_case raises InvalidStateError("Case was withdrawn on <timestamp>: <reason>")
    # Assert: state["status"] == "withdrawn" (unchanged)
    # Assert: state["withdrawn_at"] == first withdrawal's timestamp (unchanged)
```

**Test 6: Empty reason rejected**

```python
async def test_empty_reason_rejected():
    """AC 1: An empty reason (after trimming) is refused."""
    thread_id = "test-thread-6"
    # Setup: create a case in needs_input state
    # Assert: withdraw_case(thread_id, "", ...) raises ValueError("Reason is required and cannot be empty")
```

**Test 7: Whitespace-only reason rejected**

```python
async def test_whitespace_reason_rejected():
    """AC 1: A reason containing only whitespace is refused."""
    thread_id = "test-thread-7"
    # Setup: create a case in needs_input state
    # Assert: withdraw_case(thread_id, "   \t\n  ", ...) raises ValueError("Reason is required and cannot be empty")
```

**Test 8: Reason exactly 500 characters accepted**

```python
async def test_reason_exactly_500_chars():
    """AC 1: A reason of exactly 500 characters (after trimming) is accepted."""
    thread_id = "test-thread-8"
    # Setup: create a case in needs_input state
    reason = "a" * 500
    # Call withdraw_case(thread_id, reason, "reviewer-8", checkpointer_kind="memory")
    # Assert: state["withdrawal_reason"] == reason
    # Assert: state["status"] == "withdrawn"
```

**Test 9: Reason over 500 characters rejected**

```python
async def test_reason_over_500_chars_rejected():
    """AC 1: A reason exceeding 500 characters (after trimming) is refused."""
    thread_id = "test-thread-9"
    # Setup: create a case in needs_input state
    reason = "a" * 501
    # Assert: withdraw_case raises ValueError("Reason exceeds 500 characters")
```

**Test 10: History entry appended**

```python
async def test_history_entry_appended():
    """AC 2: Withdrawal appends a history entry with type, by, at, reason."""
    thread_id = "test-thread-10"
    # Setup: create a case in needs_input state with existing steps
    # Call withdraw_case(thread_id, "Test reason", "reviewer-10", checkpointer_kind="memory")
    # Assert: state["steps"][-1]["type"] == "withdrawn"
    # Assert: state["steps"][-1]["by"] == "reviewer-10"
    # Assert: state["steps"][-1]["reason"] == "Test reason"
    # Assert: state["steps"][-1]["at"] is an ISO 8601 timestamp
```

---

**Test count alignment with breakdown:** 10 tests (5 state transition cases, 4 reason validation cases, 1 idempotency test).

---

## Open Questions (carried forward from HLD/breakdown)

1. **Checkpoint locking:** Does `AsyncSqliteSaver` support optimistic locking or transactions to prevent race conditions between withdrawal and resume? If not, should a `version: int` field be added to `State` for compare-and-swap logic? (Accepted risk for this story; mitigation deferred.)
2. **Reviewer authentication:** How is `reviewer_id` determined in API (auth token?) and CLI (argument? environment variable?) contexts? (Story 2 and Story 3 will resolve this; for now, `withdraw_case` accepts it as a parameter.)

---

**End of LLD.**
