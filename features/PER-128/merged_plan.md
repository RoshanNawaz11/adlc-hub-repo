# Merged plan — PER-128

Execution order (3 ticket(s)):

1. `PER-130` (story `01-agent`)
2. `PER-131` (story `02-agent`)
3. `PER-132` (story `03-agent`)

Specs (3): `specs/PER-128-1.md`, `specs/PER-128-2.md`, `specs/PER-128-3.md`

---

## Spec `specs/PER-128-1.md`

# Tech Stub — PER-128-1

**Title:** Add token usage tracking to State and model gateways  
**Ticket:** PER-128-1  
**Epic:** PER-128  
**Repo:** agent  
**Stack:** Python (inferred from ticket context: `src/agents/` paths, `pytest`, `uv run`)

---

## Overview

This story adds per-run token usage tracking to the agent's state and all model gateway call sites. Every model call records four token counts (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) into a new `State.usage` field, which aggregates per node. The state loader handles runs saved before this change (no `usage` field) by treating the missing field as empty. Tests verify usage accumulation across multiple calls and successful migration of old runs.

---

## Stack

**Note:** Repo tree not provided. File paths below are placeholders derived from ticket context (`src/agents/nodes/orchestrator/node.py`, `src/agents/skills/client.py`, `src/agents/tools/bindings.py`). Developer must locate actual paths in the codebase before implementation.

- **Language:** Python
- **State management:** Existing `State` schema (TypedDict or dataclass) with reducer pattern
- **Model gateway:** Existing `ModelGateway.complete` and `run_skill`
- **MCP tool loop:** Existing implementation in tools/bindings module
- **Testing:** `pytest` (run via `uv run pytest -q`)

---

## Schema / API Contracts

### State extension

```python
# Add to State schema (location TBD from codebase scan)
usage: Dict[str, UsageStats]  # maps node name -> aggregated token counts
```

### UsageStats

```python
from dataclasses import dataclass

@dataclass
class UsageStats:
    calls: int = 0
    input_tokens: int = 0
    output_tokens: int = 0
    cache_creation_input_tokens: int = 0
    cache_read_input_tokens: int = 0
    
    def __add__(self, other: 'UsageStats') -> 'UsageStats':
        """Additive merge for reducer."""
        return UsageStats(
            calls=self.calls + other.calls,
            input_tokens=self.input_tokens + other.input_tokens,
            output_tokens=self.output_tokens + other.output_tokens,
            cache_creation_input_tokens=self.cache_creation_input_tokens + other.cache_creation_input_tokens,
            cache_read_input_tokens=self.cache_read_input_tokens + other.cache_read_input_tokens,
        )
```

**Reducer contract:**  
When a node update includes a `UsageStats` delta, the reducer loops over the delta's nodes and applies `existing[node] = existing.get(node, UsageStats()) + delta[node]`. The `__add__` method handles the field-by-field sum. If the node has no prior entry, it initializes with the delta (adding to a zero-initialized `UsageStats`).

**Migration contract:**  
Runs saved before this change have no `usage` field. The state loader treats missing `usage` as `{}` and execution continues normally.

---

### Model gateway usage reporting

**All model call sites must return usage deltas:**

1. **`run_skill` (skill client module)**  
   After each completion (including JSON retry calls), extract token counts from the API response and return a `UsageStats` delta with `calls: 1` and the four token fields.

2. **MCP tool loop (tool bindings module)**  
   After each tool round (including the final no-tools call), extract token counts and return a `UsageStats` delta.

3. **`ModelGateway.complete`**  
   Direct completions (e.g. for summaries or flag merges) extract token counts and return a `UsageStats` delta.

**API response fields (from Anthropic/Claude):**

```python
{
    "usage": {
        "input_tokens": int,
        "output_tokens": int,
        "cache_creation_input_tokens": int,
        "cache_read_input_tokens": int
    }
}
```

Each call site maps these fields to `UsageStats` and includes the delta in the node's state update payload.

---

## File Touch List

**Note:** All paths are placeholders. Developer must scan the codebase to locate actual file paths before implementation.

### Files to modify

- State schema definition file (e.g. `src/agents/state.py`)  
  Add `usage: Dict[str, UsageStats]` field and implement the additive reducer logic.

- State loader module (e.g. `src/agents/state_loader.py`)  
  Handle missing `usage` field by initializing to `{}` when loading old runs.

- Skill client module (e.g. `src/agents/skills/client.py`)  
  Modify `run_skill` to extract token counts from each completion response (including JSON retries) and return a `UsageStats` delta.

- Tool bindings module (e.g. `src/agents/tools/bindings.py`)  
  Modify the MCP tool loop to extract token counts after each round and return a `UsageStats` delta.

- Model gateway module (e.g. `src/agents/model/gateway.py`)  
  Modify `ModelGateway.complete` to extract token counts and return a `UsageStats` delta.

- Node update logic (e.g. `src/agents/nodes/<each_node>/node.py`)  
  Modify each node's update logic to include the `UsageStats` delta from its model calls in the state update payload.

