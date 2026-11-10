# LLD — Story 02-agent: Forced route logic in route_rules

**Story:** Forced route logic in route_rules  
**Epic:** PER-128 (Skip orchestrator model calls on deterministic hops and track token usage)  
**Repo:** `agent`  
**Version:** 1.3  
**Status:** Draft

---

## Overview

Add `forced_route(state: State) -> list[str] | None` to `src/agents/nodes/orchestrator/route_rules.py`. This function determines when exactly one routing candidate is valid, allowing the orchestrator (in Story 3) to skip its model call on deterministic hops.

**Core logic:**

1. Enumerate all routing candidates:
   - Each node individually (from verified `NODES` or node directories)
   - Every non-empty subset of verified stage agents: `{compliance, payroll_calculation, validation}`
2. For each candidate, call `broken_rules(state, candidate)` using its verified signature.
3. If exactly one candidate passes validation, return it. Otherwise return `None`.

**Delegation principle:** All routing validity logic lives in `broken_rules`. `forced_route` is pure enumeration + filtering — no rule duplication.

This story is **independent of usage tracking** (Story 1's fields aren't used here), but depends on Story 1 for the `State` schema to exist in the codebase.

**CRITICAL VERIFICATION REQUIRED BEFORE ANY CODE:** The developer MUST complete the verification checklist below and document findings in `stories/02-agent/verification-findings.md` BEFORE writing any implementation code. The implementation sketches in this LLD are placeholders only — they will be adjusted to match verified behavior.

---

## Acceptance Criteria

### AC1: `forced_route` returns single valid route for deterministic scenarios

**Given** the graph state shows planner has not run  
**When** `forced_route(state)` is called  
**Then** it returns `["planner"]` (verified node name from repo).

**Given** the graph state shows execution has finished and output validation has not run  
**When** `forced_route(state)` is called  
**Then** it returns the verified output validator node name as a single-element list (from `src/agents/nodes/` directory — exact spelling verified in Step 3).

**Given** the graph state shows all work is complete and ready to terminate  
**When** `forced_route(state)` is called  
**Then** it returns the verified terminal route marker (determined by Step 3/5 decision: either `["__end__"]` if enumerated, or `None` if terminal logic is enforced by `broken_rules` without enumerating sentinels).

---

### AC2: `forced_route` returns `None` when multiple routes are valid

**Given** two stage agents are unfinished and `broken_rules` accepts both individually  
**When** `forced_route(state)` is called  
**Then** it returns `None` (orchestrator must call model to choose).

---

### AC3: `forced_route` handles parallel stage agent sets per `broken_rules` contract

**Given** `broken_rules` accepts only a parallel set of stage agents (verified behavior from Step 1)  
**When** `forced_route(state)` is called  
**Then** it returns that parallel set as the single valid route.

**Given** `broken_rules` never requires parallel execution (verified from Step 1)  
**When** multiple individual stage agents are valid  
**Then** `forced_route` returns `None` (multiple valid candidates).

---

### AC4: `forced_route` returns `None` when no candidates are valid

**Given** the state is in an error condition where `broken_rules` rejects all candidates  
**When** `forced_route(state)` is called  
**Then** it returns `None`.

---

### AC5: No routing logic duplication

**Given** `broken_rules(state, candidate)` is the sole source of routing validity  
**When** `forced_route` evaluates candidates  
**Then** it calls `broken_rules` for each candidate and contains no hardcoded routing rules.

---

## Verification Checklist (MANDATORY BLOCKING GATE)

**NO CODE MAY BE WRITTEN UNTIL THIS CHECKLIST IS COMPLETE AND DOCUMENTED IN `stories/02-agent/verification-findings.md`.**

### Step 1: Verify `broken_rules` contract

**Tasks:**

- [ ] Read `src/agents/nodes/orchestrator/route_rules.py` (full file or search for `def broken_rules`)
- [ ] Document exact function signature (copy the `def` line verbatim)
- [ ] Document parameter types (what is `next_step`? Single string? List of strings? Other?)
- [ ] Document return type (what does it return? List of error strings? Bool? Dict? Other?)
- [ ] Document validation convention (empty list = valid? `False` = valid? No errors in dict = valid? Other?)
- [ ] Document how `broken_rules` handles parallel agent routes (does it accept `["compliance", "validation"]` as a single parameter, or must they be validated separately?)
- [ ] Test with a sample call if signature is unclear (e.g., call `broken_rules` with a mock state and observe behavior)

