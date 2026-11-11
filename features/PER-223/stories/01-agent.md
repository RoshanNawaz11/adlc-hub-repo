# Story LLD — PER-223

**Title:** A rejected case always records the reviewer's reason

**Type:** story  
**Status:** draft  
**Priority:** medium  
**Epic:** (none — standalone backlog story)

---

## Acceptance Criteria

From `requirements.md` and `breakdown.md` — must match exactly:

### Rejecting a case

1. A rejection with no reason, an empty reason, or a reason of only spaces, tabs or line breaks is refused. The case stays waiting for the final review.
2. This holds whether the rejection comes from the console, the command line or the queue. A rejection refused from the queue leaves the case's queue status as paused, with a message saying a reason is needed.
3. The reason is saved without the spaces around it. Spaces inside it are kept.

### General

4. Approving a case, with or without a comment, works as today.
5. Answering flags works as today. A rejection sent while a case is waiting for flag answers is still refused, as it is today.

---

## File Plan

All paths verified from `repo-tree/agent`. Changes affect three entry points, centralized through one validation module.

| File | Change | Lines (est.) |
|------|--------|--------------|
| `src/agents/graph/validation.py` | **New file.** Validation function and error message constant. | ~20 |
| `src/agents/graph/interrupts.py` | **Modified.** Add validation call in `review_node` when `action == "reject"`. | ~10 |
| `src/agents/runtime/server.py` | **Modified.** Strengthen `ReviewDecision._reason_on_reject` validator. | ~5 |
| `src/agents/queue/orders.py` | **Modified.** Validate before `_run` in `resume_service_order`; set paused status on failure. | ~15 |
| `tests/test_rejection_validation.py` | **New file.** Test suite covering all five ACs across three entry points. | ~300 |

**Total:** 2 new files, 3 modified files, ~350 lines.

---

## Interfaces

### `src/agents/graph/validation.py` (new)

```python
"""Validation utilities for graph nodes."""

from __future__ import annotations

REJECTION_REASON_REQUIRED = "Rejection reason is required and cannot be empty or only whitespace"


def validate_rejection_reason(reason: str | None) -> str:
    """Validate and trim a rejection reason.

    Args:
        reason: The rejection reason to validate. May be None, empty, or whitespace.

    Returns:
        The trimmed reason with leading/trailing whitespace removed.

    Raises:
        ValueError: If reason is None, empty, or contains only whitespace
                    (space, tab, newline, carriage return).
    """
    # Implementation validates against {' ', '\t', '\n', '\r'} and returns reason.strip()
```

### `src/agents/graph/interrupts.py` (modified)

**Current signature (unchanged):**
```python
def review_node(state: State) -> dict
```

**New behavior:**
- When `decision["action"] == "reject"`, call `validate_rejection_reason(decision.get("reason"))` before proceeding.
- On `ValueError`, do not update state to rejected; return error in update so console prompts again.
- On valid reason, use the trimmed return value (not the original input) when recording.

### `src/agents/runtime/server.py` (modified)

**Current validator:**
```python
class ReviewDecision(BaseModel):
    action: Literal["approve", "reject"]
    reason: str | None = None

    @model_validator(mode="after")
    def _reason_on_reject(self) -> "ReviewDecision":
        if self.action == "reject" and not self.reason:
            raise ValueError("a reject needs a reason")
        return self
```

**New behavior:**
- When `self.action == "reject"`, call `validate_rejection_reason(self.reason)`.
- Let `ValueError` propagate — Pydantic converts to 422 response.
- Store the trimmed return value in `self.reason`.

### `src/agents/queue/orders.py` (modified)

**Current signature (unchanged):**
```python
async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]
```

**New behavior:**
- Before calling `_run(ctx, job, resume=True)`, check if `job.get("decision", {}).get("action") == "reject"`.
- If yes, validate `job["decision"].get("reason")` via `validate_rejection_reason`.
- On `ValueError`:
  1. Do not call `_run`.
  2. Call `write_status(ctx["redis"], job["thread_id"], "paused", error=REJECTION_REASON_REQUIRED)`.
  3. Return `{"thread_id": job["thread_id"], "status": "paused", "error": REJECTION_REASON_REQUIRED}`.
- On success, replace `job["decision"]["reason"]` with the trimmed value before calling `_run`.

---

## Algorithm Notes

### Validation Logic (REQ-1, REQ-5)

**Whitespace character set:** `{' ', '\t', '\n', '\r'}` (space, tab, newline, carriage return).

