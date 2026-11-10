# Story 01-agent: Add reviewer notes to paused cases

**Repo:** `agent`

**Covers:** AC1, AC2, AC3, AC4, AC5, AC6, AC7

---

## Acceptance Criteria

### AC1: Add note while paused

**Given** a case is paused (for flags or for final review)  
**When** a reviewer adds a note  
**Then** the note is saved with the case, recording the note text (up to 2,000 characters), the reviewer who wrote it, and the timestamp when it was written

**Clarification:** "Paused" means `status: "needs_input"` or `status: "needs_review"` per `src/agents/graph/states.py`.

---

### AC2: Notes are immutable

**Given** a note has been added to a case  
**When** the note is saved  
**Then** it cannot be edited or deleted afterwards  
**And** a correction is made by adding a new note

---

### AC3: Display all notes

**Given** a case has one or more notes  
**When** anyone looks at the case  
**Then** all notes are displayed, oldest first, in the same place the case's flags and summary are shown

**Clarification:** "The same place" refers to the case's state structure returned by the graph and displayed in the review UI.

---

### AC4: Notes persist

**Given** a case has notes  
**When** the case is resumed, paused again, the service restarts, or the case finishes  
**Then** the notes are still present and displayed

---

### AC5: Notes only on paused cases

**Given** a case is in a state other than paused (running, done, rejected, failed)  
**When** a reviewer attempts to add a note  
**Then** the attempt is refused with a clear message saying why (e.g., "Cannot add notes to a running case" or "Cannot add notes to a finished case")

**Clarification:** Allowed states are `needs_input` and `needs_review`; disallowed states are `running`, `done`, `rejected`, `failed` per `src/agents/graph/states.py`.

---

### AC6: Validation of note content

**Given** a reviewer is adding a note  
**When** the note is empty or exceeds 2,000 characters  
**Then** the note is refused with a message saying what's wrong (e.g., "Note cannot be empty" or "Note exceeds 2,000 character limit")

---

### AC7: No disruption to existing functionality

**Given** the case review workflow today  
**When** notes are added to the system  
**Then** everything a case shows today (flags, summary, steps, results, pipeline) and how it's resumed stays the same  
**And** notes are added alongside, not in place of anything

---

## File Plan

### Modified Files

#### `src/agents/graph/states.py`

**Purpose:** Add the `Note` TypedDict and extend `State` with a `notes` field; increment `schema_version` to `2`.

**Changes:**

1. **Add `Note` TypedDict** (after `NodeTiming`, before `merge_flags`):
   ```python
   class Note(TypedDict):
       note_id: str            # UUID
       text: str               # 1-2000 chars
       author: str
       created_at: str         # ISO-8601 UTC
   ```

2. **Extend `State` TypedDict** (add field after `timings`, before `worker_results`):
   ```python
   notes: NotRequired[Annotated[list[Note], operator.add]]
   ```
   - The `operator.add` reducer appends new notes to the existing list (same pattern as `steps`, `flags`).
   - `NotRequired` because schema v1 runs have no `notes` key.

3. **Increment `schema_version`** default to `2` in `src/agents/runtime/runner.py::_initial_state` (verify location: currently `"schema_version": 1` on line 94).

**Interfaces:**

- **Exports:** `Note` (new TypedDict), `State.notes` (new field, `NotRequired[Annotated[list[Note], operator.add]]`).
- **Imports:** No new imports (uses existing `operator`, `Annotated`, `NotRequired`, `TypedDict`).

---

#### `src/agents/runtime/server.py`

**Purpose:** Add `CreateNoteRequest` Pydantic model and `POST /runs/{thread_id}/notes` endpoint.

**Changes:**

1. **Add `CreateNoteRequest` Pydantic model** (after `FlagAnswers`, before `_lifespan`):
   ```python
   class CreateNoteRequest(BaseModel):
       text: str = Field(min_length=1, max_length=2000)
       author: str
   ```
   - Pydantic validates `text` length automatically; over-limit or empty text raises 422.

2. **Add `POST /runs/{thread_id}/notes` endpoint** (after `resume_run`, before `get_run`):
   ```python
   @app.post("/runs/{thread_id}/notes", status_code=201)
   async def add_note(thread_id: str, req: CreateNoteRequest) -> dict[str, Any]:
   ```