**Required findings in `verification-findings.md`:**

```markdown
## Step 1: broken_rules Contract

**File inspected:** `src/agents/nodes/orchestrator/route_rules.py` (lines X-Y)

**Function signature (verbatim copy):**
```python
def broken_rules(state: State, next_step: <ACTUAL_TYPE>) -> <ACTUAL_RETURN_TYPE>:
```

**Parameter `next_step` expects:** (single node name string / list of node names / other — describe exactly)

**Return type:** (list of error strings / bool / dict / other — describe exactly)

**Validation convention:** (how to interpret the return value to determine if route is valid)
- Example: "Returns list[str]. Empty list means route is valid. Non-empty list contains error messages."
- Example: "Returns bool. True means route is INVALID. False means route is VALID."

**Parallel route handling:** (how `broken_rules` treats multi-node routes like `["compliance", "validation"]`)
- Example: "Accepts list of node names. Validates the entire set as a single route."
- Example: "Only accepts single node name string. Multi-node routes are not supported."

**Sample call tested:** (if behavior needed clarification)
```python
# broken_rules(mock_state, ["planner"]) → []  # Valid
# broken_rules(mock_state, ["invalid_node"]) → ["Node invalid_node does not exist"]
```

**Decision: Call pattern for forced_route:**
```python
# Based on verified signature and return type:
# If next_step expects list[str] and returns list[str]:
valid_candidates = [c for c in candidates if len(broken_rules(state, c)) == 0]

# If next_step expects str and returns bool:
# Adjust forced_route to call broken_rules(state, candidate[0]) for single-node candidates
# and handle multi-node candidates separately (or skip if not supported)
```
```

---

### Step 2: Verify `NODES` list

**Tasks:**

- [ ] Read `src/agents/nodes/__init__.py` (full file)
- [ ] Document whether a `NODES` list exists (search for `NODES =` or `__all__` containing node names)
- [ ] If `NODES` exists, copy its exact contents (verbatim)
- [ ] If `NODES` doesn't exist, document which node names are imported (look for `from .planner.node import`, etc.)
- [ ] Document the import pattern (how nodes are registered or exposed)

**Required findings in `verification-findings.md`:**

```markdown
## Step 2: NODES List

**File inspected:** `src/agents/nodes/__init__.py` (lines X-Y)

**NODES list exists?** (YES / NO)

**If YES, exact contents (verbatim copy):**
```python
NODES = [
    # Copy the exact list here
]
```

**If NO, document imports:**
```python
# from .planner.node import planner_node
# from .compliance.node import compliance_node
# (list all relevant imports)
```

**Import pattern observed:** (describe how nodes are registered)
- Example: "Nodes imported individually, no central NODES list."
- Example: "NODES list defined, contains string names matching directory names."

**Decision: Source for NODES in forced_route:**
- (If YES: import from __init__.py)
- (If NO: define in route_rules.py with verified node names from Step 3)
```

---

### Step 3: Verify node names from directory structure

**Tasks:**

- [ ] Run `ls src/agents/nodes/` or inspect the repo tree listing
- [ ] List all node directories found (from repo tree provided: `compliance/`, `execution/`, `orchestrator/`, `output_validator/`, `payroll_calculation/`, `planner/`, `validation/`)
- [ ] Verify stage agent names match breakdown assumptions (`compliance`, `payroll_calculation`, `validation`)
- [ ] Verify correct name format (underscores vs hyphens: is it `output_validator` or `output-validator`?)
- [ ] Read `src/agents/graph/build.py` (search for edge definitions or terminal state handling)
- [ ] Document terminal route marker (is `"__end__"` used? Is it in `NODES`? Is it handled specially?)

**Required findings in `verification-findings.md`:**

