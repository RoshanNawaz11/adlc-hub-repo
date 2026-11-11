# Low-Level Design — Story 05-agent

**Single-case operations remain unchanged**

---

## Acceptance Criteria

From `requirements.md` and `breakdown.md`:

- **REQ-AC14:** Single case operations unchanged — existing endpoints work as today.
- **REQ-AC12:** Cases persist across restarts — `/runs/{thread_id}` reads from SQLite as fallback.

---

## File Plan

All paths are relative to `{project-root}` (the `agent` repo root).

### New Files

None — this story verifies existing endpoints and adds a fallback read path.

### Modified Files

1. **`src/agents/runtime/server.py`**  
   Add fallback logic in `_snapshot` to load state from `repository.load_case(run_id)` when the checkpointer is empty (post-restart scenario). Add `_db_path()` helper. Modify `resume_run` to re-initialize the checkpointer from SQLite when resuming after a restart.

---

## Interfaces

### `src/agents/runtime/server.py` (modified)

```python
def _db_path() -> Path:
    """Resolve the database path for case persistence.
    
    Same logic as Story 3: project-wide data/cases.db.
    """

async def _snapshot(thread_id: str):
    """The run's latest state, read from its own checkpointer.
    
    Fallback: if the checkpointer is empty (e.g. after a restart),
    load the state from cases.state_snapshot via repository.load_case.
    
    Returns a snapshot object with:
    - values: State dict (from checkpointer or SQLite)
    - next: tuple of pending node names (empty for SQLite fallback)
    - interrupts: list of interrupt values
    - checkpointer_was_empty: bool (True when loaded from SQLite)
    """
```

**Existing signatures (unchanged):**

```python
@app.post('/runs', status_code=202)
async def create_run(req: RunRequest) -> dict[str, Any]:
    """Start a new run with a service order ID or query text."""

@app.post('/runs/{thread_id}/resume', status_code=202)
async def resume_run(thread_id: str, decision: FlagAnswers | ReviewDecision) -> dict[str, Any]:
    """Resume a paused run with flag answers or a review decision."""

@app.get('/runs/{thread_id}')
async def get_run(thread_id: str) -> dict[str, Any]:
    """Get the current state of a run."""
```

---

## Algorithm Notes

### Database Path Helper

```python
def _db_path() -> Path:
    """Resolve the database path for case persistence.
    
    Same logic as Story 3: project-wide data/cases.db.
    """
    from agents.runtime.run_dir import LOG_DIR
    db_dir = LOG_DIR.parent / "data"
    db_dir.mkdir(parents=True, exist_ok=True)
    return db_dir / "cases.db"
```

### Modified `_snapshot` Function

1. **Open the checkpointer:**
   - Determine which checkpointer kind to use (see below).
   - `async with open_graph(thread_id, checkpointer_kind=checkpointer_kind, db_path=db_path) as graph:`
   - `snapshot = await graph.aget_state({"configurable": {"thread_id": thread_id}})`

2. **Check if checkpointer has state:**
   - `if snapshot.values:` → return `snapshot` (active run case; existing behavior).

3. **Fallback to SQLite (post-restart case):**
   - Import repository: `from agents.persistence import repository`
   - Resolve database path: `db_path = _db_path()`
   - `state = await repository.load_case(thread_id, db_path)`.
   - If `state is None`: return `snapshot` (run not found in either checkpointer or SQLite; existing 404 handling in `get_run` will apply).
   - Else: construct a synthetic snapshot:
     ```python
     from types import SimpleNamespace
     return SimpleNamespace(
         values=state,
         next=(),  # no pending nodes (run finished or paused before restart)
         interrupts=[],
         checkpointer_was_empty=True,  # flag for resume_run
     )
     ```

4. **Return the snapshot:**
   - For checkpointer path: `return snapshot`.
   - For SQLite fallback: `return SimpleNamespace(...)`.

**Checkpointer configuration:** The server needs to pass the same `checkpointer_kind` it uses for runs. Add a module-level variable at the top of `server.py`:

```python
CHECKPOINTER_KIND = os.getenv("CHECKPOINTER", "sqlite")
```

Then `_snapshot` resolves the database path (if `CHECKPOINTER_KIND == "sqlite"`) and calls `open_graph(thread_id, checkpointer_kind=CHECKPOINTER_KIND, db_path=db_path)`. The `db_path` for the checkpointer is the run's own `checkpoint.db` (from `run_dir(thread_id) / "checkpoint.db"`), not the project-wide `data/cases.db` (which is only for the persistence fallback in step 3).

