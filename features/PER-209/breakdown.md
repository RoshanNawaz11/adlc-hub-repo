# Breakdown — PER-209

**Reviewers can see, filter and count every case**

---

## Overview

This epic adds persistent storage and a REST API so reviewers and leads can see all cases, filter them by status/priority/flags, sort by wait time, and retrieve aggregate counts. Cases survive restarts with their last known state.

The breakdown separates backend persistence/API from any future frontend integration. Each story is independently testable against its acceptance criteria.

---

## Dependencies

- **Repo context:** `agent` (Python, FastAPI, SQLite via `aiosqlite`, LangGraph)
- **Existing modules:**
  - `src/agents/graph/states.py` — `State` structure defines persisted fields
  - `src/agents/runtime/checkpointer.py` — Hook attachment point for persistence
  - `src/agents/runtime/server.py` — New routes added here
  - `src/agents/runtime/backend.py` — `employee_name` helper reused
  - `pyproject.toml` — Add `aiosqlite` dependency
- **New modules (to be created):**
  - `src/agents/persistence/repository.py` — Async SQLite queries
  - `src/agents/persistence/migrate.py` — Migration runner
  - `src/agents/persistence/migrations/001_create_cases.sql` — Schema initialization
  - `src/agents/runtime/persistence_hook.py` — LangGraph checkpointer write hook

---

## Allowed Paths

All new files sit under `src/agents/`, matching the existing tree:
- `src/agents/persistence/` (new directory)
  - `repository.py`
  - `migrate.py`
  - `migrations/` (new directory)
    - `001_create_cases.sql`
- `src/agents/runtime/persistence_hook.py`

Modified files:
- `src/agents/graph/build.py`
- `src/agents/runtime/server.py`
- `pyproject.toml`

---

## Stories (Build Order)

### Story 1: Case persistence schema and migration runner

**Repo:** `agent`

**Description:**  
Create the SQLite schema for case storage (`cases` and `case_flags` tables) and a migration runner that applies unapplied migrations on server startup. The schema holds every case's `run_id`, service order ID, employee name, case type, status, priority, flagged state, last-changed timestamp, and full `State` snapshot as JSON. The `case_flags` table tracks unresolved flags by type. Indexes support filtering and sorting.

**Covers:**
- REQ-AC12 (cases persist across restarts — schema holds their last state)
- REQ-AC1 (list includes all cases — schema stores every case regardless of status)
- REQ-AC2 (each case shows service order ID, employee, case type, status, priority, flagged, last changed — schema columns match)
- REQ-AC9 (filter by specific flag type — `case_flags` table and index support it)

**After:** (none — first story)

**Verification Approach:**
1. Run the migration script; verify `cases` and `case_flags` tables exist with the specified columns and indexes.
2. Insert a row; restart the server; verify the row is still readable (schema persists).
3. Query cases by status; verify the `idx_status` index is used (EXPLAIN QUERY PLAN).
4. Insert two cases with different `last_changed_at`; verify sorting by that column uses `idx_last_changed`.
5. Insert a case with `flagged=1`; verify `idx_flagged` is used when filtering flagged cases.
6. Insert `case_flags` rows; verify filtering by `flag_type` joins correctly.

**Test count:** 6

**Notes:**
- Schema details in HLD AD-3.
- Migration runner follows standard sequential-file pattern (`{seq}_{description}.sql`).
- Open question (HLD OQ1): how `case_type` is derived from `service_order` — placeholder `None` when missing; verify extraction logic before story LLD.
- Open question (HLD OQ4): wait time calculation — assume `last_changed_at` for now; may need `waiting_since` column amendment after clarification.

---

### Story 2: Repository layer for case queries

**Repo:** `agent`