```markdown
## Step 3: Node Names and Terminal Marker

**Directory listing from repo tree (confirmed):**
```
src/agents/nodes/compliance/
src/agents/nodes/execution/
src/agents/nodes/orchestrator/
src/agents/nodes/output_validator/
src/agents/nodes/payroll_calculation/
src/agents/nodes/planner/
src/agents/nodes/validation/
```

**Verified node names (match directory names exactly):**
- planner
- compliance
- payroll_calculation
- validation
- execution
- output_validator (confirmed: uses underscores, not hyphens)
- orchestrator (excluded from candidates — orchestrator doesn't route to itself)

**Verified stage agents (subset of above):**
- compliance
- payroll_calculation
- validation

**File inspected for terminal marker:** `src/agents/graph/build.py` (lines X-Y)

**Terminal route marker findings:**
- Is `"__end__"` used? (YES / NO — document what you found in edge definitions)
- If YES, is `"__end__"` a runnable node (has a directory)? (NO — confirmed no __end__/ directory in repo tree)
- How is terminal state represented in edges? (describe what you found: e.g., "edges use '__end__' as a target string")
- Does broken_rules accept '__end__' as a candidate? (test with mock call, or infer from signature — if next_step expects runnable node names only, likely NO)

**Decision: Terminal marker handling:**
- Include `"__end__"` in NODES for enumeration? (YES / NO, with reasoning from broken_rules behavior)
- If NO: How does forced_route handle terminal state? (Answer: forced_route returns None when all work is done; orchestrator routes to __end__ via model decision)
- If YES: Test 3 assertion is `assert result == ["__end__"]`
- If NO: Test 3 assertion is `assert result is None`

**Final NODES list for forced_route enumeration:**
```python
NODES = [
    "planner",
    "compliance",
    "payroll_calculation",
    "validation",
    "execution",
    "output_validator",
    # "__end__"  # Include only if decision above is YES
]
```
```

---

### Step 4: Document decisions and adjust implementation

**Tasks:**

- [ ] Create `stories/02-agent/verification-findings.md` with Sections 1-3 above filled out
- [ ] Based on findings, finalize:
  - Exact `NODES` list to use in `forced_route` (from Step 3 decision)
  - Whether `"__end__"` is included (YES/NO decision documented in Step 3)
  - Exact call signature for `broken_rules(state, candidate)` (from Step 1)
  - Exact validation check pattern (from Step 1)
  - Final candidate enumeration logic (adjustments based on Step 1 parallel route findings)
- [ ] Write concrete implementation notes (replace all placeholders in File Plan)
- [ ] Write concrete test assertions (replace all placeholders in Tests section)

**Required section in `verification-findings.md`:**

```markdown
## Step 4: Implementation Plan

**NODES list for forced_route (final):**
```python
NODES = [
    # Exact list from Step 3 decision
]
```

**STAGE_AGENTS list (final):**
```python
STAGE_AGENTS = ["compliance", "payroll_calculation", "validation"]
```

**broken_rules call pattern (concrete):**
```python
# Based on Step 1 verified signature:
# Example if next_step is list[str] and return is list[str]:
valid_candidates = [c for c in candidates if len(broken_rules(state, c)) == 0]

# Example if next_step is str and return is bool:
# (Implementation must call broken_rules(state, c[0]) for single-node candidates,
#  and skip or handle multi-node candidates based on Step 1 findings)
```

**Candidate enumeration logic:**
- Individual nodes: (for node in NODES: candidates.append([node]))
- Stage agent subsets: (all 7 non-empty subsets of STAGE_AGENTS)
- Total candidates: (N individual + 7 stage subsets — e.g., 6 + 7 = 13 if __end__ excluded)

**Test assertions (concrete):**
- Test 1 (before planner): `assert result == ["planner"]`
- Test 2 (after execution): `assert result == ["output_validator"]` (verified spelling)
- Test 3 (after completion): `assert result == ["__end__"]` OR `assert result is None` (based on Step 3 decision)
- Test 4 (multiple valid): `assert result is None`
- Test 5 (parallel set): (SKIP if Step 1 shows broken_rules never requires parallel) OR (assert set(result) == {verified agents})
- Test 6 (no valid): `assert result is None`

**Implementation adjustments summary:**
- (List all changes to placeholder code based on verification)
- Example: "Replace NODES placeholder with final list from above"
- Example: "Replace validation check with: len(broken_rules(state, c)) == 0"
- Example: "Test 3 uses assertion: assert result is None (because __end__ not enumerated)"
```