**Interfaces:**

- **Imports (new):**
  - `from uuid import uuid4` (for `note_id`)
  - `from datetime import datetime, UTC` (for `created_at`)
  - `from pydantic import Field` (for `min_length`, `max_length` constraints)
  - `from agents.graph.states import Note` (the TypedDict)

- **Exports:** `CreateNoteRequest` (new Pydantic model), `add_note` (new HTTP endpoint).

- **Calls:**
  - `_snapshot(thread_id)` (existing helper, loads the state from the checkpointer).
  - `open_graph(thread_id)` (existing context manager, yields the graph with the run's checkpointer).
  - `graph.aupdate({"notes": [new_note]}, config={"configurable": {"thread_id": thread_id}})` (LangGraph's update method; the `operator.add` reducer appends the note to `State.notes`).

---

#### `src/agents/runtime/runner.py`

**Purpose:** Increment `schema_version` to `2` in `_initial_state` for new runs.

**Changes:**

1. **In `_initial_state` function** (line 94):
   - Change `"schema_version": 1` to `"schema_version": 2`.
   - Add `"notes": []` to the initial state dict (after `"schema_version": 2`, before `"run_id": run_id`).

**Interfaces:**

- **No new imports or exports** (internal change to `_initial_state` return value).
- **Contract:** New runs write `schema_version: 2` and an empty `notes: []` list; old runs loaded from the checkpointer show `schema_version: 1` and no `notes` key.

---

### No New Files

All changes are extensions to existing files.

---

## Interfaces

### `Note` TypedDict (new, exported from `src/agents/graph/states.py`)

```python
class Note(TypedDict):
    note_id: str            # UUID (string)
    text: str               # 1-2000 chars
    author: str             # reviewer identifier (email or name)
    created_at: str         # ISO-8601 UTC timestamp
```

**Usage:** Created by the `POST /runs/{thread_id}/notes` endpoint; stored in `State.notes` as an append-only list.

---

### `State.notes` (new field in `State` TypedDict)

```python
notes: NotRequired[Annotated[list[Note], operator.add]]
```

**Behavior:**

- **Reducer:** `operator.add` appends new notes to the existing list (insertion order preserved, oldest first).
- **Schema version 2:** New runs initialize with `notes: []`.
- **Schema version 1:** Old runs have no `notes` key; the UI treats missing `notes` as `[]`.

---

### `POST /runs/{thread_id}/notes` (new endpoint in `src/agents/runtime/server.py`)

**Request:**

```json
{
  "text": "Checked with client: bonus paid separately, mismatch expected.",
  "author": "analyst@example.com"
}
```

**Success (201):**

```json
{
  "note_id": "a1b2c3d4-...",
  "text": "Checked with client: bonus paid separately, mismatch expected.",
  "author": "analyst@example.com",
  "created_at": "2025-01-24T12:34:56.789Z"
}
```

**Validation error (422):**

```json
{
  "detail": [
    {
      "loc": ["body", "text"],
      "msg": "ensure this value has at most 2000 characters",
      "type": "value_error.any_str.max_length"
    }
  ]
}
```

**State error (400):**

```json
{
  "error": "Cannot add notes to a running case"
}
```

**Logic:**

1. Load the run's latest state via `_snapshot(thread_id)` (existing helper; reads from the SQLite checkpointer).
2. Validate `status in ["needs_input", "needs_review"]`; reject with 400 + descriptive message otherwise:
   - `status == "running"` → `"Cannot add notes to a running case"`
   - `status == "done"` → `"Cannot add notes to a finished case"`
   - `status == "rejected"` → `"Cannot add notes to a finished case"`
   - `status == "failed"` → `"Cannot add notes to a finished case"`
3. Create a `Note` dict:
   - `note_id`: `str(uuid4())`
   - `text`, `author`: from request body
   - `created_at`: `datetime.now(UTC).isoformat()`
4. Update the graph state:
   ```python
   async with open_graph(thread_id) as graph:
       await graph.aupdate(
           {"notes": [new_note]},
           config={"configurable": {"thread_id": thread_id}}
       )
   ```
   - The `operator.add` reducer in `State.notes` appends `new_note` to the existing list.
5. Return 201 + the created note.

**Error handling:**

- **Pydantic validation (automatic):** Empty `text` or `text` over 2000 chars raises 422 with field-specific error message (no manual check needed).
- **State validation (manual):** Check `status` before calling `graph.aupdate`; raise `HTTPException(400, ...)` on invalid state.
- **Thread not found:** `_snapshot` returns a snapshot with `snapshot.values is None` if the thread does not exist; raise `HTTPException(404, f"Unknown run {thread_id}")`.

---

### `GET /runs/{thread_id}` (existing, extended)

**No new code:** The `_publish` helper already includes `values` in the response, and `values` is the full `State` dict once `notes` is added to `State`. The UI reads `notes` from `response["values"]["notes"]` (when present).

**Response (extended):**

```json
{
  "thread_id": "thread-123",
  "status": "needs_input",
  "notes": [
    {
      "note_id": "a1b2...",
      "text": "Checked with client: bonus paid separately.",
      "author": "analyst@example.com",
      "created_at": "2025-01-24T12:34:56.789Z"
    }
  ],
  "flags": [...],
  "summary": "..."
}
```

**Schema v1 runs:** `"notes"` key absent (the UI treats it as `[]`).

---

## Algorithm Notes

### State Validation Logic (in `POST /runs/{thread_id}/notes`)

```python
snapshot = await _snapshot(thread_id)
if not snapshot.values:
    raise HTTPException(404, f"Unknown run {thread_id}")

status = snapshot.values.get("status")
allowed_statuses = {"needs_input", "needs_review"}

if status not in allowed_statuses:
    if status == "running":
        msg = "Cannot add notes to a running case"
    else:  # "done", "rejected", "failed"
        msg = "Cannot add notes to a finished case"
    raise HTTPException(400, msg)
```

**Rationale:** The `status` field is the single source of truth for whether a case is paused (and thus eligible for notes). The error message names the state to make the rejection clear.

---

### Note Creation (in `POST /runs/{thread_id}/notes`)

```python
from datetime import UTC, datetime
from uuid import uuid4

new_note: Note = {
    "note_id": str(uuid4()),
    "text": req.text,  # already validated by Pydantic (1-2000 chars)
    "author": req.author,
    "created_at": datetime.now(UTC).isoformat(),
}
```

**Rationale:** The `note_id` is a UUID to guarantee uniqueness across runs and restarts. The `created_at` timestamp is ISO-8601 UTC (same format as `StepRecord.at`), so the UI can sort notes oldest-first without timezone conversion.

---

### Graph Update (in `POST /runs/{thread_id}/notes`)

```python
async with open_graph(thread_id) as graph:
    await graph.aupdate(
        {"notes": [new_note]},
        config={"configurable": {"thread_id": thread_id}}
    )
```

**How it works:**

1. `graph.aupdate` calls the `State.notes` reducer (`operator.add`).
2. The reducer appends `[new_note]` to the existing `notes` list (same as how `steps` and `flags` are merged).
3. The checkpointer writes the updated state to `<run_dir>/checkpoint.db`, so the note persists across service restarts.

**No race condition:** LangGraph serializes writes to the same `thread_id` (the checkpointer's `aput` method is atomic per thread). If two notes are added concurrently, both land in the list in the order the checkpointer processed them.

---

### Schema Version Handling

**New runs (schema v2):**

- `_initial_state` in `src/agents/runtime/runner.py` returns `{"schema_version": 2, "notes": [], ...}`.
- The checkpointer writes this to a new SQLite file.

**Old runs (schema v1):**

- Loaded from an existing SQLite checkpoint with `{"schema_version": 1, ...}` and no `notes` key.
- The `NotRequired[...]` annotation means Python does not raise `KeyError` when `notes` is absent.
- The `GET /runs/{thread_id}` response carries `values` as-is; the UI checks `if ("notes" in values)` or treats `values.get("notes", [])` as an empty list.

**No migration script:** The schema bump is a cold-start strategy. Old runs show no `notes` field; new runs write one. The UI handles both gracefully.

---

### Display Order (oldest first)

The `operator.add` reducer preserves insertion order: the first note appended is at index 0, the second at index 1, etc. The UI renders `notes` in array order, so oldest notes appear first (no sort needed).

---

## Tests

The breakdown lists 15 total tests (4 + 6 + 5) across the three stories. This story (01-agent) implements all three, so it covers all 15 tests.

### Story 1 Tests (schema, 4 total)

1. **Test: New runs write schema_version=2 and an empty notes list**  
   Start a new run with no service order; snapshot the state; assert `schema_version == 2` and `notes == []`.

2. **Test: Old runs (v1) loaded from checkpointer show no notes key**  
   Mock a checkpointer returning a state with `schema_version=1` and no `notes` field; load it; assert `"notes" not in state`.

3. **Test: Note structure matches the TypedDict contract**  
   Create a `Note` dict with all fields; assert keys are `note_id`, `text`, `author`, `created_at`; assert `note_id` is a valid UUID string; assert `text` is a string ≤2000 chars; assert `created_at` is an ISO-8601 UTC string.

4. **Test: Notes reducer appends new notes to the existing list**  
   Call the `State` annotated reducer with `current = [note1]` and `new = [note2]`; assert the result is `[note1, note2]` (insertion order preserved).

---

### Story 2 Tests (API endpoint, 6 total)

1. **Test: POST with valid text and author on a paused case returns 201 + the note**  
   Mock a paused state (`status="needs_input"`); POST `{"text": "checked", "author": "analyst@example.com"}`; assert response is 201; assert body has `note_id` (UUID), `text`, `author`, `created_at` (ISO-8601 UTC).

2. **Test: POST with empty text returns 422**  
   POST `{"text": "", "author": "analyst"}` to a paused case; assert response is 422; assert error message mentions "text".

3. **Test: POST with text over 2000 chars returns 422**  
   POST `{"text": "x" * 2001, "author": "analyst"}` to a paused case; assert response is 422; assert error message mentions "2000 characters".

4. **Test: POST to a running case returns 400 with "Cannot add notes to a running case"**  
   Mock a running state (`status="running"`); POST valid note; assert response is 400; assert error message is "Cannot add notes to a running case".

5. **Test: POST to a finished case returns 400 with "Cannot add notes to a finished case"**  
   Mock a finished state (`status="done"`); POST valid note; assert response is 400; assert error message is "Cannot add notes to a finished case".

6. **Test: The created note is appended to the state's notes list**  
   Mock a paused state with `notes=[note1]`; POST valid note; load the state after; assert `notes == [note1, new_note]`.

---

### Story 3 Tests (UI display, 5 total)

1. **Test: GET on a run with notes returns them in the state dict**  
   Mock a state with `notes=[{"note_id": "a1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}]`; GET `/runs/{thread_id}`; assert response `values["notes"]` matches the mocked list.

2. **Test: GET on a run with no notes (schema v1) returns values without a notes key**  
   Mock a state with `schema_version=1` and no `notes` field; GET `/runs/{thread_id}`; assert `"notes" not in response["values"]`.

3. **Test: Notes appear in insertion order (oldest first)**  
   Mock a state with `notes=[note1, note2, note3]` (distinct timestamps); GET `/runs/{thread_id}`; assert `response["values"]["notes"] == [note1, note2, note3]`.

4. **Test: A finished case still shows its notes**  
   Mock a finished state (`status="done"`) with `notes=[note1]`; GET `/runs/{thread_id}`; assert `response["values"]["notes"] == [note1]`.

5. **Test: Notes are shown alongside flags and summary (same state structure)**  
   Mock a state with `flags=[flag1]`, `summary="paused on mismatch"`, `notes=[note1]`; GET `/runs/{thread_id}`; assert all three appear in `response["values"]`.

---

## Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Old runs (schema v1) crash when UI expects `notes` | High | UI treats missing `notes` as `[]`; `NotRequired` annotation in `State` means Python does not raise `KeyError` |
| Concurrent note adds on the same `thread_id` | Low | LangGraph's checkpointer serializes writes; both notes land in order (same as concurrent `steps` or `flags`) |
| Note text over 2000 chars silently truncated | Medium | Pydantic validation rejects at the endpoint; no truncation |
| Notes added while case is running | Medium | Endpoint validates `status` before updating; rejected with clear 400 message |
| State update fails (checkpointer write error) | Medium | FastAPI raises 500; the note is not saved; client retries or surfaces error to user |
| `note_id` collision (two notes share the same UUID) | Very Low | `uuid4()` has a vanishingly small collision probability (2^122 possible values); no collision handling needed |

---

## Open Questions

None.
