# Low-Level Design — Story 01-agent

**Case persistence schema and migration runner**

---

## Acceptance Criteria

From `requirements.md` and `breakdown.md`:

- **REQ-AC12:** Cases persist across restarts — schema holds their last state.
- **REQ-AC1:** List includes all cases — schema stores every case regardless of status.
- **REQ-AC2:** Each case shows service order ID, employee, case type, status, priority, flagged, last changed — schema columns match.
- **REQ-AC9:** Filter by specific flag type — `case_flags` table and index support it.

---

## File Plan

All paths are relative to `{project-root}` (the `agent` repo root).

### New Files

1. **`src/agents/persistence/__init__.py`**  
   Empty marker file for the new `persistence` package.

2. **`src/agents/persistence/migrate.py`**  
   Migration runner: applies unapplied `.sql` files from `migrations/` on server startup.

3. **`src/agents/persistence/migrations/__init__.py`**  
   Empty marker file for the `migrations` package.

4. **`src/agents/persistence/migrations/001_create_cases.sql`**  
   Schema initialization: `CREATE TABLE cases`, `CREATE TABLE case_flags`, indexes.

### Modified Files

1. **`src/agents/runtime/server.py`**  
   Add `@app.on_event("startup")` to run `migrate.py` before serving requests.

2. **`pyproject.toml`**  
   Add dependency: `aiosqlite>=0.20.0`.

---

## Interfaces

### `src/agents/persistence/migrate.py`

```python
async def apply_migrations(db_path: Path) -> None:
    """Apply unapplied migrations from migrations/*.sql to the database at db_path.
    
    Creates a schema_version table if it doesn't exist. Each migration runs in a
    transaction. Logs each applied migration. Raises on SQL errors.
    """
```

### `src/agents/runtime/server.py` (modified)

```python
@app.on_event("startup")
async def _run_migrations() -> None:
    """Apply database migrations before the server starts accepting requests."""
```

---

## Algorithm Notes

### Migration Runner (`migrate.py`)

1. **Initialization:**
   - Accept `db_path` as a `Path` argument.
   - Open an `aiosqlite` connection to `db_path`.
   - Create `schema_version` table if it doesn't exist:
     ```sql
     CREATE TABLE IF NOT EXISTS schema_version (
         version INTEGER PRIMARY KEY,
         applied_at TEXT NOT NULL
     );
     ```

2. **Discover migrations:**
   - Scan `src/agents/persistence/migrations/` for files matching `{seq}_{description}.sql` (regex `^\d+_.*\.sql$`).
   - Sort by numeric prefix (e.g. `001`, `002`, `010`).
   - Extract `seq` as an integer for each file.

3. **Determine applied migrations:**
   - Query `SELECT version FROM schema_version ORDER BY version`.
   - Build a set of applied versions.

4. **Apply unapplied migrations:**
   - For each migration file where `seq` not in applied set:
     - Read the `.sql` file contents.
     - Execute in a transaction:
       ```python
       async with conn.execute("BEGIN"):
           await conn.executescript(sql_content)
           await conn.execute(
               "INSERT INTO schema_version (version, applied_at) VALUES (?, ?)",
               (seq, datetime.now(UTC).isoformat())
           )
           await conn.commit()
       ```
     - Log: `f"Applied migration {seq}: {filename}"`.
   - On SQL error: log error, rollback transaction, re-raise.

5. **Close connection.**

### Server Startup Hook (`server.py`)

1. **Resolve database path:**
   - Default: `{project-root}/data/cases.db` (same directory structure as checkpointer uses in `run_dir.py`).
   - Ensure parent directory exists: `db_path.parent.mkdir(parents=True, exist_ok=True)`.

2. **Call `apply_migrations(db_path)`.**

3. **Handle errors:**
   - Log failure and exit with code 1 (server does not start if migrations fail).

### Schema (`001_create_cases.sql`)

**`cases` table:**