**Assumption ASSUMPTION C6: The server's checkpointer kind is configured via the `CHECKPOINTER` environment variable (default `"sqlite"`), and `_snapshot` uses this same configuration when opening the graph. The run's checkpointer database is `run_dir(thread_id) / "checkpoint.db"` (same as what `runner.py` uses), and the persistence fallback reads from the project-wide `data/cases.db` via `repository.load_case`.**

### Modified `resume_run` Function

1. **Existing validation:**
   - Check `_runs` in-memory status (409 if running).
   - Call `snapshot = await _snapshot(thread_id)`.
   - Check `snapshot.next` (409 if not paused).
   - Validate decision type matches pause state (422 if mismatch).
   - Validate flag answers completeness (422 if missing).

2. **NEW: Re-initialize checkpointer from SQLite:**
   - After `snapshot = await _snapshot(thread_id)`:
   - Check if `getattr(snapshot, "checkpointer_was_empty", False)`:
     ```python
     if getattr(snapshot, "checkpointer_was_empty", False):
         async with open_graph(thread_id) as graph:
             await graph.update_state(
                 {"configurable": {"thread_id": thread_id}},
                 snapshot.values,
             )
     ```

3. **Continue existing flow:**
   - Convert decision to dict: `body = decision.model_dump(exclude_none=True)`.
   - Call `_start(thread_id, None, resume=True, decision=body)`.

### Existing Endpoint Behavior (No Changes)

- **`POST /runs`**: Calls `_start(thread_id, req, resume=False)`, which invokes `run_query(...)` and writes to the checkpointer as today. Story 4's persistence hook writes to SQLite in parallel.
- **`GET /runs/{thread_id}`**: Calls `_snapshot(thread_id)` and merges its `values` with the `_runs` in-memory cache (if present). The fallback now ensures `snapshot.values` is populated from SQLite when the checkpointer is empty.

---

## Tests

Five tests as specified in Verification Approach:

1. **POST /runs starts and progresses as before:**
   - `POST /runs` with `{"service_order_id": "SO-1005"}`.
   - Poll `GET /runs/{thread_id}` until `status != 'running'` (or a timeout).
   - Assert the run progresses through stages: `values['steps']` grows, `values['status']` transitions (`'running'` → `'needs_review'` or `'done'`).
   - Verify the response shape: `{"thread_id": <id>, "status": <status>, "values": {...}, "paused": <bool>, "next": [...]}`.
   - Assert `repository.save_case` was called at least once (check SQLite: `SELECT run_id FROM cases WHERE run_id = <thread_id>`; row exists).

2. **GET /runs/{thread_id} during the run returns checkpointer values:**
   - Continue from test 1 (run is still active).
   - `GET /runs/{thread_id}`.
   - Assert response includes `values` with keys: `run_id`, `status`, `steps`, `flags`, `results`, etc. (from `graph/states.py` `State` structure).
   - Verify `values['run_id'] == thread_id`.
   - Verify `values['status']` matches the current run state (e.g. `'running'`, `'needs_review'`).

3. **POST /runs/{thread_id}/resume continues the run:**
   - Start a run that pauses for review: `POST /runs` with a service order that raises flags (use a test fixture or mock `validation_node` to raise a flag).
   - Poll `GET /runs/{thread_id}` until `paused == true` and `next` includes `'flag-review'`.
   - `POST /runs/{thread_id}/resume` with:
     ```json
     {
       "answers": [
         {"flag_id": "<flag_id from values['flags']>", "action": "accept"}
       ]
     }
     ```
   - Poll `GET /runs/{thread_id}` again; verify the run continues: `paused` becomes `false`, `values['flags'][0]['answer']` is populated.
   - Verify the run eventually completes or reaches a new pause state.

4. **Restart fallback: GET /runs/{thread_id} reads from SQLite after restart:**
   - Start a run: `POST /runs` with `{"service_order_id": "SO-1006"}`.
   - Let it progress to `status='needs_review'` (paused for review).
   - Record the `values` from `GET /runs/{thread_id}` (capture `status`, `flags`, `steps`).
   - Stop the server (kill the process).
   - Delete the checkpointer database file: `rm logs/<run_folder>/checkpoint.db` (or truncate its tables if shared).
   - Start the server again.
   - `GET /runs/{thread_id}`.
   - Assert the response still includes `values` (loaded from `cases.state_snapshot` via `repository.load_case`).
   - Verify `values['status']`, `values['flags']`, and `values['steps']` match the captured state from before the restart.
   - Verify the response has `paused=false` and `next=[]` (fallback snapshot has no pending nodes; see Algorithm Notes step 3).

