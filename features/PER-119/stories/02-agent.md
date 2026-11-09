# Story 02-agent LLD

**Story:** Forced routing function in route_rules.py

**Repo:** `app`

**Epic:** PER-119

---

## Acceptance Criteria

From requirements and breakdown only:

1. **AC-1 (REQ-1 AC-1.1):** `forced_route(state)` returns the route as `list[str]` when exactly one candidate route passes `broken_rules`
2. **AC-2 (REQ-1 AC-1.2):** `forced_route(state)` returns `None` when two or more candidate routes pass `broken_rules`
3. **AC-3 (REQ-1 AC-1.3):** Candidates are: each node in `NODES` on its own, plus every non-empty set of stage agents
4. **AC-4 (REQ-1 AC-1.4):** `broken_rules` is the only source of routing rules (no duplication of rule logic in `forced_route`)

---

## HLD Binding Outcomes

From `hld-per-119.md`:

- **AD-4:** `forced_route(state) -> list[str] | None` enumerates candidate routes (each node in `NODES` on its own, plus every non-empty set of stage agents), calls `broken_rules(state, route)` on each, collects routes where `broken_rules` returns an empty list (valid routes), and returns the route if exactly one is valid, else `None`. Stage agents are identified by examining the orchestrator's existing route-building code (the code that constructs multi-agent routes like `["agent-analyst", "agent-writer"]`); the set of stage agents is the set of nodes that appear in such routes. `forced_route` derives this from the orchestrator's route space, not hard-coded names. `forced_route` does not implement routing rules; it only enumerates candidates and calls `broken_rules`. `broken_rules` is the single source of truth for routing logic.

---

## File Plan

**Note:** Repo tree for `app` is not provided. The story specifies `src/agents/nodes/orchestrator/route_rules.py` as the file location; implementation must verify the actual path in the repo and use it.

### Modified Files

#### `src/agents/nodes/orchestrator/route_rules.py`

**Changes:**

1. **Identify stage agents from existing orchestrator route-building code:**
   - Examine the orchestrator's route-building logic (likely in `node.py` or `route_rules.py`) to identify which nodes appear in multi-agent routes
   - The authoritative source is the code that constructs routes like `["agent-analyst", "agent-writer"]`
   - Extract the set of stage agents from this route space (e.g., by reading a constant, examining route construction logic, or parsing existing route patterns)
   - Do NOT hard-code stage agent names in `forced_route`; derive them from the orchestrator's route space

2. **Add `forced_route(state) -> list[str] | None` function:**
   - Enumerate candidate routes:
     - Each node in `NODES` as a single-element list
     - Every non-empty subset of stage agents (power set of stage agents, excluding empty set)
   - For each candidate route, call `broken_rules(state, candidate)`
   - Collect valid routes (where `broken_rules` returns empty list `[]`)
   - Return the route if exactly one valid route exists
   - Return `None` if zero or multiple valid routes exist

**Algorithm notes:**

- **Stage agent identification (implementation must locate this in existing code):**
  
  The orchestrator's route-building code already knows which nodes are stage agents because it constructs multi-agent routes for them. Common patterns to look for:
  
  - **Pattern 1: Constant or list defining stage agents:**
    ```python
    # Existing code (likely in node.py or route_rules.py)
    STAGE_AGENTS = ["agent-analyst", "agent-writer", "agent-planner"]
    ```
    If such a constant exists, use it directly in `forced_route`.
  
  - **Pattern 2: Route construction logic that builds multi-agent routes:**
    ```python
    # Existing code (example from orchestrator route-building)
    if condition:
        return ["agent-analyst", "agent-writer"]
    elif other_condition:
        return ["agent-planner"]
    ```
    Extract the unique nodes that appear in multi-agent routes (routes with more than one element). Collect them into a set `stage_agents`.
  
  - **Pattern 3: Node metadata (e.g., node type or category):**
    ```python
    # Existing code (example node registry)
    NODES = {
        "planner": {"type": "singleton"},
        "agent-analyst": {"type": "stage_agent"},
        "agent-writer": {"type": "stage_agent"},
        "orchestrator": {"type": "singleton"},
        "review": {"type": "singleton"}
    }
    ```
    Filter `NODES` for entries where `type == "stage_agent"`.
  
  **Implementation guidance:** The developer must examine the orchestrator's existing route-building code to identify the mechanism. If no mechanism exists (i.e., stage agents are not distinguished from other nodes), candidate enumeration simplifies to individual nodes only (each node in `NODES` on its own, no multi-agent sets).