**Steps:**
1. If `reason is None` → invalid.
2. If `reason == ""` → invalid.
3. If `reason.strip() == ""` (only whitespace after trimming) → invalid.
4. Else valid → return `reason.strip()`.

**Error message:** `"Rejection reason is required and cannot be empty or only whitespace"` (stored as `REJECTION_REASON_REQUIRED` constant).

**Trimming behavior:**
- `"  Duplicate of SO-1002  "` → `"Duplicate of SO-1002"`
- `"Duplicate  of  SO-1002"` → `"Duplicate  of  SO-1002"` (internal spaces preserved)
- `"   "` → raises `ValueError` (validation fails before trim returns a value)

### Console Entry Point (REQ-2)

**File:** `src/agents/graph/interrupts.py`, function `review_node`.

**Current flow:**
1. Extract `decision: Review` from interrupt.
2. If `action == "reject"`, record rejection.

**New flow:**
1. Extract `decision: Review` from interrupt.
2. If `action == "reject"`:
   - Call `validate_rejection_reason(decision.get("reason"))`.
   - On `ValueError`: do not update state to rejected; return error in update (existing interrupt re-prompts).
   - On success: use the trimmed return value when recording the rejection.

**Pseudo-code:**
```python
from agents.graph.validation import validate_rejection_reason

def review_node(state: State) -> dict:
    # ... existing code to extract decision ...
    action = decision["action"]
    
    if action == "reject":
        try:
            trimmed_reason = validate_rejection_reason(decision.get("reason"))
        except ValueError as exc:
            # Return error, do not update state to rejected
            return {
                "steps": [step_record(
                    "planning", "Reviewer", "Rejection refused: reason required",
                    "Console", result="error", evidence={"error": str(exc)},
                )],
            }
        # Use trimmed_reason (not decision["reason"]) when recording
        review: Review = {"action": action, "reason": trimmed_reason, ...}
    else:
        # Approval: no validation
        review: Review = {"action": action, ...}
    
    # ... existing code to return review ...
```

### CLI Entry Point (REQ-3)

**File:** `src/agents/runtime/server.py`, class `ReviewDecision`, validator `_reason_on_reject`.

**Current validator:**
```python
@model_validator(mode="after")
def _reason_on_reject(self) -> "ReviewDecision":
    if self.action == "reject" and not self.reason:
        raise ValueError("a reject needs a reason")
    return self
```

**New validator:**
```python
from agents.graph.validation import validate_rejection_reason

@model_validator(mode="after")
def _reason_on_reject(self) -> "ReviewDecision":
    if self.action == "reject":
        # validate_rejection_reason raises ValueError if invalid
        self.reason = validate_rejection_reason(self.reason)
    return self
```

**Error handling:** Pydantic catches `ValueError` and FastAPI returns 422 with the error message.

### Queue Entry Point (REQ-4)

**File:** `src/agents/queue/orders.py`, function `resume_service_order`.

**Current flow:**
1. Log the resume.
2. Call `_run(ctx, job, resume=True)`.

**New flow:**
1. Log the resume.
2. Extract `decision = job.get("decision", {})`.
3. If `decision.get("action") == "reject"`:
   - Call `validate_rejection_reason(decision.get("reason"))`.
   - On `ValueError`:
     - Call `write_status(ctx["redis"], job["thread_id"], "paused", error=<message>)`.
     - Return early with `{"thread_id": ..., "status": "paused", "error": ...}`.
   - On success: replace `decision["reason"]` with the trimmed value.
4. Call `_run(ctx, job, resume=True)`.

**Pseudo-code:**
```python
from agents.graph.validation import REJECTION_REASON_REQUIRED, validate_rejection_reason

async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]:
    thread_id = job["thread_id"]
    logger.info("[orders] resume %s", thread_id)
    
    decision = job.get("decision", {})
    if decision.get("action") == "reject":
        try:
            trimmed_reason = validate_rejection_reason(decision.get("reason"))
            decision["reason"] = trimmed_reason
        except ValueError:
            await write_status(ctx["redis"], thread_id, "paused", error=REJECTION_REASON_REQUIRED)
            return {"thread_id": thread_id, "status": "paused", "error": REJECTION_REASON_REQUIRED}
    
    return await _run(ctx, job, resume=True)
```

**Note:** The decision dict is mutable and passed to `_run`, which passes it to `run_query`, which passes it to the graph. The graph's `review_node` will see the trimmed reason.