```sql
CREATE TABLE cases (
    run_id TEXT PRIMARY KEY,
    service_order_id TEXT,
    employee_name TEXT,
    case_type TEXT,
    status TEXT NOT NULL,
    priority TEXT,
    flagged BOOLEAN NOT NULL DEFAULT 0,
    last_changed_at TEXT NOT NULL,
    created_at TEXT NOT NULL,
    state_snapshot TEXT NOT NULL
);

CREATE INDEX idx_status ON cases(status);
CREATE INDEX idx_last_changed ON cases(last_changed_at DESC);
CREATE INDEX idx_flagged ON cases(flagged) WHERE flagged = 1;
```

**`case_flags` table:**

```sql
CREATE TABLE case_flags (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id TEXT NOT NULL,
    flag_id TEXT NOT NULL,
    flag_type TEXT NOT NULL,
    resolved BOOLEAN NOT NULL DEFAULT 0,
    FOREIGN KEY (run_id) REFERENCES cases(run_id) ON DELETE CASCADE
);

CREATE INDEX idx_case_flags_run_type ON case_flags(run_id, flag_type, resolved);
```

**Column mapping to `State` (from `graph/states.py`):**

- `run_id` ← `State['run_id']`
- `service_order_id` ← `State['service_order_id']` (optional key)
- `employee_name` ← derived via `backend.employee_name(State['service_order'])` (see Story 4)
- `case_type` ← extracted from `State['service_order']` structure (placeholder `None` for now; see Open Question 1)
- `status` ← `State['status']` (one of `{'running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed'}`)
- `priority` ← `State['priority']` (optional; one of `{'high', 'medium', 'low'}`)
- `flagged` ← `State['flagged']` (boolean)
- `last_changed_at` ← current ISO 8601 UTC timestamp on every write
- `created_at` ← ISO 8601 UTC timestamp on first insert
- `state_snapshot` ← full `State` dict as JSON (via `json.dumps(state)`)

**Flag mapping:**

- `run_id` ← `State['run_id']`
- `flag_id` ← `Flag['flag_id']` from `State['flags']`
- `flag_type` ← `Flag['type']`
- `resolved` ← `Flag['resolved']`

---

## Tests

Six tests as specified in Verification Approach:

1. **Migration creates tables:**
   - Run `apply_migrations(tmp_db_path)`.
   - Query SQLite schema: `SELECT name FROM sqlite_master WHERE type='table'`.
   - Assert `cases` and `case_flags` exist.
   - Assert `schema_version` exists.
   - Query `PRAGMA table_info(cases)` and verify columns: `run_id`, `service_order_id`, `employee_name`, `case_type`, `status`, `priority`, `flagged`, `last_changed_at`, `created_at`, `state_snapshot`.
   - Query `PRAGMA table_info(case_flags)` and verify: `id`, `run_id`, `flag_id`, `flag_type`, `resolved`.
   - Query `PRAGMA index_list(cases)` and verify: `idx_status`, `idx_last_changed`, `idx_flagged`.
   - Query `PRAGMA index_list(case_flags)` and verify: `idx_case_flags_run_type`.

2. **Schema persists across restarts:**
   - Run `apply_migrations(tmp_db_path)`.
   - Insert a row: `INSERT INTO cases (run_id, status, flagged, last_changed_at, created_at, state_snapshot) VALUES ('test-run-1', 'running', 0, '2025-01-15T10:00:00Z', '2025-01-15T10:00:00Z', '{}')`.
   - Close connection.
   - Reopen connection to same `tmp_db_path`.
   - Query: `SELECT run_id, status FROM cases WHERE run_id='test-run-1'`.
   - Assert row is readable.

3. **Index used for status filter:**
   - Insert 10 cases with different statuses.
   - Run: `EXPLAIN QUERY PLAN SELECT * FROM cases WHERE status='running'`.
   - Assert query plan contains `SEARCH TABLE cases USING INDEX idx_status`.

