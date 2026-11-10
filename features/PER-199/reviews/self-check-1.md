## What would go wrong

### C1
- file: breakdown.md
- section: ## Story 1: Add notes schema to State and increment schema version
- quote: Verify in `src/agents/runtime/runner.py` where `_initial_state` sets `schema_version` (currently `1`)
- match: stories/01-agent.md › ## File Plan › `src/agents/runtime/runner.py`
- problem: The breakdown says "Verify in" but the story's File Plan section specifies the exact line number (94) and exact change needed. The breakdown's "Verify in" language suggests uncertainty about where the change goes, but the story is definite about it. When the breakdown lists a file path as something to check rather than a place to change, but the story changes it, Dev would wonder whether the change was actually approved.
- change: Replace "Verify in `src/agents/runtime/runner.py` where `_initial_state` sets `schema_version` (currently `1`)" with "Modify `src/agents/runtime/runner.py` in `_initial_state` to set `schema_version` to `2` (currently `1` on line 94 per the story LLD)".

---

### C2
- file: stories/01-agent.md
- section: ## File Plan › `src/agents/runtime/runner.py`
- quote: In `_initial_state` function (line 94): Change `"schema_version": 1` to `"schema_version": 2`. Add `"notes": []` to the initial state dict (after `"schema_version": 2`, before `"run_id": run_id`).
- match: breakdown.md › ## Story 1 › **Allowed paths**
- problem: The breakdown lists `src/agents/runtime/runner.py` as something to "verify" in the "Allowed paths" note, but the story's File Plan section says to modify it and gives exact line numbers and field positions. The breakdown treats runner.py as context to check, but the story treats it as a file to change, so Dev would follow the story and modify runner.py without knowing whether that was in scope.
- change: Add `src/agents/runtime/runner.py` to the breakdown's Story 1 "Allowed paths" section with a note: "Modify `_initial_state` to set `schema_version: 2` and initialize `notes: []`."

---

## Notes for the gate

- The breakdown says Story 1 must complete before Story 2 starts ("the endpoint writes to the new `notes` field"), but it also says "No story blocks another from starting in parallel". The dependency logic is sound (the schema must exist before the API writes to it), and the merge order (schema → API → UI) clarifies sequencing, so the "in parallel" note likely means reviewers can read the stories concurrently. The person at the gate may want to confirm the dev team understands the order.

- The story lists 15 tests (4 + 6 + 5) and the breakdown lists the same count (4 + 6 + 5), but the story combines all three into one delivery because it implements the full feature in one ticket. The breakdown's three-story split was architectural (schema, API, UI), and the story collapsed them. The test count matches, so nothing is missing, but the gate may want to confirm the test grouping still makes sense when delivered as one story.

---

VERDICT: FAIL