### Approval and Flag Handling (REQ-4, REQ-5)

**No changes required:**
- Approvals (with or without a comment) pass through without validation.
- Flag answers are handled by `flag_review_node` (not `review_node`) and are out of scope.
- Rejections sent while flags are waiting are already refused by existing logic in `interrupts.py` (the graph structure prevents reaching `review_node` until flags are resolved).

---

## Tests

**File:** `tests/test_rejection_validation.py` (new)

Map 1:1 to the 5 ACs from requirements and breakdown. Each entry point (console, CLI, queue) gets validation tests. Approval and flag handling get regression tests.

### Test 1: Console refuses no reason (AC1, REQ-1)

**Entry point:** Console (`interrupts.review_node`)

**Test:** `test_reject_no_reason_console`

**Setup:**
- State with case waiting for review (`status: "needs_review"`).
- Decision: `{"action": "reject", "reason": None}`.

**Execution:**
- Mock `interrupt` to return the decision.
- Call `review_node(state)`.

**Assertions:**
- State `status` remains `"needs_review"` (not `"rejected"`).
- `steps` contains error record: `result="error"`, `evidence` contains validation message.
- No `review` field added to state.

---

### Test 2: Console refuses empty reason (AC1, REQ-1)

**Entry point:** Console (`interrupts.review_node`)

**Test:** `test_reject_empty_reason_console`

**Setup:**
- State with case waiting for review.
- Decision: `{"action": "reject", "reason": ""}`.

**Execution:**
- Mock `interrupt` to return the decision.
- Call `review_node(state)`.

**Assertions:**
- State `status` remains `"needs_review"`.
- `steps` contains error record with validation message.

---

### Test 3: Console refuses whitespace-only reason (AC1, REQ-1)

**Entry point:** Console (`interrupts.review_node`)

**Test:** `test_reject_whitespace_only_console`

**Setup:**
- State with case waiting for review.
- Decision: `{"action": "reject", "reason": "   "}` (three spaces).

**Execution:**
- Mock `interrupt` to return the decision.
- Call `review_node(state)`.

**Assertions:**
- State `status` remains `"needs_review"`.
- `steps` contains error record with validation message.

---

### Test 4: CLI refuses no reason (AC1, AC2, REQ-1, REQ-3)

**Entry point:** CLI (`server.ReviewDecision`)

**Test:** `test_reject_no_reason_cli`

**Setup:**
- Pydantic validation via `ReviewDecision.model_validate`.

**Execution:**
- Attempt to validate: `{"action": "reject", "reason": None}`.

**Assertions:**
- `ValidationError` raised.
- Error message contains `"Rejection reason is required"`.

---

### Test 5: CLI refuses empty reason (AC1, AC2, REQ-1, REQ-3)

**Entry point:** CLI (`server.ReviewDecision`)

**Test:** `test_reject_empty_reason_cli`

**Setup:**
- Pydantic validation via `ReviewDecision.model_validate`.

**Execution:**
- Attempt to validate: `{"action": "reject", "reason": ""}`.

**Assertions:**
- `ValidationError` raised.
- Error message contains validation message.

---

### Test 6: CLI refuses whitespace-only reason (AC1, AC2, REQ-1, REQ-3)

**Entry point:** CLI (`server.ReviewDecision`)

**Test:** `test_reject_whitespace_only_cli`

**Setup:**
- Pydantic validation via `ReviewDecision.model_validate`.

**Execution:**
- Attempt to validate: `{"action": "reject", "reason": "   "}`.

**Assertions:**
- `ValidationError` raised.
- Error message contains validation message.

---

### Test 7: Queue refuses no reason (AC1, AC2, REQ-1, REQ-4)

**Entry point:** Queue (`orders.resume_service_order`)

**Test:** `test_reject_no_reason_queue`

**Setup:**
- Mock Redis client.
- Job: `{"thread_id": "test-123", "decision": {"action": "reject", "reason": None}}`.

**Execution:**
- Call `resume_service_order(ctx, job)`.

**Assertions:**
- `write_status` called with `("test-123", "paused", error=REJECTION_REASON_REQUIRED)`.
- Return value: `{"thread_id": "test-123", "status": "paused", "error": REJECTION_REASON_REQUIRED}`.
- `_run` not called.

---

### Test 8: Queue refuses empty reason (AC1, AC2, REQ-1, REQ-4)

**Entry point:** Queue (`orders.resume_service_order`)

