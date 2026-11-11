# Low-Level Design — Story 02-agent

**Repository layer for case queries**

---

## Acceptance Criteria

From `requirements.md` and `breakdown.md`:

- **REQ-AC1:** List endpoint returns all cases — `list_cases` selects from `cases` with optional filters.
- **REQ-AC4:** Filter by status — `list_cases` builds `WHERE status IN (...)`.
- **REQ-AC5:** Filter by priority — `WHERE priority IN (...)`.
- **REQ-AC6:** Filter by flagged state — `WHERE flagged = ...`.
- **REQ-AC7:** Filter by case type — `WHERE case_type IN (...)`.
- **REQ-AC8:** Combine multiple filters — `WHERE` clauses are ANDed.
- **REQ-AC9:** Filter by flag type — join `case_flags` where `flag_type = ... AND resolved = 0`.
- **REQ-AC10:** Sort by wait time — `ORDER BY last_changed_at ASC` when `status = 'needs_review'`.
- **REQ-AC11:** Counts endpoint — `get_counts` groups by status and counts flagged waiting cases.
- **REQ-AC12:** Cases persist across restarts — `save_case` writes synchronously; `list_cases` reads back.

---

## File Plan

All paths are relative to `{project-root}` (the `agent` repo root).

### New Files

1. **`src/agents/persistence/repository.py`**  
   Async SQLite queries: `save_case`, `list_cases`, `get_counts`, `load_case`.

### Modified Files

None — this story creates the repository layer without integrating it into routes or hooks.

---

## Interfaces

### `src/agents/persistence/repository.py`

```python
from pathlib import Path
from typing import Any

async def save_case(state: dict[str, Any], db_path: Path) -> None:
    """Insert or update a case in the cases table and bulk-replace its case_flags rows.
    
    Extracts run_id, service_order_id, employee_name (via backend.employee_name),
    case_type, status, priority, flagged, last_changed_at, created_at, and state_snapshot
    from state. Flags from state['flags'] are bulk-inserted into case_flags after deleting
    existing rows for the run_id.
    
    Args:
        state: State dict from graph/states.py.
        db_path: Path to the SQLite database file (e.g. data/cases.db).
    """

async def list_cases(
    db_path: Path,
    statuses: list[str] | None = None,
    priorities: list[str] | None = None,
    flagged: bool | None = None,
    flag_type: str | None = None,
    case_types: list[str] | None = None,
    sort_by: str = "last_changed",
    page: int = 1,
    page_size: int = 25,
) -> tuple[list[dict[str, Any]], int]:
    """Query cases with filters, sorting, and pagination.
    
    Builds WHERE clauses from filters (ANDed), ORDER BY from sort_by, and LIMIT/OFFSET
    from page/page_size. Joins case_flags when flag_type is specified.
    
    Args:
        db_path: Path to the SQLite database file.
        statuses: Filter by status values (e.g. ['running', 'needs_review']).
        priorities: Filter by priority values (e.g. ['high', 'medium']).
        flagged: Filter by flagged state (True/False).
        flag_type: Filter by unresolved flag type (e.g. 'netPayMismatch').
        case_types: Filter by case type values (e.g. ['Offboarding']).
        sort_by: 'last_changed' (default, descending) or 'wait_time' (ascending).
        page: Page number (1-indexed).
        page_size: Cases per page (default 25).
    
    Returns:
        Tuple of (list of case dicts, total count).
    """

async def get_counts(
    db_path: Path,
    by_status: bool = True,
    flagged: bool = True,
) -> dict[str, Any]:
    """Retrieve aggregate counts: cases per status and flagged cases waiting for review.
    
    Args:
        db_path: Path to the SQLite database file.
        by_status: Include counts per status in the result.
        flagged: Include count of flagged cases with status='needs_review'.
    
    Returns:
        Dict with 'by_status' (status -> count) and 'flagged_waiting' (int).
    """

async def load_case(run_id: str, db_path: Path) -> dict[str, Any] | None:
    """Load a case's full State dict from state_snapshot by run_id.
    
    Args:
        run_id: Thread ID of the run.
        db_path: Path to the SQLite database file.
    
    Returns:
        State dict parsed from state_snapshot JSON, or None if run_id not found.
    """
```

---

## Algorithm Notes

### `save_case`

## Algorithm Notes

### `save_case`

