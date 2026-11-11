# Low-Level Design — Story 04-agent

**Persistence hook for state updates**

---

## Acceptance Criteria

From `requirements.md` and `breakdown.md`:

- **REQ-AC12:** Cases persist across restarts — hook writes to SQLite after every state update.
- **REQ-AC1:** List includes all cases — hook ensures every case is saved as it runs.

---

## File Plan

All paths are relative to `{project-root}` (the `agent` repo root).

### New Files

1. **`src/agents/persistence/persistence_hook.py`**  
   LangGraph checkpointer write hook: calls `repository.save_case(state)` after each checkpoint write.

### Modified Files

1. **`src/agents/graph/build.py`**  
   Register `persistence_hook` with the checkpointer (attach in `build_graph()` before calling `graph.compile()`).

2. **`src/agents/runtime/checkpointer.py`**  
   Add a wrapper function `get_checkpointer_with_hook` that attaches the persistence hook to the checkpointer returned by `get_checkpointer`.

---

## Interfaces

### `src/agents/persistence/persistence_hook.py`

```python
from pathlib import Path
from typing import Any

async def on_checkpoint_write(state: dict[str, Any], db_path: Path) -> None:
    """Called after each LangGraph checkpoint write.
    
    Extracts case metadata and full state from the State dict, then writes
    to the cases and case_flags tables via repository.save_case.
    
    Wraps the checkpointer's aput method since LangGraph 1.2.11 does not
    provide a native write hook API.
    
    Args:
        state: State dict from graph/states.py (after a node completes).
        db_path: Path to the SQLite database file (e.g. data/cases.db).
    
    Raises:
        Any exception raised by repository.save_case (caller logs and fails the run).
    """
```

### `src/agents/runtime/checkpointer.py` (modified)

```python
def get_checkpointer_with_hook(
    kind: str = "memory",
    db_path: Path | None = None,
    cases_db_path: Path | None = None,
) -> BaseCheckpointSaver:
    """Get a checkpointer with the persistence hook attached.
    
    Wraps the checkpointer's aput method to call repository.save_case after
    each checkpoint write, since LangGraph 1.2.11 does not provide a native
    write hook API.
    
    Args:
        kind: Checkpointer kind ('memory', 'sqlite', 'redis').
        db_path: Path to the checkpointer's database file (for 'sqlite').
        cases_db_path: Path to the case persistence database file (for the hook).
    
    Returns:
        A checkpointer with aput wrapped to call the persistence hook.
    """
```

### `src/agents/graph/build.py` (modified)

```python
def build_graph(
    checkpointer_kind: str = "memory",
    db_path: Path | None = None,
    cases_db_path: Path | None = None,
):
    # ... (existing node setup)
    
    # NEW: use get_checkpointer_with_hook instead of get_checkpointer
    checkpointer = get_checkpointer_with_hook(checkpointer_kind, db_path, cases_db_path)
    return graph.compile(checkpointer=checkpointer)
```

## Algorithm Notes

### Persistence Hook (`persistence_hook.py`)

1. **Extract fields from state:**
   - `run_id` ← `state['run_id']`
   - `service_order_id` ← `state.get('service_order_id')` (optional key)
   - `service_order` ← `state.get('service_order')` (dict or `None`)
   - `employee_name` ← `backend.employee_name(service_order)` (may return `None`)
   - `case_type` ← `service_order.get('caseType')` if `service_order` else `None` (placeholder logic; see Open Question 1)
   - `status` ← `state['status']`
   - `priority` ← `state.get('priority')` (optional)
   - `flagged` ← `state.get('flagged', False)` (boolean, default `False`)
   - `flags` ← `state.get('flags', [])` (list of Flag dicts)

2. **Call repository:**
   - `await repository.save_case(state, db_path)`
   - The repository handles `last_changed_at`, `created_at`, `state_snapshot`, and bulk-replacing `case_flags` rows (Story 2 implementation).

3. **Error handling:**
   - If `save_case` raises an exception, the hook does **not** catch it — the caller (LangGraph checkpointer write path) logs the error and fails the run (per HLD AD-5: "write failure fails the run").
   - This ensures the checkpoint write is atomic: if the case persistence fails, the checkpoint is rolled back (SQLite transaction semantics).

### Checkpointer Hook Registration (`checkpointer.py`)

