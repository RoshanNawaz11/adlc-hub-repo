# Low-Level Design — Story 03-agent

**Case list and counts API endpoints**

---

## Acceptance Criteria

From `requirements.md` and `breakdown.md`:

- **REQ-AC1:** List endpoint returns all cases — `GET /cases` route exists.
- **REQ-AC2:** Each case shows service order ID, employee, case type, status, priority, flagged, last changed — response format matches.
- **REQ-AC3:** Default sort and pagination — default `page_size=25`, `sort=last_changed` descending.
- **REQ-AC4:** Filter by status — parameters validated and passed to repository.
- **REQ-AC5:** Filter by priority.
- **REQ-AC6:** Filter by flagged state.
- **REQ-AC7:** Filter by case type.
- **REQ-AC8:** Combine multiple filters.
- **REQ-AC9:** Filter by flag type.
- **REQ-AC10:** Sort by wait time.
- **REQ-AC11:** Counts endpoint — `GET /cases/counts` route exists.
- **REQ-AC13:** Invalid filter values rejected — validation at API entry per HLD AD-7.

---

## File Plan

All paths are relative to `{project-root}` (the `agent` repo root).

### New Files

None — this story adds routes to an existing file.

### Modified Files

1. **`src/agents/runtime/server.py`**  
   Add two routes: `GET /cases` and `GET /cases/counts`.

---

## Interfaces

### `src/agents/runtime/server.py` (additions)

```python
@app.get("/cases")
async def list_cases(
    status: str | None = None,
    priority: str | None = None,
    flagged: str | None = None,
    flag_type: str | None = None,
    case_type: str | None = None,
    sort: str = "last_changed",
    page: int = 1,
    page_size: int = 25,
) -> dict[str, Any]:
    """List cases with optional filters, sorting, and pagination.
    
    Query parameters (all optional):
    - status: comma-separated statuses (e.g. "running,needs_review")
    - priority: comma-separated priorities (e.g. "high,medium")
    - flagged: "true" or "false"
    - flag_type: filter by unresolved flag type (e.g. "netPayMismatch")
    - case_type: comma-separated case types (e.g. "Offboarding")
    - sort: "last_changed" (default, descending) or "wait_time" (ascending)
    - page: page number (default 1)
    - page_size: cases per page (default 25, max 100)
    
    Returns:
    {
        "cases": [
            {
                "run_id": "abc123",
                "service_order_id": "SO-1001",
                "employee_name": "Jane Doe",
                "case_type": "Offboarding",
                "status": "needs_review",
                "priority": "high",
                "flagged": true,
                "last_changed_at": "2025-01-15T14:30:00Z"
            },
            ...
        ],
        "total": 42,
        "page": 1,
        "page_size": 25
    }
    
    Raises HTTPException(400) for invalid filter values or page_size.
    """

@app.get("/cases/counts")
async def get_case_counts(
    by_status: bool = True,
    flagged: bool = True,
) -> dict[str, Any]:
    """Get aggregate counts of cases by status and flagged waiting count.
    
    Query parameters:
    - by_status: include counts per status (default true)
    - flagged: include count of flagged cases waiting for review (default true)
    
    Returns:
    {
        "by_status": {
            "running": 5,
            "needs_input": 2,
            "needs_review": 8,
            "done": 120,
            "rejected": 3,
            "failed": 1
        },
        "flagged_waiting": 3
    }
    """
```

---

## Algorithm Notes

### Route: `GET /cases`

1. **Parse and validate filter parameters:**
   - `status`: split on comma, validate each against `ALLOWED_STATUSES`.
   - `priority`: split on comma, validate each against `ALLOWED_PRIORITIES`.
   - `flagged`: parse to `bool` (`"true"` → `True`, `"false"` → `False`, `None` → `None`); invalid value → `400 Bad Request`.
   - `flag_type`: accept as-is (freeform string, no validation).
   - `case_type`: split on comma, accept as-is (freeform strings).
   - `sort`: validate against `{"last_changed", "wait_time"}`; invalid → `400 Bad Request`.
   - If `sort == "wait_time"` and `status` does not contain `"needs_review"` → `400 Bad Request` with message `"sort=wait_time requires status=needs_review"`.
   - `page`: must be `>= 1`; else → `400 Bad Request`.
   - `page_size`: must be `1 <= page_size <= 100`; else → `400 Bad Request` with message `"page_size must be between 1 and 100"`.

   **Constants:**
   ```python
   ALLOWED_STATUSES = {"running", "needs_input", "needs_review", "done", "rejected", "failed"}
   ALLOWED_PRIORITIES = {"high", "medium", "low"}
   ```