- **Candidate enumeration:**
  ```python
  from itertools import combinations
  
  def forced_route(state) -> list[str] | None:
      """
      Returns the single valid route if exactly one candidate passes validation,
      otherwise None.
      
      Enumerates candidate routes (individual nodes + stage agent sets), delegates
      validation to broken_rules, and returns the route if exactly one is valid.
      
      Args:
          state: Current run state
          
      Returns:
          list[str]: The single valid route (e.g., ["planner"], ["__end__"])
          None: When zero or multiple candidates are valid
      """
      candidates = []
      
      # Each node individually
      for node in NODES:
          candidates.append([node])
      
      # Identify stage agents from existing orchestrator route-building code
      # (see implementation guidance above)
      # Example (Pattern 1): stage_agents = STAGE_AGENTS
      # Example (Pattern 2): stage_agents = extract_stage_agents_from_route_logic()
      # Example (Pattern 3): stage_agents = [n for n, meta in NODES.items() if meta["type"] == "stage_agent"]
      
      # For this algorithm, assume stage_agents is derived per guidance above
      stage_agents = derive_stage_agents()  # Placeholder: developer must implement
      
      # Every non-empty set of stage agents (power set excluding empty set)
      if stage_agents:
          for r in range(1, len(stage_agents) + 1):
              for combo in combinations(stage_agents, r):
                  candidates.append(list(combo))
      
      # Validate each candidate by delegating to broken_rules
      valid_routes = []
      for candidate in candidates:
          if not broken_rules(state, candidate):  # empty list means valid
              valid_routes.append(candidate)
      
      # Return route if exactly one is valid, else None
      if len(valid_routes) == 1:
          return valid_routes[0]
      else:
          return None
  
  def derive_stage_agents():
      """
      Derives the set of stage agents from the orchestrator's existing route-building code.
      
      Implementation depends on the codebase's pattern (see implementation guidance above).
      This is a placeholder; the developer must implement this by examining existing code.
      
      Returns:
          list[str]: Names of stage agents (nodes that appear in multi-agent routes)
      """
      # Pattern 1 example:
      # return STAGE_AGENTS
      
      # Pattern 2 example (extract from route logic):
      # all_multi_agent_routes = [["agent-analyst", "agent-writer"], ["agent-planner"]]
      # return list({node for route in all_multi_agent_routes for node in route})
      
      # Pattern 3 example (filter NODES by metadata):
      # return [n for n, meta in NODES.items() if meta.get("type") == "stage_agent"]
      
      # If no stage agents exist (or cannot be identified), return empty list
      # In that case, candidate enumeration simplifies to individual nodes only
      pass  # Developer must implement
  ```

- **Delegation to `broken_rules`:** No routing logic (e.g., checking planner state, review state, execution state) lives in `forced_route`. All validation is delegated to `broken_rules(state, route)`. `forced_route` only enumerates and filters.

- **Performance note:** Generating all non-empty subsets of stage agents (power set minus empty set) has complexity 2^N - 1 for N stage agents. For typical orchestrator graphs with 3-5 stage agents, this is 7-31 candidates per type (individual nodes + stage agent sets). If performance becomes a concern (e.g., >10 stage agents), enumeration logic may need optimization (limit subset size, cache results). However, the story and HLD do not specify such optimizations, so the straightforward power set approach is correct.

---

## Interfaces

### Function Signature