### Files to create

- `UsageStats` definition (new file or inline in state module)  
  Define the `UsageStats` dataclass with `__add__` method.

- `tests/test_usage_tracking.py`  
  New test file with 5 tests (see Tests section below).

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Missing usage from a call site (e.g. forgot to extract counts in a retry path) | Underreported costs, incorrect savings measurement | Code review checklist: every path calling the model API must extract and return usage. Add integration test asserting usage appears for JSON retries, tool rounds, and direct completions. |
| State migration breaks existing runs | Runs fail to load after deploy, blocking production | Loader explicitly treats missing `usage` as `{}` (HLD AD-1). Test 4 uses a state dict with all required schema fields except `usage` to avoid schema validation errors before the loader's check runs. |
| Reducer logic overwrites instead of adds | Usage resets to last call's counts, losing prior history | Reducer uses `existing.get(node, UsageStats()) + delta[node]` pattern; `__add__` handles field-by-field sum. Test 1 verifies accumulation across two updates. |
| UsageStats fields don't match API response structure | Runtime errors or zero counts | Match field names exactly to Anthropic API. Test 5 mocks an API response and asserts correct extraction. |

---

## Acceptance Criteria (from parent ticket AC1, AC4)

**AC1 scenarios:**

1. Every model call records `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` in `State.usage` under the current node's key.
2. Multiple calls for the same node add counts to existing ones (not replace).
3. A run saved before this change (no `usage` field) loads and resumes without error.

