# Story 02-agent LLD

**Ticket:** PER-128-2  
**Epic:** PER-128  
**Repo:** agent  
**Title:** Implement forced routing logic in route_rules.py

---

## Acceptance Criteria

From `requirements.md` AC2 and AC4 (this story's scope):

1. **AC2.1:** `forced_route(state)` returns a single route when exactly one candidate passes `broken_rules`, `None` otherwise.
2. **AC2.2:** Before planner has run, `forced_route` returns `["planner"]`.
3. **AC2.3:** After a review hop, `forced_route` returns `["__end__"]`.
4. **AC2.4:** Once execution has finished, `forced_route` returns `["output-validator"]`.
5. **AC2.5:** When two stage agents are both unfinished with nothing waiting, `forced_route` returns `None`.
6. **AC2.6:** `broken_rules` is the sole source of routing rules; `forced_route` does not duplicate any rule logic.
7. **AC4.1:** New tests verify `forced_route` behavior per AC2 scenarios.
8. **AC4.2:** All existing `route_rules` tests pass unchanged.

---

## HLD Outcomes (Binding Constraints)

From `hld.md`:

- **AD-2 (forced_route contract):** Returns `list[str]` (single route) when exactly one candidate passes `broken_rules`, else `None`. Candidates: each node in `NODES` individually + every non-empty combination of stage agents. `broken_rules` remains the **sole** source of routing rules (no duplication).

---

## File Plan

**CRITICAL:** Repo tree not mapped. Developer MUST locate actual file paths by scanning the agent codebase before implementation. All paths below are placeholders that will fail if used directly.

### Files to Create

1. **Test file for `forced_route` (path TBD from codebase scan)**
   - Purpose: Test suite for `forced_route` behavior
   - Content: 7 tests per Verification Approach below
   - Placeholder name: `tests/test_forced_route.py`

### Files to Modify

2. **Route rules module (path TBD from codebase scan)**
   - Add `forced_route(state) -> list[str] | None` function
   - Import `NODES` constant (developer must locate; see Open Questions)
   - Import stage agent list (developer must locate; see Open Questions)
   - Generate candidates: individual nodes + non-empty agent combinations
   - Test each candidate via `broken_rules`, count passing routes
   - Return single route if count == 1, else `None`
   - Placeholder path: `src/agents/nodes/orchestrator/route_rules.py`

---

## Interfaces and Data Structures

### forced_route API

```python
# Route rules module (path TBD from codebase scan)

def forced_route(state: State) -> list[str] | None:
    """
    Returns a single route if exactly one candidate passes broken_rules.
    
    Candidates are generated from:
    - Each node in NODES as a single-element list (e.g. ["planner"])
    - Every non-empty combination of stage agents (e.g. ["analyst"], ["architect"], 
      ["analyst", "architect"])
    
    Args:
        state: Current execution state, passed through to broken_rules for validation
    
    Returns:
        list[str]: A single valid route when exactly one candidate passes broken_rules
        None: When zero candidates pass OR multiple candidates pass
    
    Contract:
        - broken_rules is the SOLE source of routing validity logic
        - This function does NOT duplicate any routing rules
        - All validation delegates to broken_rules(state, candidate)
    """
```

### Candidate Generation Logic

```python
def _generate_candidates(nodes: list[str], stage_agents: list[str]) -> list[list[str]]:
    """
    Generate all candidate routes to test.
    
    Returns:
        - Each node in NODES as a single-element list
        - All non-empty subsets of stage_agents (using itertools.combinations)
    
    Example output:
        [
            ["planner"],
            ["orchestrator"],
            ["output-validator"],
            ["analyst"],
            ["architect"],
            ["analyst", "architect"],
            # ... other agent combinations
        ]
    """
```

### broken_rules Contract (existing, read-only)

```python
def broken_rules(state: State, route: list[str]) -> list[str]:
    """
    Existing function — returns list of broken rules for given route.
    
    Empty list = route is valid
    Non-empty list = route violates rules (invalid)
    
    forced_route delegates ALL routing logic to this function.
    """
```

---

## Algorithm Notes

### 1. Candidate Generation Algorithm

```python
from itertools import combinations

def _generate_candidates(nodes: list[str], stage_agents: list[str]) -> list[list[str]]:
    """
    Generate candidate routes:
    1. Each node individually
    2. All non-empty combinations of stage agents
    """
    candidates = []
    
    # Individual nodes
    for node in nodes:
        candidates.append([node])
    
    # Non-empty combinations of stage agents (size 1 to len(stage_agents))
    for size in range(1, len(stage_agents) + 1):
        for combo in combinations(stage_agents, size):
            candidates.append(list(combo))
    
    return candidates
```

**Rationale:**
- Individual nodes cover system transitions (e.g. `["planner"]`, `["output-validator"]`)
- Agent combinations cover parallel execution phases (e.g. `["analyst", "architect"]`)
- Exhaustive generation ensures we catch forced routes in all phases

**Performance note:**
- With N stage agents, generates O(2^N) agent combinations
- Typical case: 3-5 agents → 7-31 combinations (acceptable)
- If agent count grows large, consider deferred optimization (per HLD)

### 2. forced_route Core Logic

```python
def forced_route(state: State) -> list[str] | None:
    """
    Implementation:
    
    1. Generate all candidates from discovered NODES and STAGE_AGENTS
    2. For each candidate:
       - Call broken_rules(state, candidate)
       - Count as valid if result is empty list
    3. If exactly one valid candidate:
       - Return that route
    4. Else:
       - Return None (zero or multiple valid routes)
    """
    candidates = _generate_candidates(NODES, STAGE_AGENTS)
    
    valid_routes = []
    for candidate in candidates:
        # Delegate validity check to broken_rules (no duplication)
        rules_broken = broken_rules(state, candidate)
        if not rules_broken:  # Empty list = valid route
            valid_routes.append(candidate)
    
    # Return single route if exactly one is valid
    if len(valid_routes) == 1:
        return valid_routes[0]
    else:
        return None  # Zero or multiple valid routes
```

**Contract enforcement:**
- All routing logic lives in `broken_rules` (read-only)
- `forced_route` only counts and returns; no rule duplication

### 3. NODES and STAGE_AGENTS Discovery

**Developer must locate these before implementing `forced_route`:**

```python
# Expected patterns (actual location TBD from codebase scan):

# Pattern 1: Constants in route_rules module or orchestrator node module
NODES = ["planner", "orchestrator", "output-validator", ...]
STAGE_AGENTS = ["analyst", "architect", "designer", ...]

# Pattern 2: Derived from graph structure
# NODES = list(graph.nodes.keys())
# STAGE_AGENTS = [n for n in NODES if is_stage_agent(n)]

# Pattern 3: Registered dynamically
# NODES = NodeRegistry.get_all()
# STAGE_AGENTS = NodeRegistry.get_by_type("stage")
```

**Implementation approach:**
1. Scan the codebase for `NODES` constant, node registry, or graph definition
2. Identify stage agents by convention or type (nodes that execute work vs. orchestration nodes)
3. If not centralized, document the discovered list in a module-level comment and import it
4. **DO NOT hardcode placeholder values** — tests would pass for wrong reasons

**If discovery fails:**
- Flag for architect review before implementing `forced_route`
- Document the assumed node set and stage agent list
- Mark as blocking until resolved

---

## Tests

### Test File: `tests/test_forced_route.py` (path TBD)

7 tests matching the Verification Approach exactly. **Developer must construct realistic State instances by examining the actual State schema before writing these tests.**

#### Test 1: Pre-planner State Returns ["planner"]

```python
def test_forced_route_pre_planner():
    """
    Given state before planner has run
    When forced_route(state) is called
    Then returns ["planner"]
    
    Developer: Examine State schema to determine which field(s) indicate 
    pre-planner state (e.g. state.plan is None, state.planner_complete is False).
    """
    state = State(
        # Developer: Add all required State fields here
        # Include field(s) that indicate planner has not run yet
    )
    
    route = forced_route(state)
    
    assert route == ["planner"]
```

#### Test 2: Post-review State Returns ["__end__"]

```python
def test_forced_route_post_review():
    """
    Given state after review hop
    When forced_route(state) is called
    Then returns ["__end__"]
    
    Developer: Examine State schema to determine which field(s) indicate 
    post-review state (e.g. state.review_complete is True).
    """
    state = State(
        # Developer: Add all required State fields here
        # Include field(s) that indicate review is complete
    )
    
    route = forced_route(state)
    
    assert route == ["__end__"]
```

#### Test 3: Post-execution State Returns ["output-validator"]

```python
def test_forced_route_post_execution():
    """
    Given execution finished
    When forced_route(state) is called
    Then returns ["output-validator"]
    
    Developer: Examine State schema to determine which field(s) indicate 
    execution is complete (e.g. state.execution_status == "complete").
    """
    state = State(
        # Developer: Add all required State fields here
        # Include field(s) that indicate execution finished
    )
    
    route = forced_route(state)
    
    assert route == ["output-validator"]
```

#### Test 4: Multiple Valid Routes Returns None

```python
def test_forced_route_multiple_valid():
    """
    Given two stage agents unfinished, nothing waiting
    When forced_route(state) is called
    Then returns None (multiple valid routes)
    
    Developer: Construct state where both agents could proceed 
    (e.g. both have work to do, no blocking flags).
    """
    state = State(
        # Developer: Add all required State fields here
        # Set up state where multiple routes are valid
        # (e.g. analyst_done=False, architect_done=False, flags=[])
    )
    
    route = forced_route(state)
    
    assert route is None
```

#### Test 5: broken_rules Called for Each Candidate, No Other Logic

```python
def test_forced_route_delegates_to_broken_rules(mocker):
    """
    Given forced_route is called
    When checking candidate validity
    Then broken_rules is called for each candidate
    And no routing logic exists outside those calls
    
    Developer: After discovering NODES and STAGE_AGENTS, calculate the 
    expected candidate count: len(NODES) + (2^len(STAGE_AGENTS) - 1).
    This test verifies forced_route generates that count.
    """
    # Developer: Determine the actual path to broken_rules from codebase scan
    mock_broken_rules = mocker.patch(
        'path.to.broken_rules',  # Developer: replace with actual path
        return_value=[]  # All candidates pass
    )
    
    # Developer: Construct a minimal valid State instance
    state = State(
        # Developer: Add all required State fields here
    )
    
    forced_route(state)
    
    # Developer: Replace expected_candidate_count with the actual count
    # after discovering NODES and STAGE_AGENTS
    expected_candidate_count = 999  # Placeholder — replace with real count
    assert mock_broken_rules.call_count == expected_candidate_count
    
    # Assert all calls passed the same state
    for call in mock_broken_rules.call_args_list:
        assert call[0][0] == state  # First arg is state
```

#### Test 6: Single Valid Route Among Many Candidates

```python
def test_forced_route_single_valid(mocker):
    """
    Given state where only one candidate passes broken_rules
    When forced_route(state) is called
    Then returns that single valid route
    
    Developer: Construct state where only one route is valid, then mock 
    broken_rules to enforce that (return [] for the valid route, 
    non-empty list for all others).
    """
    # Developer: Construct state where only one route should be valid
    # (e.g. only analyst has work, all others are done or blocked)
    state = State(
        # Developer: Add all required State fields here
    )
    
    # Mock broken_rules to reject all except one candidate
    def mock_broken_rules_impl(state, route):
        if route == ["analyst"]:  # Developer: adjust to match expected valid route
            return []  # Valid
        else:
            return ["some_rule"]  # Invalid
    
    # Developer: Determine actual path to broken_rules
    mocker.patch(
        'path.to.broken_rules',  # Developer: replace with actual path
        side_effect=mock_broken_rules_impl
    )
    
    route = forced_route(state)
    
    # Developer: Replace ["analyst"] with the expected valid route
    assert route == ["analyst"]
```

#### Test 7: Zero Valid Routes Returns None

```python
def test_forced_route_zero_valid(mocker):
    """
    Given state where no candidate passes broken_rules
    When forced_route(state) is called
    Then returns None
    
    Developer: Construct an impossible/broken state, or mock broken_rules 
    to reject all candidates.
    """
    # Developer: Construct a state that should have no valid routes
    state = State(
        # Developer: Add all required State fields here
    )
    
    # Mock broken_rules to reject all candidates
    # Developer: Determine actual path to broken_rules
    mocker.patch(
        'path.to.broken_rules',  # Developer: replace with actual path
        return_value=["blocking_rule"]  # All invalid
    )
    
    route = forced_route(state)
    
    assert route is None
```

### Existing Tests

All existing tests covering `route_rules` behavior must pass unchanged. Key areas:

- `broken_rules` behavior for various states
- Individual route validation logic
- Graph traversal rules

**Regression risk mitigation:**
- `forced_route` is new; does not modify `broken_rules` or other existing functions
- Only reads `NODES` and stage agent registry (no writes)
- Test isolation via mocks ensures no side effects

---

## Out of Scope

- State schema changes (`State.usage`, `State.seen_flags` — that's PER-128-1 and PER-128-3)
- Orchestrator integration (`_decide` skip logic — that's PER-128-3)
- Runtime usage aggregation (`run_query` logging — that's PER-128-3)
- API response extension (`GET /runs/{thread_id}` — that's PER-128-3)
- Changing models, `policy.yaml`, or prompt content
- Modifying `broken_rules` behavior or routing rules themselves

---

## Open Questions

**BLOCKING — Must resolve before implementing `forced_route`:**

1. **Where is `NODES` defined?**  
   - Developer must scan route_rules module, orchestrator node module, graph definition, or node registry
   - If not found, document the discovery failure and flag for architect review
   - **DO NOT hardcode a placeholder list** — tests would pass for wrong reasons

2. **Where is the stage agent list defined?**  
   - Developer must identify which nodes are "stage agents" (nodes that execute work) vs. system nodes (planner, orchestrator, output-validator, flag-review, review)
   - Likely stage agents: analyst, architect, designer, test_architect
   - If not centralized, infer from node types or conventions and document in code

3. **Does `broken_rules` return empty list for valid routes?**  
   - Developer must verify signature and return type before implementing `forced_route`
   - If returns boolean or raises exceptions, adjust `forced_route` logic accordingly

**Non-blocking (affects test quality):**

4. **What State fields indicate planner/review/execution completion?**  
   - Tests 1-3 require these fields to construct valid state snapshots
   - Developer must examine State schema to determine correct fields

---

## Implementation Checklist

- [ ] Scan codebase to locate `NODES` constant or registry (BLOCKING)
- [ ] Identify stage agent list (BLOCKING)
- [ ] Verify `broken_rules` signature and return type (BLOCKING)
- [ ] Examine State schema to identify fields for test state construction
- [ ] Implement `_generate_candidates` helper (individual nodes + agent combinations)
- [ ] Implement `forced_route` core logic (delegate to `broken_rules`, count valid routes)
- [ ] Calculate expected candidate count for Test 5 using discovered NODES/STAGE_AGENTS
- [ ] Write 7 tests with realistic State instances per schema
- [ ] Replace all placeholder paths in tests with actual paths from codebase scan
- [ ] Run existing `route_rules` tests, confirm no regressions
- [ ] Document discovered node lists in code comments if not already centralized

---

**End of Story 02-agent LLD**