```python
def forced_route(state) -> list[str] | None:
    """
    Returns the single valid route if exactly one candidate passes validation,
    otherwise None.
    
    Enumerates candidate routes (individual nodes + stage agent sets), delegates
    validation to broken_rules, and returns the route if exactly one is valid.
    
    Args:
        state: Current run state
        
    Returns:
        list[str]: The single valid route (e.g., ["planner"], ["__end__"])
        None: When zero or multiple candidates are valid
    """
```

### Inputs

**`state`:**
- Current run state (type depends on existing state schema)
- Contains flags, planner result, agent results, review status, execution status, etc.
- Same state object passed to `broken_rules`

### Outputs

**When exactly one candidate is valid:**
- `list[str]` — the route (e.g., `["planner"]`, `["__end__"]`, `["output-validator"]`, `["agent-analyst", "agent-writer"]`)

**When zero or multiple candidates are valid:**
- `None`

### Dependencies

**`broken_rules(state, route) -> list[str]`:**
- Existing function in `route_rules.py`
- Returns empty list `[]` if route is valid
- Returns list of reason strings if route is invalid
- `forced_route` calls this for each candidate route

**`NODES`:**
- Existing constant or collection in the codebase listing all node names
- `forced_route` iterates over it to enumerate individual-node candidates

**Stage agent identification mechanism:**
- Derived from the orchestrator's existing route-building code (see implementation guidance in Algorithm notes)
- May be a `STAGE_AGENTS` constant, route construction patterns, or node metadata
- Developer must examine existing code to identify the mechanism
- If no mechanism exists, candidate enumeration simplifies to individual nodes only

---

## Tests

From breakdown Verification Approach (5 tests, exact count matching breakdown):

### Test 1: forced_route returns ["planner"] before planner has run

**Mapped from:** Breakdown verification item 1 (REQ-1 AC-1.1)

**Setup:**
- Create a fake state where:
  - Planner has not run (e.g., `state.planner_result` is `None` or missing)
  - No other valid routes exist (e.g., no agents finished, no review complete)
- Mock `broken_rules` to return:
  - `[]` (valid) for `["planner"]`
  - Non-empty list (invalid) for all other candidates

**Expected outcome:**
- `forced_route(state)` returns `["planner"]`

**Verification:**
- AC-1 (REQ-1 AC-1.1): `forced_route` returns the route when exactly one candidate route passes `broken_rules`
- Only one candidate route passes validation
- `forced_route` delegates to `broken_rules` for each candidate (AC-4)

---

### Test 2: forced_route returns ["__end__"] after review

**Mapped from:** Breakdown verification item 2 (REQ-1 AC-1.1)

**Setup:**
- Create a fake state where:
  - Review is complete (e.g., `state.review_result` exists and is approved)
  - No other valid routes exist
- Mock `broken_rules` to return:
  - `[]` (valid) for `["__end__"]`
  - Non-empty list (invalid) for all other candidates

**Expected outcome:**
- `forced_route(state)` returns `["__end__"]`

**Verification:**
- AC-1 (REQ-1 AC-1.1): `forced_route` returns the route when exactly one candidate route passes `broken_rules`
- Only one candidate route passes validation
- `forced_route` delegates to `broken_rules` for each candidate (AC-4)

---

### Test 3: forced_route returns ["output-validator"] once execution finished

**Mapped from:** Breakdown verification item 3 (REQ-1 AC-1.1)

**Setup:**
- Create a fake state where:
  - Execution has finished (e.g., all agents complete, tests run)
  - Review has not started
  - No other valid routes exist
- Mock `broken_rules` to return:
  - `[]` (valid) for `["output-validator"]`
  - Non-empty list (invalid) for all other candidates

**Expected outcome:**
- `forced_route(state)` returns `["output-validator"]`

**Verification:**
- AC-1 (REQ-1 AC-1.1): `forced_route` returns the route when exactly one candidate route passes `broken_rules`
- Only one candidate route passes validation
- `forced_route` delegates to `broken_rules` for each candidate (AC-4)