---

**BLOCKING RULE:** The developer MUST NOT proceed to File Plan or Implementation until `verification-findings.md` exists and contains all required sections (Steps 1-4) with concrete, verified values (not placeholders). Code review will reject any PR that skips verification or contains placeholder code.

---

## File Plan

**NOTE:** All file paths, function signatures, and logic below are PLACEHOLDERS until `verification-findings.md` Step 4 provides concrete values. Replace every `<VERIFIED_*>` marker with actual findings.

### Modified Files

#### `src/agents/nodes/orchestrator/route_rules.py`

**Changes:**

1. Add `forced_route(state: State) -> list[str] | None` function.
2. Define or import `NODES` based on Step 2/4 findings.
3. Define `STAGE_AGENTS` constant with verified names from Step 3.
4. Enumerate candidates (individual nodes + stage agent subsets).
5. For each candidate, call `broken_rules(state, candidate)` using Step 1 verified signature.
6. Filter valid candidates based on Step 1 verified return type.
7. Return single valid candidate or `None`.

**Implementation template (ADJUST AFTER VERIFICATION):**

```python
from itertools import chain, combinations
from src.agents.graph.states import State

# <VERIFIED_FROM_STEP_2>: NODES source
# If NODES exists in __init__.py:
from src.agents.nodes import NODES
# If NODES doesn't exist, define here:
# NODES = [<VERIFIED_LIST_FROM_STEP_3>]

# <VERIFIED_FROM_STEP_3>: Stage agents
STAGE_AGENTS = ["compliance", "payroll_calculation", "validation"]

def _powerset(iterable):
    """Generate all non-empty subsets of iterable."""
    s = list(iterable)
    return chain.from_iterable(combinations(s, r) for r in range(1, len(s)+1))

def forced_route(state: State) -> list[str] | None:
    """
    Returns a single valid route if exactly one candidate passes broken_rules.
    Otherwise returns None.
    
    Candidates:
    - Each node in NODES individually
    - Every non-empty subset of STAGE_AGENTS
    
    Delegates all validity checks to broken_rules.
    
    Implementation is based on verified broken_rules contract in
    stories/02-agent/verification-findings.md Step 1 and Step 4.
    """
    candidates = []
    
    # Individual nodes
    for node in NODES:
        candidates.append([node])
    
    # Stage agent subsets
    for subset in _powerset(STAGE_AGENTS):
        candidates.append(list(subset))
    
    # <VERIFIED_FROM_STEP_1_AND_STEP_4>: Validation logic
    # Replace this placeholder with exact pattern from verification-findings.md Step 4:
    # Example A: If broken_rules returns list[str] (empty = valid):
    # valid_candidates = [c for c in candidates if len(broken_rules(state, c)) == 0]
    #
    # Example B: If broken_rules returns bool (True = invalid):
    # valid_candidates = [c for c in candidates if not broken_rules(state, c)]
    #
    # Example C: If broken_rules expects single string for next_step:
    # valid_candidates = []
    # for candidate in candidates:
    #     if len(candidate) == 1:
    #         if <VERIFIED_CHECK>(broken_rules(state, candidate[0])):
    #             valid_candidates.append(candidate)
    #     else:
    #         # Handle multi-node candidates per Step 1 findings
    #         pass
    
    <VERIFIED_VALIDATION_PATTERN>
    
    if len(valid_candidates) == 1:
        return valid_candidates[0]
    else:
        return None
```

**Critical reminder:** Replace every `<VERIFIED_*>` marker with concrete code from `verification-findings.md` Step 4 before committing. The placeholder code above will NOT compile.

---

#### `src/agents/nodes/__init__.py`

**Changes (conditional on Step 2 findings):**

**If Step 2 shows `NODES` does NOT exist:**