1. **Open async connection:**
   - `async with aiosqlite.connect(str(db_path)) as conn:`
   - Enable foreign key enforcement: `await conn.execute("PRAGMA foreign_keys = ON")`.

2. **Extract fields from state:**
   - `run_id` ← `state['run_id']`
   - `service_order_id` ← `state.get('service_order_id')` (optional key)
   - `employee_name` ← `state.get('service_order', {}).get('subjectEmployee', {}).get('fullName') or state.get('service_order', {}).get('subject_employee', {}).get('full_name')` (matches `backend.employee_name` logic from `src/agents/runtime/backend.py`)
   - `case_type` ← `state.get('service_order', {}).get('caseType')` (placeholder logic; see Open Question 1 — may be `None`)
   - `status` ← `state['status']`
   - `priority` ← `state.get('priority')` (optional)
   - `flagged` ← `state.get('flagged', False)` (boolean, default `False`)
   - `last_changed_at` ← current ISO 8601 UTC timestamp (`datetime.now(UTC).isoformat()`)
   - `created_at` ← current timestamp on first insert; preserve existing value on update (see step 4)
   - `state_snapshot` ← `json.dumps(state)` (full `State` dict as JSON)

3. **Check if row exists:**
   - `cursor = await conn.execute("SELECT created_at FROM cases WHERE run_id = ?", (run_id,))`
   - `row = await cursor.fetchone()`
   - If `row` exists: `created_at = row[0]` (preserve original creation timestamp).
   - Else: `created_at = last_changed_at` (first insert).

4. **Insert or replace case row:**
   - Use `INSERT OR REPLACE INTO cases (run_id, service_order_id, employee_name, case_type, status, priority, flagged, last_changed_at, created_at, state_snapshot) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`.
   - Bind: `(run_id, service_order_id, employee_name, case_type, status, priority, int(flagged), last_changed_at, created_at, state_snapshot)`.
   - Note: SQLite `BOOLEAN` is stored as `INTEGER` (`0`/`1`); convert `flagged` with `int()`.

5. **Bulk-replace case_flags rows:**
   - Delete existing rows: `await conn.execute("DELETE FROM case_flags WHERE run_id = ?", (run_id,))`.
   - Extract flags: `flags = state.get('flags', [])`.
   - For each `flag` in `flags`:
     - Extract: `flag_id = flag['flag_id']`, `flag_type = flag['type']`, `resolved = flag['resolved']`.
     - Insert: `await conn.execute("INSERT INTO case_flags (run_id, flag_id, flag_type, resolved) VALUES (?, ?, ?, ?)", (run_id, flag_id, flag_type, int(resolved)))`.
   - Optimization: batch insert with `executemany` if `flags` is non-empty:
     ```python
     if flags:
         rows = [(run_id, f['flag_id'], f['type'], int(f['resolved'])) for f in flags]
         await conn.executemany("INSERT INTO case_flags (run_id, flag_id, flag_type, resolved) VALUES (?, ?, ?, ?)", rows)
     ```

6. **Commit transaction:**
   - `await conn.commit()`.

### `list_cases`

1. **Open async connection:**
   - `async with aiosqlite.connect(str(db_path)) as conn:`
   - Set row factory: `conn.row_factory = aiosqlite.Row` (enables column access by name).

2. **Build WHERE clauses (ANDed):**
   - Start with `where_clauses = []` and `params = []`.
   - If `statuses`:
     - `placeholders = ', '.join(['?'] * len(statuses))`
     - `where_clauses.append(f"cases.status IN ({placeholders})")`
     - `params.extend(statuses)`
   - If `priorities`:
     - `placeholders = ', '.join(['?'] * len(priorities))`
     - `where_clauses.append(f"cases.priority IN ({placeholders})")`
     - `params.extend(priorities)`
   - If `flagged is not None`:
     - `where_clauses.append("cases.flagged = ?")`
     - `params.append(int(flagged))`
   - If `case_types`:
     - `placeholders = ', '.join(['?'] * len(case_types))`
     - `where_clauses.append(f"cases.case_type IN ({placeholders})")`
     - `params.extend(case_types)`
   - If `flag_type`:
     - `where_clauses.append("case_flags.flag_type = ? AND case_flags.resolved = 0")`
     - `params.append(flag_type)`
   - Join: `where_clause = " AND ".join(where_clauses) if where_clauses else "1=1"`.

