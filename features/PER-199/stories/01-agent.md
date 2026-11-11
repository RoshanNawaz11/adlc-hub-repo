# Story 01-agent: Add reviewer notes to paused cases (LLD)

**Repo:** `agent` (Python, LangGraph)  
**Date:** 2025-01-24

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

1. **`src/agents/graph/states.py`**
   - Add `Note` TypedDict (4 fields: `note_id`, `text`, `author`, `created_at`) after `NodeTiming`, before `merge_flags`.
   - Add `notes: NotRequired[Annotated[list[Note], operator.add]]` to `State` (after `timings`, before `worker_results`).
   - No `schema_version` increment here (it's in `runner.py`).

2. **`src/agents/runtime/runner.py`**
   - In `_initial_state` function (line ~116): change `"schema_version": 1` to `"schema_version": 2`.
   - Add `"notes": []` to the initial state dict (after `"schema_version": 2`, before `"run_id": run_id`).

3. **`src/agents/runtime/server.py`**
   - Add `CreateNoteRequest` Pydantic model (after `FlagAnswers`, before `_lifespan`).
   - Add `POST /runs/{thread_id}/notes` endpoint (after `resume_run`, before `get_run`).
   - Imports (new):
     - `from uuid import uuid4` (for `note_id`)
     - `from datetime import datetime, UTC` (for `created_at`)
     - `from pydantic import Field` (for `min_length`, `max_length` constraints)
     - `from agents.graph.states import Note` (the TypedDict)

### Test File

- **`tests/test_reviewer_notes.py`** (new)
  - Covers all 15 tests listed in the Tests section below.

## Interfaces

### `Note` TypedDict (new, in `src/agents/graph/states.py`)

```python
class Note(TypedDict):
    note_id: str            # UUID
    text: str               # 1-2000 chars
    author: str
    created_at: str         # ISO-8601 UTC
```

### `State` extension (in `src/agents/graph/states.py`)

Add one field to the existing `State` TypedDict:

```python
notes: NotRequired[Annotated[list[Note], operator.add]]
```

- **Schema version:** `schema_version` incremented to `2` in new runs (in `runner.py::_initial_state`); existing runs stay at `1` and show no `notes` key.
- **Reducer:** `operator.add` appends new notes to the existing list (same pattern as `steps`, `flags`).
- **Graceful degradation:** UI treats missing `notes` as `[]`; no migration script.

### `CreateNoteRequest` Pydantic model (new, in `src/agents/runtime/server.py`)

```python
class CreateNoteRequest(BaseModel):
    text: str = Field(min_length=1, max_length=2000)
    author: str
```

FastAPI validates automatically; over-limit or empty text raises 422.

### POST `/runs/{thread_id}/notes` (new endpoint, in `src/agents/runtime/server.py`)

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

1. Load the run's latest state via `_snapshot(thread_id)` (existing helper in `server.py`).
2. Validate `status in ["needs_input", "needs_review"]`; reject with 400 otherwise.
3. Create a `Note` dict: `note_id` = new UUID (via `uuid4()`), `text` and `author` from request, `created_at` = ISO-8601 UTC now (via `datetime.now(UTC).isoformat()`).
4. Update the graph state: `async with open_graph(thread_id) as graph: await graph.aupdate({"notes": [new_note]}, config={"configurable": {"thread_id": thread_id}})` (the `operator.add` reducer appends it).
5. Return 201 + the created note.

### GET `/runs/{thread_id}` (existing, extended)

The response now includes `notes` when present (existing `_publish` helper already carries `values`):

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

No new code needed; `notes` appears in `State.values` once the field is added to `State`, and `_publish` already returns the full `values` dict.

---

## Algorithm Notes

### Schema Version Increment

- **Where:** `src/agents/runtime/runner.py::_initial_state` (line ~94).
- **What:** Change `"schema_version": 1` to `"schema_version": 2`, and add `"notes": []` to the initial state dict.
- **Why:** New runs write `schema_version: 2` and an empty `notes` list; old runs (loaded from SQLite) show `schema_version: 1` and no `notes` key. The UI treats missing `notes` as `[]` (graceful degradation).

### Note Creation

- **Where:** `src/agents/runtime/server.py::add_note` (new endpoint).
- **What:**
  1. Call `_snapshot(thread_id)` to load the run's latest state.
  2. Check `snapshot.values["status"] in ["needs_input", "needs_review"]`; if not, raise `HTTPException(400, "Cannot add notes to a <status> case")`.
  3. Create `note: Note = {"note_id": str(uuid4()), "text": req.text, "author": req.author, "created_at": datetime.now(UTC).isoformat()}`.
  4. Call `async with open_graph(thread_id) as graph: await graph.aupdate({"notes": [note]}, config={"configurable": {"thread_id": thread_id}})`.
  5. Return `201, note`.
- **Why:** The `operator.add` reducer (in `State.notes`) appends `[note]` to the existing `notes` list. LangGraph's checkpointer persists the updated state to SQLite, so the note survives service restarts.

### Display

- **Where:** `src/agents/runtime/server.py::get_run` (existing).
- **What:** No code change. Once `notes` is added to `State`, `snapshot.values` (returned by `_publish`) includes `notes` if present.
- **Why:** The existing `_publish` helper already returns `{**run, "values": snapshot.values}` or `{"values": snapshot.values, ...}`, so `notes` appears alongside `flags`, `summary`, `steps`.

### Validation

- **Where:** `src/agents/runtime/server.py::CreateNoteRequest` (new Pydantic model).
- **What:** `text: str = Field(min_length=1, max_length=2000)` and `author: str`.
- **Why:** FastAPI's automatic validation raises 422 (with a descriptive error message) if `text` is empty or over 2000 chars, satisfying AC6.

### Immutability

- **Where:** No delete/edit endpoints are created.
- **What:** Notes are append-only (via `operator.add` reducer); no endpoint allows editing or deleting a note.
- **Why:** Satisfies AC2 (notes are immutable once saved).

---

## Tests

### Test 1: New runs write schema_version=2 and an empty notes list

**Setup:** Start a new run (no service order, or a fresh `thread_id`).  
**Action:** Call `run_query` (or `POST /runs`), then load the state via `_snapshot`.  
**Assert:** `snapshot.values["schema_version"] == 2` and `snapshot.values["notes"] == []`.

**Code:**

```python
@pytest.mark.asyncio
async def test_new_runs_write_schema_version_2_and_empty_notes(monkeypatch: pytest.MonkeyPatch) -> None:
    from agents.runtime.runner import _initial_state

    state = _initial_state(
        run_id="test-run",
        query="",
        service_order_id="SO-1003",
        service_order={"id": "SO-1003", "caseType": "Offboarding"},
    )
    assert state["schema_version"] == 2
    assert state["notes"] == []
```

---

### Test 2: Old runs (v1) loaded from checkpointer show no notes key

**Setup:** Mock `AsyncSqliteSaver` to return a state with `schema_version=1` and no `notes` field.  
**Action:** Call `_snapshot(thread_id)`.  
**Assert:** `"notes" not in snapshot.values`.

**Code:**

```python
@pytest.mark.asyncio
async def test_old_runs_show_no_notes_key(monkeypatch: pytest.MonkeyPatch) -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"schema_version": 1, "run_id": "old-run", "status": "done"}
    mock_snapshot.next = []

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.get("/runs/old-run")
        assert response.status_code == 200
        assert "notes" not in response.json()["values"]
```

---

### Test 3: Note structure matches the TypedDict contract

**Setup:** None.  
**Action:** Create a `Note` dict: `{"note_id": str(uuid4()), "text": "checked", "author": "analyst", "created_at": datetime.now(UTC).isoformat()}`.  
**Assert:**
- Keys are exactly `{"note_id", "text", "author", "created_at"}`.
- `note_id` is a valid UUID string (36 chars, matches UUID pattern).
- `text` is a string, `len(text) <= 2000`.
- `created_at` is an ISO-8601 UTC string (ends with `Z` or `+00:00`).

**Code:**

```python
def test_note_structure_matches_contract() -> None:
    note: Note = {
        "note_id": str(uuid.uuid4()),
        "text": "checked",
        "author": "analyst",
        "created_at": datetime.now(UTC).isoformat(),
    }
    assert set(note.keys()) == {"note_id", "text", "author", "created_at"}
    # UUID pattern: 8-4-4-4-12 hex digits
    assert len(note["note_id"]) == 36
    assert note["note_id"].count("-") == 4
    assert len(note["text"]) <= 2000
    assert note["created_at"].endswith("Z") or "+00:00" in note["created_at"]
```

---

### Test 4: Notes reducer appends new notes to the existing list

**Setup:** Mock the `State` annotated reducer for `notes` (i.e., call `operator.add`).  
**Action:** Call the reducer with `current = [{"note_id": "n1", ...}]` and `new = [{"note_id": "n2", ...}]`.  
**Assert:** Result is `[{"note_id": "n1", ...}, {"note_id": "n2", ...}]` (insertion order preserved).

**Code:**

```python
def test_notes_reducer_appends() -> None:
    current = [{"note_id": "n1", "text": "first", "author": "analyst1", "created_at": "2025-01-24T10:00:00Z"}]
    new = [{"note_id": "n2", "text": "second", "author": "analyst2", "created_at": "2025-01-24T11:00:00Z"}]
    # The State field is `notes: NotRequired[Annotated[list[Note], operator.add]]`
    result = operator.add(current, new)
    assert result == [
        {"note_id": "n1", "text": "first", "author": "analyst1", "created_at": "2025-01-24T10:00:00Z"},
        {"note_id": "n2", "text": "second", "author": "analyst2", "created_at": "2025-01-24T11:00:00Z"},
    ]
```

---

### Test 5: POST with valid text and author on a paused case returns 201 + the note

**Setup:** Mock `_snapshot` to return a state with `status="needs_input"` and `notes=[]`.  
**Action:** `POST /runs/{thread_id}/notes {"text": "checked", "author": "analyst@example.com"}`.  
**Assert:**
- Response status is 201.
- Response body has keys `{"note_id", "text", "author", "created_at"}`.
- `note_id` is a valid UUID string.
- `text == "checked"`, `author == "analyst@example.com"`.
- `created_at` is an ISO-8601 UTC string.

**Code:**

```python
def test_post_valid_note_on_paused_case_returns_201(monkeypatch: pytest.MonkeyPatch) -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"status": "needs_input", "notes": []}

    async def mock_aupdate(self: Any, payload: dict[str, Any], config: dict[str, Any]) -> None:
        pass

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        with patch("agents.runtime.server.open_graph") as mock_graph:
            mock_graph_instance = AsyncMock()
            mock_graph_instance.aupdate = mock_aupdate
            mock_graph.return_value.__aenter__.return_value = mock_graph_instance

            client = TestClient(app)
            response = client.post("/runs/test-thread/notes", json={"text": "checked", "author": "analyst@example.com"})
            assert response.status_code == 201
            body = response.json()
            assert set(body.keys()) == {"note_id", "text", "author", "created_at"}
            assert len(body["note_id"]) == 36
            assert body["text"] == "checked"
            assert body["author"] == "analyst@example.com"
            assert body["created_at"].endswith("Z") or "+00:00" in body["created_at"]
```

---

### Test 6: POST with empty text returns 422

**Setup:** Mock `_snapshot` to return a paused state.  
**Action:** `POST /runs/{thread_id}/notes {"text": "", "author": "analyst"}`.  
**Assert:**
- Response status is 422.
- Response body `detail` includes an error mentioning `"text"` and `"min_length"` or `"ensure this value has at least 1 characters"`.

**Code:**

```python
def test_post_empty_text_returns_422() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"status": "needs_input"}

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.post("/runs/test-thread/notes", json={"text": "", "author": "analyst"})
        assert response.status_code == 422
        assert "text" in str(response.json()["detail"]).lower()
```

---

### Test 7: POST with text over 2000 chars returns 422

**Setup:** Mock `_snapshot` to return a paused state.  
**Action:** `POST /runs/{thread_id}/notes {"text": "x" * 2001, "author": "analyst"}`.  
**Assert:**
- Response status is 422.
- Response body `detail` includes an error mentioning `"text"` and `"max_length"` or `"2000 characters"`.

**Code:**

```python
def test_post_over_limit_text_returns_422() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"status": "needs_input"}

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.post("/runs/test-thread/notes", json={"text": "x" * 2001, "author": "analyst"})
        assert response.status_code == 422
        detail_str = str(response.json()["detail"]).lower()
        assert "text" in detail_str and ("2000" in detail_str or "max" in detail_str)
```

---

### Test 8: POST to a running case returns 400 with "Cannot add notes to a running case"

**Setup:** Mock `_snapshot` to return a state with `status="running"`.  
**Action:** `POST /runs/{thread_id}/notes {"text": "checked", "author": "analyst"}`.  
**Assert:**
- Response status is 400.
- Response body `error == "Cannot add notes to a running case"`.

**Code:**

```python
def test_post_to_running_case_returns_400() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"status": "running"}

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.post("/runs/test-thread/notes", json={"text": "checked", "author": "analyst"})
        assert response.status_code == 400
        assert response.json()["error"] == "Cannot add notes to a running case"
```

---

### Test 9: POST to a finished case returns 400 with "Cannot add notes to a finished case"

**Setup:** Mock `_snapshot` to return a state with `status="done"`.  
**Action:** `POST /runs/{thread_id}/notes {"text": "checked", "author": "analyst"}`.  
**Assert:**
- Response status is 400.
- Response body `error == "Cannot add notes to a finished case"`.

**Code:**

```python
def test_post_to_finished_case_returns_400() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"status": "done"}

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.post("/runs/test-thread/notes", json={"text": "checked", "author": "analyst"})
        assert response.status_code == 400
        assert response.json()["error"] == "Cannot add notes to a finished case"
```

---

### Test 10: The created note is appended to the state's notes list

**Setup:** Mock `_snapshot` to return a paused state with `notes=[{"note_id": "n1", "text": "first", "author": "analyst1", "created_at": "2025-01-24T12:00:00Z"}]`.  
**Action:** `POST /runs/{thread_id}/notes {"text": "second", "author": "analyst2"}`.  
**Assert:**
- Load the state after the POST (`_snapshot`).
- `notes` has 2 elements: the old note (`n1`) and the new one.
- The new note's `text == "second"`, `author == "analyst2"`.

**Code:**

```python
@pytest.mark.asyncio
async def test_created_note_is_appended(monkeypatch: pytest.MonkeyPatch) -> None:
    existing_notes = [{"note_id": "n1", "text": "first", "author": "analyst1", "created_at": "2025-01-24T12:00:00Z"}]
    mock_snapshot_before = MagicMock()
    mock_snapshot_before.values = {"status": "needs_input", "notes": existing_notes}

    new_note_captured = None

    async def mock_aupdate(self: Any, payload: dict[str, Any], config: dict[str, Any]) -> None:
        nonlocal new_note_captured
        new_note_captured = payload["notes"][0]

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot_before):
        with patch("agents.runtime.server.open_graph") as mock_graph:
            mock_graph_instance = AsyncMock()
            mock_graph_instance.aupdate = mock_aupdate
            mock_graph.return_value.__aenter__.return_value = mock_graph_instance

            client = TestClient(app)
            response = client.post("/runs/test-thread/notes", json={"text": "second", "author": "analyst2"})
            assert response.status_code == 201
            assert new_note_captured is not None
            assert new_note_captured["text"] == "second"
            assert new_note_captured["author"] == "analyst2"
```

---

### Test 11: GET on a run with notes returns them in the state dict

**Setup:** Mock `_snapshot` to return a state with `notes=[{"note_id": "a1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}]`.  
**Action:** `GET /runs/{thread_id}`.  
**Assert:**
- Response status is 200.
- Response body `values["notes"] == [{"note_id": "a1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}]`.

**Code:**

```python
def test_get_run_with_notes_returns_them() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {
        "status": "needs_input",
        "notes": [{"note_id": "a1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}],
    }
    mock_snapshot.next = []

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.get("/runs/test-thread")
        assert response.status_code == 200
        assert response.json()["values"]["notes"] == [
            {"note_id": "a1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}
        ]
```

---

### Test 12: GET on a run with no notes (schema v1) returns values without a notes key

**Setup:** Mock `_snapshot` to return a state with `schema_version=1` and no `notes` field.  
**Action:** `GET /runs/{thread_id}`.  
**Assert:**
- Response status is 200.
- `"notes" not in response["values"]`.

**Code:**

```python
def test_get_run_schema_v1_no_notes_key() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {"schema_version": 1, "status": "done"}
    mock_snapshot.next = []

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.get("/runs/test-thread")
        assert response.status_code == 200
        assert "notes" not in response.json()["values"]
```

---

### Test 13: Notes appear in insertion order (oldest first)

**Setup:** Mock `_snapshot` to return a state with `notes=[{"note_id": "n1", "created_at": "2025-01-24T10:00:00Z", ...}, {"note_id": "n2", "created_at": "2025-01-24T11:00:00Z", ...}, {"note_id": "n3", "created_at": "2025-01-24T12:00:00Z", ...}]`.  
**Action:** `GET /runs/{thread_id}`.  
**Assert:**
- Response `values["notes"]` is a list of 3 elements.
- The list order is `[n1, n2, n3]` (i.e., `created_at` ascending, matching insertion order).

**Code:**

```python
def test_notes_appear_in_insertion_order() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {
        "notes": [
            {"note_id": "n1", "created_at": "2025-01-24T10:00:00Z", "text": "first", "author": "a1"},
            {"note_id": "n2", "created_at": "2025-01-24T11:00:00Z", "text": "second", "author": "a2"},
            {"note_id": "n3", "created_at": "2025-01-24T12:00:00Z", "text": "third", "author": "a3"},
        ]
    }
    mock_snapshot.next = []

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.get("/runs/test-thread")
        assert response.status_code == 200
        notes = response.json()["values"]["notes"]
        assert len(notes) == 3
        assert [n["note_id"] for n in notes] == ["n1", "n2", "n3"]
```

---

### Test 14: A finished case still shows its notes

**Setup:** Mock `_snapshot` to return a state with `status="done"` and `notes=[{"note_id": "n1", ...}]`.  
**Action:** `GET /runs/{thread_id}`.  
**Assert:**
- Response status is 200.
- Response `values["notes"] == [{"note_id": "n1", ...}]`.

**Code:**

```python
def test_finished_case_shows_notes() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {
        "status": "done",
        "notes": [{"note_id": "n1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}],
    }
    mock_snapshot.next = []

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.get("/runs/test-thread")
        assert response.status_code == 200
        assert response.json()["values"]["notes"] == [
            {"note_id": "n1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}
        ]
```

---

### Test 15: Notes are shown alongside flags and summary (same state structure)

**Setup:** Mock `_snapshot` to return a state with `flags=[{"flag_id": "f1", ...}]`, `summary="paused on mismatch"`, `notes=[{"note_id": "n1", ...}]`.  
**Action:** `GET /runs/{thread_id}`.  
**Assert:**
- Response status is 200.
- Response `values` includes all three keys: `"flags"`, `"summary"`, `"notes"`.
- `values["flags"] == [{"flag_id": "f1", ...}]`.
- `values["summary"] == "paused on mismatch"`.
- `values["notes"] == [{"note_id": "n1", ...}]`.

**Code:**

```python
def test_notes_shown_alongside_flags_and_summary() -> None:
    mock_snapshot = MagicMock()
    mock_snapshot.values = {
        "flags": [{"flag_id": "f1", "type": "duplicateSO", "title": "Duplicate SO"}],
        "summary": "paused on mismatch",
        "notes": [{"note_id": "n1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}],
    }
    mock_snapshot.next = []

    with patch("agents.runtime.server._snapshot", return_value=mock_snapshot):
        client = TestClient(app)
        response = client.get("/runs/test-thread")
        assert response.status_code == 200
        values = response.json()["values"]
        assert "flags" in values
        assert "summary" in values
        assert "notes" in values
        assert values["flags"] == [{"flag_id": "f1", "type": "duplicateSO", "title": "Duplicate SO"}]
        assert values["summary"] == "paused on mismatch"
        assert values["notes"] == [
            {"note_id": "n1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}
        ]
```

---

**End of LLD.**