Add `NODES` list with verified node names from Step 3/4:

```python
# <VERIFIED_FROM_STEP_4>: Final NODES list
NODES = [
    # Copy exact list from verification-findings.md Step 4
]
```

Ensure `NODES` is exported in `__all__` if that pattern exists in the file.

**If Step 2 shows `NODES` already exists:**

No changes needed to `__init__.py`. Just import it in `route_rules.py`.

---

### New Test Files

#### `tests/test_forced_routing.py`

**CRITICAL:** All test code below uses `<VERIFIED_*>` placeholders. Replace with concrete values from `verification-findings.md` Step 4 before running tests.

**Setup:**

```python
import pytest
from src.agents.graph.states import State
from src.agents.nodes.orchestrator.route_rules import forced_route

# <VERIFIED_STATE_SCHEMA>: Read from src/agents/graph/states.py
# Document actual field names in verification-findings.md
# Example state constructor (adjust to match actual State):
def make_state(**kwargs):
    """Construct State with verified field names from states.py."""
    return State(**kwargs)
```

**Test 1: Before planner runs → returns `["planner"]`**

```python
def test_forced_route_before_planner():
    # <VERIFIED_STATE_FIELDS>: Adjust to actual State schema
    state = make_state(
        # Configure state where planner has not run yet
    )
    result = forced_route(state)
    
    # <VERIFIED_FROM_STEP_4>: Planner node name
    assert result == ["planner"]
```

**Test 2: After execution finishes → returns `["output_validator"]`**

```python
def test_forced_route_after_execution():
    # <VERIFIED_STATE_FIELDS>: Adjust to actual State schema
    state = make_state(
        # Configure state where execution finished, output_validator not run
    )
    result = forced_route(state)
    
    # <VERIFIED_FROM_STEP_3>: Output validator node name (exact spelling)
    assert result == ["output_validator"]
```

**Test 3: After all work complete → returns terminal marker or None**

```python
def test_forced_route_after_completion():
    # <VERIFIED_STATE_FIELDS>: Adjust to actual State schema
    state = make_state(
        # Configure state where all work is done, ready to terminate
    )
    result = forced_route(state)
    
    # <VERIFIED_FROM_STEP_3_AND_STEP_4>: Terminal marker decision
    # If __end__ is enumerated:
    # assert result == ["__end__"]
    # If __end__ is NOT enumerated:
    # assert result is None
    <VERIFIED_ASSERTION>
```

**Test 4: Multiple stage agents valid → returns None**

```python
def test_forced_route_multiple_valid():
    # <VERIFIED_STATE_FIELDS>: Adjust to actual State schema
    state = make_state(
        # Configure state where two stage agents are both valid
    )
    result = forced_route(state)
    
    assert result is None
```

**Test 5: Parallel stage agent set (conditional on Step 1 findings)**

```python
def test_forced_route_parallel_set():
    # <VERIFIED_FROM_STEP_1>: Skip if broken_rules never requires parallel
    # If Step 1 shows parallel routes are supported:
    
    # <VERIFIED_STATE_FIELDS>: Configure state where only parallel set is valid
    state = make_state(
        # ...
    )
    result = forced_route(state)
    
    # <VERIFIED_FROM_STEP_1_AND_STEP_4>: Expected parallel set
    # assert set(result) == {"compliance", "validation"}  # example
    <VERIFIED_ASSERTION_OR_SKIP>
```

**Test 6: No valid candidates → returns None**

```python
def test_forced_route_no_valid():
    # <VERIFIED_STATE_FIELDS>: Configure state where broken_rules rejects all
    state = make_state(
        # ...
    )
    result = forced_route(state)
    
    assert result is None
```

---

## Tests

### Test Coverage Summary

| Test | AC Coverage | Verified Elements Required (from verification-findings.md Step 4) |
|------|-------------|---------------------------|
| `test_forced_route_before_planner` | AC1 | Planner node name, State schema, broken_rules validation |
| `test_forced_route_after_execution` | AC1 | Output validator node name (exact spelling: output_validator), State schema, broken_rules validation |
| `test_forced_route_after_completion` | AC1 | Terminal marker decision (__end__ enumerated YES/NO), State schema, broken_rules validation |
| `test_forced_route_multiple_valid` | AC2 | Stage agent names, State schema, broken_rules validation |
| `test_forced_route_parallel_set` | AC3 | Step 1 parallel route behavior (skip if not supported), State schema |
| `test_forced_route_no_valid` | AC4 | broken_rules rejection behavior, State schema |

