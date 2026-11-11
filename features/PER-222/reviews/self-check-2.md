# ADLC spec critic

You are the senior engineer on this team who reads the whole spec before a developer picks it up.

## What would go wrong

### C1
- file: stories/01-agent.md
- section: ## File Plan
- quote: Add `type: NotRequired[str]`, `by: NotRequired[str]`, and `reason: NotRequired[str]` fields to `StepRecord` TypedDict (after line ~97, before `plan`):
- match: repo-file/agent/src/agents/graph/states.py › class StepRecord(TypedDict)
- severity: blocking
- blocks because: The `StepRecord` TypedDict shown in `repo-file/agent/src/agents/graph/states.py` ends at line 45, and the fields listed come before `plan` at line 97, which is in the `State` TypedDict, not `StepRecord`. Dev would look for line 97 in the wrong class and either fail to find the right location or put the fields in the wrong TypedDict.
- problem: The file plan says "after line ~97, before `plan`" but `StepRecord` ends at line 45 and the `plan` field is in `State` (line 97), not in `StepRecord`.
- change: Say "after line ~45, after the existing `pipeline` field" or "Add three new `NotRequired` fields to `StepRecord` TypedDict (after the existing fields, before the closing)".

---

### C2
- file: stories/01-agent.md
- section: ## File Plan
- quote: The modification to `StepRecord` is simpler and follows the pattern of other `NotRequired` fields already present (e.g. `result`, `evidence`, `rule`).
- match: repo-file/agent/src/agents/graph/states.py › class StepRecord(TypedDict)
- severity: blocking
- blocks because: The existing `StepRecord` fields shown in `repo-file` are `stage`, `agent`, `title`, `source`, `at` (all required), then `result`, `evidence`, `feed`, `rule`, `pipeline` (all `NotRequired`). Dev would add `type`, `by`, `reason` as additional `NotRequired` fields after these. However, the `feed` field (line 42) is also `NotRequired` but is not mentioned in the LLD's list of examples. If the pattern includes all `NotRequired` fields, the list should be complete; if it's just examples, it should say "e.g." more clearly.
- problem: The LLD says the new fields follow the pattern of `NotRequired` fields like `result`, `evidence`, `rule`, but omits `feed` and `pipeline` from the list. The list is incomplete or unclear whether it's exhaustive.
- change: Say "follows the pattern of other `NotRequired` fields already present (`result`, `evidence`, `feed`, `rule`, `pipeline`)" or "follows the pattern of other `NotRequired` fields (e.g. `result`, `evidence`, `rule`)" with "e.g." to show it's not exhaustive.

---

### C3
- file: stories/02-agent.md
- section: ## File Plan
- quote: 1. Add imports (after line ~19, with other imports):
- match: repo-file/agent/src/agents/runtime/server.py
- severity: blocking
- blocks because: `repo-file/agent/src/agents/runtime/server.py` shows imports ending around line 31 (`from agents.runtime.runner import _setup_logging, open_graph, run_query`), not line 19. Line 19 is in the middle of the imports (`from typing import Any, Literal`). Dev would place the new imports after line 19 (mid-block) instead of after line 31 (end of imports), breaking Python's import grouping convention.
- problem: The file plan says "after line ~19, with other imports" but line 19 is mid-import-block, and imports end around line 31. The new imports should go after line 31.
- change: Say "after line ~31, after the existing imports" or "Add imports at the top (after the existing imports)".

---

### C4
- file: stories/02-agent.md
- section: ## File Plan
- quote: 3. Add `POST /runs/{thread_id}/withdraw` route (after `resume_run` endpoint, around line ~150):
- match: repo-file/agent/src/agents/runtime/server.py
- severity: blocking
- blocks because: The `resume_run` endpoint in `repo-file/agent/src/agents/runtime/server.py` ends at line 148. Line 150 is inside the `get_run` endpoint definition (line 151 starts `async def get_run`). Dev would insert the new route inside `get_run` instead of between `resume_run` and `get_run`.
- problem: The file plan says "after `resume_run` endpoint, around line ~150" but `resume_run` ends at line 148, and line 150 is the blank line before `get_run` starts at line 151. Saying "around line 150" implies inside that blank line or at the start of `get_run`.
- change: Say "after `resume_run` endpoint (after line ~148, before `get_run`)" or "Add a new route between `resume_run` and `get_run`".

---

