# Open items from the critic for PER-209

Written by the critic, not approved by a person. Delete the items you disagree with and edit the rest; what is left is applied on the next run:
`adlc resume PER-209` (or `adlc start PER-209 --skip-dev`).

### C1 (open)
- file: requirements.md
- section: ## Acceptance Criteria
- decided: The ticket says "waiting for input" and "waiting for review" but the existing State in states.py uses `needs_input` and `needs_review`. The API will normalize these terms: it accepts both forms in query parameters (mapping "waiting for input" → `needs_input`, "waiting for review" → `needs_review`) but always returns the normalized `needs_input`/`needs_review` in responses. This keeps the implementation aligned with the existing codebase while honoring the ticket's language.

### C2 (open)
- file: stories/02-agent.md
- section: ## Algorithm Notes › `save_case
- problem: The story's extraction logic uses a **different order** than the real function. The story says:
```python
state.get('service_order', {}).get('subjectEmployee', {}).get('fullName') or state.get('service_order', {}).get('subject_employee', {}).get('full_name')
```
This tries `subjectEmployee.fullName` first, and only falls back to `subject_employee.full_name` if the first chain returned a falsy value (including `{}`). But the **real function** in `backend.py` first resolves **which key exists** (`subjectEmployee` or `subject_employee`), then tries both name keys on that one object. The story's logic would fail if `service_order = {'subjectEmployee': {}}` (no `fullName`), because the first chain returns `{}` (truthy), so the `or` never tries the snake_case fallback.

### C3 (open)
- file: stories/02-agent.md
- section: ## Dependencies
- problem: This note says the logic "matches the exact structure" of `backend.employee_name`, but the algorithm in the same file (see C2 above) uses a different order that would fail in the `subjectEmployee` exists but has no `fullName` case. The note is wrong either way: if the algorithm is right (handles snake_case), the note should say "extends" not "matches"; if the note is right (matches exactly), the algorithm is wrong.