4. **Index used for last_changed sort:**
   - Insert two cases: `run_id='a'` with `last_changed_at='2025-01-15T10:00:00Z'`, `run_id='b'` with `last_changed_at='2025-01-15T11:00:00Z'`.
   - Run: `EXPLAIN QUERY PLAN SELECT * FROM cases ORDER BY last_changed_at DESC`.
   - Assert query plan contains `SCAN TABLE cases USING INDEX idx_last_changed`.
   - Query: `SELECT run_id FROM cases ORDER BY last_changed_at DESC`.
   - Assert order is `['b', 'a']`.

5. **Index used for flagged filter:**
   - Insert case with `flagged=1`.
   - Run: `EXPLAIN QUERY PLAN SELECT * FROM cases WHERE flagged=1`.
   - Assert query plan contains `SEARCH TABLE cases USING INDEX idx_flagged`.

6. **Flag type filter joins correctly:**
   - Insert case `run_id='c'`, `status='needs_review'`.
   - Insert `case_flags`: `(run_id='c', flag_id='c-1', flag_type='netPayMismatch', resolved=0)`.
   - Insert `case_flags`: `(run_id='c', flag_id='c-2', flag_type='missingData', resolved=0)`.
   - Query:
     ```sql
     SELECT DISTINCT cases.run_id FROM cases
     JOIN case_flags ON cases.run_id = case_flags.run_id
     WHERE case_flags.flag_type = 'netPayMismatch' AND case_flags.resolved = 0
     ```
   - Assert result includes `'c'`.
   - Query same with `flag_type='missingData'`: assert result includes `'c'`.
   - Query with `flag_type='unknownType'`: assert empty result.
   - Run: `EXPLAIN QUERY PLAN` on above query; assert index `idx_case_flags_run_type` is used.

---

## Open Questions (from HLD/Breakdown)

1. **Case type derivation (HLD OQ1):** How is `case_type` extracted from `service_order`? Is it `service_order['caseType']` verbatim, or does it map to a normalized set?  
   **Impact:** Affects Story 2 extraction logic in `repository.save_case`. For this story (schema only), the column is `TEXT` and accepts any value or `NULL`.

2. **Wait time calculation (HLD OQ4):** How is "how long they have been waiting" computed for AC10? Is it `last_changed_at` when status became `needs_review`, or a separate `waiting_since` timestamp?  
   **Impact:** Current schema has no `waiting_since` column; Story 3 (API) uses `last_changed_at` as a proxy for sorting. If clarification requires a separate field, a second migration (`002_add_waiting_since.sql`) will be needed after this story.

---

## Dependencies

- **External:** `aiosqlite>=0.20.0` (added to `pyproject.toml`).
- **Internal:**
  - `src/agents/runtime/server.py` — startup hook calls `migrate.py`.
  - `src/agents/graph/states.py` — `State` structure defines the fields persisted in `cases.state_snapshot`.
  - `src/agents/runtime/backend.py` — `employee_name` helper will be used in Story 2 extraction logic (not this story).

---

## Notes

- **No placeholder data:** The schema is created empty; Story 4 (persistence hook) writes the first rows.
- **Transaction safety:** Each migration runs in a transaction; partial application is rolled back on error.
- **Idempotency:** Rerunning migrations is safe (already-applied versions are skipped).
- **Restart behavior:** The server fails to start if migrations fail (exit code 1); this prevents serving requests with an inconsistent schema.
- **Column types:** SQLite `TEXT` for strings, `BOOLEAN` (stored as `INTEGER 0/1`), `INTEGER` for autoincrement. ISO 8601 timestamps are stored as `TEXT` (standard SQLite practice).
- **Foreign key enforcement:** SQLite foreign keys are enabled by default in `aiosqlite`; `ON DELETE CASCADE` ensures deleting a case also deletes its flags.
- **Index selectivity:** Partial index `idx_flagged` (where `flagged=1`) saves space and speeds up the common query (flagged cases are rare).