2. **Build repository call parameters:**
   - `statuses = _validate_statuses(status)` (helper validates and splits CSV).
   - `priorities = _validate_priorities(priority)` (helper validates and splits CSV).
   - `case_types = _split_csv(case_type)` (helper splits CSV, no validation).
   - `flagged_bool = _parse_flagged(flagged)` (helper: `"true"` → `True`, `"false"` → `False`, `None` → `None`, else raise `HTTPException(400, "flagged must be 'true' or 'false'")`)
   - `_validate_sort(sort, statuses)` (helper checks `wait_time` requires `needs_review`).

3. **Call repository:**
   - `db_path = _db_path()` (helper returns project-wide `data/cases.db`).
   - `cases, total = await repository.list_cases(db_path, statuses=statuses, priorities=priorities, flagged=flagged_bool, flag_type=flag_type, case_types=case_types, sort_by=sort, page=page, page_size=page_size)`

4. **Return response:**
   - `return {"cases": cases, "total": total, "page": page, "page_size": page_size}`

5. **Error handling:**
   - Invalid `status` value: `raise HTTPException(400, f"Invalid status: {value}. Allowed: {sorted(ALLOWED_STATUSES)}")`
   - Invalid `priority` value: `raise HTTPException(400, f"Invalid priority: {value}. Allowed: {sorted(ALLOWED_PRIORITIES)}")`
   - Invalid `sort`: `raise HTTPException(400, f"Invalid sort: {sort}. Allowed: ['last_changed', 'wait_time']")`
   - `page < 1`: `raise HTTPException(400, "page must be >= 1")`
   - `page_size < 1` or `page_size > 100`: `raise HTTPException(400, "page_size must be between 1 and 100")`

---

### Route: `GET /cases/counts`

1. **Parse query parameters:**
   - `by_status` and `flagged` are already booleans via FastAPI's type coercion.

2. **Call repository:**
   - `db_path = _db_path()`
   - `counts = await repository.get_counts(db_path, by_status=by_status, flagged=flagged)`

3. **Return response:**
   - `return counts`

---

### Helper Functions

```python
def _parse_flagged(value: str | None) -> bool | None:
    """Parse 'true'/'false' query param to bool, None if absent."""
    if value is None:
        return None
    if value.lower() == "true":
        return True
    if value.lower() == "false":
        return False
    raise HTTPException(400, "flagged must be 'true' or 'false'")

def _db_path() -> Path:
    """Resolve the database path for case persistence.
    
    Project-wide data/cases.db (easier for listing all cases).
    Story 1 migration runner uses the same path.
    """
    from agents.runtime.run_dir import LOG_DIR
    db_dir = LOG_DIR.parent / "data"
    db_dir.mkdir(parents=True, exist_ok=True)
    return db_dir / "cases.db"

def _validate_statuses(raw: str | None) -> list[str] | None:
    """Split comma-separated statuses and validate each."""
    if not raw:
        return None
    statuses = [s.strip() for s in raw.split(",")]
    invalid = [s for s in statuses if s not in ALLOWED_STATUSES]
    if invalid:
        raise HTTPException(
            400,
            f"Invalid status: {', '.join(invalid)}. Allowed: {sorted(ALLOWED_STATUSES)}"
        )
    return statuses

def _validate_priorities(raw: str | None) -> list[str] | None:
    """Split comma-separated priorities and validate each."""
    if not raw:
        return None
    priorities = [p.strip() for p in raw.split(",")]
    invalid = [p for p in priorities if p not in ALLOWED_PRIORITIES]
    if invalid:
        raise HTTPException(
            400,
            f"Invalid priority: {', '.join(invalid)}. Allowed: {sorted(ALLOWED_PRIORITIES)}"
        )
    return priorities

def _validate_sort(sort: str, statuses: list[str] | None) -> str:
    """Validate sort parameter and check wait_time requires needs_review."""
    if sort not in {"last_changed", "wait_time"}:
        raise HTTPException(
            400,
            f"Invalid sort: {sort}. Allowed: ['last_changed', 'wait_time']"
        )
    if sort == "wait_time" and (not statuses or "needs_review" not in statuses):
        raise HTTPException(
            400,
            "sort=wait_time requires status=needs_review"
        )
    return sort

def _split_csv(raw: str | None) -> list[str] | None:
    """Split comma-separated values (for freeform fields like case_type)."""
    if not raw:
        return None
    return [s.strip() for s in raw.split(",") if s.strip()]
```