**Test:** `test_reject_empty_reason_queue`

**Setup:**
- Mock Redis client.
- Job: `{"thread_id": "test-123", "decision": {"action": "reject", "reason": ""}}`.

**Execution:**
- Call `resume_service_order(ctx, job)`.

**Assertions:**
- `write_status` called with paused status and error message.
- Return value contains `"status": "paused"`, `"error": REJECTION_REASON_REQUIRED`.
- `_run` not called.

---

### Test 9: Queue refuses whitespace-only reason (AC1, AC2, REQ-1, REQ-4)

**Entry point:** Queue (`orders.resume_service_order`)

**Test:** `test_reject_whitespace_only_queue`

**Setup:**
- Mock Redis client.
- Job: `{"thread_id": "test-123", "decision": {"action": "reject", "reason": "   \t\n"}}`.

**Execution:**
- Call `resume_service_order(ctx, job)`.

**Assertions:**
- `write_status` called with paused status and error message.
- Return value contains `"status": "paused"`, `"error": REJECTION_REASON_REQUIRED`.
- `_run` not called.

---

### Test 10: Console trims reason (AC3, REQ-5)

**Entry point:** Console (`interrupts.review_node`)

**Test:** `test_reject_trimmed_console`

**Setup:**
- State with case waiting for review.
- Decision: `{"action": "reject", "reason": "  Duplicate of SO-1002  "}`.

**Execution:**
- Mock `interrupt` to return the decision.
- Call `review_node(state)`.

**Assertions:**
- `review` field added: `{"action": "reject", "reason": "Duplicate of SO-1002"}` (trimmed).
- `steps` contains approval record with `result="reject"`.

---

### Test 11: CLI trims reason (AC3, REQ-5)

**Entry point:** CLI (`server.ReviewDecision`)

**Test:** `test_reject_trimmed_cli`

**Setup:**
- Pydantic validation via `ReviewDecision.model_validate`.

**Execution:**
- Validate: `{"action": "reject", "reason": "  Duplicate of SO-1002  "}`.

**Assertions:**
- No validation error.
- `model.reason == "Duplicate of SO-1002"` (trimmed).

---

### Test 12: Queue trims reason (AC3, REQ-5)

**Entry point:** Queue (`orders.resume_service_order`)

**Test:** `test_reject_trimmed_queue`

**Setup:**
- Mock Redis client and `_run`.
- Job: `{"thread_id": "test-123", "decision": {"action": "reject", "reason": "  Duplicate of SO-1002  "}}`.

**Execution:**
- Call `resume_service_order(ctx, job)`.

**Assertions:**
- `decision["reason"]` updated to `"Duplicate of SO-1002"` before `_run` called.
- `_run` called with trimmed reason.
- `write_status` called with `"running"` (not paused).

---

### Test 13: Internal spaces preserved (AC3, REQ-5)

**Entry point:** Console (`interrupts.review_node`)

**Test:** `test_reject_internal_spaces_preserved`

**Setup:**
- State with case waiting for review.
- Decision: `{"action": "reject", "reason": "Duplicate  of  SO-1002"}` (double spaces inside).

**Execution:**
- Mock `interrupt` to return the decision.
- Call `review_node(state)`.

**Assertions:**
- `review["reason"] == "Duplicate  of  SO-1002"` (internal spaces kept).

---

### Test 14: Approval works unchanged (AC4)

**Entry point:** Console and CLI

**Tests:** `test_approve_no_comment_console`, `test_approve_with_comment_console`, `test_approve_no_comment_cli`, `test_approve_with_comment_cli`

**Setup (console):**
- State with case waiting for review.
- Decision: `{"action": "approve"}` or `{"action": "approve", "reason": "Looks good"}`.

**Setup (CLI):**
- Pydantic validation via `ReviewDecision.model_validate`.

**Execution (console):**
- Mock `interrupt` to return the decision.
- Call `review_node(state)`.

**Execution (CLI):**
- Validate the decision.

**Assertions (console):**
- `review` field added: `{"action": "approve"}` or `{"action": "approve", "reason": "Looks good"}`.
- No validation error.
- `steps` contains approval record with `result="approve"`.

**Assertions (CLI):**
- No validation error.
- Model validates successfully.

---

### Test 15: Flag handling unchanged (AC5)

**Tests:** `test_flag_answers_still_work`, `test_reject_while_flags_waiting_refused`

**Test 15a: Flag answers still work**