**All tests require:**

- `verification-findings.md` Steps 1-4 complete with concrete values (not placeholders)
- State schema read from `src/agents/graph/states.py` and documented
- Test assertions adjusted to match verified behavior

---

## Interfaces

### `forced_route(state: State) -> list[str] | None`

**Module:** `src/agents/nodes/orchestrator/route_rules.py`

**Signature:**

```python
def forced_route(state: State) -> list[str] | None:
    """
    Returns a single valid route if exactly one candidate passes broken_rules.
    Otherwise returns None.
    
    Candidates enumerated:
    - Each node in verified NODES individually (from verification-findings.md Step 4)
    - Every non-empty subset of verified STAGE_AGENTS (7 subsets total)
    
    Validity determined by calling broken_rules(state, candidate) with
    verified signature and validation convention from verification-findings.md Step 1.
    
    Args:
        state: Current graph state
    
    Returns:
        list[str]: Single valid route (node names as strings)
        None: Zero or multiple valid routes
    
    Contract:
    - Exhaustive enumeration of all possible routes
    - Delegates all validity checks to broken_rules (no logic duplication)
    - Deterministic (same state → same result)
    - No side effects (pure function)
    """
```

---

### `broken_rules(state: State, next_step: <VERIFIED_TYPE>) -> <VERIFIED_RETURN_TYPE>`

**Module:** `src/agents/nodes/orchestrator/route_rules.py` (existing function)

**Signature:** Verified in Step 1, documented in verification-findings.md

**Contract:** Sole source of routing validity logic. `forced_route` calls it, never duplicates its rules.

---

### `NODES` List

**Module:** `src/agents/nodes/__init__.py` (if exists) or `route_rules.py` (if added)

**Type:** `list[str]`

**Content:** Verified in Steps 2-4, documented in verification-findings.md Step 4

**Contract:**

- Contains all node names used in graph routing (excluding orchestrator)
- Names match exactly what `broken_rules` expects (verified spelling)
- Order doesn't matter (used for enumeration only)

---

### `STAGE_AGENTS` Constant

**Module:** `src/agents/nodes/orchestrator/route_rules.py` (to be added)

**Type:** `list[str]`

**Value (from repo tree and Step 3 verification):**

```python
STAGE_AGENTS = ["compliance", "payroll_calculation", "validation"]
```

**Contract:**

- Contains names of stage agents that may execute in parallel
- Names match exactly what `broken_rules` expects
- Used to enumerate all 2^3 - 1 = 7 non-empty subsets

---

## Edge Cases

### Edge Case 1: No valid candidates (all routes rejected)

**Scenario:** State is in error condition, `broken_rules` rejects all candidates.

**Handling:** `forced_route` returns `None`. Orchestrator (Story 3) calls model.

**Test:** `test_forced_route_no_valid`

---

### Edge Case 2: Multiple stage agents individually valid

**Scenario:** Planner ran, two stage agents unfinished, `broken_rules` accepts both.

**Handling:** `forced_route` returns `None` (multiple valid).

**Test:** `test_forced_route_multiple_valid`

---

### Edge Case 3: Only parallel set valid (if supported)

**Scenario:** `broken_rules` enforces parallel execution of specific agents (verified in Step 1).

**Handling:** `forced_route` returns that set.

**Test:** `test_forced_route_parallel_set` (skip if Step 1 shows this never happens)

---

### Edge Case 4: Terminal marker enumeration

**Scenario:** `"__end__"` is a LangGraph sentinel (confirmed: no `__end__/` directory in repo tree).

**Handling decision (from Step 3/4):**