5. **Resume after restart:**
   - Continue from test 4 (run exists in SQLite, checkpointer is empty).
   - `POST /runs/{thread_id}/resume` with a review decision:
     ```json
     {"action": "approve"}
     ```
   - **Expected behavior:** The resume re-initializes the checkpointer from `cases.state_snapshot` (via the `checkpointer_was_empty` flag in `_snapshot`), applies the decision, and continues the run.
   - **Verification:** Poll `GET /runs/{thread_id}`; verify the run progresses past the pause state (new steps appear in `values['steps']`).
   - Verify the run eventually reaches `'done'` or `'failed'` status.

---

## Open Questions (from HLD/Breakdown)

1. **Resume from SQLite initialization (RESOLVED):** The current `run_query(..., resume=True)` does not handle re-seeding the checkpointer from SQLite. The fix is in `resume_run`: after `_snapshot(thread_id)` returns a snapshot with `checkpointer_was_empty=True`, call `graph.update_state(config, snapshot.values)` before invoking `_start`.  
   **Status:** Addressed in Algorithm Notes above. `_snapshot` sets the flag, `resume_run` checks it and writes the state back to the checkpointer.

2. **Case type derivation (HLD OQ1):** How is `case_type` extracted from `service_order`? Is it `service_order['caseType']` verbatim, or does it map to a normalized set?  
   **Impact:** Not directly relevant to this story (no extraction logic changed), but test 1 assumes `repository.save_case` was called and wrote a row. If `case_type` extraction is broken, the row may be incomplete (see Story 2 and Story 4 notes).

3. **Wait time calculation (HLD OQ4):** Story 2 uses `last_changed_at` as a proxy for wait time. If a separate `waiting_since` field is required, the schema (Story 1) will need amendment.  
   **Impact:** Not directly relevant to this story.

---

## Dependencies

- **External:** `aiosqlite>=0.20.0` (added to `pyproject.toml` in Story 1).
- **Internal:**
  - `src/agents/persistence/repository.py` — `load_case` function (Story 2).
  - `src/agents/runtime/runner.py` — `run_query` function (existing; unchanged in this story).
  - `src/agents/runtime/checkpointer.py` — `get_checkpointer` and `close_checkpointer` (existing).
  - `src/agents/graph/build.py` — `build_graph` (existing).
  - `src/agents/runtime/run_dir.py` — `LOG_DIR` for resolving `data/` folder (existing).
  - `src/agents/graph/states.py` — `State` TypedDict structure (existing).
  - `types.SimpleNamespace` (stdlib) — for synthetic snapshot construction.
  - `pathlib.Path` (stdlib) — for database path resolution.

---

## Notes