3. **Build ORDER BY clause:**
   - If `sort_by == "last_changed"`: `order_by = "cases.last_changed_at DESC"`.
   - Else if `sort_by == "wait_time"`: `order_by = "cases.last_changed_at ASC"`.
   - Else: raise `ValueError(f"Invalid sort_by: {sort_by}")` (enforced by API layer in Story 3; repository should validate defensively).

4. **Build pagination LIMIT/OFFSET:**
   - `offset = (page - 1) * page_size`.
   - `limit_clause = f"LIMIT {page_size} OFFSET {offset}"`.

5. **Build SELECT query:**
   - Base: `SELECT cases.* FROM cases`.
   - If `flag_type`: `JOIN case_flags ON cases.run_id = case_flags.run_id`.
   - Add: `WHERE {where_clause}`.
   - Add: `ORDER BY {order_by}`.
   - Add: `{limit_clause}`.
   - Full query:
     ```sql
     SELECT cases.* FROM cases
     [JOIN case_flags ON cases.run_id = case_flags.run_id]
     WHERE {where_clause}
     ORDER BY {order_by}
     LIMIT {page_size} OFFSET {offset}
     ```
   - If `flag_type` filter is present, add `DISTINCT` to avoid duplicate rows: `SELECT DISTINCT cases.* FROM cases JOIN ...`.

6. **Execute query:**
   - `cursor = await conn.execute(query, params)`.
   - `rows = await cursor.fetchall()`.
   - Convert rows to dicts: `cases = [dict(row) for row in rows]`.

7. **Get total count:**
   - Build count query: replace `SELECT cases.*` with `SELECT COUNT(DISTINCT cases.run_id)` (distinct needed when joining `case_flags`).
   - Strip `ORDER BY` and `LIMIT/OFFSET` (not needed for count).
   - `count_query = f"SELECT COUNT(DISTINCT cases.run_id) FROM cases [JOIN ...] WHERE {where_clause}"`.
   - `cursor = await conn.execute(count_query, params)`.
   - `total = (await cursor.fetchone())[0]`.

8. **Return:**
   - `return (cases, total)`.

---

### `get_counts`

1. **Open async connection:**
   - `async with aiosqlite.connect(str(db_path)) as conn:`

2. **Counts by status:**
   - If `by_status`:
     - Query: `SELECT status, COUNT(*) as count FROM cases GROUP BY status`.
     - `cursor = await conn.execute(query)`.
     - `rows = await cursor.fetchall()`.
     - Build dict: `by_status = {row[0]: row[1] for row in rows}`.
     - Fill in missing statuses with `0`:
       ```python
       all_statuses = ['running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed']
       by_status = {s: by_status.get(s, 0) for s in all_statuses}
       ```
   - Else: `by_status = {}`.

3. **Flagged waiting count:**
   - If `flagged`:
     - Query: `SELECT COUNT(*) FROM cases WHERE status = 'needs_review' AND flagged = 1`.
     - `cursor = await conn.execute(query)`.
     - `flagged_waiting = (await cursor.fetchone())[0]`.
   - Else: `flagged_waiting = 0`.

4. **Return:**
   - `return {"by_status": by_status, "flagged_waiting": flagged_waiting}`.

---

### `load_case`

1. **Open async connection:**
   - `async with aiosqlite.connect(str(db_path)) as conn:`

2. **Query state_snapshot:**
   - `cursor = await conn.execute("SELECT state_snapshot FROM cases WHERE run_id = ?", (run_id,))`.
   - `row = await cursor.fetchone()`.

3. **Parse and return:**
   - If `row is None`: return `None`.
   - Else: `return json.loads(row[0])` (deserialize JSON to `State` dict).

---

## Tests

Fifteen tests as specified in Verification Approach:

1. **save_case inserts a new case:**
   - Create a `State` dict with `run_id='test-1'`, `status='running'`, `flagged=False`, `service_order={'caseType': 'Offboarding'}`.
   - Call `save_case(state, tmp_db_path)`.
   - Query: `SELECT run_id, status, case_type, flagged, service_order_id, priority FROM cases WHERE run_id='test-1'`.
   - Assert row exists with correct values: `run_id='test-1'`, `status='running'`, `case_type='Offboarding'`, `flagged=0`, `service_order_id=None`, `priority=None`.
   - Verify `created_at` and `last_changed_at` are ISO 8601 timestamps.
   - Verify `state_snapshot` is parseable JSON matching the input `state`.

