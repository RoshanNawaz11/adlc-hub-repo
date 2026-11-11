# ADLC Critic Re-check — PER-209

I have re-checked the documents against my earlier items and the sections listed in `changed-since-last-review.md`. I compared each earlier item to the current text and checked the changed sections for new problems.

## What would go wrong

### C1
- file: requirements.md
- section: ## Acceptance Criteria
- quote: (missing)
- severity: blocking
- problem: The fixer decided to normalize "waiting for input" → `needs_input` and "waiting for review" → `needs_review` in AD-2, but the requirements document was never updated to note this normalization. The ticket uses "waiting for input" and "waiting for review" in AC1, AC7, AC10, and the examples, but the requirements ACs still use those terms without explaining they become `needs_input`/`needs_review` in the API. Dev would read the requirements and build endpoints that return `waiting_for_input` instead of `needs_input`, or they would wonder which is right.
- change: Add a note at the top of the Acceptance Criteria section: "The ticket uses the terms 'waiting for input' and 'waiting for review' to match business language. The implementation normalizes these to `needs_input` and `needs_review` (matching the existing `State['status']` values in `states.py`). The API accepts both forms in query parameters but always returns the normalized forms in responses."

### C2
- file: stories/02-agent.md
- section: ## Algorithm Notes › `save_case`
- quote: `employee_name` ← `state.get('service_order', {}).get('subjectEmployee', {}).get('fullName') or state.get('service_order', {}).get('subject_employee', {}).get('full_name')` (matches `backend.employee_name` logic from `src/agents/runtime/backend.py`)
- match: repo-file/agent/src/agents/runtime/backend.py
- severity: blocking
- problem: The algorithm says this matches `backend.employee_name` logic, but `repo-file/agent/src/agents/runtime/backend.py` shows `employee_name(service_order)` returns `service_order.get('subjectEmployee', {}).get('fullName')` with no fallback to `subject_employee.full_name`. The story's extraction tries both snake_case and camelCase, but the real function only handles camelCase. Dev would build the wrong extraction path.
- change: Either use the real logic from `backend.py` (`state.get('service_order', {}).get('subjectEmployee', {}).get('fullName')` only), or if snake_case is needed, say "extends `backend.employee_name` by adding a fallback for snake_case keys" and explain why.

### C3
- file: stories/02-agent.md
- section: ## Dependencies
- quote: Employee name extraction logic is implemented directly in `save_case` by reading `state['service_order']['subjectEmployee']['fullName']` or `state['service_order']['subject_employee']['full_name']`, matching the structure used by `employee_name(service_order)` in `src/agents/runtime/backend.py`. No external call to `backend.employee_name` is required.
- match: stories/02-agent.md › ## Algorithm Notes › `save_case`
- severity: blocking
- problem: The Dependencies note says the extraction "matches the structure used by `employee_name`", but C2 shows it does not match (it adds a snake_case fallback the real function does not have). These two parts of the same document contradict each other about what the code does.
- change: Make the Dependencies note agree with the algorithm. If the algorithm is right (extraction includes both camelCase and snake_case), say "extends the `backend.employee_name` logic by adding a snake_case fallback". If the algorithm is wrong (should only use camelCase like the real function), fix the algorithm.

### C4
- file: breakdown.md
- section: ## Stories (Build Order) › Story 4
- quote: `employee_name` is derived via `backend.employee_name(state.get('service_order'))` inside `save_case`.
- match: stories/02-agent.md › ## Algorithm Notes › `save_case`
- severity: blocking
- problem: The breakdown says `employee_name` is derived via `backend.employee_name(...)`, but Story 2's algorithm (C2) shows it is extracted inline without calling that function. The breakdown and the story LLD disagree on how the field is populated.
- change: In the breakdown, change the note to match the story: "`employee_name` is extracted directly from `state['service_order']['subjectEmployee']['fullName']` inside `save_case` (see Story 2 algorithm notes)."

## Notes for the gate

### N1
- file: stories/03-agent.md
- section: ## Algorithm Notes › Helper Functions
- item: The `_validate_sort` helper raises `HTTPException` on error but does not return the validated `sort` value. The route algorithm in step 2 does not capture a return value (`_validate_sort(sort, statuses)` with no assignment). Either the helper should return `sort` and the route should use `sort = _validate_sort(sort, statuses)`, or the route should just reference the parameter `sort` directly after validation. The current code would work (validation side-effects are enough), but the pattern is inconsistent with `_validate_statuses` and `_validate_priorities` which do return values.

VERDICT: FAIL