- **No new routes:** This story verifies existing endpoints (`POST /runs`, `GET /runs/{thread_id}`, `POST /runs/{thread_id}/resume`) and adds fallback logic to `_snapshot` and re-initialization logic to `resume_run`.
- **Checkpointer as source of truth:** For active runs, the checkpointer is authoritative. SQLite is only consulted when the checkpointer is empty (post-restart).
- **Fallback snapshot shape:** The synthetic snapshot from `repository.load_case` has `values=<state>`, `next=()`, `interrupts=[]`, `checkpointer_was_empty=True`. This matches the shape `get_run` expects, but loses the "pending nodes" metadata (what the run was paused on). Acceptable because the run was paused *before* the restart; after restart, it must be resumed explicitly (the reviewer or analyst provides the decision again).
- **Test isolation:** Tests use a temporary database (`tmp_path` fixture) to avoid polluting the project-wide `data/cases.db`. Test 4 and 5 simulate a restart by deleting the checkpointer file and restarting the server (or, in a unit test, closing and re-opening `open_graph`).
- **Resume re-initialization:** Open Question 1 is resolved by the implementation: `resume_run` checks `snapshot.checkpointer_was_empty` and writes the fallback state back to the checkpointer before calling `_start`. This ensures `run_query(..., resume=True)` sees the state it expects.
- **Database path consistency:** `_db_path()` uses the same logic as Story 3 (project-wide `data/cases.db`). If the database were per-run (in each run's `logs/` folder), the fallback would need to resolve the run's folder via `run_dir(thread_id)` instead.
- **Error handling:** If `repository.load_case` fails (e.g. database locked, corrupted snapshot JSON), the fallback returns `None` and the existing 404 logic in `get_run` applies (raises `HTTPException(404, f"Unknown run {thread_id}")`). No special error message for "SQLite fallback failed" — it's treated the same as "run not found".
- **State snapshot JSON:** The `state_snapshot` column in SQLite holds the full `State` dict as JSON. `repository.load_case` deserializes it with `json.loads(row[0])` (Story 2 implementation). The returned dict matches the `State` TypedDict structure (`graph/states.py`).
- **Import placement:** `from agents.persistence import repository` and `from types import SimpleNamespace` are added at the top of `server.py` (existing imports section). `from pathlib import Path` already exists in the file.

---

## Implementation Sketch

### Modified `_snapshot` in `src/agents/runtime/server.py`

```python
async def _snapshot(thread_id: str):
    """The run's latest state, read from its own checkpointer.
    
    Fallback: if the checkpointer is empty (e.g. after a restart),
    load the state from cases.state_snapshot via repository.load_case.
    """
    from types import SimpleNamespace
    from agents.persistence import repository
    from agents.runtime.run_dir import run_dir
    
    # Use the same checkpointer kind the server is configured with
    checkpointer_kind = CHECKPOINTER_KIND
    db_path = run_dir(thread_id) / "checkpoint.db" if checkpointer_kind == "sqlite" else None
    
    async with open_graph(thread_id, checkpointer_kind=checkpointer_kind, db_path=db_path) as graph:
        snapshot = await graph.aget_state({"configurable": {"thread_id": thread_id}})
    
    # Active run case: checkpointer has state
    if snapshot.values:
        return snapshot
    
    # Fallback case: checkpointer is empty (post-restart)
    cases_db_path = _db_path()
    state = await repository.load_case(thread_id, cases_db_path)
    if state is None:
        # Run not found in either checkpointer or SQLite
        return snapshot  # empty snapshot; get_run will raise 404
    
    # Synthetic snapshot from SQLite
    return SimpleNamespace(
        values=state,
        next=(),  # no pending nodes (run finished or paused before restart)
        interrupts=[],
        checkpointer_was_empty=True,  # flag for resume_run
    )
```

### New `_db_path` helper in `src/agents/runtime/server.py`

```python
def _db_path() -> Path:
    """Resolve the database path for case persistence.
    
    Same logic as Story 3: project-wide data/cases.db.
    """
    from agents.runtime.run_dir import LOG_DIR
    db_dir = LOG_DIR.parent / "data"
    db_dir.mkdir(parents=True, exist_ok=True)
    return db_dir / "cases.db"
```

### Modified `resume_run` in `src/agents/runtime/server.py`

```python
@app.post('/runs/{thread_id}/resume', status_code=202)
async def resume_run(thread_id: str, decision: FlagAnswers | ReviewDecision) -> dict[str, Any]:
    if _runs.get(thread_id, {}).get("status") == "running":
        raise HTTPException(409, f"Run {thread_id} is running")
    # the checkpointer, not _runs, says whether it is paused — it outlives a restart
    snapshot = await _snapshot(thread_id)
    if not snapshot.next:
        raise HTTPException(409, f"Run {thread_id} is not paused")
    on_flags = FLAG_REVIEW in snapshot.next
    if on_flags != isinstance(decision, FlagAnswers):
        expected = "the answers to its flags" if on_flags else "approve or reject"
        raise HTTPException(422, f"Run {thread_id} is paused at {list(snapshot.next)} — send {expected}")
    body = decision.model_dump(exclude_none=True)
    if on_flags and (missing := missing_answers(snapshot.values or {}, body)):
        raise HTTPException(422, f"Answer every flag before the case continues — no answer yet for {missing}")
    
    # Re-initialize checkpointer from SQLite if it was empty
    if getattr(snapshot, "checkpointer_was_empty", False):
        async with open_graph(thread_id) as graph:
            await graph.update_state(
                {"configurable": {"thread_id": thread_id}},
                snapshot.values,
            )
    
    return _start(thread_id, None, resume=True, decision=body)
```

### Imports to add at top of `src/agents/runtime/server.py`

```python
from types import SimpleNamespace
# (Path already imported from pathlib)

# Module-level configuration for the checkpointer kind
CHECKPOINTER_KIND = os.getenv("CHECKPOINTER", "sqlite")
```

## Final Test Count

Five tests as specified:

1. `POST /runs` starts and progresses.
2. `GET /runs/{thread_id}` during the run returns checkpointer values.
3. `POST /runs/{thread_id}/resume` continues the run.
4. Restart fallback: `GET /runs/{thread_id}` reads from SQLite.
5. Resume after restart.

**Total:** 5 tests.
