I will read through all the documents systematically and compare them against each other before deciding.

## Checking the documents

Let me trace through the acceptance criteria, schema, validation rules, and file paths across all documents.

**Acceptance criteria consistency:**

- Ticket lists: AC1-AC7
- Requirements lists: AC1-AC7 (same wording)
- Breakdown Story 1 covers: AC2, AC4, AC7
- Breakdown Story 2 covers: AC1, AC5, AC6
- Breakdown Story 3 covers: AC3, AC4, AC7
- Story 01-agent covers: AC1-AC7 (all of them)
- All seven ACs map to coverage across the breakdown and story

**Schema/contract consistency:**

The `Note` TypedDict appears identically in:
- HLD § Data Contracts
- Tech spec (specs/1.md) § Schema / API Contracts
- Story 01-agent § Interfaces

All say: `note_id: str`, `text: str`, `author: str`, `created_at: str`.

The `State.notes` field appears identically in:
- HLD § AD-1 (with `operator.add` reducer)
- Tech spec § Schema / API Contracts
- Story 01-agent § Interfaces

All say: `notes: NotRequired[Annotated[list[Note], operator.add]]`.

The `CreateNoteRequest` Pydantic model appears identically in:
- HLD § Data Contracts
- Tech spec § Schema / API Contracts
- Story 01-agent (in File Plan for `server.py`)

All say: `text: str = Field(min_length=1, max_length=2000)`, `author: str`.

**File paths:**

- Breakdown Story 1 lists: `src/agents/graph/states.py`, and says "Modify `src/agents/runtime/runner.py` in `_initial_state`"
- Story 01-agent File Plan lists: `src/agents/graph/states.py`, `src/agents/runtime/server.py`, `src/agents/runtime/runner.py`
- All three files match what the breakdown promised

**Validation rules:**

- Requirements AC5: allowed states are `needs_input` and `needs_review`; disallowed are `running`, `done`, `rejected`, `failed`
- HLD § AD-2: validates `status in ["needs_input", "needs_review"]`
- Story 01-agent § Algorithm Notes: same four disallowed statuses and same two allowed
- Breakdown Story 2: "validates the case is paused (`status in ["needs_input", "needs_review"]`)"

All agree.

**Error messages:**

- Requirements AC5: "e.g., 'Cannot add notes to a running case' or 'Cannot add notes to a finished case'"
- HLD § Data Contracts POST endpoint: `"Cannot add notes to a running case"` and `"Cannot add notes to a finished case"`
- Story 01-agent § Interfaces POST endpoint: same two messages
- Breakdown Story 2 Test 4: `"Cannot add notes to a running case"`
- Breakdown Story 2 Test 5: `"Cannot add notes to a finished case"`

All agree.

**Test counts:**

- Breakdown Story 1: 4 tests listed
- Breakdown Story 2: 6 tests listed
- Breakdown Story 3: 5 tests listed
- Breakdown § Closure: "15 total tests (4 + 6 + 5)"
- Story 01-agent § Tests: 15 tests listed (4 + 6 + 5)

Counts match.

**The C1 item:**

The breakdown says "Verify in `src/agents/runtime/runner.py`" in the "Allowed paths" note of Story 1, but then immediately adds "(currently `1` on line 94 per the story LLD)". This shows the breakdown already knows the exact location from the story, so the "Verify in" language is softened by the parenthetical. The story then gives the full detail in its File Plan. The breakdown is consistent with the story here because the parenthetical commits to the line number and change, and "Verify in" is used to mean "this is the file where the change goes" rather than "we are uncertain about this". The C1 item says this would confuse Dev, but the breakdown's own text already says "currently `1` on line 94", which is the same fact the story states. The story's File Plan also matches: it says line 94, `schema_version: 1` → `2`. The two documents agree.

**Checking C1's concern more carefully:**