**Description:**  
Implement `src/agents/persistence/repository.py` with async functions for saving a case (`save_case`), listing cases with filters (`list_cases`), retrieving counts (`get_counts`), and loading a case by `run_id` (`load_case`). All functions use `aiosqlite` for async I/O. `save_case` extracts what it needs from the `State` dict it receives: `run_id`, `service_order_id`, `employee_name` (via `backend.employee_name(state.get('service_order'))`), `case_type` (from `service_order` or `None`), `status`, `priority`, `flagged`, `last_changed_at` (current timestamp), and serializes the full `state` to JSON for `state_snapshot`. It then inserts or updates the `cases` row and bulk-replaces `case_flags` rows (deleting existing flags for the `run_id` first, then inserting rows for each flag in `state['flags']`). `list_cases` builds `WHERE` clauses from filter parameters and `ORDER BY` from `sort_by`. `get_counts` uses `COUNT(*)` with `GROUP BY status`.

**Covers:**
- REQ-AC1 (list endpoint returns all cases — `list_cases` selects from `cases` with optional filters)
- REQ-AC4 (filter by status — `list_cases` builds `WHERE status IN (...)`)
- REQ-AC5 (filter by priority — `WHERE priority IN (...)`)
- REQ-AC6 (filter by flagged state — `WHERE flagged = ...`)
- REQ-AC7 (filter by case type — `WHERE case_type IN (...)`)
- REQ-AC8 (combine multiple filters — `WHERE` clauses are ANDed)
- REQ-AC9 (filter by flag type — join `case_flags` where `flag_type = ... AND resolved = 0`)
- REQ-AC10 (sort by wait time — `ORDER BY last_changed_at ASC` when `status = 'needs_review'`)
- REQ-AC11 (counts endpoint — `get_counts` groups by status and counts flagged waiting cases)
- REQ-AC12 (cases persist across restarts — `save_case` writes synchronously; `list_cases` reads back)

**After:** Story 1 (schema must exist before repository can query it)

**Verification Approach:**
1. Call `save_case` with a `State` dict; verify a `cases` row is inserted with correct columns.
2. Call `save_case` again with the same `run_id` but changed `status`; verify the row is updated.
3. Save a case with flags; verify `case_flags` rows are inserted; save again with different flags; verify old rows are deleted and new ones inserted.
4. Call `list_cases` with no filters; verify all cases are returned.
5. Call `list_cases` with `statuses=['running']`; verify only `running` cases are returned.
6. Call `list_cases` with `priorities=['high', 'medium']`; verify only those priorities are returned.
7. Call `list_cases` with `flagged=True`; verify only flagged cases are returned.
8. Call `list_cases` with `case_types=['Offboarding']`; verify only that type is returned.
9. Call `list_cases` with `statuses=['needs_review']` and `flagged=True`; verify intersection (AC8).
10. Call `list_cases` with `flag_type='netPayMismatch'`; verify only cases with unresolved `netPayMismatch` flags are returned (join `case_flags` where `resolved=0`).
11. Call `list_cases` with `sort_by='last_changed'`; verify cases are ordered by `last_changed_at DESC`.
12. Call `list_cases` with `statuses=['needs_review']` and `sort_by='wait_time'`; verify `ORDER BY last_changed_at ASC`.
13. Call `list_cases` with `page=2` and `page_size=10`; verify `LIMIT 10 OFFSET 10`.
14. Call `get_counts` with `by_status=True` and `flagged=True`; verify counts per status and flagged waiting count.
15. Call `load_case` with a known `run_id`; verify the full `State` dict is returned from `state_snapshot`.

**Test count:** 15

**Notes:**
- HLD AD-4 specifies function signatures.
- `save_case` owns the extraction logic: it takes the `State` dict and pulls out the fields it needs. For each field: `run_id` from `state['run_id']`, `service_order_id` from `state.get('service_order_id')`, `employee_name` via `backend.employee_name(state.get('service_order'))`, `case_type` from `state.get('service_order', {}).get('caseType')` or `None`, `status` from `state['status']`, `priority` from `state.get('priority')`, `flagged` from `state['flagged']`, `last_changed_at` as the current ISO 8601 timestamp, `state_snapshot` as `json.dumps(state)`. Flags come from `state['flags']`: for each flag, insert a row into `case_flags` with `run_id`, `flag['flag_id']`, `flag['type']`, and `flag.get('resolved', False)`.
- Default `sort_by='last_changed'` (descending); `wait_time` only valid when `status=needs_review` (enforced in Story 3 API layer).