---

### Test 4: forced_route returns None when two agents are valid

**Mapped from:** Breakdown verification item 4 (REQ-1 AC-1.2)

**Setup:**
- Create a fake state where:
  - Two stage agents are unfinished and have no blockers (e.g., `agent-analyst` and `agent-writer`)
  - Both routes `["agent-analyst"]` and `["agent-writer"]` are valid
- Mock `broken_rules` to return:
  - `[]` (valid) for `["agent-analyst"]`
  - `[]` (valid) for `["agent-writer"]`
  - Non-empty list (invalid) for all other candidates

**Expected outcome:**
- `forced_route(state)` returns `None`

**Verification:**
- AC-2 (REQ-1 AC-1.2): `forced_route` returns `None` when two or more candidate routes pass `broken_rules`
- Multiple candidate routes pass validation (two or more)
- `forced_route` returns `None` instead of choosing one

---

### Test 5: forced_route delegates validation to broken_rules for every candidate

**Mapped from:** Breakdown verification item 5 (REQ-1 AC-1.3, AC-1.4)

**Setup:**
- Create a fake state (any valid state)
- Mock `broken_rules` to track all calls (record each `(state, route)` invocation)
- Mock `broken_rules` to return `[]` for one candidate (e.g., `["planner"]`) and non-empty list for all others

**Expected outcome:**
- `forced_route(state)` calls `broken_rules(state, candidate)` for every enumerated candidate
- No routing logic (e.g., checking `state.planner_result`) exists in `forced_route` itself

**Verification:**
- AC-4 (REQ-1 AC-1.4): `broken_rules` is the only source of routing rules (no duplication of rule logic in `forced_route`)
- AC-3 (REQ-1 AC-1.3): Candidates are: each node in `NODES` on its own, plus every non-empty set of stage agents
- Verify `broken_rules` is called for each candidate (via mock call tracking)
- Verify `forced_route` contains no routing logic (code review: only enumeration and filtering)

---

## Notes

- **No repo tree provided:** Implementation must verify the actual path of `route_rules.py` in the `app` repo. The story specifies `src/agents/nodes/orchestrator/route_rules.py` as the likely location.
- **Stage agent identification is implementation-dependent:** The developer must examine the orchestrator's existing route-building code to identify how stage agents are distinguished from other nodes. HLD AD-4 clarifies that stage agents are derived from the orchestrator's route space (nodes that appear in multi-agent routes), not hard-coded. Common patterns are documented in the Algorithm notes (constants, route construction logic, node metadata). If no mechanism exists, candidate enumeration simplifies to individual nodes only.
- **Candidate enumeration complexity:** Generating all non-empty subsets of stage agents (power set minus empty set) has complexity 2^N - 1 for N stage agents. For typical orchestrator graphs with 3-5 stage agents, this is 7-31 candidates. If performance becomes a concern (e.g., >10 stage agents), the enumeration logic may need optimization (limit subset size, cache results). However, the story and HLD do not specify such optimizations, so the straightforward power set approach is correct.
- **No live runs in tests:** All tests use fake states and mock `broken_rules`. No live model calls, no live runs.
- **Existing tests unchanged:** After this story, `uv run pytest -q` must pass, and existing tests must pass unchanged.
- **No rule duplication:** `forced_route` must not implement any routing logic directly. All validation is delegated to `broken_rules`. Code review must verify this (see Test 5).
- **Test count matches breakdown:** 5 tests exactly as specified in breakdown Verification Approach for story 02-agent.
- **Alignment with HLD AD-4:** The algorithm and implementation guidance fully incorporate the HLD AD-4 clarification that stage agents are identified by examining the orchestrator's route-building code, not hard-coded. The `derive_stage_agents()` placeholder function documents the three common patterns (constant, route logic, metadata) and emphasizes that the developer must locate and use the mechanism already present in the codebase.
