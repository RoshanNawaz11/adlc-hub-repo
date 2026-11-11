# breakdown.md — PER-222

**Status:** Draft  
**Source:** Requirements PER-222, HLD PER-222  
**Epic:** _(none — standalone ticket)_

---

## Overview

Implements a withdrawal operation for paused cases, allowing reviewers to cleanly stop a case waiting for flag answers or final review. Work is split by technical area: domain model and state transitions, API endpoint, CLI command, queue handler updates, persistence schema, console notification, and end-to-end validation.

---

## Implementation Order and Dependencies

Stories are sequenced to build foundational components first (domain logic, persistence), then integrations (API, CLI, queue), then notifications and end-to-end validation.

### Story 1: Domain logic and state transitions

**Repo:** `agent`

**Description:**  
Adds the `withdrawn` state to the case status enumeration, implements state transition validation (from `needs_input` or `needs_review` to `withdrawn`, with rejection from all other states), validates the withdrawal reason (non-empty after trimming, ≤500 characters), and implements the core withdrawal logic in a shared service function. Includes persistence of withdrawal fields (`status`, `withdrawn_at`, `withdrawn_by`, `withdrawal_reason`) and appending the history entry.

**Covers:**  
- REQ-AC1 (withdrawing a case: state transitions, reason validation)
- REQ-AC2 (after withdrawing: state persistence, history entry)
- REQ-AC3 (cases that can't be withdrawn: state validation)

**After:** _(none — foundational story)_

**Allowed paths:**
- `src/agents/graph/states.py` — add `"withdrawn"` to `status` Literal, add `withdrawn_at`, `withdrawn_by`, `withdrawal_reason` fields to `State` TypedDict
- `src/agents/runtime/withdrawal.py` (new file) — `withdraw_case` function
- `src/agents/runtime/checkpointer.py` — _(no changes; checkpointer API used by `withdrawal.py`)_

**Verification Approach:**  
Unit tests verify:
1. State transition rules: valid transitions from `needs_input` and `needs_review` to `withdrawn`, invalid transitions from `running`, `done`, `rejected`, `failed`, `withdrawn` (AC 1, 3, 6, 8, 9).
2. Reason validation: empty string rejected, whitespace-only rejected, exactly 500 characters accepted, over 500 characters rejected (AC 1).
3. Withdrawal fields persisted: `status`, `withdrawn_at`, `withdrawn_by`, `withdrawal_reason` written to state (AC 2).
4. History entry appended: type `withdrawn`, `by`, `at`, `reason` fields present, is final entry (AC 2).
5. Idempotency: second withdrawal call on same case returns error with existing details, does not modify state (AC 6).

**Test count:** 10 tests (5 state transition cases, 4 reason validation cases, 1 idempotency test)

---

### Story 2: API endpoint for withdrawal

**Repo:** `agent`

**Description:**  
Adds `POST /runs/{thread_id}/withdraw` endpoint to the FastAPI server, with request/response validation per the API contract. Calls the shared `withdraw_case` function from Story 1. Returns appropriate HTTP status codes (200 OK, 400 Bad Request, 404 Not Found, 409 Conflict) and error messages per the HLD.

**Covers:**  
- REQ-AC1 (withdrawing a case: API interface)
- REQ-AC2 (after withdrawing: API response)
- REQ-AC3 (cases that can't be withdrawn: API error responses)
- REQ-AC4 (everywhere a case is handled: API)

**After:** Story 1

**Allowed paths:**
- `src/agents/runtime/server.py` — add `POST /runs/{thread_id}/withdraw` route
- `src/agents/runtime/withdrawal.py` — import and call `withdraw_case`

**Verification Approach:**  
Integration tests verify:
1. Successful withdrawal: returns 200 OK with `status`, `reason`, `withdrawn_at` (AC 1, 2).
2. Empty reason: returns 400 Bad Request with "Reason is required" message (AC 1).
3. Reason over 500 characters: returns 400 Bad Request with "Reason exceeds 500 characters" message (AC 1).
4. Case not found: returns 404 Not Found with "Case not found" message (AC 9).
5. Case running: returns 409 Conflict with "wait until it pauses" message (AC 8).
6. Case done/rejected/failed: returns 409 Conflict with case's status (AC 9).
7. Case already withdrawn: returns 409 Conflict with existing withdrawal details (AC 6).

**Test count:** 7 tests (1 success case, 6 error cases)

---

### Story 3: CLI command for withdrawal

**Repo:** `agent`

**Description:**  
Adds a `withdraw-case` CLI command (`python -m agents.cli withdraw-case <thread_id> --reason "<reason>"`) that validates the reason locally and calls the shared `withdraw_case` function from Story 1. Prints success or error messages to stdout/stderr per the CLI contract.

**Covers:**  
- REQ-AC1 (withdrawing a case: CLI interface)
- REQ-AC2 (after withdrawing: CLI output)
- REQ-AC3 (cases that can't be withdrawn: CLI error messages)
- REQ-AC4 (everywhere a case is handled: CLI)

**After:** Story 1

**Allowed paths:**
- `src/agents/cli.py` (new file) — `withdraw-case` command implementation
- `src/agents/runtime/withdrawal.py` — import and call `withdraw_case`

**Verification Approach:**  
End-to-end tests (invoking CLI as a subprocess) verify:
1. Successful withdrawal: prints "Case <thread_id> withdrawn: <reason>" and timestamp (AC 1, 2).
2. Empty reason: prints "Error: Reason is required" (AC 1).
3. Case running: prints "Error: Case is running, wait until it pauses" (AC 8).
4. Case not found: prints "Error: Case not found" (AC 9).
5. Case already withdrawn: prints existing withdrawal details (AC 6).

**Test count:** 5 tests (1 success case, 4 error cases)

---

### Story 4: Queue handler integration for resume no-op

**Repo:** `agent`

**Description:**  
Modifies the `resume_service_order` queue handler in `src/agents/queue/orders.py` to check the case's status after loading state. If `status == "withdrawn"`, skips graph invocation, writes queue status to Redis (`{"status": "withdrawn", "reason": "<reason>", "withdrawn_at": "<timestamp>"}`), marks the job as successful, and logs the no-op.

**Covers:**  
- REQ-AC4 (everywhere a case is handled: queue resume handling)
- REQ-AC2 (after withdrawing: no resume)

**After:** Story 1

**Allowed paths:**
- `src/agents/queue/orders.py` — modify `resume_service_order` function

**Verification Approach:**  
Integration tests verify:
1. Resume job for withdrawn case: graph is not invoked, queue status written with `status: "withdrawn"`, job marked successful (AC 11).
2. Resume job for non-withdrawn case: graph invoked as usual (AC 13, general behavior unchanged).

**Test count:** 2 tests (1 withdrawn case, 1 control case)

---

### Story 5: Console notification integration

**Repo:** `agent`

**Description:**  
Adds console notification of withdrawal events to `src/agents/runtime/backend.py`. After persisting the withdrawal, dispatches an activity feed event to the console backend (`{"kind": "case_withdrawn", "employee_name": <name or null>, "reason": "<reason>", "withdrawn_at": "<timestamp>"}`). Notification is best-effort: logs failure but does not roll back withdrawal or return an error to the caller.

**Covers:**  
- REQ-AC4 (everywhere a case is handled: console notification)
- REQ-AC2 (after withdrawing: activity feed event)

**After:** Story 1

**Allowed paths:**
- `src/agents/runtime/backend.py` — add `report_withdrawal` function or extend `report_progress`
- `src/agents/runtime/withdrawal.py` — call notification dispatch after persisting withdrawal

**Verification Approach:**  
Integration tests verify:
1. Successful withdrawal: console backend receives `case_withdrawn` event with correct payload (AC 12).
2. Console backend unreachable: withdrawal still succeeds, failure logged at WARNING level (AC 12).

**Test count:** 2 tests (1 success, 1 failure handling)

---

### Story 6: Persistence across restarts

**Repo:** `agent`

**Description:**  
Integration test that withdraws a case, restarts the service (or re-opens the checkpointer), and verifies the case is still `withdrawn` with all fields intact (`status`, `withdrawn_at`, `withdrawn_by`, `withdrawal_reason`). Ensures the SQLite checkpointer correctly persists withdrawn state.

**Covers:**  
- REQ-AC2 (after withdrawing: persistence across restarts)

**After:** Story 1

**Allowed paths:**
- `tests/test_withdrawal.py` (new file or existing test file) — restart integration test

**Verification Approach:**  
Integration test:
1. Withdraws a case, closes and re-opens the checkpointer (simulating a restart), verifies case shows as `withdrawn` with reason and timestamp (AC 7).

**Test count:** 1 test

---

### Story 7: Prevent operations on withdrawn cases

**Repo:** `agent`

**Description:**  
Ensures all operations that mutate case state (resume, approve, reject, start again, withdraw again) check the current status and refuse if `status == "withdrawn"`, returning an appropriate error message ("Case was withdrawn"). Covers API endpoint checks and queue handler checks.

**Covers:**  
- REQ-AC2 (after withdrawing: operations refused)

**After:** Story 1, Story 2, Story 4

**Allowed paths:**
- `src/agents/runtime/server.py` — add status check to `resume_run` and other mutating endpoints
- `src/agents/queue/orders.py` — add status check to `process_service_order`
- `src/agents/runtime/runner.py` — verify in `repo-file/<repo>/<path>` whether `run_query` needs a status check

**Verification Approach:**  
Integration tests verify:
1. Attempt to resume withdrawn case via API: returns 409 Conflict with "Case was withdrawn" (AC 6).
2. Attempt to approve withdrawn case via API: returns 409 Conflict with "Case was withdrawn" (AC 6).
3. Attempt to reject withdrawn case via API: returns 409 Conflict with "Case was withdrawn" (AC 6).
4. Attempt to start withdrawn case again via queue: job fails with "Case was withdrawn" (AC 6).

**Test count:** 4 tests (1 per mutating operation)

---

### Story 8: End-to-end validation

**Repo:** `agent`

**Description:**  
End-to-end tests covering the examples from the requirements and all integration points. Verifies the complete withdrawal flow from API/CLI call through persistence, history recording, queue handling, and console notification.

**Covers:**  
- REQ-Examples (SO-1004 duplicate, Ben withdraws at review, Asha tries empty reason, restart persistence)
- REQ-AC5 (general: operations on non-withdrawn cases unchanged)

**After:** Story 1, Story 2, Story 3, Story 4, Story 5, Story 6, Story 7

**Allowed paths:**
- `tests/test_withdrawal.py` (new file or existing test file) — end-to-end scenario tests

**Verification Approach:**  
End-to-end tests verify:
1. Example 1 (SO-1004 duplicate): case waiting for flag answers is withdrawn, shows as withdrawn, flags not answered (AC 1, 2).
2. Example 2 (Ben withdraws at review): case at final review is withdrawn, then approval attempt is refused (AC 1, 6).
3. Example 3 (Asha empty reason): withdrawal with whitespace-only reason is refused, case still paused (AC 1).
4. Example 4 (restart persistence): case withdrawn before restart still shows as withdrawn after restart (AC 7).
5. General case: starting, answering flags, approving, rejecting work unchanged for non-withdrawn cases (AC 13).

**Test count:** 5 tests (4 examples, 1 general behavior)

---

## Summary

- **8 stories** in total
- **Story 1** (domain logic) is foundational; Stories 2, 3, 4, 5, 6 depend on it
- **Story 7** (prevent operations) depends on Stories 1, 2, 4
- **Story 8** (end-to-end validation) depends on all prior stories
- **Total test count:** 31 tests (10 + 7 + 5 + 2 + 2 + 1 + 4 + 5 = 36 tests across all stories)

**Dependencies visualized:**

```
Story 1: Domain logic
  ├─> Story 2: API endpoint (after 1)
  ├─> Story 3: CLI command (after 1)
  ├─> Story 4: Queue handler (after 1)
  ├─> Story 5: Console notification (after 1)
  ├─> Story 6: Persistence test (after 1)
  └─> Story 7: Prevent operations (after 1, 2, 4)
        └─> Story 8: End-to-end validation (after 1-7)
```

**Open questions carried forward from HLD:**
1. Checkpoint locking: Does `AsyncSqliteSaver` support optimistic locking or transactions to prevent race conditions? If not, a `version` field may be needed in `State` for compare-and-swap logic.
2. Reviewer authentication: How is `reviewer_id` determined in API (auth token?) and CLI (argument? environment variable?) contexts?
3. Console backend endpoint: What is the exact URL and payload format for the console's activity feed notification? Verify in `src/agents/runtime/backend.py`.
4. Status key collision: Should queue handler write to a separate key or extend the existing `order-run:{thread_id}` payload? Clarify before implementing Story 4.

---

**Acceptance criteria alignment:**  
All 13 ACs from requirements.md are covered across the stories:
- AC 1: Stories 1, 2, 3
- AC 2: Stories 1, 2, 3, 4, 5, 7
- AC 3: Stories 1, 2, 3
- AC 4: Stories 2, 3, 4, 5
- AC 5: Story 8