- **If Step 3/4 decision is YES (enumerate __end__):** Include `"__end__"` in NODES. Test 3 expects `result == ["__end__"]` after all work is complete.
- **If Step 3/4 decision is NO (don't enumerate __end__):** Exclude `"__end__"` from NODES. `broken_rules` enforces terminal state logic. Test 3 expects `result is None` after all work is complete (orchestrator calls model to route to `__end__`).

**Test:** `test_forced_route_after_completion` (assertion adjusted per Step 3/4 decision)

---

### Edge Case 5: `broken_rules` expects different signature than assumed

**Scenario:** After Step 1 verification, `broken_rules` signature differs from HLD assumptions (e.g., expects single string, not list).

**Handling:** Step 1 verification documents actual signature. Implementation adjusts `forced_route` to match (e.g., call `broken_rules(state, c[0])` for single-node candidates, skip or handle multi-node candidates per Step 1 findings).

**Mitigation:** Step 1 verification prevents runtime discovery. Implementation must not proceed until Step 1 is complete.

---

## Implementation Checklist

**BLOCKING PRE-IMPLEMENTATION:**

- [ ] Complete Step 1: Verify `broken_rules` signature, return type, validation convention, parallel route handling
- [ ] Complete Step 2: Verify `NODES` existence and content in `__init__.py`
- [ ] Complete Step 3: Verify node names from directory structure, terminal marker handling, output_validator spelling
- [ ] Complete Step 4: Document final NODES list, __end__ enumeration decision, concrete broken_rules call pattern, concrete test assertions
- [ ] Read `src/agents/graph/states.py` to determine actual State field names (for test setup)
- [ ] **BLOCK: Code review rejects PR if verification-findings.md is missing, incomplete, or contains placeholder values**

**Implementation (after verification complete):**

- [ ] Replace all `<VERIFIED_*>` markers in `route_rules.py` implementation with concrete code from Step 4
- [ ] Add or import `NODES` in `route_rules.py` or `__init__.py` per Step 2/4 decision
- [ ] Define `STAGE_AGENTS` in `route_rules.py` with verified names
- [ ] Implement `forced_route` function with verified `broken_rules` call pattern from Step 4
- [ ] Replace all `<VERIFIED_*>` markers in `tests/test_forced_routing.py` with concrete values from Step 4
- [ ] Adjust Test 3 assertion to match Step 3/4 __end__ enumeration decision
- [ ] Skip or adjust Test 5 based on Step 1 parallel route findings
- [ ] Run `uv run pytest tests/test_forced_routing.py -v`
- [ ] Verify all tests pass (6 tests, or 5 if Test 5 skipped)
- [ ] Verify existing tests still pass: `uv run pytest -q`

---

## Dependencies

**Depends on:**

- Story 1 (State schema) — `State` class must exist in `src/agents/graph/states.py`

**Blocks:**

- Story 3 (Orchestrator guards) — needs `forced_route` function to call

**Critical path note:** Story 2 can begin verification steps immediately (Steps 1-3 inspect existing code, don't depend on Story 1's new fields). Implementation of `forced_route` code waits for Story 1 merge. Tests can be written during verification (using `<VERIFIED_*>` markers), then finalized after Step 4.

---

## Notes

- **Verification is mandatory.** No exceptions. Code review will reject any PR that contains placeholder code (`<VERIFIED_*>` markers not replaced) or implements based on assumptions instead of verified behavior.
- **Terminal marker decision must be made in Step 3/4.** Test 3's assertion depends on it. The decision is: "Does `broken_rules` accept `"__end__"` as a valid `next_step` parameter?" Test with a mock call if needed.
- **Parallel route support must be verified in Step 1.** If `broken_rules` never requires parallel execution of stage agents, Test 5 is skipped. Document the finding.
- **State schema field names matter for tests.** Tests construct `State` objects. Inspect `states.py` and document the actual field names used to track node results (e.g., `planner_result`, `execution_result`, or other).
- **Performance is not a concern.** Enumerating ~13 candidates (6 individual nodes + 7 stage subsets, if `__end__` excluded) and calling `broken_rules` 13 times is negligible vs. a 1-2 second model call.
- **`output_validator` spelling verified from repo tree:** underscore format confirmed (`output_validator/` directory exists, no `output