### C5
- file: stories/02-agent.md
- section: ## Interfaces
- quote: **Response (200 OK):**
```json
{
  "status": "withdrawn",
  "reason": "Duplicate of SO-1003",
  "withdrawn_at": "2026-10-15T14:32:00Z"
}
```
- match: hld.md › ## Component Interaction › Withdrawal Endpoint (API)
- severity: blocking
- blocks because: The HLD's API contract shows the error responses as `{"error": "<message>"}`, but the success response shown here uses the keys `status`, `reason`, `withdrawn_at` directly (no wrapping key). The endpoint implementation in the File Plan returns `result` (from `withdraw_case`), which is `{"status": "withdrawn", "reason": "<trimmed>", "withdrawn_at": "<timestamp>"}`. Dev would build this as written, but if FastAPI's response model or a middleware wraps it differently, the actual response shape could differ. The HLD and LLD agree on this shape, so this is fine unless there's an implicit wrapping.
- problem: (No problem — checked and consistent.)
- change: (No change needed.)

---

### C6
- file: stories/02-agent.md
- section: ## Tests Section
- quote: **Test 1: Successful withdrawal from `needs_input`**
```python
@pytest.mark.asyncio
async def test_withdraw_from_needs_input_returns_200(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None
    """AC 1: A case waiting for flag answers can be withdrawn via API, returns 200 OK."""
    thread_id = "test-api-1"
    # Mock withdraw_case to return success
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        return {"status": "withdrawn", "reason": "Test reason", "withdrawn_at": "2026-10-15T14:32:00Z"}
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "Test reason"})
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "withdrawn"
    assert data["reason"] == "Test reason"
    assert "withdrawn_at" in data
```
- match: None (test code, no repo-file to check against)
- severity: blocking
- blocks because: The test mocks `withdraw_case` at the path `"agents.runtime.server.withdraw_case"`, but the File Plan shows the import in `server.py` as `from agents.runtime.withdrawal import CaseNotFoundError, InvalidStateError, withdraw_case`. The function is imported into `server.py`'s namespace as `withdraw_case`, so the monkeypatch path should be `"agents.runtime.server.withdraw_case"` (correct as written), but the test doesn't mock `run_dir` (which `withdraw_case` calls to resolve the `db_path`). If `withdraw_case` is mocked, `run_dir` is never called, so that's fine. However, the test doesn't set up a checkpointer or state, so if the mock is wrong and the real `withdraw_case` runs, it will fail. The test pattern is sound: it mocks the function the endpoint calls. No problem here.
- problem: (No problem — checked and the mock path is correct.)
- change: (No change needed.)

---

### C7
- file: stories/07-agent.md
- section: ## File Plan
- quote: **No changes to `resume_service_order`:**
Story 4 already added the withdrawn check to `resume_service_order` (around line 93). That check reads the state from the checkpointer before calling `_run`, and if `status == "withdrawn"`, it skips graph invocation and writes the queue status as `"withdrawn"` (not `"failed"`). The existing implementation from Story 4 handles resume operations on withdrawn cases; this story (Story 7) does not modify `resume_service_order`.
- match: stories/04-agent.md › ## File Plan › Modified Files › src/agents/queue/orders.py
- severity: note
- blocks because: (Not blocking — this is consistent now, as requested in C3 from the last review.)
- problem: Story 4's file plan adds the withdrawn check to `resume_service_order`, and Story 7 correctly states "no changes" to that function. The division of labor is clear: Story 4 handles resume (queue + API both delegate to the queue handler's check), Story 7 handles start/approve/reject. This is now consistent.
- change: (No change needed — the clarification was made.)

---