---

### Integration into `server.py`

The existing `server.py` uses `async with open_graph(thread_id)` to access the checkpointer. The new routes do **not** use the checkpointer (it holds in-memory run state); they read from SQLite via `repository.py`.

**Imports to add:**

```python
from pathlib import Path
from agents.persistence import repository
```

**Placement:**

Add the new routes after the existing `/runs/{thread_id}` route and before `/health`.

**Constants to add (module-level):**

```python
ALLOWED_STATUSES = {"running", "needs_input", "needs_review", "done", "rejected", "failed"}
ALLOWED_PRIORITIES = {"high", "medium", "low"}
```

---

## Tests

Sixteen tests as specified in Verification Approach:

1. **Default pagination and response shape:**
   - Insert 30 cases (via `repository.save_case` or direct SQL).
   - `GET /cases` with no query params.
   - Assert response: `{"cases": [...], "total": 30, "page": 1, "page_size": 25}`.
   - Assert `len(response["cases"]) == 25`.

2. **Case object fields:**
   - Insert one case with all fields populated (`run_id`, `service_order_id`, `employee_name`, `case_type`, `status`, `priority`, `flagged`, `last_changed_at`).
   - `GET /cases`.
   - Assert each case object has those keys.

3. **Default page_size is 25:**
   - Insert 30 cases.
   - `GET /cases` without `page_size` param.
   - Assert `len(response["cases"]) == 25`.

4. **Custom page_size:**
   - Insert 60 cases.
   - `GET /cases?page_size=50`.
   - Assert `len(response["cases"]) == 50`.

5. **page_size too large:**
   - `GET /cases?page_size=150`.
   - Assert `400 Bad Request` with message `"page_size must be between 1 and 100"`.

6. **Filter by status (multiple):**
   - Insert cases: `status='running'`, `'needs_review'`, `'done'`.
   - `GET /cases?status=running,done`.
   - Assert only `running` and `done` cases appear (not `needs_review`).

7. **Filter by priority:**
   - Insert cases: `priority='high'`, `'medium'`, `'low'`.
   - `GET /cases?priority=high`.
   - Assert only `high` cases appear.

8. **Filter by flagged:**
   - Insert cases: `flagged=True`, `flagged=False`.
   - `GET /cases?flagged=true`.
   - Assert all returned cases have `flagged == True`.

9. **Filter by case_type:**
   - Insert cases: `case_type='Offboarding'`, `'Onboarding'`.
   - `GET /cases?case_type=Offboarding`.
   - Assert only `Offboarding` cases appear.

10. **Combine filters (AC8):**
    - Insert cases:
      - `run_id='a'`, `status='needs_review'`, `flagged=True`.
      - `run_id='b'`, `status='needs_review'`, `flagged=False`.
      - `run_id='c'`, `status='running'`, `flagged=True`.
    - `GET /cases?status=needs_review&flagged=true`.
    - Assert only `run_id='a'` is returned.

11. **Filter by flag_type (AC9):**
    - Insert case `run_id='d'` with `case_flags` row: `(flag_type='netPayMismatch', resolved=0)`.
    - Insert case `run_id='e'` with `case_flags` row: `(flag_type='missingData', resolved=0)`.
    - `GET /cases?flag_type=netPayMismatch`.
    - Assert only `run_id='d'` is returned.

12. **Sort by wait_time (AC10):**
    - Insert cases:
      - `run_id='f'`, `status='needs_review'`, `last_changed_at='2025-01-15T10:00:00Z'`.
      - `run_id='g'`, `status='needs_review'`, `last_changed_at='2025-01-15T11:00:00Z'`.
    - `GET /cases?status=needs_review&sort=wait_time`.
    - Assert order is `['f', 'g']` (oldest first).

