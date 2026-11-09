# Story 03-agent LLD

**Ticket:** PER-128-3  
**Epic:** PER-128  
**Repo:** agent  
**Title:** Integrate forced routing into orchestrator node

---

## Acceptance Criteria

From `requirements.md` AC3, AC4, AC5 (this story's scope):

1. **AC3.1:** When `forced_route` returns a route and no model work is needed, the orchestrator skips `_decide` and uses the forced route without calling the model.
2. **AC3.2:** A forced hop records a step with title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`, and updates `status`, `flagged`, `priority`, `confidence` as a model hop would, leaving `summary` unchanged.
3. **AC3.3:** When the forced route is `["flag-review"]` or `["review"]`, the orchestrator calls the model (to write reviewer summary).
4. **AC3.4:** When an unresolved flag exists whose ID is not in `state.seen_flags`, the orchestrator calls the model (to merge/address the flag).
5. **AC3.5:** After every model call, the orchestrator extracts flag IDs from the brief and adds them to `state.seen_flags`.
6. **AC4.1:** New tests verify the orchestrator makes no model call on forced hops (via gateway mock).
7. **AC4.2:** New tests verify the model is called when forced route requires reviewer summary or unseen flag handling.
8. **AC4.3:** New tests verify forced hop step structure matches expected format.
9. **AC4.4:** New tests verify `usage` accumulates correctly across calls, including runs without prior `usage` field.
10. **AC4.5:** All existing tests pass unchanged (`uv run pytest -q`).
11. **AC5.1:** PR description reports before/after metrics (model calls, tokens) for one service order.

---

## HLD Outcomes (Binding Constraints)

From `hld.md`:

- **AD-1 (State schema extensions):** `State` gains `usage: Dict[str, UsageStats]` (from Story 1) and `seen_flags: Set[str]`. The `seen_flags` reducer unions incoming IDs with existing ones. Missing `seen_flags` in old runs treated as `set()`.
- **AD-2 (forced_route contract):** Returns `list[str]` when exactly one candidate passes `broken_rules`, else `None` (from Story 2).
- **AD-3 (Orchestrator skip conditions):** Skip `_decide` when `forced_route` returns a route, UNLESS: (1) route is `["flag-review"]` or `["review"]`, or (2) unresolved flag exists with ID not in `state.seen_flags`.
- **AD-4 (Usage recording contract):** All model call sites record token counts (from Story 1). This story adds orchestrator's contribution.
- **AD-5 (Runtime aggregation and logging):** `run_query` sums `state.usage` across nodes and logs totals at run end/pause. `GET /runs/{thread_id}` returns totals in `values`.

---

## File Plan

**CRITICAL:** Repo tree not mapped. Developer MUST locate actual file paths by scanning the agent codebase before implementation. All paths below are placeholders derived from ticket context and will fail if used directly.

### Files to Create

1. **Test file for orchestrator forced routing (path TBD from codebase scan)**
   - Purpose: Test orchestrator's forced routing integration
   - Content: 6 tests per Verification Approach below
   - Placeholder name: `tests/agents/nodes/test_orchestrator_forced_routing.py`

2. **Test file for usage tracking integration (path TBD from codebase scan)**
   - Purpose: Integration tests for usage aggregation and API response
   - Content: 4 tests per Verification Approach below
   - Placeholder name: `tests/agents/integration/test_usage_tracking.py`

### Files to Modify

3. **State schema module (path TBD from codebase scan)**
   - Add `seen_flags: Set[str]` field to `State` schema
   - Implement union reducer: `_reduce_seen_flags(existing, delta) -> Set[str]`
   - Placeholder path: `src/agents/state.py`

4. **State loader module (path TBD from codebase scan)**
   - Handle missing `seen_flags` field by initializing to `set()` when loading old runs
   - Placeholder path: `src/agents/state_loader.py`

5. **Orchestrator node module (path TBD from codebase scan)**
   - Import `forced_route` from route_rules module
   - At each decision point, call `forced_route(state)`
   - Implement skip logic (two exceptions per AD-3)
   - Record forced hop step (title, result, evidence, status updates)
   - Extract flag IDs from brief after model calls
   - Update `state.seen_flags` via reducer
   - Placeholder path: `src/agents/nodes/orchestrator/node.py`

6. **Runtime query handler module (path TBD from codebase scan)**
   - In `run_query`: aggregate `state.usage` across nodes
   - Log totals at run end/pause
   - Placeholder path: `src/agents/runtime/runner.py`

7. **API routes module (path TBD from codebase scan)**
   - In `GET /runs/{thread_id}` handler: read `state.usage`, sum across nodes, return totals in response payload
   - Developer must verify existing response structure before nesting `total_usage`
   - Placeholder path: `src/agents/api/routes.py`

---

## Interfaces and Data Structures

### State Extension

```python
# State schema module (path TBD from codebase scan)
from typing import Set

# Add to State schema (TypedDict or dataclass)
seen_flags: Set[str] = field(default_factory=set)
```

### State Reducer for seen_flags

```python
# State schema module (path TBD from codebase scan)
def _reduce_seen_flags(
    existing: Set[str],
    delta: Set[str]
) -> Set[str]:
    """
    Union reducer: merge incoming flag IDs with existing.
    
    Args:
        existing: Current set of seen flag IDs
        delta: New flag IDs from latest model call's brief
    
    Returns:
        Union of existing and delta
    """
    return existing.union(delta)
```

### Orchestrator Decision Flow

```python
# Orchestrator node module (path TBD from codebase scan)

def _should_skip_model_call(route: list[str], state: State) -> bool:
    """
    Determines if we can skip _decide for this forced route.
    
    Returns False (call model) when:
    1. Route is ["flag-review"] or ["review"] — model writes reviewer summary
    2. Any unresolved flag exists with ID not in state.seen_flags — model must merge
    
    Otherwise returns True (skip model call).
    """
    # Exception 1: reviewer summary required
    if route in [["flag-review"], ["review"]]:
        return False
    
    # Exception 2: unseen flag requires model attention
    unresolved_flags = [f for f in state.flags if not f.resolved]
    unseen_flags = [f for f in unresolved_flags if f.id not in state.seen_flags]
    if unseen_flags:
        return False
    
    return True


def decide_next_hop(state: State) -> StateUpdate:
    """
    Main orchestrator decision point.
    """
    route = forced_route(state)
    
    if route is not None and _should_skip_model_call(route, state):
        # Forced hop: no model call
        return _record_forced_hop(route, state)
    else:
        # Model hop: call _decide as usual
        result = _decide(state)
        
        # Extract flag IDs from brief and update seen_flags
        brief_flag_ids = _extract_flag_ids_from_brief(result.brief)
        result.seen_flags_delta = brief_flag_ids
        
        return result
```

### Forced Hop Recording

```python
# Orchestrator node module (path TBD from codebase scan)

def _record_forced_hop(route: list[str], state: State) -> StateUpdate:
    """
    Record a forced hop step without calling the model.
    
    Updates status, priority, confidence as a model hop would.
    Leaves summary unchanged.
    
    Note: Developer must locate _compute_status_for_route, 
    _compute_priority_for_route, and _compute_confidence_for_route 
    in the codebase before implementing. These functions are assumed 
    to exist and be used by model hops today.
    """
    step = Step(
        title=f"Next: {route}",
        result="forced: only one valid route",
        evidence={"attempts": 0}
    )
    
    # Compute status, priority, confidence (same logic as model hop)
    new_status = _compute_status_for_route(route, state)
    new_priority = _compute_priority_for_route(route, state)
    new_confidence = _compute_confidence_for_route(route, state)
    
    return StateUpdate(
        route=route,
        step=step,
        status=new_status,
        flagged=state.flagged,  # unchanged
        priority=new_priority,
        confidence=new_confidence,
        summary=state.summary,  # unchanged per AC3.2
    )
```

### Brief Flag ID Extraction

```python
# Orchestrator node module (path TBD from codebase scan)

def _extract_flag_ids_from_brief(brief: dict) -> Set[str]:
    """
    Extract flag IDs from the brief passed to the model.
    
    Args:
        brief: The input dict sent to run_skill, containing flags list
    
    Returns:
        Set of flag IDs present in the brief
    """
    flags = brief.get("flags", [])
    return {flag["id"] for flag in flags if "id" in flag}
```

---

## Algorithm Notes

### 1. Orchestrator Integration Logic

```python
# Orchestrator node module (path TBD from codebase scan)

def run_orchestrator(state: State) -> StateUpdate:
    """
    Orchestrator node execution with forced routing.
    
    Flow:
    1. Call forced_route(state)
    2. If route is forced:
       a. Check skip conditions (no pause, no unseen flags)
       b. If can skip: record forced hop, no model call
       c. Else: proceed to model call
    3. If route is None (multiple valid routes): call model
    4. After any model call: extract brief flag IDs, update seen_flags
    """
    route = forced_route(state)
    
    if route is not None:
        # Route is forced — check if we can skip model call
        if _should_skip_model_call(route, state):
            # No model work needed: record forced hop
            return _record_forced_hop(route, state)
    
    # Model call needed (route is None OR skip conditions failed)
    model_result = _decide(state)
    
    # Extract flag IDs from brief
    brief_flag_ids = _extract_flag_ids_from_brief(model_result.brief)
    
    # Update seen_flags via reducer (union with existing)
    return StateUpdate(
        **model_result.__dict__,
        seen_flags_delta=brief_flag_ids,
    )
```

### 2. Usage Aggregation and Logging

```python
# Runtime query handler module (path TBD from codebase scan)

def run_query(thread_id: str, query: str) -> dict:
    """
    Execute query, aggregate usage, log totals at end/pause.
    
    Note: Developer must locate the exact condition in run_query that 
    signals run end (status transitions to "complete" or "failed") or 
    pause (status becomes "paused") before implementing the logging call.
    """
    # ... existing run logic ...
    
    # At run end or pause (developer must identify the exact condition):
    total_usage = _aggregate_usage(state.usage)
    
    _log_usage_totals(total_usage)
    
    return {
        "result": final_result,
        "usage": total_usage,
    }


def _aggregate_usage(usage_per_node: Dict[str, UsageStats]) -> UsageStats:
    """
    Sum usage across all nodes.
    Uses UsageStats.__add__ from Story 1.
    """
    total = UsageStats()
    for node, stats in usage_per_node.items():
        total = total + stats
    return total


def _log_usage_totals(usage: UsageStats):
    """
    Log total usage at INFO level.
    """
    logger.info(
        f"Run completed — "
        f"calls: {usage.calls}, "
        f"input_tokens: {usage.input_tokens}, "
        f"output_tokens: {usage.output_tokens}, "
        f"cache_creation_input_tokens: {usage.cache_creation_input_tokens}, "
        f"cache_read_input_tokens: {usage.cache_read_input_tokens}"
    )
```

### 3. API Response Extension

```python
# API routes module (path TBD from codebase scan)

@app.get("/runs/{thread_id}")
def get_run(thread_id: str) -> dict:
    """
    Return run status and values, including total usage.
    
    CRITICAL: Developer must inspect the existing GET /runs/{thread_id} 
    response structure before implementing. Verify where "values" nests 
    and what existing fields it contains. The placeholder below assumes 
    "values" is a root-level dict per HLD AD-5, but this must be confirmed 
    from the actual API handler code.
    
    After codebase scan, adjust the return structure and Test 4's assertions 
    to match the discovered schema.
    """
    state = load_state(thread_id)
    
    # Aggregate usage across nodes
    total_usage = _aggregate_usage(state.usage)
    
    # Developer: Verify existing response structure before implementing.
    # Placeholder assumes values.total_usage per HLD AD-5.
    return {
        "thread_id": thread_id,
        "status": state.status,
        "values": {
            # Existing values fields — developer must preserve these
            # (inspect current handler to identify them)
            # ...
            
            # New total_usage field
            "total_usage": {
                "calls": total_usage.calls,
                "input_tokens": total_usage.input_tokens,
                "output_tokens": total_usage.output_tokens,
                "cache_creation_input_tokens": total_usage.cache_creation_input_tokens,
                "cache_read_input_tokens": total_usage.cache_read_input_tokens,
            }
        }
    }
```

### 4. State Migration for seen_flags

```python
# State loader module (path TBD from codebase scan)

def load_state(serialized_state: dict) -> State:
    """
    Load state from persistence, handling missing fields.
    
    Pre-change runs have no 'usage' or 'seen_flags'.
    Default both to empty to allow execution to continue.
    
    Note: This extends Story 1's migration logic. If Story 1's loader 
    already handles 'usage', add 'seen_flags' handling in the same pattern.
    """
    if "usage" not in serialized_state:
        serialized_state["usage"] = {}
    
    if "seen_flags" not in serialized_state:
        serialized_state["seen_flags"] = set()
    
    return State(**serialized_state)
```

---

## Tests

### Test File 1: Orchestrator Forced Routing Tests (path TBD)

6 tests matching the breakdown.md verification count exactly. Developer must construct realistic State instances by examining the actual State schema before writing these tests.

#### Test 1: Forced Hop with No Pause, No Unseen Flags → No Model Call

```python
def test_forced_hop_no_model_call(fake_gateway, mocker):
    """
    Given forced_route returns ["planner"] and no skip conditions apply
    When orchestrator decides next hop
    Then no model call is made (via gateway mock)
    And forced hop step is recorded
    
    Verifies AC3.1, AC4.1
    
    Developer: Replace placeholder import paths with actual paths from codebase scan.
    """
    # Developer: Replace 'path.to.forced_route' with actual module path
    mock_forced_route = mocker.patch(
        'path.to.forced_route',  # Placeholder — replace with actual path
        return_value=["planner"]
    )
    
    # Developer: Replace 'path.to._decide' with actual module path
    mock_decide = mocker.patch(
        'path.to._decide'  # Placeholder — replace with actual path
    )
    
    # Developer: Construct State with all required fields from schema scan
    state = State(
        # Add all required State fields here after schema scan
        flags=[],
        seen_flags=set(),
    )
    
    # Developer: Replace 'run_orchestrator' with actual function name if different
    result = run_orchestrator(state)
    
    # Assert forced_route was called
    mock_forced_route.assert_called_once_with(state)
    
    # Assert _decide was NOT called
    mock_decide.assert_not_called()
    
    # Assert forced hop step recorded
    assert result.step.title == "Next: ['planner']"
    assert result.step.result == "forced: only one valid route"
    assert result.step.evidence["attempts"] == 0
```

#### Test 2: Forced Route ["flag-review"] → Model Called

```python
def test_forced_route_flag_review_calls_model(fake_gateway, mocker):
    """
    Given forced_route returns ["flag-review"]
    When orchestrator decides next hop
    Then model is called (to write reviewer summary)
    
    Verifies AC3.3, AC4.2
    
    Developer: Replace placeholder import paths with actual paths from codebase scan.
    """
    # Developer: Replace with actual paths
    mock_forced_route = mocker.patch(
        'path.to.forced_route',
        return_value=["flag-review"]
    )
    
    mock_decide = mocker.patch(
        'path.to._decide',
        return_value=StateUpdate(
            route=["flag-review"],
            summary="Reviewer summary",
            brief={"flags": []},
        )
    )
    
    # Developer: Construct State with all required fields
    state = State(
        # Add all required State fields here
        flags=[],
        seen_flags=set()
    )
    
    result = run_orchestrator(state)
    
    # Assert _decide WAS called
    mock_decide.assert_called_once()
```

#### Test 3: Unresolved Flag Not in seen_flags → Model Called

```python
def test_unseen_flag_calls_model(fake_gateway, mocker):
    """
    Given forced_route returns ["analyst"] (forced)
    And an unresolved flag exists with ID not in seen_flags
    When orchestrator decides next hop
    Then model is called (to merge/address the flag)
    
    Verifies AC3.4, AC4.2
    
    Developer: Replace placeholder paths and construct Flag instances per actual schema.
    """
    # Developer: Replace with actual paths
    mock_forced_route = mocker.patch(
        'path.to.forced_route',
        return_value=["analyst"]
    )
    
    mock_decide = mocker.patch(
        'path.to._decide',
        return_value=StateUpdate(
            route=["analyst"],
            brief={"flags": [{"id": "flag-123"}]},
        )
    )
    
    # Developer: Construct State and Flag instances per actual schema
    state = State(
        # Add all required State fields here
        flags=[Flag(id="flag-123", resolved=False)],
        seen_flags=set(),
    )
    
    result = run_orchestrator(state)
    
    # Assert _decide WAS called (flag requires model attention)
    mock_decide.assert_called_once()
```

#### Test 4: Forced Hop Step Structure Matches Expectations

```python
def test_forced_hop_step_structure(fake_gateway, mocker):
    """
    Given a forced hop is recorded
    When the step is written
    Then title, result, evidence, status/priority/confidence match expected format
    And summary is unchanged
    
    Verifies AC3.2, AC4.3
    
    Developer: Replace placeholder paths and construct State with required fields.
    """
    # Developer: Replace with actual path
    mocker.patch(
        'path.to.forced_route',
        return_value=["planner"]
    )
    
    # Developer: Construct State with all required fields
    state = State(
        # Add all required State fields here
        flags=[],
        seen_flags=set(),
        summary="Previous summary",
        status="running",
        priority=5,
        confidence=0.8,
    )
    
    result = run_orchestrator(state)
    
    # Assert step structure
    assert result.step.title == "Next: ['planner']"
    assert result.step.result == "forced: only one valid route"
    assert result.step.evidence == {"attempts": 0}
    
    # Assert status/priority/confidence updated (implementation-specific values)
    # Developer: Adjust expected values after discovering what 
    # _compute_status_for_route, _compute_priority_for_route, 
    # _compute_confidence_for_route actually return
    assert result.status in ["running", "waiting", "complete"]
    assert isinstance(result.priority, int)
    assert 0 <= result.confidence <= 1
    
    # Assert summary unchanged
    assert result.summary == "Previous summary"
```

#### Test 5: seen_flags Updated After Model Call

```python
def test_seen_flags_updated_after_model_call(fake_gateway, mocker):
    """
    Given model call includes flags in brief
    When orchestrator extracts flag IDs
    Then seen_flags is updated with those IDs
    
    Verifies AC3.5
    
    Developer: Replace placeholder paths and construct State with required fields.
    """
    # Developer: Replace with actual paths
    mocker.patch(
        'path.to.forced_route',
        return_value=None  # Multiple routes, model called
    )
    
    mock_decide = mocker.patch(
        'path.to._decide',
        return_value=StateUpdate(
            route=["analyst"],
            brief={"flags": [
                {"id": "flag-123"},
                {"id": "flag-456"},
            ]},
        )
    )
    
    # Developer: Construct State with all required fields
    state = State(
        # Add all required State fields here
        flags=[],
        seen_flags=set()
    )
    
    result = run_orchestrator(state)
    
    # Assert seen_flags delta contains flag IDs from brief
    assert result.seen_flags_delta == {"flag-123", "flag-456"}
```

#### Test 6: Forced Hop and Model Hop Produce Same Status Updates from Different Initial States

```python
def test_forced_hop_status_matches_model_hop_different_states(fake_gateway, mocker):
    """
    Given two different initial states (A and B)
    When forced hop is taken with each state
    Then status/priority/confidence differ between the two forced hops
    (proving the helpers are actually computing, not hardcoded)
    
    And when model hop is taken with state A
    Then status/priority/confidence match forced hop A's values
    (proving forced and model hops use the same computation logic)
    
    Verifies AC3.2 (forced hop updates use same helper functions as model hop)
    
    CRITICAL: This test avoids circular mocking by comparing outcomes from 
    different initial states. It verifies that:
    1. The helpers produce different results for different inputs (not hardcoded)
    2. Forced hops and model hops compute the same values for the same input
    
    Developer: After locating _compute_status_for_route, _compute_priority_for_route, 
    _compute_confidence_for_route, replace the mock_decide_impl placeholder calls 
    with actual invocations of those functions.
    """
    # State A: initial conditions where status/priority/confidence compute to one set
    state_a = State(
        # Developer: Add all required State fields here
        flags=[],
        seen_flags=set(),
        status="running",
        priority=5,
        confidence=0.8,
    )
    
    # State B: different conditions where helpers should produce different values
    # Developer: Adjust these values to create a state that actually differs in 
    # ways that affect status/priority/confidence computation
    state_b = State(
        # Developer: Add all required State fields here
        flags=[Flag(id="flag-999", resolved=False)],  # Different flag state
        seen_flags=set(),
        status="paused",  # Different status
        priority=2,  # Different priority
        confidence=0.3,  # Different confidence
    )
    
    # Path 1: Forced hop with state A
    # Developer: Replace with actual path
    mocker.patch(
        'path.to.forced_route',
        return_value=["planner"]
    )
    forced_result_a = run_orchestrator(state_a)
    
    # Path 2: Forced hop with state B
    forced_result_b = run_orchestrator(state_b)
    
    # Assert different initial states produce different outcomes
    # (proves the helpers are actually computing based on state, not hardcoded)
    # Developer: If this assertion fails, adjust state_b to create a meaningful difference
    assert (
        forced_result_a.status != forced_result_b.status or 
        forced_result_a.priority != forced_result_b.priority or 
        forced_result_a.confidence != forced_result_b.confidence
    ), "Forced hops with different initial states should produce different status/priority/confidence"
    
    # Path 3: Model hop with state A (for same route as forced hop A)
    # Reset mocks
    mocker.patch(
        'path.to.forced_route',
        return_value=None  # Force model call
    )
    
    # Mock _decide to call the same _compute_X_for_route functions
    # Developer: After discovering the actual helper functions, replace the 
    # placeholder return values with calls to those functions
    def mock_decide_impl(state):
        route = ["planner"]
        return StateUpdate(
            route=route,
            # Developer: Replace these placeholders with actual function calls:
            # status=_compute_status_for_route(route, state),
            # priority=_compute_priority_for_route(route, state),
            # confidence=_compute_confidence_for_route(route, state),
            status="placeholder_status",  # Replace after discovering function
            priority=999,  # Replace after discovering function
            confidence=0.999,  # Replace after discovering function
            summary="Model summary",  # Different from forced (forced leaves unchanged)
            brief={"flags": []},
        )
    
    # Developer: Replace with actual path
    mocker.patch(
        'path.to._decide',
        side_effect=mock_decide_impl
    )
    
    model_result_a = run_orchestrator(state_a)
    
    # Assert forced hop A and model hop A produce same status/priority/confidence
    # Developer: After replacing the mock_decide_impl placeholders with actual 
    # function calls, this assertion will verify both code paths compute the same values
    assert forced_result_a.status == model_result_a.status, \
        "Forced hop and model hop should compute same status for same initial state"
    assert forced_result_a.priority == model_result_a.priority, \
        "Forced hop and model hop should compute same priority for same initial state"
    assert forced_result_a.confidence == model_result_a.confidence, \
        "Forced hop and model hop should compute same confidence for same initial state"
    
    # Assert summary behavior differs as expected (forced unchanged, model changed)
    assert forced_result_a.summary == state_a.summary, \
        "Forced hop should leave summary unchanged"
    assert model_result_a.summary == "Model summary", \
        "Model hop should update summary"
```

---

### Test File 2: Usage Tracking Integration Tests (path TBD)

4 tests matching the breakdown.md verification count exactly.

#### Test 1: Usage Accumulates Across Multiple Calls

```python
def test_usage_accumulates_across_calls(fake_gateway):
    """
    Given multiple model calls occur for different nodes
    When run completes
    Then state.usage correctly sums counts per node
    
    Verifies AC4.4
    
    Developer: Construct State and StateUpdate instances per actual schema.
    """
    # Developer: Construct State with all required fields
    state = State(
        # Add all required State fields here
        usage={}
    )
    
    # Node 1: two calls
    update1 = StateUpdate(usage={
        "orchestrator": UsageStats(calls=1, input_tokens=100, output_tokens=50, cache_creation_input_tokens=10, cache_read_input_tokens=5)
    })
    state = apply_update(state, update1)
    
    update2 = StateUpdate(usage={
        "orchestrator": UsageStats(calls=1, input_tokens=200, output_tokens=75, cache_creation_input_tokens=20, cache_read_input_tokens=10)
    })
    state = apply_update(state, update2)
    
    # Node 2: one call
    update3 = StateUpdate(usage={
        "planner": UsageStats(calls=1, input_tokens=150, output_tokens=60, cache_creation_input_tokens=15, cache_read_input_tokens=7)
    })
    state = apply_update(state, update3)
    
    # Assert totals per node
    assert state.usage["orchestrator"].calls == 2
    assert state.usage["orchestrator"].input_tokens == 300
    assert state.usage["planner"].calls == 1
    assert state.usage["planner"].input_tokens == 150
```

#### Test 2: Run with No usage/seen_flags Fields Loads and Resumes

```python
def test_run_without_usage_seen_flags_resumes(fake_gateway):
    """
    Given a run was saved before usage/seen_flags tracking existed
    When the run is loaded and resumed
    Then no error occurs and both fields initialize to empty
    
    Verifies AC4.4 (state migration)
    
    CRITICAL: Developer must identify all required State schema fields before 
    writing this test. The old_state_data dict must include those fields to 
    avoid schema validation errors before the loader's migration check runs.
    
    Without all required fields, the test will fail with schema validation 
    error instead of testing the migration logic.
    
    Developer: After scanning State schema, replace the placeholder fields 
    below with the actual required fields.
    """
    # Developer: Replace with ALL required State fields from schema scan
    old_state_data = {
        "run_id": "test-123",
        "status": "paused",
        # Developer: Add ALL other required State fields here 
        # (e.g. flags, plan, thread_id, values, etc.)
        # Do NOT include 'usage' or 'seen_flags' — those are being tested
    }
    
    state = load_state(old_state_data)
    
    # Assert state loaded successfully
    assert state.run_id == "test-123"
    assert state.usage == {}
    assert state.seen_flags == set()
    
    # Resume execution (no error)
    result = run_orchestrator(state)
    assert result is not None
```

#### Test 3: run_query Logs Totals at Run End

```python
def test_run_query_logs_usage_totals(fake_gateway, caplog):
    """
    Given a run completes
    When run_query finishes
    Then total usage is logged at INFO level
    
    Verifies AC4.4
    
    Developer: Construct State with all required fields and verify the 
    condition that triggers logging in run_query before implementing.
    """
    # Developer: Construct State with all required fields
