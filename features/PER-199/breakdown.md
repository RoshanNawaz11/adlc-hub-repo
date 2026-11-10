# Breakdown — PER-199

**Feature:** Reviewers can leave notes on a case  
**Repo:** `agent` (Python, LangGraph)  
**Date:** 2025-01-24

---

## Overview

This feature adds persistent, immutable reviewer notes to paused cases. A reviewer can add a note when a case is paused (for flags or final review), see all notes in the case timeline, and every note records who wrote it and when. Notes persist across pause/resume cycles and service restarts, and cannot be edited or deleted once saved.

Work splits into three implementation stories: database schema extension and migration, REST API endpoint with validation, and UI integration with the case state structure.

---

## Dependencies

All stories build on the existing `State` structure (`src/agents/graph/states.py`), the FastAPI server (`src/agents/runtime/server.py`), and the SQLite checkpointer (`src/agents/runtime/checkpointer.py`). The schema story must complete before the API story starts (the endpoint writes to the new `notes` field). The API story must complete before the UI story (the UI reads the `notes` field from the state snapshot). No story blocks another from starting in parallel, but the merge order is: schema → API → UI.

---

## Story 1: Add notes schema to State and increment schema version

**Repo:** `agent`

**Description:**  
Extend the `State` TypedDict with a `notes` field holding an append-only list of `Note` dicts. Increment `schema_version` to `2` for new runs. Existing runs (schema v1) show no `notes` key; the UI treats missing `notes` as an empty list (no migration script). Each `Note` carries `note_id` (UUID), `text` (≤2000 chars), `author`, and `created_at` (ISO-8601 UTC). The `notes` reducer uses `operator.add` to append new notes to the existing list (same pattern as `steps` and `flags`).

**Covers:** AC2 (immutable notes), AC4 (persistence), AC7 (no disruption to existing functionality)

**Prerequisites:** None (opens the epic)

**Allowed paths:**
- `src/agents/graph/states.py` (add `Note` TypedDict, extend `State` with `notes`, increment `schema_version`)
- Modify `src/agents/runtime/runner.py` in `_initial_state` to set `schema_version` to `2` (currently `1` on line 94 per the story LLD)

**Verification Approach:**

1. **Test: New runs write schema_version=2 and an empty notes list**  
   Start a new run with no service order; snapshot the state; assert `schema_version == 2` and `notes == []`.

2. **Test: Old runs (v1) loaded from checkpointer show no notes key**  
   Mock a checkpointer returning a state with `schema_version=1` and no `notes` field; load it; assert `"notes" not in state`.

3. **Test: Note structure matches the TypedDict contract**  
   Create a `Note` dict with all fields; assert keys are `note_id`, `text`, `author`, `created_at`; assert `note_id` is a valid UUID string; assert `text` is a string ≤2000 chars; assert `created_at` is an ISO-8601 UTC string.

4. **Test: Notes reducer appends new notes to the existing list**  
   Call the `State` annotated reducer with `current = [note1]` and `new = [note2]`; assert the result is `[note1, note2]` (insertion order preserved).

**Total tests:** 4

## Story 2: Add POST /runs/{thread_id}/notes endpoint with validation

**Repo:** `agent`

**Description:**  
Add a `POST /runs/{thread_id}/notes` endpoint in `src/agents/runtime/server.py`. The endpoint accepts `text` (1–2000 chars) and `author`, validates the case is paused (`status in ["needs_input", "needs_review"]`), creates a `Note` dict with a new UUID and timestamp, updates the graph state with `{"notes": [note]}`, and returns 201 + the created note. Pydantic validates `text` length (reject with 422 if empty or over 2000 chars). State validation rejects with 400 + a message if the case is not paused (running, done, rejected, failed). The endpoint uses the existing `_snapshot` helper to load the state and `open_graph` to update it.

**Covers:** AC1 (add note while paused), AC5 (notes only on paused cases), AC6 (validation of note content)

**Prerequisites:** Story 1 (schema must exist before the endpoint writes to it)

**Allowed paths:**
- `src/agents/runtime/server.py` (add `CreateNoteRequest` Pydantic model, add `POST /runs/{thread_id}/notes` handler)
- Use existing helpers: `_snapshot(thread_id)` to load state, `open_graph(thread_id)` to update it
- Verify in `src/agents/graph/states.py` the allowed status literals (`needs_input`, `needs_review`, `running`, `done`, `rejected`, `failed`)

**Verification Approach:**

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

**Total tests:** 6

---

## Story 3: Display notes in GET /runs/{thread_id} response

**Repo:** `agent`

**Description:**  
Extend the `GET /runs/{thread_id}` endpoint (already in `src/agents/runtime/server.py`) to include `notes` in the returned state dict when the field is present. No new endpoint: the `_snapshot` helper already loads the full state, and `_publish` already wraps it with the pipeline stages. The UI reads `notes` from the state snapshot and displays them oldest-first (insertion order) in the case timeline, alongside flags and summary. The UI treats missing `notes` (schema v1 runs) as an empty list.

**Covers:** AC3 (display all notes), AC4 (notes persist), AC7 (no disruption to existing functionality)

**Prerequisites:** Story 2 (notes must be writable before the UI can read them)

**Allowed paths:**
- `src/agents/runtime/server.py` (no new code: `_publish` already includes `values`, and `values` carries `notes` once Story 1's schema is in place)
- Verify in `src/agents/graph/states.py` that `notes` is a top-level `State` field (not nested under another dict)

**Verification Approach:**

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

**Total tests:** 5

---

## Closure

Story 3 closes the epic: once the UI displays notes from the state snapshot, the feature is complete. All acceptance criteria (AC1–AC7) are covered across the three stories, with 15 total tests (4 + 6 + 5). The verification count matches the test count in each story's LLD (to be written in the `tests` role). No new dependencies or frameworks are introduced; the feature uses the existing Python stack, FastAPI validation, and LangGraph's annotated state merging.