13. **Invalid status:**
    - `GET /cases?status=invalid_status`.
    - Assert `400 Bad Request` with message `"Invalid status: invalid_status. Allowed: ['done', 'failed', 'needs_input', 'needs_review', 'rejected', 'running']"`.

14. **Invalid priority:**
    - `GET /cases?priority=unknown_priority`.
    - Assert `400 Bad Request` with message `"Invalid priority: unknown_priority. Allowed: ['high', 'low', 'medium']"`.

15. **Counts endpoint response:**
    - Insert cases:
      - 2 with `status='running'`.
      - 3 with `status='needs_review'`, 2 of which have `flagged=True`.
      - 1 with `status='done'`.
    - `GET /cases/counts?by_status=true&flagged=true`.
    - Assert response:
      ```python
      {
        "by_status": {
          "running": 2,
          "needs_input": 0,
          "needs_review": 3,
          "done": 1,
          "rejected": 0,
          "failed": 0,
        },
        "flagged_waiting": 2,
      }
      ```

16. **Counts endpoint does not load full case list:**
    - Enable SQLite query logging or use a profiler (e.g. `EXPLAIN QUERY PLAN`).
    - `GET /cases/counts`.
    - Assert no `SELECT * FROM cases` (only `COUNT(*)` queries appear).
    - Verify query is `SELECT COUNT(*) FROM cases WHERE status = 'needs_review' AND flagged = 1` for `flagged_waiting`.

---

## Open Questions (from HLD/Breakdown)

1. **Database path resolution:** Is `cases.db` per-run (in each run's `logs/...` folder) or project-wide (in `data/cases.db`)?  
   **Impact:** This story assumes a **project-wide `data/cases.db`** (easier for listing all cases across runs). Story 1's migration runner and Story 4's persistence hook must use the same path. If per-run is required, the `/cases` endpoint needs a way to aggregate across all run folders (expensive; not recommended).

2. **Wait time calculation (HLD OQ4):** Story 2 uses `last_changed_at` as a proxy for wait time. If a separate `waiting_since` field is required (timestamp when status became `needs_review`), Story 1's schema will need amendment (`002_add_waiting_since.sql` migration).  
   **Impact:** This story's `sort=wait_time` validation logic is correct either way (it only checks that `status=needs_review` is present).

3. **sort=wait_time without status filter:** Should `sort=wait_time` be allowed when no `status` filter is given (assumes all cases are `needs_review`), or must it always be paired with `status=needs_review`?  
   **Current decision:** Reject if `status` filter does not include `needs_review` (safer; prevents sorting non-waiting cases by an irrelevant field).

---

## Dependencies

- **External:** None (FastAPI and `aiosqlite` already in `pyproject.toml` from Story 1).
- **Internal:**
  - `src/agents/persistence/repository.py` — `list_cases` and `get_counts` functions (Story 2).
  - `src/agents/runtime/run_dir.py` — `LOG_DIR` for resolving `data/` folder (already exists).
  - `src/agents/runtime/server.py` — existing routes and `_lifespan` (already exists).

---

## Notes

- **No checkpointer integration:** These routes read from SQLite directly via `repository.py`, not from the LangGraph checkpointer (which holds in-memory state for active runs).
- **Validation at API layer:** All filter values are validated in the route handler before calling `repository.list_cases`. This makes `repository.py` simpler (it trusts its inputs) and keeps error messages consistent.
- **Comma-separated values:** Follow existing FastAPI patterns in the repo (no Pydantic model for query params; parse manually in the route).
- **Response format:** Matches HLD AD-2 exactly. Each case is a flat dict (no nested objects).
- **Error messages:** Always name the allowed values (per HLD AD-7). Use sorted lists for deterministic test assertions.
- **sort=wait_time guard:** Prevents a misleading sort when the filter doesn't include `needs_review` (e.g. sorting `done` cases by wait time makes no sense).
- **Project-wide database:** Resolves Open Question 1 by choosing a single `data/cases.db` for all runs. This matches the "list every case" requirement better than per-run databases.
