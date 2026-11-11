# ADLC spec critic re-check

I will re-check each earlier item against the current files, and raise new items only inside changed sections or where those changes disagree with other documents.

---

## C1: Terminology note in requirements.md

**File:** requirements.md  
**Section:** ## Acceptance Criteria  
**Current text (first paragraph):**

> **Note on terminology:** The ticket uses the terms "waiting for input" and "waiting for review" to match business language. The implementation normalizes these to `needs_input` and `needs_review` (matching the existing `State['status']` values in `states.py`). The API accepts both forms in query parameters but always returns the normalized forms in responses.

**Verdict:** Fixed. The note is present and matches the decision. No disagreement with other documents.

---

## C2: Employee name extraction in stories/02-agent.md

**File:** stories/02-agent.md  
**Section:** ## Algorithm Notes › `save_case`  
**Current text (step 2, `employee_name` line):**

> - `employee_name` ← `state.get('service_order', {}).get('subjectEmployee', {}).get('fullName') or state.get('service_order', {}).get('subject_employee', {}).get('full_name')` (matches `backend.employee_name` logic from `src/agents/runtime/backend.py`)

**Checked against:** I need to see what `backend.employee_name` really does.

**From repo-file/agent/src/agents/runtime/backend.py** (lines 1–40):

```python
def employee_name(service_order: dict | None) -> str | None:
    """Extract the employee's full name from the service order.
    
    Args:
        service_order: The service order dictionary.
    
    Returns:
        The employee's full name, or None if not found.
    """
    if not service_order:
        return None
    subject = service_order.get('subjectEmployee') or service_order.get('subject_employee')
    if not subject:
        return None
    return subject.get('fullName') or subject.get('full_name')
```

**Problem:** The story's extraction logic uses a **different order** than the real function. The story says:

```python
state.get('service_order', {}).get('subjectEmployee', {}).get('fullName') or state.get('service_order', {}).get('subject_employee', {}).get('full_name')
```

This tries `subjectEmployee.fullName` first, and only falls back to `subject_employee.full_name` if the first chain returned a falsy value (including `{}`). But the **real function** in `backend.py` first resolves **which key exists** (`subjectEmployee` or `subject_employee`), then tries both name keys on that one object. The story's logic would fail if `service_order = {'subjectEmployee': {}}` (no `fullName`), because the first chain returns `{}` (truthy), so the `or` never tries the snake_case fallback.

**Severity:** blocking. A developer following the story would write extraction logic that disagrees with the real `backend.employee_name` function and would fail when `subjectEmployee` exists but has no `fullName` key.

**Match:** backend.py › `employee_name` function.

---

## C3: Dependencies note in stories/02-agent.md

**File:** stories/02-agent.md  
**Section:** ## Dependencies  
**Current text (last paragraph):**

> **Note:** Employee name extraction logic is implemented directly in `save_case` by reading `state['service_order']['subjectEmployee']['fullName']`, matching the exact structure used by `employee_name(service_order)` in `src/agents/runtime/backend.py`. No external call to `backend.employee_name` is required.

**Problem:** This note says the logic "matches the exact structure" of `backend.employee_name`, but the algorithm in the same file (see C2 above) uses a different order that would fail in the `subjectEmployee` exists but has no `fullName` case. The note is wrong either way: if the algorithm is right (handles snake_case), the note should say "extends" not "matches"; if the note is right (matches exactly), the algorithm is wrong.

**Severity:** blocking. This is the same disagreement as C2, but in the Dependencies section instead of the algorithm.

**Match:** Same issue as C2; the note contradicts the algorithm in the same file and the real code in `backend.py`.

---

## New items inside changed sections

I checked every changed section listed in `changed-since-last-review.md`:

- **requirements.md › ## Acceptance Criteria:** The terminology note is present and correct (C1 above). No new problems.
- **stories/02-agent.md › ## Algorithm Notes › `save_case`:** Employee name extraction disagrees with the real function (C2 above).
- **stories/02-agent.md › ## Dependencies:** The note contradicts the algorithm and the real code (C3 above).

No other disagreements or new problems in those sections.

---

## Notes for the gate

None. Every earlier note-level item (C4) was already passed to the gate in the last review and does not need re-raising.

---

## What would go wrong

### C2
- file: stories/02-agent.md
- section: ## Algorithm Notes › `save_case`
- quote: `employee_name` ← `state.get('service_order', {}).get('subjectEmployee', {}).get('fullName') or state.get('service_order', {}).get('subject_employee', {}).get('full_name')`
- match: backend.py › `employee_name` function
- severity: blocking
- problem: The story's extraction tries `subjectEmployee.fullName` first and only falls back to `subject_employee.full_name` if the first chain is falsy. But when `service_order = {'subjectEmployee': {}}` (key exists, no `fullName`), the first chain returns `{}` (truthy), so the `or` never tries the snake_case keys. The real `backend.employee_name` function resolves which top-level key exists first (`subjectEmployee` or `subject_employee`), then tries both name keys on that one object. A developer following the story would write logic that fails when the camelCase keys exist but are empty.
- change: Use the real function's logic: `subject = service_order.get('subjectEmployee') or service_order.get('subject_employee')` (resolve top-level key first), then `subject.get('fullName') or subject.get('full_name')` (try both name keys on the resolved object). Or call `backend.employee_name(state.get('service_order'))` directly instead of reimplementing it.

### C3
- file: stories/02-agent.md
- section: ## Dependencies
- quote: Employee name extraction logic is implemented directly in `save_case` by reading `state['service_order']['subjectEmployee']['fullName']`, matching the exact structure used by `employee_name(service_order)` in `src/agents/runtime/backend.py`.
- match: backend.py › `employee_name` function, and the same file's ## Algorithm Notes › `save_case`
- severity: blocking
- problem: The note says the extraction "matches the exact structure" of `backend.employee_name`, but the algorithm in this file (step 2 of `save_case`) uses a different order that would fail when `subjectEmployee` exists but has no `fullName` (see C2). The note contradicts the algorithm and the real code.
- change: If the algorithm is fixed to match the real function (resolve top-level key first, then try both name keys), update the note to say "matches the logic" and include the snake_case fallback in the description. If the algorithm stays as-is (tries each chain in sequence), change the note to "extends `backend.employee_name` by handling snake_case" and fix the algorithm to resolve the top-level key first (like the real function does).

---

VERDICT: FAIL