### Story 3: Case list and counts API endpoints

**Repo:** `agent`

**Description:**  
Add `GET /cases` and `GET /cases/counts` routes to `src/agents/runtime/server.py`. Validate filter parameters (status, priority, flag_type, case_type) against allowed values; reject invalid values with a `400 Bad Request` naming allowed options. Validate `page_size` (max 100); reject out-of-range values. Call `repository.list_cases` and `repository.get_counts` with validated inputs and return JSON responses per the HLD API contracts.

**Covers:**
- REQ-AC1 (list endpoint returns all cases — `GET /cases` route exists)
- REQ-AC2 (each case shows service order ID, employee, case type, status, priority, flagged, last changed — response format matches)
- REQ-AC3 (default sort and pagination — default `page_size=25`, `sort=last_changed` descending)
- REQ-AC4 through REQ-AC10 (filtering and sorting — parameters validated and passed to repository)
- REQ-AC11 (counts endpoint — `GET /cases/counts` route exists)
- REQ-AC13 (invalid filter values rejected — validation at API entry per HLD AD-7)

**After:** Story 2 (repository functions must exist before routes can call them)

**Verification Approach:**
1. `GET /cases` with no query params; verify response has `cases`, `total`, `page=1`, `page_size=25`.
2. Verify each case object has `run_id`, `service_order_id`, `employee_name`, `case_type`, `status`, `priority`, `flagged`, `last_changed_at`.
3. `GET /cases` without `page_size`; verify 25 cases returned.
4. `GET /cases?page_size=50`; verify 50 cases returned.
5. `GET /cases?page_size=150`; verify `400 Bad Request` with message `"page_size must be between 1 and 100"`.
6. `GET /cases?status=running,done`; verify only those statuses appear.
7. `GET /cases?priority=high`; verify only high-priority cases appear.
8. `GET /cases?flagged=true`; verify all returned cases have `flagged=true`.
9. `GET /cases?case_type=Offboarding`; verify only that type appears.
10. `GET /cases?status=needs_review&flagged=true`; verify intersection of both filters.
11. `GET /cases?flag_type=netPayMismatch`; verify only cases with unresolved `netPayMismatch` flags appear.
12. `GET /cases?status=needs_review&sort=wait_time`; verify cases ordered by `last_changed_at ASC`.
13. `GET /cases?status=invalid_status`; verify `400 Bad Request` with `"Invalid status: invalid_status. Allowed: ['running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed']"`.
14. `GET /cases?priority=unknown_priority`; verify `400 Bad Request` naming allowed priorities.
15. `GET /cases/counts?by_status=true&flagged=true`; verify response has `by_status` (counts per status) and `flagged_waiting` (count of flagged `needs_review` cases).
16. Verify `/cases/counts` does not load the full case list (check no `SELECT * FROM cases` in logs/profiler).

**Test count:** 16

**Notes:**
- HLD AD-2 specifies response formats and AD-7 specifies validation rules.
- Allowed status values: `{'running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed'}`.
- Allowed priority values: `{'high', 'medium', 'low'}`.
- `flag_type` and `case_type` are freeform strings (no enum).
- `sort=wait_time` is only valid when `status=needs_review`; reject otherwise with `400 Bad Request`.

---

### Story 4: Persistence hook for state updates

**Repo:** `agent`

**Description:**  
Implement `src/agents/runtime/persistence_hook.py` to call `repository.save_case(state)` after each LangGraph checkpoint write. The hook receives the `State` dict and passes it to `save_case`, which does the extraction (see Story 2). Register the hook in `src/agents/graph/build.py` by attaching it to the checkpointer before compilation.

**Covers:**
- REQ-AC12 (cases persist across restarts — hook writes to SQLite after every state update)
- REQ-AC1 (list includes all cases — hook ensures every case is saved as it runs)

**After:** Story 2 (repository layer must exist before hook can call `save_case`)