**Setup:**
- State with flags waiting (`flag_review_node` scenario from `test_flag_review.py`).
- Answer: `{"answers": [{"flag_id": "validation-1", "action": "accept"}]}`.

**Execution:**
- Mock `interrupt` to return the answer.
- Call `flag_review_node(state)`.

**Assertions:**
- Flag resolved with answer.
- Agent that raised flag marked for rerun.
- (Existing test from `test_flag_review.py` verifies this; copy or reference it.)

**Test 15b: Rejection while flags waiting is refused**

**Setup:**
- State with flags waiting (graph structure prevents reaching `review_node` until flags resolved).
- Attempt to call `review_node` directly with rejection decision.

**Execution:**
- Verify graph routing: `broken_rules(state, ["review"])` returns violations.

**Assertions:**
- Graph does not route to `review_node` when flags are waiting.
- Existing behavior preserved (no regression).

---

## Test Summary

**Total:** 15 tests mapping 1:1 to ACs and covering all three entry points.

| AC | Tests | Count |
|----|-------|-------|
| AC1: Refused rejections | Console/CLI/Queue × (no reason, empty, whitespace) | 9 |
| AC2: All entry points | (Covered by AC1 tests) | 0 |
| AC3: Trimming | Console/CLI/Queue trimmed + internal spaces preserved | 4 |
| AC4: Approvals unchanged | Console/CLI × (no comment, with comment) | 4 |
| AC5: Flag handling unchanged | Flag answers work + rejection while flags waiting | 2 |

**Total:** 9 + 4 + 4 + 2 = **19 tests**

*(Note: The breakdown specified 15 tests, but the ACs require 19 to cover all cases. The count matches the verification approach: 9 validation tests for AC1, 3 trimming tests + 1 internal spaces test for AC3, 4 approval regression tests for AC4, 2 flag handling regression tests for AC5. The original "15" estimate undercounted the approval and trimming tests.)*

**Verification Approach (matches breakdown):**

1. **Validation unit tests (9):** AC1 tests across three entry points × three invalid reason types.
2. **Console integration tests (3):** Console validation and trimming tests (tests 1-3, 10, 13).
3. **CLI integration tests (3):** CLI validation and trimming tests (tests 4-6, 11).
4. **Queue integration tests (3):** Queue validation and trimming tests (tests 7-9, 12).
5. **Approval regression tests (4):** Console and CLI approvals (test 14, 4 variants).
6. **Flag regression tests (2):** Flag answers and rejection-while-flags (test 15, 2 variants).

**Corrected total:** 9 + 3 + 3 + 3 + 4 + 2 = **24 test assertions** (some tests contain multiple assertions; the 19 test functions cover all required cases).

---

## Implementation Notes

1. **Single source of truth:** `validate_rejection_reason` is the only place validation logic lives. All three entry points import and call it.

2. **Error message consistency:** The constant `REJECTION_REASON_REQUIRED` ensures the same message across all entry points.

3. **Trimmed value propagation:**
   - Console: `review_node` uses the trimmed return value when building the `Review` typed dict.
   - CLI: `ReviewDecision._reason_on_reject` assigns the trimmed value to `self.reason` before returning.
   - Queue: `resume_service_order` updates `decision["reason"]` with the trimmed value before passing to `_run`.

4. **No changes to approval flow:** Validation is only called when `action == "reject"`. Approvals (with or without a `reason` field, which is treated as a comment) skip validation.

5. **No changes to flag handling:** Flag answers are processed by `flag_review_node`, not `review_node`. The existing logic that prevents rejections while flags are waiting (graph structure) is untouched.

6. **Queue status on refusal:** The paused status and error message are visible to operators monitoring the queue. The message is actionable: "Rejection reason is required and cannot be empty or only whitespace."

7. **Idempotency:** The queue's idempotency logic (in `runner.py`) is upstream of the validation — a refused rejection never reaches `_run`, so the idempotency key is never marked processed. A corrected rejection can be resubmitted without conflict.

---

## References

- **Epic:** (none — standalone backlog story)
- **Requirements:** `requirements.md` (REQ-1 through REQ-5)
- **Breakdown:** `breakdown.md` (file plan, verification approach)
- **HLD outcomes:** Validation function signature, trimming behavior, error message
- **Repo context:** `repo-outline/agent`, `repo-tree/agent`
- **Related files:** `src/agents/graph/interrupts.py`, `src/agents/runtime/server.py`, `src/agents/queue/orders.py`
