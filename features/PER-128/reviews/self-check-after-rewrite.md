## What would go wrong

### Story 3 Test 2 schema validation timing

Test 2 in `stories/03-agent.md` (`test_run_without_usage_seen_flags_resumes`) says in its docstring:

> The old_state_data dict must include those fields to avoid schema validation errors before the loader's migration check runs.

But the test code shows:

```python
old_state_data = {
    "run_id": "test-123",
    "status": "paused",
    # Developer: Add ALL other required State fields here 
}

state = load_state(old_state_data)
```

The test assumes `load_state` will see the missing `usage` and `seen_flags` and default them, then pass the result to `State(**serialized_state)`. If the State schema validates **before** the loader's migration check adds the missing fields, the test will fail with a schema error (missing required fields) instead of testing migration.

The developer would either add `usage` and `seen_flags` to `old_state_data` to make the test pass (defeating the migration test), or the test would fail for the wrong reason. The loader implementation in the story shows the migration check happens **before** deserialization, so this works—but the test's own comment warns about a risk the implementation avoids. A developer following the comment might write the test differently and get stuck.

**Fix:** Remove the warning from the test's docstring (lines starting "Without all required fields, the test will fail..."). The implementation already handles this correctly by checking for missing fields before calling `State(**serialized_state)`. The comment creates doubt where none exists.

---

### Story 3 Test 6 mock circular dependency

Test 6 in `stories/03-agent.md` (`test_forced_hop_status_matches_model_hop_different_states`) mocks `_decide` to call the helper functions `_compute_status_for_route`, etc., to prove forced hops and model hops use the same logic. The mock implementation is:

```python
def mock_decide_impl(state):
    route = ["planner"]
    return StateUpdate(
        route=route,
        # Developer: Replace these placeholders with actual function calls:
        # status=_compute_status_for_route(route, state),
        # ...
        status="placeholder_status",  # Replace after discovering function
```

If the developer replaces the placeholders with real function calls, the mock calls the helpers, and the test compares the mock's output (which uses the helpers) to the forced hop's output (which also uses the helpers). Both paths call the same functions, so they match—but that doesn't prove forced hops **actually** use the helpers in production code, only that the mock does.

The test's docstring says "This test avoids circular mocking by comparing outcomes from different initial states," but it still has a circular dependency: the assertion that forced and model hops match only holds if the mock correctly calls the helpers. If the developer writes the mock wrong (e.g., hardcodes values or calls different logic), the test passes even when the production code is broken.

**What would go wrong:** A developer could write `_record_forced_hop` to hardcode `status="complete"`, write the mock to also return `"complete"`, and the test would pass. The assertion `forced_result_a.status == model_result_a.status` is true, but for the wrong reason. The test claims to verify both paths compute the same way, but it only verifies the mock and the forced hop produce the same output—it doesn't verify the forced hop uses the helpers.

A sound test would either:
- Spy on the helper function calls in `_record_forced_hop` (assert the helpers were called with the expected args), or
- Compare forced hop outcomes to **known-good expected values** (not to a mock that might be wrong).

The current test relies on the developer writing the mock correctly, which is exactly what needs verification.

---

### Story 2 Test 5 candidate count placeholder

Test 5 in `stories/02-agent.md` (`test_forced_route_delegates_to_broken_rules`) says:

```python
# Developer: Replace expected_candidate_count with the actual count
# after discovering NODES and STAGE_AGENTS
expected_candidate_count = 999  # Placeholder — replace with real count
assert mock_broken_rules.call_count == expected_candidate_count
```

The test mocks `broken_rules` to return `[]` (all candidates pass), calls `forced_route`, and asserts the call count matches the expected candidate count. But the expected count is a placeholder (`999`). If the developer forgets to replace it, the test fails with a clear mismatch (`expected 999, got 12`), so they know to fix it—but the ticket goes to Dev with a broken test.

The breakdown says Story 2 has "no upstream blockers," but Test 5 **blocks on discovering NODES and STAGE_AGENTS** (an Open Question marked BLOCKING). If the developer can't find those constants, they can't calculate the candidate count, and they can't finish Test 5. The test is written as if the discovery already happened, but the story's Open Questions say it hasn't.

**What would go wrong:** The developer writes the other six tests, writes `forced_route` using discovered or assumed node lists, but can't finalize Test 5 until they know the candidate count. If they hardcoded a node list (against the LLD's instruction), Test 5 would pass with the wrong count and miss bugs. If they did discover the real list, they still have to do the combinatorics by hand (`len(NODES) + sum(C(len(stage_agents), k) for k in 1..len)`), which is error-prone.

A better test structure would calculate the count from the same `NODES` and `STAGE_AGENTS` that `forced_route` uses, so they stay in sync. Example:

```python
from forced_route_module import NODES, STAGE_AGENTS
expected_count = len(NODES) + sum(1 for _ in combinations(STAGE_AGENTS, r) for r in range(1, len(STAGE_AGENTS)+1))
assert mock_broken_rules.call_count == expected_count
```

That way the test doesn't depend on the developer hand-calculating a number.

---

VERDICT: FAIL