1. **Resolve cases database path:**
   - If `cases_db_path` is provided, use it.
   - Else, default to `{project-root}/data/cases.db` (same logic as `_db_path()` in Story 3).
   - Ensure parent directory exists: `cases_db_path.parent.mkdir(parents=True, exist_ok=True)`.

2. **Get base checkpointer:**
   - `saver = get_checkpointer(kind, db_path)` (existing function).

3. **Attach write hook:**
   - LangGraph checkpointers support `on_checkpoint_write` hooks.
   - Define a closure that captures `cases_db_path`:
     ```python
     async def _hook(checkpoint: dict, config: dict, **kwargs):
         # Extract state from checkpoint
         state = checkpoint.get("channel_values", {})
         if not state:
             return  # empty checkpoint (initial state before first node)
         await on_checkpoint_write(state, cases_db_path)
     ```
   - Register the hook: `saver.on_checkpoint_write = _hook` (or use the checkpointer's API if it provides a registration method).
   - **Note:** LangGraph's `AsyncSqliteSaver` and `InMemorySaver` do not natively support write hooks as of LangGraph 1.2.11. Instead, wrap the checkpointer's `aput` method:
     ```python
     original_aput = saver.aput
     async def aput_with_hook(config, checkpoint, metadata, **kwargs):
         result = await original_aput(config, checkpoint, metadata, **kwargs)
         state = checkpoint.get("channel_values", {})
         if state:
             await on_checkpoint_write(state, cases_db_path)
         return result
     saver.aput = aput_with_hook
     ```

4. **Return wrapped checkpointer:**
   - `return saver`

### Build Graph Integration (`build.py`)

1. **Resolve cases database path:**
   - Default to `{project-root}/data/cases.db` (same as Story 3).
   - Accept `cases_db_path` as an optional parameter to `build_graph` for test flexibility.

2. **Call `get_checkpointer_with_hook`:**
   - Replace `get_checkpointer(checkpointer_kind, db_path)` with `get_checkpointer_with_hook(checkpointer_kind, db_path, cases_db_path)`.

3. **Compile graph:**
   - `return graph.compile(checkpointer=checkpointer)` (unchanged).

---

## Tests

Six tests as specified in Verification Approach:

1. **Hook writes a case row after the first node completes:**
   - Start a run: `POST /runs` with `{"service_order_id": "SO-1001"}`.
   - Wait for the first node (e.g. `planner`) to complete (poll `GET /runs/{thread_id}` until `len(state['steps']) > 0`).
   - Query `cases` table: `SELECT run_id, status, service_order_id FROM cases WHERE run_id = <thread_id>`.
   - Assert row exists with `status='running'`, `service_order_id='SO-1001'`.
   - Verify `last_changed_at` and `created_at` are ISO 8601 timestamps.

2. **Hook updates the case row after each node:**
   - Continue the run from test 1 (let it progress through `validation`, `compliance`, etc.).
   - After each node completion, query `cases.last_changed_at`.
   - Assert `last_changed_at` increases (later timestamp) after each checkpoint write.
   - Verify `status` changes from `'running'` to `'needs_review'` (or `'done'`) as the run progresses.

3. **Hook inserts case_flags rows:**
   - Let the run from test 1 progress until a flag is raised (e.g. `validation` raises a `missingData` flag).
   - Query `case_flags`: `SELECT flag_id, flag_type, resolved FROM case_flags WHERE run_id = <thread_id>`.
   - Assert at least one row exists with `flag_type='missingData'`, `resolved=0`.
   - Verify the `flag_id` matches the one in `state['flags']` (e.g. `validation-1`).

4. **Cases persist across restarts:**
   - Start a run: `POST /runs` with `{"service_order_id": "SO-1002"}`.
   - Let it progress to `status='needs_review'` (paused for review).
   - Record the `status` and `last_changed_at` from `cases` table.
   - Restart the server (kill the process, start it again).
   - Query `GET /cases?status=needs_review`.
   - Assert the case with `service_order_id='SO-1002'` appears in the list with the same `status` and `last_changed_at` as before the restart.

5. **Resume reads from persisted state:**
   - Continue from test 4: the run is paused at `needs_review`.
   - Resume the run: `POST /runs/{thread_id}/resume` with `{"action": "approve"}`.
   - Verify the run continues (check `GET /runs/{thread_id}` for new steps).
   - Query `cases.state_snapshot` for the resumed run.
   - Assert the `state_snapshot` JSON contains the `review` decision (`{"action": "approve"}`).

6. **Write failure fails the run:**
   - Mock `repository.save_case` to raise an exception (e.g. `sqlite3.OperationalError("database is locked")`).
   - Start a run: `POST /runs` with `{"service_order_id": "SO-1003"}`.
   - Wait for the first node to complete.
   - Assert the run status is `'failed'` (check `GET /runs/{thread_id}`).
   - Assert the error message in the response contains `"database is locked"` (or similar).
   - Verify the checkpoint write was rolled back (no row in `cases` table for this `run_id`).

---

## Open Questions (from HLD/Breakdown)

1. **Case type derivation (HLD OQ1):** How is `case_type` extracted from `service_order`? Is it `service_order['caseType']` verbatim, or does it map to a normalized set?  
   **Impact:** Current extraction logic in the hook uses `service_order.get('caseType')` as a placeholder. If a mapping is required, add a helper function `_normalize_case_type(service_order)` in `persistence_hook.py` after clarification.

2. **Checkpointer write hook API:** LangGraph 1.2.11 does not expose a native `on_checkpoint_write` hook. The current implementation wraps `aput` (see Algorithm Notes). Verify whether a future LangGraph version provides a cleaner API.  
   **Impact:** If LangGraph adds a native hook API, refactor `get_checkpointer_with_hook` to use it instead of wrapping `aput`.

3. **Database path resolution:** Should `cases.db` be per-run (in each run's `logs/...` folder) or project-wide (in `data/cases.db`)?  
   **Impact:** This story assumes **project-wide `data/cases.db`** (matching Story 3's decision). If per-run is required, `cases_db_path` would need to be derived from `run_dir(thread_id)` instead.

4. **Wait time calculation (HLD OQ4):** Story 2 uses `last_changed_at` as a proxy for wait time. If a separate `waiting_since` field is required (timestamp when status became `needs_review`), the hook would need to detect status transitions and set `waiting_since` accordingly.  
   **Impact:** Current implementation updates `last_changed_at` on every write; no special handling for `needs_review` transitions. If `waiting_since` is added, the hook must check `state['status'] == 'needs_review' and previous_status != 'needs_review'` before calling `save_case`.

---

## Dependencies

- **External:** `aiosqlite>=0.20.0` (added to `pyproject.toml` in Story 1).
- **Internal:**
  - `src/agents/persistence/repository.py` — `save_case` function (Story 2).
  - `src/agents/runtime/backend.py` — `employee_name(service_order)` helper.
  - `src/agents/graph/states.py` — `State` structure defines the fields extracted by the hook.
  - `src/agents/runtime/checkpointer.py` — `get_checkpointer` function (existing).
  - `src/agents/graph/build.py` — `build_graph` function (existing).
  - `src/agents/runtime/run_dir.py` — `LOG_DIR` for resolving `data/` folder (existing).
  - `json` (stdlib) — for `json.dumps(state)` in `repository.save_case` (not directly used in the hook, but a dependency of Story 2).

---

## Notes

- **Synchronous execution:** The hook runs in the checkpoint write path, so `save_case` is called synchronously (awaited) before the checkpoint write completes. A write failure prevents the checkpoint from being committed.
- **No checkpointer changes:** The existing `get_checkpointer` function is unchanged; `get_checkpointer_with_hook` is a new wrapper function that delegates to it.
- **Database path default:** `cases_db_path` defaults to `{project-root}/data/cases.db` (same logic as Story 3's `_db_path()` helper).
- **Case type placeholder:** The hook uses `service_order.get('caseType')` as a placeholder for `case_type`. If a mapping is required (e.g. normalizing `'Offboarding'` to `'termination'`), add a helper function after clarifying Open Question 1.
- **Flag extraction:** Flags in `state['flags']` are passed to `repository.save_case`, which bulk-replaces `case_flags` rows (Story 2 implementation).
- **Empty checkpoint handling:** The hook checks whether `state` is empty (initial checkpoint before the first node runs) and skips the `save_case` call if so, to avoid inserting a row with incomplete data.
- **Error logging:** The hook does **not** catch exceptions from `repository.save_case` — the caller (LangGraph checkpointer) logs the error and fails the run. This ensures atomicity: no checkpoint without a persisted case.
- **Test isolation:** Tests use a temporary `cases.db` file (via `tmp_path` fixture) to avoid polluting the project-wide database.
