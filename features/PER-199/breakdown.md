# Breakdown — PER-199

**Feature:** Reviewers can leave notes on a case  
**Repo:** `agent` (Python, LangGraph)  
**Date:** 2025-01-24

---

## Overview

This feature adds persistent, immutable reviewer notes to paused cases. A reviewer can add a note when a case is paused (for flags or final review), see all notes in the case timeline, and every note records who wrote it and when. Notes persist across pause/resume cycles and service restarts, and cannot be edited or deleted once saved.

Work splits into one implementation story: extend the `State` schema with a `notes` field, add a `POST /runs/{thread_id}/notes` endpoint with validation, and include `notes` in the existing `GET /runs/{thread_id}` response. The story owns the schema change, the API endpoint, and the display integration together, because splitting them would require testing incomplete functionality (e.g., an endpoint that writes to a field the GET doesn't return, or a GET returning a field no endpoint writes).

---

## Dependencies

The single story extends the existing `State` structure (`src/agents/graph/states.py`), the FastAPI server (`src/agents/runtime/server.py`), and the SQLite checkpointer (`src/agents/runtime/checkpointer.py`). No external dependencies are added. The story touches `src/agents/runtime/runner.py` to increment `schema_version` for new runs.

---

## Story 1: Add reviewer notes to paused cases

**Repo:** `agent`

**Description:**  
Extend the `State` TypedDict with a `notes` field holding an append-only list of `Note` dicts. Add a `POST /runs/{thread_id}/notes` endpoint that validates the case is paused (`status in ["needs_input", "needs_review"]`), validates note text (1–2000 chars, via Pydantic), creates a `Note` with a UUID, timestamp, and author, and updates the graph state. The existing `GET /runs/{thread_id}` endpoint already returns the full `State.values` via the `_publish` helper in `server.py`, so once `notes` is added to `State`, it appears in the response automatically—no new code is needed in the GET endpoint. Increment `schema_version` to `2` for new runs; old runs (schema v1) show no `notes` key (graceful degradation: the UI treats missing `notes` as `[]`).

Assumption: No code change is needed in `GET /runs/{thread_id}` to display `notes`. The endpoint's three return paths all call `_publish`, which returns `{..., "values": snapshot.values}` or `{..., "values": with_pipeline(payload["values"])}`, and `snapshot.values` is the full `State` dict from the checkpointer, so once `notes` is added to `State`, it appears in every response. The first path returns `_publish(run)` when `run["status"] not in ("running", "failed")` and `"values" in run` (line 145 of `server.py`); the second returns `_publish({**run, "values": snapshot.values})` when `run` exists (line 149); the third returns `_publish({"thread_id": ..., "values": snapshot.values, ...})` when `run` does not exist (line 151). All three paths include `snapshot.values`, which is the checkpointed `State`, and `_publish` calls `with_pipeline(payload["values"])` when `payload.get("values")` is a dict (line 35 of `server.py`), so notes appear in the response automatically once the field is added to `State`. This matches the existing pattern for `flags`, `steps`, and `summary`.

Each `Note` carries `note_id` (UUID), `text` (≤2000 chars), `author`, and `created_at` (ISO-8601 UTC). The `notes` reducer uses `operator.add` to append new notes to the existing list (same pattern as `steps` and `flags`). Notes are immutable once saved (no edit/delete), persist across pause/resume cycles and service restarts (SQLite checkpointer), and are displayed oldest-first (insertion order).

**Covers:** AC1 (add note while paused), AC2 (notes are immutable), AC3 (display all notes), AC4 (notes persist), AC5 (notes only on paused cases), AC6 (validation of note content), AC7 (no disruption to existing functionality)

**Prerequisites:** None (opens the epic)

**Allowed paths:**
- `src/agents/graph/states.py` (add `Note` TypedDict, extend `State` with `notes: NotRequired[Annotated[list[Note], operator.add]]`)
- `src/agents/runtime/runner.py` (increment `schema_version` to `2` in `_initial_state`, add `"notes": []` to initial state dict)
- `src/agents/runtime/server.py` (add `CreateNoteRequest` Pydantic model, add `POST /runs/{thread_id}/notes` endpoint; no change to `GET /runs/{thread_id}` needed—it already returns the full `State.values` via `_publish`)

**Verification Approach:**

1. **Test: New runs write schema_version=2 and an empty notes list**  
   Start a new run with no service order; snapshot the state; assert `schema_version == 2` and `notes == []`.

2. **Test: Old runs (v1) loaded from checkpointer show no notes key**  
   Mock a checkpointer returning a state with `schema_version=1` and no `notes` field; load it; assert `"notes" not in state`.

3. **Test: Note structure matches the TypedDict contract**  
   Create a `Note` dict with all fields; assert keys are `note_id`, `text`, `author`, `created_at`; assert `note_id` is a valid UUID string; assert `text` is a string ≤2000 chars; assert `created_at` is an ISO-8601 UTC string.

4. **Test: Notes reducer appends new notes to the existing list**  
   Call the `State` annotated reducer with `current = [note1]` and `new = [note2]`; assert the result is `[note1, note2]` (insertion order preserved).

5. **Test: POST with valid text and author on a paused case returns 201 + the note**  
   Mock a paused state (`status="needs_input"`); POST `{"text": "checked", "author": "analyst@example.com"}`; assert response is 201; assert body has `note_id` (UUID), `text`, `author`, `created_at` (ISO-8601 UTC).

6. **Test: POST with empty text returns 422**  
   POST `{"text": "", "author": "analyst"}` to a paused case; assert response is 422; assert error message mentions "text".

7. **Test: POST with text over 2000 chars returns 422**  
   POST `{"text": "x" * 2001, "author": "analyst"}` to a paused case; assert response is 422; assert error message mentions "2000 characters".

8. **Test: POST to a running case returns 400 with "Cannot add notes to a running case"**  
   Mock a running state (`status="running"`); POST valid note; assert response is 400; assert error message is "Cannot add notes to a running case".

9. **Test: POST to a finished case returns 400 with "Cannot add notes to a finished case"**  
   Mock a finished state (`status="done"`); POST valid note; assert response is 400; assert error message is "Cannot add notes to a finished case".

10. **Test: The created note is appended to the state's notes list**  
    Mock a paused state with `notes=[note1]`; POST valid note; load the state after; assert `notes == [note1, new_note]`.

11. **Test: GET on a run with notes returns them in the state dict**  
    Mock a state with `notes=[{"note_id": "a1", "text": "checked", "author": "analyst", "created_at": "2025-01-24T12:00:00Z"}]`; GET `/runs/{thread_id}`; assert response `values["notes"]` matches the mocked list.

12. **Test: GET on a run with no notes (schema v1) returns values without a notes key**  
    Mock a state with `schema_version=1` and no `notes` field; GET `/runs/{thread_id}`; assert `"notes" not in response["values"]`.

13. **Test: Notes appear in insertion order (oldest first)**  
    Mock a state with `notes=[note1, note2, note3]` (distinct timestamps); GET `/runs/{thread_id}`; assert `response["values"]["notes"] == [note1, note2, note3]`.

14. **Test: A finished case still shows its notes**  
    Mock a finished state (`status="done"`) with `notes=[note1]`; GET `/runs/{thread_id}`; assert `response["values"]["notes"] == [note1]`.

15. **Test: Notes are shown alongside flags and summary (same state structure)**  
    Mock a state with `flags=[flag1]`, `summary="paused on mismatch"`, `notes=[note1]`; GET `/runs/{thread_id}`; assert all three appear in `response["values"]`.

**Total tests:** 15

## Closure

Story 1 closes the epic: once the schema is extended, the endpoint writes notes, and the GET endpoint returns them, the feature is complete. All acceptance criteria (AC1–AC7) are covered in the single story, with 15 total tests. The verification count matches the test count in the story's LLD (to be written in the `tests` role). No new dependencies or frameworks are introduced; the feature uses the existing Python stack, FastAPI validation, and LangGraph's annotated state merging.