The breakdown Story 1 "Allowed paths" section says:
- `src/agents/graph/states.py` (add Note TypedDict, extend State with notes, increment schema_version)
- "Modify `src/agents/runtime/runner.py` in `_initial_state` to set `schema_version` to `2` (currently `1` on line 94 per the story LLD)"

The word "Modify" is there. The file is listed as something to change. The parenthetical "(currently `1` on line 94 per the story LLD)" says the breakdown already checked the story and knows where the change goes. The C1 item says the breakdown's "Verify in" language contradicts the story, but the breakdown text I just quoted does not say "Verify in" anymore—it says "Modify". The C1 item quotes: "Verify in `src/agents/runtime/runner.py` where `_initial_state` sets `schema_version` (currently `1`)". That quote is not in the breakdown now. The breakdown now says "Modify `src/agents/runtime/runner.py` in `_initial_state` to set `schema_version` to `2` (currently `1` on line 94 per the story LLD)". The C1 concern was already fixed.

**Checking the rest of the documents:**

The HLD § AD-5 says "Increment `schema_version` to `2`" and "New runs write `schema_version: 2`". It does not name `runner.py` as the place to change (the HLD describes outcomes, not file paths). The tech spec says "Increment `schema_version` to `2` in new runs; existing runs stay at `1`" and later says "verify where initial state is created; likely `src/agents/runtime/runner.py::_initial_state`". The word "verify" appears in the tech spec, but the tech spec also says "likely", which is hedged. Then the breakdown and story both commit to `runner.py` line 94. The tech spec's "verify" is a writer's note, not a gap; the breakdown and story resolved it. No contradiction.

**Persistence:**

- Requirements AC4: "Notes are kept when the case is resumed, paused again, or the service restarts. A case that's finished still shows its notes."
- HLD § AD-1: "added by a new `/runs/{thread_id}/notes` POST endpoint" and "kept across resume/pause cycles and service restarts"
- Breakdown Story 1: "AC4 (persistence)"
- Story 01-agent: "The checkpointer writes the updated state to `<run_dir>/checkpoint.db`, so the note persists across service restarts."

All agree.

**Display location:**

- Requirements AC3: "in the same place the case's flags and summary are shown"
- HLD § AD-4: "`notes` appears alongside `flags`, `summary`, `steps` in the snapshot"
- Breakdown Story 3: "alongside flags and summary"
- Story 01-agent § Interfaces GET endpoint: `"notes": [...], "flags": [...], "summary": "..."`

All agree.

**Edge case: empty vs. over-limit text:**

- Requirements AC6: "empty or exceeds 2,000 characters" → "refused with a message"
- HLD § AD-3: Pydantic `min_length=1`, `max_length=2000`, raises 422 on schema violations
- Story 01-agent § Interfaces POST endpoint: Pydantic validation raises 422, error message names the field and constraint
- Breakdown Story 2 Test 2: POST with empty text returns 422
- Breakdown Story 2 Test 3: POST with text over 2000 chars returns 422

All agree.

**Schema version handling:**

- HLD § AD-5: "no migration script, cold-start schema bump"
- Tech spec § Schema / API Contracts: "Schema version: Increment `schema_version` to `2` in new runs; existing runs stay at `1`"
- Story 01-agent § Algorithm Notes: "No migration script: The schema bump is a cold-start strategy."

All agree.

**Nothing contradicts anything else.** The documents describe the same behavior, use the same validation rules, name the same files (or describe outcomes that the files implement), and list the same test counts.

## Notes for the gate

The breakdown's Story 1 section now says "Modify `src/agents/runtime/runner.py`" rather than "Verify in", so the C1 item's concern is already resolved. The C2 item is marked stale because the quoted text is not in the current story file.

The tech spec uses "verify in" once ("verify where initial state is created; likely `src/agents/runtime/runner.py::_initial_state`"), but the breakdown and story both commit to the exact file and line, so the hedged language in the tech spec does not create a gap. The person at the gate may want to confirm `runner.py` line 94 is still the right place before Dev starts, but the documents themselves agree on it.

VERDICT: PASS
