## What would go wrong

### C1
- file: hld.md
- section: ## Seed: Component Structure
- quote: src/agents/runtime/runner.py        # schema_version increment in _initial_state
- match: repo-file/agent/src/agents/runtime/runner.py › _initial_state
- severity: blocking
- blocks because: Dev would increment `schema_version` to 2 in `_initial_state` (line ~94), but `_initial_state` is at line 116 in the actual code, and it writes `"schema_version": 1` on line 122, so the location and current value in the HLD are wrong.
- problem: The HLD says to increment `schema_version` in `_initial_state` at "line ~94", but `_initial_state` starts at line 116 and `schema_version` is on line 122.
- change: Update the line reference to match the actual code: `_initial_state` function (line ~116), and confirm the change is `"schema_version": 1` → `"schema_version": 2`.

---

### C2
- file: breakdown.md
- section: ## Story 1: Add reviewer notes to paused cases
- quote: Extend the existing `GET /runs/{thread_id}` endpoint to include `notes` in the returned state dict.
- match: hld.md › ### GET `/runs/{thread_id}` (existing, extended)
- severity: blocking
- blocks because: Dev would add code to extend the GET endpoint, but the endpoint already returns the full `State.values` via `_publish`, so adding `notes` to `State` makes it appear in the response automatically — no new code is needed in `server.py` for the GET endpoint.
- problem: The breakdown says to "extend the existing `GET /runs/{thread_id}` endpoint", but the HLD (and the tech stub) say "No new code; `notes` appears in `State.values` once the field is added."
- change: Clarify that the GET endpoint needs no code change: once `notes` is added to `State`, it appears in the response automatically via the existing `_publish` helper.

---

### C3
- file: stories/01-agent.md
- section: ## File Plan
- quote: In `_initial_state` function (line ~94): change `"schema_version": 1` to `"schema_version": 2`.
- match: repo-file/agent/src/agents/runtime/runner.py › _initial_state
- severity: blocking
- blocks because: Dev would look for `_initial_state` at line ~94, but it starts at line 116, so they would either edit the wrong line or waste time searching.
- problem: The story says `_initial_state` is at line ~94, but it is at line 116 in the actual code.
- change: Update the line reference to ~116 (where `_initial_state` actually starts), and confirm `schema_version` is on line 122.

---

### C4
- file: stories/01-agent.md
- section: ## Interfaces
- quote: No new code needed; `notes` appears in `State.values` once the field is added to `State`, and `_publish` already returns the full `values` dict.
- match: breakdown.md › Extend the existing `GET /runs/{thread_id}` endpoint
- severity: blocking
- blocks because: The story (Interfaces section) says no new code is needed for the GET endpoint, but the breakdown says "Extend the existing `GET /runs/{thread_id}` endpoint", so Dev would add code for something that already works.
- problem: The story's Interfaces section correctly says no GET endpoint code is needed, but the breakdown says to extend it, so Dev would write unnecessary code.
- change: In the breakdown, clarify that the GET endpoint needs no code change: `notes` appears in the response once the field is added to `State`.

---

### C5
- file: specs/PER-201.md
- section: ## File Touch List
- quote: Increment `schema_version` default to `2` in new runs (verify where initial state is created; likely `src/agents/runtime/runner.py::_initial_state`).
- match: repo-file/agent/src/agents/runtime/runner.py › _initial_state
- severity: blocking
- blocks because: The tech stub says "verify where initial state is created; likely `src/agents/runtime/runner.py::_initial_state`", but the code shows `_initial_state` is exactly there (line 116), so Dev would waste time verifying something the documents should state.
- problem: The spec says "verify where initial state is created; likely `_initial_state`", but `_initial_state` is exactly where it is (line 116), so the "verify" instruction is unnecessary.
- change: State the location as a fact: "`src/agents/runtime/runner.py::_initial_state` (line ~116): change `'schema_version': 1` to `'schema_version': 2`."

---

## Notes for the gate

- The documents agree on the schema change (`schema_version: 1` → `2`), the `notes` field structure, the POST endpoint validation logic, and the persistence mechanism. The line number mismatch (C1, C3, C5) is a direct consequence of the code being at line 116, not ~94, so fixing it once in the HLD cascades to the story and spec.
- The GET endpoint confusion (C2, C4) stems from the breakdown saying "extend" when the HLD and story correctly say "no new code." Once the breakdown is clarified, the story and spec already match.
- The POST endpoint logic, validation (Pydantic), state update (`graph.aupdate`), and the `Note` TypedDict structure are consistent across all documents and match the existing code patterns (`Flag`, `Need`, `StepRecord`).
- The documents correctly name the existing helpers (`_snapshot`, `_publish`, `open_graph`) and their locations in `server.py` and `runner.py`.

VERDICT: FAIL