2. **save_case updates an existing case:**
   - Insert a case with `run_id='test-2'`, `status='running'`, `created_at='2025-01-15T10:00:00Z'`.
   - Call `save_case` again with same `run_id` but `status='needs_review'`.
   - Query: `SELECT status, created_at, last_changed_at FROM cases WHERE run_id='test-2'`.
   - Assert `status='needs_review'`, `created_at='2025-01-15T10:00:00Z'` (unchanged), `last_changed_at > created_at`.

3. **save_case bulk-replaces case_flags:**
   - Create a `State` dict with `run_id='test-3'`, `flags=[{"flag_id": "test-3-1", "type": "netPayMismatch", "resolved": False}]`.
   - Call `save_case(state, tmp_db_path)`.
   - Query: `SELECT flag_id, flag_type, resolved FROM case_flags WHERE run_id='test-3'`.
   - Assert one row: `flag_id='test-3-1'`, `flag_type='netPayMismatch'`, `resolved=0`.
   - Update `state['flags']` to `[{"flag_id": "test-3-2", "type": "missingData", "resolved": False}]`.
   - Call `save_case(state, tmp_db_path)` again.
   - Query `case_flags` again.
   - Assert old row (`test-3-1`) is deleted and new row (`test-3-2`) is inserted.

4. **list_cases with no filters returns all cases:**
   - Insert 3 cases with different statuses: `'running'`, `'needs_review'`, `'done'`.
   - Call `list_cases(tmp_db_path)`.
   - Assert `total == 3` and all 3 cases are in the returned list.

5. **list_cases filters by status:**
   - Insert cases with `status='running'`, `'needs_review'`, `'done'`.
   - Call `list_cases(tmp_db_path, statuses=['running'])`.
   - Assert only the `'running'` case is returned.

6. **list_cases filters by priority:**
   - Insert cases with `priority='high'`, `'medium'`, `'low'`, `None`.
   - Call `list_cases(tmp_db_path, priorities=['high', 'medium'])`.
   - Assert only `'high'` and `'medium'` cases are returned (not `'low'` or `None`).

7. **list_cases filters by flagged state:**
   - Insert cases with `flagged=True` and `flagged=False`.
   - Call `list_cases(tmp_db_path, flagged=True)`.
   - Assert only `flagged=True` cases are returned.

8. **list_cases filters by case_type:**
   - Insert cases with `case_type='Offboarding'`, `'Onboarding'`, `None`.
   - Call `list_cases(tmp_db_path, case_types=['Offboarding'])`.
   - Assert only `'Offboarding'` case is returned.

9. **list_cases combines multiple filters (AC8):**
   - Insert cases:
     - `run_id='a'`, `status='needs_review'`, `flagged=True`.
     - `run_id='b'`, `status='needs_review'`, `flagged=False`.
     - `run_id='c'`, `status='running'`, `flagged=True`.
   - Call `list_cases(tmp_db_path, statuses=['needs_review'], flagged=True)`.
   - Assert only `run_id='a'` is returned (intersection of both filters).

10. **list_cases filters by flag_type (AC9):**
    - Insert case `run_id='d'`, `status='needs_review'`.
    - Insert `case_flags`: `(run_id='d', flag_id='d-1', flag_type='netPayMismatch', resolved=0)`.
    - Insert another case `run_id='e'` with `(run_id='e', flag_id='e-1', flag_type='missingData', resolved=0)`.
    - Insert a resolved flag: `(run_id='d', flag_id='d-2', flag_type='netPayMismatch', resolved=1)`.
    - Call `list_cases(tmp_db_path, flag_type='netPayMismatch')`.
    - Assert only `run_id='d'` is returned (join `case_flags` where `flag_type='netPayMismatch' AND resolved=0`).
    - Verify `run_id='e'` (different flag type) and the resolved `d-2` flag do not appear.

11. **list_cases sorts by last_changed (default, descending):**
    - Insert cases:
      - `run_id='f'`, `last_changed_at='2025-01-15T10:00:00Z'`.
      - `run_id='g'`, `last_changed_at='2025-01-15T11:00:00Z'`.
    - Call `list_cases(tmp_db_path, sort_by='last_changed')`.
    - Assert order is `['g', 'f']` (most recent first).

12. **list_cases sorts by wait_time (ascending) when status=needs_review:**
    - Insert cases:
      - `run_id='h'`, `status='needs_review'`, `last_changed_at='2025-01-15T11:00:00Z'`.
      - `run_id='i'`, `status='needs_review'`, `last_changed_at='2025-01-15T10:00:00Z'`.
    - Call `list_cases(tmp_db_path, statuses=['needs_review'], sort_by='wait_time')`.
    - Assert order is `['i', 'h']` (oldest first).