### C8
- file: stories/03-agent.md
- section: ## File Plan
- quote: **Imports:**
```python
from __future__ import annotations

import argparse
import asyncio
import logging
import os
import sys
from pathlib import Path

# Ensure src/ is on the path (matches the pattern in runner.py and server.py)
_SRC_ROOT = str(Path(__file__).resolve().parent.parent)
if _SRC_ROOT not in sys.path:
    sys.path.insert(0, _SRC_ROOT)

from agents.runtime.withdrawal import CaseNotFoundError, InvalidStateError, withdraw_case
from agents.runtime.run_dir import run_dir

logger = logging.getLogger(__name__)
```
- match: repo-file/agent/src/agents/runtime/runner.py › _SRC_ROOT
- severity: blocking
- blocks because: The CLI module is at `src/agents/cli.py` (new file), so `Path(__file__).resolve().parent` is `src/agents/`, and `.parent` again is `src/`. But `runner.py` shows `_SRC_ROOT = str(Path(__file__).resolve().parent.parent.parent)` (three `.parent` calls), because `runner.py` is at `src/agents/runtime/runner.py` (three levels deep: `runtime/` → `agents/` → `src/`). The CLI is two levels deep (`agents/` → `src/`), so it needs `.parent.parent`, not `.parent.parent`. Dev would copy the pattern and use two `.parent` calls, which gives `src/` correctly, but the comment says "matches the pattern in runner.py", which uses three. Either the code or the comment is wrong.
- problem: The LLD says the path setup "matches the pattern in runner.py and server.py", but `runner.py` uses `.parent.parent.parent` (three levels), and `cli.py` is two levels deep (should use `.parent.parent`). The code shown (`.parent.parent`) is correct for `cli.py`, but the comment claiming it matches `runner.py` is wrong.
- change: Change the comment to "Ensure src/ is on the path" (without "matches the pattern in runner.py") or show the correct number of `.parent` calls and note "two levels deep, so `.parent.parent`".

---

## Notes for the gate

### N1
The HLD specifies the API error response key as `"error"` (e.g. `{"error": "Reason is required"}`), but FastAPI's default `HTTPException` returns `{"detail": "<message>"}`. The fixer added a custom exception handler to `server.py` (per the `decided` note in `critic-items.md` → C1) to map `HTTPException` to `{"error": "<message>"}`. Verify that the handler is present in the implementation when Story 2 is coded, and that all error responses (400, 404, 409) return `{"error": ...}` as the HLD shows. If the handler is missing, the API will return `{"detail": ...}` instead, breaking the contract.

### N2
The LLD for Story 5 (console notification) says the notification event includes `"employee_name": employee` (extracted from `service_order.subjectEmployee.fullName` or `service_order.subject_employee.full_name`). The existing `employee_name` function in `backend.py` (line 35) tries both paths. If the console's service order schema uses a different field name or structure, the employee name will be `None` in the notification. The LLD notes this as acceptable (the notification still succeeds), but the console team should verify the field path before this ships.

### N3
The breakdown says Story 7 (prevent operations on withdrawn cases) has 4 tests, and the LLD shows 4 test functions. However, Test 6 in Story 2's LLD uses `@pytest.mark.parametrize("status", ["done", "rejected", "failed"])`, which pytest counts as 3 test cases (one per parameter value). The breakdown's test count for Story 2 is 7, and the LLD lists 7 test function names, but the parameterized test will report as 3 when run. This is a counting style difference (stories count "test functions", pytest counts "test cases"), not a missing test. Just note it for reconciliation: the total test count when run will be higher than the sum of story test counts.

### N4
The HLD (AD-6) and Story 4 say the queue handler writes `{"status": "withdrawn", "reason": "<reason>", "withdrawn_at": "<timestamp>"}` to Redis for a withdrawn case's resume job. Story 7 says the start handler writes `{"status": "failed", "error": "<message>"}` for a withdrawn case's start job. The Redis key (`order-run:{thread_id}`) is the same, so a start attempt after a resume will overwrite the "withdrawn" status with "failed". This is probably fine (the case is still terminal, and "failed" with a "case was withdrawn" error is clear), but if the console expects "withdrawn" to be stable, the start handler should also write `"status": "withdrawn"` instead of `"failed"`. The open question in Story 7 (risk 2) mentions this. Check with the console team whether "withdrawn" must be the final status or "failed" is acceptable.

### N5
Story 1's LLD says the `withdraw_case` function returns a dict with keys `status`, `reason`, `withdrawn_at`. Story 2's API endpoint returns this dict directly (as JSON). The HLD's API contract shows the same keys. This is consistent. However, the `State` TypedDict has four withdrawal fields (`status`, `withdrawn_at`, `withdrawn_by`, `withdrawal_reason`), and the function returns only three (`status`, `withdrawn_at`, `reason` — note `reason`, not `withdrawal_reason`, and no `withdrawn_by`). The mismatch is intentional: the return value is the API response shape, not the full state. Just confirm that the console doesn't need `withdrawn_by` in the response. If it does, the API contract should include it.

VERDICT: FAIL