**AC4 scenarios (this story's scope):**

4. New tests with fake gateway verify usage accumulates correctly across calls.
5. Test verifies a run loads without error when `usage` is missing.
6. Existing tests pass unchanged after the changes.

---

## Out of Scope

- Logging or returning usage totals (that's PER-128-3: `run_query` aggregation and API response).
- `seen_flags` field (that's PER-128-3: orchestrator integration).
- Forced routing logic (that's PER-128-2 and PER-128-3).
- Changing models, `policy.yaml`, or prompt content.

---

## Tests

### New tests (in `tests/test_usage_tracking.py`)

**Test count:** 5 (matching breakdown.md Verification Approach exactly)

1. **test_usage_accumulation_same_node**  
   - **Given:** A node makes two model calls  
   - **When:** Usage is recorded for both calls  
   - **Then:** `state.usage[node].calls == 2` and token counts are summed (not replaced)  
   - **Verify:** Assert `calls == 2`, `input_tokens == sum`, `output_tokens == sum`, `cache_creation_input_tokens == sum`, `cache_read_input_tokens == sum`

2. **test_usage_from_json_retry**  
   - **Given:** `run_skill` makes initial call that returns invalid JSON  
   - **When:** It retries and succeeds  
   - **Then:** Both calls contribute to usage (`calls == 2`, tokens summed)  
   - **Verify:** Assert `usage_delta[node].calls == 2` and token counts include both attempts

3. **test_usage_from_mcp_tool_rounds**  
   - **Given:** MCP tool loop makes three rounds (two tool calls, one final no-tools call)  
   - **When:** Usage is recorded per round  
   - **Then:** `state.usage[node].calls == 3` and token counts sum across all rounds  
   - **Verify:** Assert `calls == 3` and token totals match sum of three mocked responses

4. **test_state_migration_no_usage_field**  
   - **Given:** A serialized state from before this change (no `usage` field)  
   - **When:** The state is loaded  
   - **Then:** It loads successfully and `state.usage == {}`  
   - **Verify:** Use a state dict with all required schema fields except `usage` (e.g. `run_id`, `status`, `flags`, `plan`, any other required fields discovered from schema scan). Assert load succeeds and `state.usage == {}`. **Developer must identify required fields from State schema before writing this test.**

5. **test_usage_extraction_from_api_response**  
   - **Given:** A mocked Anthropic API response with all four token fields  
   - **When:** `extract_usage_from_response` (or equivalent extraction logic) is called  
   - **Then:** `UsageStats` is correctly populated  
   - **Verify:** Assert `calls == 1`, `input_tokens == 123`, `output_tokens == 456`, `cache_creation_input_tokens == 78`, `cache_read_input_tokens == 90` (match mocked response values)

### Existing tests

- All existing tests in the agent repo must pass unchanged after adding `usage` (verified via `uv run pytest -q`).

**Regression risk mitigation:**
- `usage` field defaults to `{}` in State schema (no impact on tests that don't inspect it)
- State loader handles missing `usage` gracefully (old test fixtures continue to load)
- Reducer only processes `usage` delta if present in update (non-usage updates unaffected)

---

## Open Questions

**Resolved before implementation:**

1. **What are the required fields in the State schema?** — Developer must scan the State definition (e.g. `src/agents/state.py`) to identify all required fields. Test 4 depends on this (must include all required fields except `usage` to avoid schema validation errors before the loader's migration check runs).

2. **Where do the model gateway modules live?** — Developer must locate `run_skill`, MCP tool loop, and `ModelGateway.complete` in the codebase. Paths in this spec are placeholders derived from ticket context.

3. **How does the state reducer work today?** — Developer must examine the existing reducer pattern (likely in the state module or update application logic) to understand how to add the `usage` additive merge without breaking existing field merges.

---

**End of Tech Stub**

## Spec `specs/PER-128-2.md`

# Tech Stub — PER-128-2

**Ticket:** PER-128-2  
**Title:** Implement forced routing logic in route_rules.py  
**Repo:** agent  
**Status:** Draft  

---

## Stack

Python codebase (repo path not mapped; developer must locate actual file paths before implementation).

---

## Schema & API Contracts

### `forced_route` API

**Module:** route_rules module (exact path TBD from codebase scan)

```python
def forced_route(state: State) -> list[str] | None:
    """
    Returns a single route if exactly one candidate passes broken_rules.
    
    Candidates:
    - Each node in NODES as a single-element list
    - Every non-empty combination of stage agents
    
    Returns:
        list[str]: A single valid route when exactly one candidate passes broken_rules
        None: When zero or multiple candidates pass broken_rules
    
    Rule source: broken_rules is the sole source of routing validity logic.
    This function does NOT duplicate any routing rules.
    """
```

### Inputs

- **`state: State`** — current execution state, passed through to `broken_rules` for each candidate test

### Outputs

- **`list[str]` (single route)** — returned when exactly one candidate route is valid per `broken_rules`
- **`None`** — returned when zero candidates pass OR multiple candidates pass

### Candidate Generation

**Individual nodes:**
```python
# For each node in NODES:
candidate = [node]
```

**Stage agent combinations:**
```python
# All non-empty subsets of stage agents:
# e.g. ["analyst"], ["architect"], ["analyst", "architect"], ...
```

### Contract with `broken_rules`

`forced_route` must:
1. Generate all candidate routes (individual nodes + agent combinations)
2. Call `broken_rules(state, candidate)` for each candidate
3. Count candidates that return empty broken_rules (valid routes)
4. Return the single valid route if count == 1, else None

**No duplication:** `forced_route` MUST NOT implement any routing logic itself. All validity checks delegate to `broken_rules`.

---

## File Touch List

**Note:** All paths below are placeholders. Developer must locate actual files in the agent repo before implementation.

### New or Modified Files

1. **Route rules module (path TBD)**
   - Add `forced_route(state) -> list[str] | None` function
   - Locate and import `NODES` constant (see Open Questions)
   - Locate and import stage agent list (see Open Questions)
   - Candidate generator for individual nodes
   - Candidate generator for non-empty agent combinations
   - Logic to test each candidate via `broken_rules` and count passing routes

2. **Test file for forced_route (new, path TBD)**
   - Test: `forced_route` returns `["planner"]` before planner has run
   - Test: `forced_route` returns `["__end__"]` after review hop
   - Test: `forced_route` returns `["output-validator"]` once execution finished
   - Test: `forced_route` returns `None` when two stage agents are both unfinished
   - Test: `forced_route` uses `broken_rules` as sole source (no rule duplication)
   - Test: single valid route among many candidates
   - Test: zero valid routes
   - Fake gateway, no live services

### Files to Read (for context)

- Route rules module — existing `broken_rules` signature and behavior
- Orchestrator node module — `NODES` constant, stage agent definitions (if present)
- State schema module — `State` definition
- Existing routing tests — patterns for state snapshots and route validation

---

## Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| **Candidate generation is slow with many agents** | High latency at each orchestrator decision point | Medium | Profile with realistic agent counts; if needed, implement caching or topology-based pruning (deferred per HLD). Add performance test asserting forced_route completes in <100ms for typical state. |
| **`NODES` or stage agent list location unknown** | Cannot generate candidates correctly; hardcoding would make tests pass for wrong reasons | High (repo-tree not mapped) | Developer MUST locate actual constants before implementing candidate generation. Do not hardcode placeholder values. If not centralized, document the discovery process and flag for architect review. |
| **`broken_rules` signature or behavior differs from assumption** | `forced_route` contract breaks | Low | Verify `broken_rules(state, route) -> list[str]` returns empty list for valid routes before implementing the counting logic; adjust if it returns boolean or raises exceptions. |
| **Exhaustive agent combinations cause combinatorial explosion** | Memory or time blowup with many agents | Low | Limit combinations to realistic sizes (e.g. max 3 agents per combo) or implement lazy generation. Document assumption in code comments. |
| **Missing edge case in forced route logic** | Wrong route returned, orchestrator skips model when it shouldn't | Medium | Comprehensive test coverage per AC2 scenarios; add integration test with real state snapshots from before-planner, after-review, post-execution phases. |

---

## Dependencies

**Upstream (must exist before this ticket):**
- `broken_rules` function in route rules module (already exists per context)
- `State` schema (already exists)
- `NODES` constant or equivalent node registry (location TBD)
- Stage agent list or registry (location TBD)

**Downstream (blocks or is blocked by):**
- **Blocks PER-128-3** — orchestrator integration requires `forced_route` API to exist
- **No upstream blockers** — this ticket reads existing `broken_rules`, does not modify state or orchestrator

---

## Test Strategy

### Unit Tests (new file, path TBD)

**Test count:** 7 (matching breakdown.md Verification Approach exactly)

1. **Test: pre-planner forced route**
   - Given: state before planner has run (developer must identify the state field indicating planner not run)
   - When: `forced_route(state)` called
   - Then: returns `["planner"]`
   - Verify: `assert route == ["planner"]`

2. **Test: post-review forced route**
   - Given: state after review hop (developer must identify the state field indicating review complete)
   - When: `forced_route(state)` called
   - Then: returns `["__end__"]`
   - Verify: `assert route == ["__end__"]`

3. **Test: post-execution forced route**
   - Given: execution finished (developer must identify the state field indicating execution complete)
   - When: `forced_route(state)` called
   - Then: returns `["output-validator"]`
   - Verify: `assert route == ["output-validator"]`

4. **Test: multiple valid routes**
   - Given: two stage agents unfinished, nothing waiting (developer must construct state where both agents are valid next hops)
   - When: `forced_route(state)` called
   - Then: returns `None`
   - Verify: `assert route is None`

5. **Test: no duplication of broken_rules logic**
   - Given: mock `broken_rules` spy tracking calls
   - When: `forced_route(state)` called
   - Then: assert `broken_rules` was called for each candidate (count matches len(NODES) + number of agent combinations), and no routing logic executed outside those calls
   - Verify: Mock call count matches expected candidate count; no conditional logic in `forced_route` beyond the count check

6. **Test: single valid route among many candidates**
   - Given: state where only one candidate passes `broken_rules` (developer must construct state with specific completion flags)
   - When: `forced_route(state)` called
   - Then: returns that single route
   - Verify: `assert route == <expected_single_route>`

7. **Test: zero valid routes**
   - Given: state where no candidate passes `broken_rules` (impossible/broken state; mock `broken_rules` to reject all)
   - When: `forced_route(state)` called
   - Then: returns `None`
   - Verify: `assert route is None`

### Integration Tests (existing test suite)

- Ensure existing `route_rules` tests still pass after adding `forced_route`
- Add integration test using real state snapshots from end-to-end runs (pre-planner, between agents, post-review) to verify forced routes match expected behavior

### Acceptance Criteria Coverage

**AC2 (all scenarios covered by tests 1-5):**
- ✅ Test 1: AC2.2 (`["planner"]` before planner)
- ✅ Test 2: AC2.3 (`["__end__"]` after review)
- ✅ Test 3: AC2.4 (`["output-validator"]` after execution)
- ✅ Test 4: AC2.5 (`None` when multiple routes valid)
- ✅ Test 5: AC2.6 (`broken_rules` as sole source)
- ✅ Tests 1, 6: AC2.1 (single route when exactly one candidate passes)

**AC4 (forced_route behavior tests):**
- ✅ All seven unit tests map to AC4.1 forced_route test requirements

---

## Open Questions

**MUST resolve before implementation:**

1. **Where is `NODES` defined?**  
   - Developer must scan codebase to locate node registry or constant
   - Do NOT hardcode a placeholder list — tests would pass for wrong reasons
   - If not centralized, document the node list discovery process and flag for architect review

2. **Where is the stage agent list?**  
   - Same as above; if agents are dynamically registered, locate the registry API
   - Identify which nodes are "stage agents" vs. system nodes (planner, orchestrator, output-validator, etc.)

3. **Does `broken_rules` return empty list for valid routes?**  
   - Verify signature and behavior before implementing the counting logic
   - If it returns boolean or raises exceptions, adjust `forced_route` logic accordingly

4. **What state fields indicate planner/review/execution completion?**  
   - Tests 1-3 require these fields to construct valid state snapshots
   - Developer must locate these in the State schema

---

## Notes

- **Deferred per HLD:** If candidate generation proves slow, implement caching or topology-based pruning as a follow-up optimization.
- **No state changes in this ticket:** PER-128-2 only adds `forced_route` function; state schema changes are in PER-128-1.
- **No orchestrator changes in this ticket:** Integration of `forced_route` into orchestrator decision flow is in PER-128-3.
- **Repo-tree not mapped:** All file paths are placeholders. Developer MUST locate actual paths from the codebase before implementation. Do NOT invent paths or hardcode constants that should be discovered.

---

**End of Tech Stub**

## Spec `specs/PER-128-3.md`

# Tech Stub — PER-128-3

**Ticket:** PER-128-3  
**Epic:** PER-128  
**Repo:** agent  
**Title:** Integrate forced routing into orchestrator node

---

## Overview

This story integrates forced routing into the orchestrator's decision flow, adds per-run token usage aggregation and logging, and extends the run status API to return usage totals. The orchestrator calls `forced_route` at each decision point and skips the model when the route is deterministic and no model work remains (two exceptions: reviewer summary required, or an unseen flag exists). A new `state.seen_flags` field tracks which flags the model has observed. The runtime aggregates `state.usage` across nodes and logs totals at run end/pause. The API returns these totals in the response payload.

---

## Stack

**Note:** Repo tree not provided. File paths below are placeholders derived from ticket context and HLD references. Developer must locate actual paths in the agent codebase before implementation.

- **Language:** Python
- **State management:** Existing `State` schema (extended in PER-128-1 with `usage` field; this story adds `seen_flags`)
- **Orchestrator:** Existing node in `src/agents/nodes/orchestrator/node.py` (path TBD)
- **Routing:** `forced_route` function from PER-128-2 in `src/agents/route_rules.py` (path TBD)
- **Runtime:** Query handler in `src/agents/runtime/runner.py` (path TBD)
- **API:** Run status endpoint in `src/agents/api/routes.py` (path TBD)
- **Testing:** `pytest` (run via `uv run pytest -q`)

---

## Schema / API Contracts

### State Extension (seen_flags)

```python
# Add to State schema (location TBD from codebase scan)
from typing import Set

seen_flags: Set[str] = field(default_factory=set)
```

**Reducer contract:**  
When a state update includes a `seen_flags` delta (set of new flag IDs), the reducer unions it with the existing set:

```python
def _reduce_seen_flags(existing: Set[str], delta: Set[str]) -> Set[str]:
    """Union reducer: merge incoming flag IDs with existing."""
    return existing.union(delta)
```

**Migration contract:**  
Runs saved before this change have no `seen_flags` field. The state loader treats missing `seen_flags` as `set()` and execution continues normally (same pattern as `usage` migration in PER-128-1).

---

### forced_route API (from PER-128-2)

```python
def forced_route(state: State) -> list[str] | None:
    """
    Returns a single route if exactly one candidate passes broken_rules.
    Candidates: each node in NODES (alone), plus every non-empty set of stage agents.
    Returns None if zero or multiple candidates pass.
    
    Delegates all routing rules to broken_rules (no duplication).
    """
```

**This story imports and calls this function; it does not implement it.**

---

### Orchestrator Skip Decision Logic

```python
# In orchestrator node.py, at each decision point:

def _should_skip_model_call(route: list[str], state: State) -> bool:
    """
    Determines if we can skip _decide for this forced route.
    
    Returns False (call model) when:
    1. Route is ["flag-review"] or ["review"] — model must write reviewer summary
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
```

---

### Forced Hop Recording

```python
def _record_forced_hop(route: list[str], state: State) -> StateUpdate:
    """
    Record a forced hop step without calling the model.
    
    Step format:
    - title: "Next: <route>"
    - result: "forced: only one valid route"
    - evidence: {"attempts": 0}
    
    Updates status, priority, confidence as a model hop would.
    Leaves summary unchanged.
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
        flagged=state.flagged,
        priority=new_priority,
        confidence=new_confidence,
        summary=state.summary,  # unchanged per AC3.2
    )
```

**Note:** `_compute_status_for_route`, `_compute_priority_for_route`, and `_compute_confidence_for_route` are existing functions in the orchestrator that model hops use. Developer must locate these in the codebase and reuse them for forced hops to ensure consistency.

---

### Brief Flag ID Extraction

```python
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

### Usage Aggregation and Logging

```python
# In runtime/runner.py

def _aggregate_usage(usage_per_node: Dict[str, UsageStats]) -> UsageStats:
    """
    Sum usage across all nodes.
    
    Uses UsageStats.__add__ from PER-128-1.
    """
    total = UsageStats()
    for node, stats in usage_per_node.items():
        total = total + stats
    return total


def _log_usage_totals(usage: UsageStats):
    """
    Log total usage at INFO level.
    
    Format: "Run completed — calls: X, input_tokens: Y, output_tokens: Z, 
             cache_creation_input_tokens: A, cache_read_input_tokens: B"
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

**When to log:**  
At run end (when `state.status` transitions to `"complete"` or `"failed"`) or at pause (when `state.status` becomes `"paused"`). Developer must locate the exact condition in `run_query` that signals these transitions.

---

### API Response Extension

```python
# In api/routes.py

@app.get("/runs/{thread_id}")
def get_run(thread_id: str) -> dict:
    """
    Return run status and values, including total usage.
    
    Developer must verify existing response structure before implementing.
    The placeholder below assumes values.total_usage nesting per HLD AD-5,
    but actual structure TBD from codebase scan.
    """
    state = load_state(thread_id)
    
    # Aggregate usage across nodes
    total_usage = _aggregate_usage(state.usage)
    
    return {
        "thread_id": thread_id,
        "status": state.status,
        "values": {
            # Existing values fields (developer must preserve these)
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

**Open Question (must resolve before implementation):**  
Developer must inspect the current API handler to verify the exact nesting of `total_usage` in the response. The HLD specifies `values.total_usage`, but Test 4 must match the actual structure after codebase scan.

---

## File Touch List

**Note:** All paths are placeholders. Developer must locate actual file paths in the agent repo before implementation.

### Files to Modify

1. **State schema definition (e.g. `src/agents/state.py`)**  
   Add `seen_flags: Set[str]` field to State schema.  
   Implement union reducer: `_reduce_seen_flags(existing, delta) -> Set[str]`.

2. **State loader module (e.g. `src/agents/state_loader.py`)**  
   Handle missing `seen_flags` field by initializing to `set()` when loading old runs.  
   (Same migration pattern as `usage` field from PER-128-1.)

3. **Orchestrator node module (e.g. `src/agents/nodes/orchestrator/node.py`)**  
   Import `forced_route` from route_rules module.  
   At each decision point, call `forced_route(state)`.  
   Implement `_should_skip_model_call(route, state)` with two exceptions.  
   Implement `_record_forced_hop(route, state)` for forced hop steps.  
   Implement `_extract_flag_ids_from_brief(brief)`.  
   After every model call, extract flag IDs and update `state.seen_flags` via reducer.

4. **Route rules module (e.g. `src/agents/route_rules.py`)**  
   No changes needed (reads `forced_route` implemented in PER-128-2).

5. **Runtime query handler (e.g. `src/agents/runtime/runner.py`)**  
   Implement `_aggregate_usage(usage_per_node)`.  
   Implement `_log_usage_totals(usage)`.  
   At run end/pause, aggregate `state.usage` and log totals.

6. **API routes module (e.g. `src/agents/api/routes.py`)**  
   Extend `GET /runs/{thread_id}` handler to include `total_usage` in response payload.  
   Developer must verify existing response structure before nesting `total_usage`.

### Files to Create

7. **`tests/agents/nodes/test_orchestrator_forced_routing.py`**  
   New test file with 6 tests (see Tests section below).

8. **`tests/agents/integration/test_usage_tracking.py`**  
   New test file with 4 tests (see Tests section below).

---

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Forced route taken when model work needed** | High | Explicit tests for two skip exceptions (reviewer summary, unseen flags). Code review checks both conditions in `_should_skip_model_call`. |
| **Missing usage from orchestrator call sites** | Medium | PER-128-1 already added usage recording to all model gateways; this story only aggregates. Integration test asserts usage delta appears for orchestrator calls. |
| **Incorrect `seen_flags` tracking across pauses** | Medium | Store `seen_flags` in state (not ephemeral). Test 2 in integration suite loads a run without `seen_flags` and resumes. |
| **Forced hop step diverges from model hop format** | Low | Reuse existing `_compute_status_for_route`, `_compute_priority_for_route`, `_compute_confidence_for_route` functions. Test 4 asserts step structure matches. Test 6 compares call patterns or outcomes from different initial states. |
| **API response schema breaks existing clients** | Low | Extend `values` payload with `total_usage` (additive change). Test 4 in integration suite asserts correct nesting after verifying actual structure. |
| **State migration breaks existing runs** | Low | Loader treats missing `seen_flags` as `set()` (same pattern as `usage` from PER-128-1). Test 2 verifies migration. |

---

## Acceptance Criteria (from parent ticket AC3, AC4, AC5)

**AC3 scenarios:**

1. **AC3.1:** When `forced_route` returns a route and no model work is needed (no pause, no unseen flags), the orchestrator skips `_decide` and uses the forced route without calling the model.
2. **AC3.2:** A forced hop records a step with title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`, and updates `status`, `flagged`, `priority`, `confidence` as a model hop would, leaving `summary` unchanged.
3. **AC3.3:** When the forced route is `["flag-review"]` or `["review"]`, the orchestrator calls the model to write the reviewer's `summary`.
4. **AC3.4:** When an unresolved flag exists whose ID is not in `state.seen_flags`, the orchestrator calls the model to merge/address the flag.
5. **AC3.5:** After every model call, the orchestrator extracts flag IDs from the brief and adds them to `state.seen_flags`.

**AC4 scenarios (this story's scope):**

6. **AC4.1:** New tests verify the orchestrator makes no model call on forced hops (via gateway mock).
7. **AC4.2:** New tests verify the model is called when forced route requires reviewer summary or unseen flag handling.
8. **AC4.3:** New tests verify forced hop step structure matches expected format.
9. **AC4.4:** New tests verify `usage` accumulates correctly across calls, including runs without prior `usage` field.
10. **AC4.5:** All existing tests pass unchanged (`uv run pytest -q`).

**AC5 scenario:**

11. **AC5.1:** PR description reports before/after metrics (model calls, tokens) for one service order.

---

## Out of Scope

- State schema for `usage` field (added in PER-128-1)
- `forced_route` implementation (added in PER-128-2)
- Usage recording at model gateway call sites (added in PER-128-1)
- Changing models, `policy.yaml` limits, skill or prompt content
- Shrinking `_brief`, or removing the skill container and code-execution tool from orchestrator calls
- The MCP server, the GraphQL service, and fastapi-backend
- Graph routing rules themselves: `broken_rules` behaves as today

---

## Tests

### Test File 1: `tests/agents/nodes/test_orchestrator_forced_routing.py`

**Test count:** 6 (matching breakdown.md Verification Approach exactly)

#### Test 1: Forced Hop with No Pause, No Unseen Flags → No Model Call

```python
def test_forced_hop_no_model_call(fake_gateway, mocker):
    """
    Given forced_route returns ["planner"] and no skip conditions apply
    When orchestrator decides next hop
    Then no model call is made (via gateway mock)
    And forced hop step is recorded
    
    Covers AC3.1, AC4.1
    """
    mock_forced_route = mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=["planner"]
    )
    
    mock_decide = mocker.patch(
        'src.agents.nodes.orchestrator.node._decide'
    )
    
    state = State(
        flags=[],
        seen_flags=set(),
    )
    
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
    
    Covers AC3.3, AC4.2
    """
    mock_forced_route = mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=["flag-review"]
    )
    
    mock_decide = mocker.patch(
        'src.agents.nodes.orchestrator.node._decide',
        return_value=StateUpdate(
            route=["flag-review"],
            summary="Reviewer summary",
            brief={"flags": []},
        )
    )
    
    state = State(flags=[], seen_flags=set())
    
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
    
    Covers AC3.4, AC4.2
    """
    mock_forced_route = mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=["analyst"]
    )
    
    mock_decide = mocker.patch(
        'src.agents.nodes.orchestrator.node._decide',
        return_value=StateUpdate(
            route=["analyst"],
            brief={"flags": [{"id": "flag-123"}]},
        )
    )
    
    state = State(
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
    
    Covers AC3.2, AC4.3
    """
    mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=["planner"]
    )
    
    state = State(
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
    
    Covers AC3.5
    """
    mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=None  # Multiple routes, model called
    )
    
    mock_decide = mocker.patch(
        'src.agents.nodes.orchestrator.node._decide',
        return_value=StateUpdate(
            route=["analyst"],
            brief={"flags": [
                {"id": "flag-123"},
                {"id": "flag-456"},
            ]},
        )
    )
    
    state = State(flags=[], seen_flags=set())
    
    result = run_orchestrator(state)
    
    # Assert seen_flags delta contains flag IDs from brief
    assert result.seen_flags_delta == {"flag-123", "flag-456"}
```

#### Test 6: Forced Hop and Model Hop Produce Same Status Updates

```python
def test_forced_hop_status_matches_model_hop(fake_gateway, mocker):
    """
    Given two different initial states
    When forced hop and model hop compute status/priority/confidence for same route
    Then both paths use the same helper functions (_compute_X_for_route)
    And outcomes match when inputs match
    
    Covers AC3.2 (forced hop updates use same logic as model hop)
    
    Note: This test verifies that both code paths call the same helper functions
    by comparing outcomes from different initial states where the helpers produce
    different results. It does NOT mock _decide to return forced hop values.
    """
    # State A: initial conditions where status/priority/confidence compute to one set of values
    state_a = State(
        flags=[],
        seen_flags=set(),
        status="running",
        priority=5,
        confidence=0.8,
    )
    
    # State B: different conditions where helpers produce different values
    state_b = State(
        flags=[Flag(id="flag-999", resolved=False)],
        seen_flags=set(),
        status="paused",
        priority=2,
        confidence=0.3,
    )
    
    # Path 1: Forced hop with state A
    mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=["planner"]
    )
    forced_result_a = run_orchestrator(state_a)
    
    # Path 2: Forced hop with state B
    forced_result_b = run_orchestrator(state_b)
    
    # Assert different initial states produce different outcomes
    # (proves the helpers are actually computing, not hardcoded)
    assert forced_result_a.status != forced_result_b.status or \
           forced_result_a.priority != forced_result_b.priority or \
           forced_result_a.confidence != forced_result_b.confidence
    
    # Now verify model hop uses same helpers by mocking _decide to call them
    mocker.patch(
        'src.agents.nodes.orchestrator.node.forced_route',
        return_value=None  # Force model call
    )
    
    def mock_decide_impl(state):
        route = ["planner"]
        return StateUpdate(
            route=route,
            status=_compute_status_for_route(route, state),
            priority=_compute_priority_for_route(route, state),
            confidence=_compute_confidence_for_route(route, state),
            summary="Model summary",
            brief={"flags": []},
        )
    
    mocker.patch(
        'src.agents.nodes.orchestrator.node._decide',
        side_effect=mock_decide_impl
    )
    
    # Path 3: Model hop with state A
    model_result_a = run_orchestrator(state_a)
    
    # Assert forced and model hops produce same status/priority/confidence for same input
    assert forced_result_a.status == model_result_a.status
    assert forced_result_a.priority == model_result_a.priority
    assert forced_result_a.confidence == model_result_a.confidence
    
    # Verify summary behavior differs as expected
    assert forced_result_a.summary == state_a.summary  # unchanged
    assert model_result_a.summary == "Model summary"  # changed
```

---

### Test File 2: `tests/agents/integration/test_usage_tracking.py`

**Test count:** 4 (matching breakdown.md Verification Approach exactly)

#### Test 1: Usage Accumulates Across Multiple Calls

```python
def test_usage_accumulates_across_calls(fake_gateway):
    """
    Given multiple model calls occur for different nodes
    When run completes
    Then state.usage correctly sums counts per node
    
    Covers AC4.4 (usage accumulation)
    """
    state = State(usage={})
    
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
    
    Covers AC4.4 (state migration)
    
    Note: Developer must identify all required State schema fields before 
    writing this test. The state dict must include those fields to avoid 
    schema validation errors before the loader's migration check runs.
    """
    old_state_data = {
        "run_id": "test-123",
        "status": "paused",
        # Developer: add all other required State fields here (e.g. flags, plan, etc.)
        # 'usage' field missing
        # 'seen_flags' field missing
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
    
    Covers AC4.4 (usage logging)
    """
    state = State(
        usage={
            "orchestrator": UsageStats(calls=2, input_tokens=300, output_tokens=100, cache_creation_input_tokens=30, cache_read_input_tokens=15),
            "planner": UsageStats(calls=1, input_tokens=150, output_tokens=60, cache_creation_input_tokens=15, cache_read_input_tokens=7),
        }
    )
    
    with caplog.at_level("INFO"):
        run_query("thread-123", "test query", state)
    
    # Assert log contains total usage
    log_output = caplog.text
    assert "calls: 3" in log_output
    assert "input_tokens: 450" in log_output
    assert "output_tokens: 160" in log_output
    assert "cache_creation_input_tokens: 45" in log_output
    assert "cache_read_input_tokens: 22" in log_output
```

#### Test 4: GET /runs/{thread_id} Returns Totals in Response

```python
def test_get_run_returns_usage_totals(fake_gateway, test_client):
    """
    Given a run has usage data
    When GET /runs/{thread_id} is called
    Then response includes total usage in the payload
    
    Covers AC4.4 (API response)
    
    Note: Developer must verify the existing response structure before implementing.
    Adjust assertion path after inspecting the current API handler to match actual nesting.
    """
    state = State(
        thread_id="thread-123",
        usage={
            "orchestrator": UsageStats(calls=2, input_tokens=300, output_tokens=100, cache_creation_input_tokens=30, cache_read_input_tokens=15),
            "planner": UsageStats(calls=1, input_tokens=150, output_tokens=60, cache_creation_input_tokens=15, cache_read_input_tokens=7),
        }
    )
    save_state(state)
    
    response = test_client.get("/runs/thread-123")
    
    assert response.status_code == 200
    data = response.json()
    
    # Developer: Verify actual nesting from codebase scan before asserting.
    # Placeholder below assumes values.total_usage per HLD AD-5.
    assert "values" in data
    assert "total_usage" in data["values"]
    assert data["values"]["total_usage"]["calls"] == 3
    assert data["values"]["total_usage"]["input_tokens"] == 450
    assert data["values"]["total_usage"]["output_tokens"] == 160
    assert data["values"]["total_usage"]["cache_creation_input_tokens"] == 45
    assert data["values"]["total_usage"]["cache_read_input_tokens"] == 22
```

---

### Existing Tests

All existing tests in the agent repo must pass unchanged (`uv run pytest -q`).

**Regression risk mitigation:**
- `seen_flags` field defaults to `set()` in State schema (no impact on tests that don't inspect it)
- State loader handles missing `seen_flags` gracefully (old test fixtures continue to load)
- Forced routing only affects orchestrator decision flow; other nodes unchanged
- Usage aggregation in `run_query` and API response is additive (existing response fields unaffected)

---

## Open Questions

**Must resolve before implementation:**

1. **What is the existing structure of the `GET /runs/{thread_id}` response?**  
   - Is `values` a dict at the root level, or nested under another key?
   - Are there existing fields in `values` that must be preserved?
   - Does the response already have a `total_usage` field that this would conflict with?
   
   Developer must inspect `src/agents/api/routes.py` (or equivalent) to answer these questions and adjust the implementation and Test 4 accordingly.

2. **What are all the required fields in the State schema?**  
   - Test 2 in the integration suite requires a state dict with all required fields except `usage` and `seen_flags`.
   - Developer must scan the State definition (e.g. `src/agents/state.py`) to identify required fields before writing Test 2.

3. **Where do `_compute_status_for_route`, `_compute_priority_for_route`, and `_compute_confidence_for_route` live?**  
   - These functions are assumed to exist in the orchestrator module and are used by model hops today.

---

# Story summaries

- `01-agent`: Story 01-agent LLD
- `02-agent`: Story 02-agent LLD
- `03-agent`: Story 03-agent LLD

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