**Verification Approach:**
1. Run a case (via `POST /runs`); verify a `cases` row is inserted after the first node completes.
2. Let the case progress through multiple nodes; verify the `cases` row is updated after each checkpoint write (check `last_changed_at` changes).
3. Add flags to the state; verify `case_flags` rows are inserted after the checkpoint write.
4. Restart the server mid-run; verify the case still appears in `GET /cases` with the status it had before restart.
5. Resume the run; verify the case continues from its last persisted state (read from `cases.state_snapshot`).
6. Verify the hook runs synchronously in the checkpoint write path (a write failure fails the run and logs an error).

**Test count:** 6

**Notes:**
- The hook's job is to call `save_case(state)` after each checkpoint write. `save_case` (Story 2) owns the extraction of fields from `state`. The hook does not duplicate that logic.
- `employee_name` is derived via `backend.employee_name(state.get('service_order'))` inside `save_case`.
- `case_type` extraction heuristic is still an open question (HLD OQ1); `save_case` uses placeholder logic for now (verify in LLD).
- Flags in `state['flags']` are bulk-inserted into `case_flags` by `save_case`; existing rows for the `run_id` are deleted first to avoid duplicates.

### Story 5: Single-case operations remain unchanged

**Repo:** `agent`

**Description:**  
Verify that `GET /runs/{thread_id}`, `POST /runs`, and `POST /runs/{thread_id}/resume` still work as they do today. These routes read from the LangGraph checkpointer, not from SQLite. When a run existed before a restart and the checkpointer is empty, `repository.load_case(run_id)` returns the `State` from `cases.state_snapshot` as a fallback.

**Covers:**
- REQ-AC14 (single case operations unchanged — existing endpoints work as today)
- REQ-AC12 (cases persist across restarts — `/runs/{thread_id}` reads from SQLite as fallback)

**After:** Story 4 (persistence hook must be writing before we can test the fallback)

**Verification Approach:**
1. `POST /runs` with a service order ID; verify the run starts and progresses as before.
2. `GET /runs/{thread_id}` during the run; verify the response includes `values` from the checkpointer.
3. `POST /runs/{thread_id}/resume` with a flag answer; verify the run resumes and continues.
4. Restart the server; `GET /runs/{thread_id}` for a run that existed before restart; verify `values` is loaded from `cases.state_snapshot` (fallback when checkpointer is empty).
5. `POST /runs/{thread_id}/resume` after restart; verify the run resumes from the persisted state.

**Test count:** 5

**Notes:**
- HLD AD-6 specifies the fallback logic.
- No changes to `src/agents/runtime/server.py` single-case routes beyond adding `repository.load_case` as a fallback in `_snapshot`.
- Checkpointer is source of truth for active runs; SQLite is the fallback for runs that existed before restart.

---

## Open Questions (from HLD)

1. **Case type derivation:** How is `case_type` extracted from `service_order`? Is it `service_order['caseType']` verbatim, or does it map to a normalized set? *(Affects Story 1 schema and Story 2 extraction logic.)*
2. **Flag type normalization:** Are flag types (`netPayMismatch`, `missingData`, etc.) hardcoded enums or freeform strings? *(Affects Story 3 validation.)*
3. **"Waiting for review" vs "needs_review":** AC uses both terms; are they the same status? *(Assume `needs_review` for now; verify in requirements.)*
4. **Wait time calculation:** How is "how long they have been waiting" computed for AC10? Is it `last_changed_at` when status became `needs_review`, or a separate `waiting_since` timestamp? *(No field in current schema; Story 1 may need amendment after clarification.)*

---

## Summary

- **5 stories** in build order: schema → repository → API → hook → verification of unchanged operations.
- **Backend-focused:** All stories implement persistence, query layer, and API routes. No UI changes.
- **Independently testable:** Each story has explicit verification steps and test counts that match its ACs.
- **Consistent with HLD:** Schema, API contracts, and repository functions match HLD architectural decisions.
- **Total test count:** 6 + 15 + 16 + 6 + 5 = **48 tests** across all stories.