13. **list_cases paginates correctly:**
    - Insert 30 cases.
    - Call `list_cases(tmp_db_path, page=2, page_size=10)`.
    - Assert `total == 30` and 10 cases are returned (LIMIT 10 OFFSET 10).
    - Verify the returned cases are the 11th–20th when sorted by default (`last_changed_at DESC`).

14. **get_counts returns counts per status and flagged waiting count:**
    - Insert cases:
      - 2 cases with `status='running'`.
      - 3 cases with `status='needs_review'`, 2 of which have `flagged=True`.
      - 1 case with `status='done'`.
    - Call `get_counts(tmp_db_path, by_status=True, flagged=True)`.
    - Assert result:
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

15. **load_case returns the full State dict:**
    - Insert a case with `run_id='j'`, `state_snapshot='{"run_id": "j", "status": "running", "flagged": false}'`.
    - Call `load_case('j', tmp_db_path)`.
    - Assert returned dict is `{"run_id": "j", "status": "running", "flagged": False}` (parsed JSON).
    - Call `load_case('unknown', tmp_db_path)`.
    - Assert `None` is returned.

---

## Open Questions (from HLD/Breakdown)

1. **Case type derivation (HLD OQ1):** How is `case_type` extracted from `service_order`? Is it `service_order['caseType']` verbatim, or does it map to a normalized set?  
   **Impact:** Current extraction logic in `save_case` uses `state.get('service_order', {}).get('caseType')` as a placeholder. If a mapping is required, add a helper function `_normalize_case_type(service_order)` in `repository.py` after clarification.

2. **Flag type normalization (HLD OQ2):** Are flag types (`netPayMismatch`, `missingData`, etc.) hardcoded enums or freeform strings?  
   **Impact:** This story treats `flag_type` as a freeform string (no validation in repository). Story 3 (API layer) will validate if they are enums.

3. **Wait time calculation (HLD OQ4):** How is "how long they have been waiting" computed for AC10? Is it `last_changed_at` when status became `needs_review`, or a separate `waiting_since` timestamp?  
   **Impact:** Current implementation uses `last_changed_at` as a proxy (sorted ascending for `sort_by='wait_time'`). If a separate `waiting_since` field is required, Story 1's schema will need amendment (`002_add_waiting_since.sql` migration).

---

## Dependencies

- **External:** `aiosqlite>=0.20.0` (added to `pyproject.toml` in Story 1).
- **Internal:**
  - `src/agents/graph/states.py` — `State` structure defines the fields persisted in `cases.state_snapshot`.
  - `src/agents/persistence/migrations/001_create_cases.sql` — Schema must exist (Story 1 completed).
  - `json` (stdlib) — for `json.dumps` and `json.loads`.
  - `datetime` (stdlib) — for `datetime.now(UTC).isoformat()`.

**Note:** Employee name extraction logic is implemented directly in `save_case` by reading `state['service_order']['subjectEmployee']['fullName']`, matching the exact structure used by `employee_name(service_order)` in `src/agents/runtime/backend.py`. No external call to `backend.employee_name` is required.

## Notes

- **No route integration yet:** This story creates the repository layer; Story 3 wires it into `GET /cases` and `GET /cases/counts` routes.
- **No hook integration yet:** Story 4 calls `save_case` from the LangGraph checkpointer write hook.
- **Defensive validation:** `list_cases` validates `sort_by` (raises `ValueError` for unknown values), even though Story 3's API layer will enforce allowed values before calling the repository.
- **DISTINCT for flag_type filter:** When joining `case_flags`, `SELECT DISTINCT cases.*` prevents duplicate rows when a case has multiple flags of the same type.
- **Foreign key enforcement:** `PRAGMA foreign_keys = ON` ensures `ON DELETE CASCADE` works (deleting a case also deletes its flags).
- **Row factory:** `aiosqlite.Row` enables column access by name (`row['run_id']`) instead of index (`row[0]`), making the code more readable.
- **Batch insert optimization:** `executemany` for `case_flags` rows avoids N individual `INSERT` statements when a case has many flags.
- **Missing statuses in counts:** `get_counts` fills in `0` for statuses with no cases, so the response always has all six statuses (matches API contract in Story 3).
