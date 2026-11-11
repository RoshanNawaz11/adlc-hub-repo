## What would go wrong

### C1
- file: stories/01-agent.md
- section: ## File Plan
- quote: In `_initial_state` function (line ~94): change `"schema_version": 1` to `"schema_version": 2`.
- match: repo-file/agent/src/agents/runtime/runner.py › _initial_state starts at line 116
- severity: blocking
- blocks because: Dev would look for `_initial_state` at line ~94 and find the wrong place in the file, wasting time or editing the wrong line.
- problem: The line number is wrong. `_initial_state` starts at line 116, and the `schema_version` assignment is at line 122.
- change: Change "line ~94" to "line ~116" (where `_initial_state` starts), or "line ~122" (where `schema_version` is assigned).

---

### C2
- file: breakdown.md
- section: ## Story 1: Add reviewer notes to paused cases
- quote: Assumption: No code change is needed in `GET /runs/{thread_id}` to display `notes`.
- match: repo-file/agent/src/agents/runtime/server.py › get_run function (line ~154)
- severity: blocking
- blocks because: Dev would assume the GET endpoint already returns `notes` once the field is added to `State`, but the code shows that `_publish` is only called when `run["status"]` is not `"running"` or `"failed"` and `"values" in run`, and also when `snapshot.values` exists. The assumption needs to state which code path makes `notes` appear.
- problem: The assumption says no GET code change is needed, but does not say which existing code returns `notes`. The story and specs say `_publish` returns `State.values`, but the actual `get_run` code has three return paths: one that calls `_publish(run)` when `run` is finished, one that calls `_publish({**run, "values": snapshot.values})` when `run` exists, and one that calls `_publish({"thread_id": ..., "values": snapshot.values, ...})` when `run` does not exist. The assumption should name the path.
- change: After "No code change is needed in `GET /runs/{thread_id}` to display `notes`", add: "The endpoint's three return paths all call `_publish`, which returns `{..., "values": snapshot.values}` or `{..., "values": with_pipeline(payload["values"])}`, and `snapshot.values` is the full `State` dict from the checkpointer, so once `notes` is added to `State`, it appears in every response."

---

VERDICT: FAIL
