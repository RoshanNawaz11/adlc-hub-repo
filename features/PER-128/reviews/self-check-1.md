## What would go wrong

**1. Missing repo-tree makes file paths unverifiable**

The breakdown and all three stories list concrete file paths (`src/agents/state.py`, `src/agents/nodes/orchestrator/node.py`, `tests/test_usage_tracking.py`, etc.), but `repo-tree/agent` is not mapped. The developer cannot verify these paths exist or match the actual codebase structure. They might build the feature in the wrong locations or create files that conflict with existing ones.

**Consequence:** Code placed in invented paths would fail integration, and the PR would need rework to move everything to the correct locations.

---

**2. Story 1 duplicates the UsageStats `__add__` method in two places**

In `stories/01-agent.md`, the `UsageStats` dataclass includes a `__add__` method for the reducer. The same story's "State Reducer" section defines a separate `_reduce_usage` function that iterates over nodes and calls `result[node] + new_stats`. The dataclass's `__add__` already handles the addition; the reducer only needs to loop and assign. One of these is redundant, and the developer would have to guess which to implement.

**Consequence:** If both are implemented, the code has unnecessary duplication. If the developer picks only one, it might not match what the tests expect (Test 1 in Story 1 asserts `state.usage[node]` after `apply_update`, which implies the reducer is called, but the test doesn't show whether it uses `__add__` directly or `_reduce_usage`).

---

**3. Story 2 cannot generate candidates without knowing NODES and STAGE_AGENTS**

The story's algorithm requires `NODES` and `STAGE_AGENTS` to generate candidates, but it lists this as an "Open Question" and suggests hardcoding if not found. The forced_route logic is the core of AC2, and hardcoding would make the tests pass for the wrong reasons (they would test the hardcoded list, not the actual routing logic). Without the real constants, the developer cannot verify the candidate count in Test 5, and the feature would ship with placeholder data.

**Consequence:** Tests would pass against a fake candidate set, and the forced_route function would return wrong answers in production when actual NODES differ from the hardcoded list.

---

**4. Story 3's "forced hop matches model hop" test cannot work as written**

Test 6 in `stories/03-agent.md` patches `forced_route` twice in the same test function to simulate both a forced hop and a model hop, then asserts the status/priority/confidence updates are equivalent. But the test mocks `_decide` to return the forced hop's results, which makes the assertion circular (it's comparing the forced hop's values to themselves). The test should call `_compute_status_for_route` directly for both paths, or run two separate hops with the same initial state and compare outcomes, but it does neither.

**Consequence:** The test would always pass (comparing X to X), and it wouldn't catch a bug where forced hops and model hops compute status differently.

---

**5. No concrete API response structure for GET /runs/{thread_id}**

Story 3 modifies `src/agents/api/routes.py` to add `total_usage` to the `values` payload, and the HLD says the API "returns the totals in `values`", but neither document shows what `values` looks like today or where `total_usage` goes within it. The story's pseudocode has `**state.values, "total_usage": {...}`, which assumes `state.values` is a dict, but that's not in the State schema from Story 1. The developer would have to guess the existing response format.

**Consequence:** The developer might nest `total_usage` incorrectly (e.g. `{"values": {"total_usage": ...}}` vs `{"total_usage": ..., "values": {...}}`), and Test 4 in Story 3 would fail because it asserts `data["values"]["total_usage"]` exists.

---

**6. Story 1's migration test cannot run without a real serialized state**

Test 4 in `stories/01-agent.md` loads a "minimal state snapshot without the `usage` field", but the test code shows `old_state_data = {"run_id": "test-123", "status": "running"}`. The State schema likely has required fields beyond `run_id` and `status` (e.g. `flags`, `plan`, `usage` itself), and a dict with only two fields would fail deserialization before the loader's `if "usage" not in...` check runs. The test would error instead of proving migration works.

**Consequence:** The test would fail with a schema validation error (missing required fields), not test the migration logic at all.

---

VERDICT: FAIL
