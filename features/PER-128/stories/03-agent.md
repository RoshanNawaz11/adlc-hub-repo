# LLD — Story 03-agent: Orchestrator decision guards and forced hops

**Story:** Orchestrator decision guards and forced hops  
**Epic:** PER-128 (Skip orchestrator model calls on deterministic hops and track token usage)  
**Repo:** `agent`  
**Version:** 1.2  
**Status:** Draft

---

## Overview

Modify the orchestrator (`src/agents/nodes/orchestrator/node.py`) to skip model calls when the graph has only one valid route and the model has no other work to do. Before calling the model decision function, extract flag IDs from the current state and update `State.orchestrator_seen_flag_ids`, then call `forced_route(state)` (from Story 2) to check if a route is deterministic.

**Model work guards:** The orchestrator still calls the model when:

1. **Guard 1 (Pause detection):** Pause is required (detected by checking state conditions verified from `src/agents/graph/interrupts.py`, NOT by routing to a pause node—no separate pause nodes exist per repo tree).
2. **Guard 2 (Unseen flags):** An unresolved flag exists whose ID is not in `State.orchestrator_seen_flag_ids` (model must merge duplicate flags).

**Forced hops:** When no model work is required, skip the model decision function and write a step with:

- `title: "Next: {route}"`
- `result: "forced: only one valid route"`
- `evidence: {attempts: 0}`
- `status`, `flagged`, `priority`, `confidence` updated as a model hop would
- `summary` unchanged

This story integrates Stories 1 (State schema) and 2 (forced_route logic) into the orchestrator's decision flow.

**CRITICAL VERIFICATION MANDATE:** Before implementing ANY code, the developer MUST complete the Verification Checklist below and document ALL findings in `stories/03-agent/verification-findings.md`. The implementation sketches in this LLD are PLACEHOLDERS ONLY—they will be adjusted to match verified behavior. NO CODE may be written until verification is complete and documented.

---

## Acceptance Criteria

### AC1: Orchestrator skips model decision function on forced hop when no model work required

**Given** `forced_route(state)` returns a route  
**And** pause is not required (using verified pause detection mechanism from `src/agents/graph/interrupts.py`)  
**And** no unresolved flag exists whose ID is missing from `State.orchestrator_seen_flag_ids` (using verified flag schema from `src/agents/graph/states.py`)  
**When** the orchestrator processes the hop  
**Then** the model decision function (verified name from `src/agents/nodes/orchestrator/node.py`) is not called, and a forced hop step is written with `attempts: 0`.

---

### AC2: Forced hop step format

**Given** a forced hop completes  
**When** the step is written  
**Then** the step has (using verified step structure from `src/agents/nodes/orchestrator/node.py`):

- `title: "Next: {route}"` (e.g., `"Next: ['planner']"`)
- `result: "forced: only one valid route"`
- `evidence: {attempts: 0, ...}` (plus any other evidence fields a normal hop includes, verified from existing step structure)

---

### AC3: Forced hop state updates

**Given** a forced hop completes  
**When** state is updated  
**Then** `status`, `flagged`, `priority`, and `confidence` are updated exactly as a model hop would (using verified field names and update pattern from `src/agents/nodes/orchestrator/node.py`), and `summary` remains unchanged.

---

### AC4: Model call proceeds when pause is required

**Given** `forced_route(state)` returns a route  
**And** pause is required (detected by verified pause mechanism from `src/agents/graph/interrupts.py`—NOT by checking route names, as no separate pause nodes exist)  
**When** the orchestrator processes the hop  
**Then** the model decision function is called (model must write `summary` before pause).

---

### AC5: Model call proceeds on unseen flags

**Given** `forced_route(state)` returns a route  
**And** an unresolved flag exists whose ID is not in `State.orchestrator_seen_flag_ids` (using verified flag schema from `src/agents/graph/states.py`)  
**When** the orchestrator processes the hop  
**Then** the model decision function is called (model must merge flags).

---

### AC6: `orchestrator_seen_flag_ids` updated before decision

**Given** the orchestrator receives control with flags present in state  
**When** the orchestrator processes the hop  
**Then** the flag IDs are extracted (using verified flag schema from `src/agents/graph/states.py`) and added to `State.orchestrator_seen_flag_ids` before any decision logic runs.

---

## Verification Checklist (MANDATORY BLOCKING GATE)

**NO CODE MAY BE WRITTEN UNTIL THIS CHECKLIST IS COMPLETE AND DOCUMENTED IN `stories/03-agent/verification-findings.md`.**

### Step 1: Verify flag schema in `State`

**Tasks:**

- [ ] Read `src/agents/graph/states.py` (full file or search for class `State`)
- [ ] Document actual flag field name (search for "flag" in the file—is it `flags`, `open_flags`, `active_flags`, `flagged`, other?)
- [ ] Document flag dict schema if flags are a list: what keys exist? (e.g., `id`/`flag_id`, `resolved`/`is_resolved`, `message`, other?)
- [ ] Document flag storage type: list of dicts? dict keyed by ID? other?
- [ ] Provide example flag from code or docstring

**Required findings in `verification-findings.md`:**

```markdown
## Step 1: Flag Schema from states.py

**File inspected:** `src/agents/graph/states.py` (lines X-Y)

**Flag field name:** state.<verified_name> (e.g., `state.flags`, `state.open_flags`, etc.)

**Flag storage type:** (list[dict] / dict[str, dict] / other)

**If list[dict], flag dict structure:**
```python
{
  "<verified_id_field>": str,  # e.g., "id" or "flag_id"
  "<verified_resolved_field>": bool,  # e.g., "resolved" or "is_resolved"
  # Other keys found: message, source, etc.
}
```

**Example flag from code/docstring:**
```python
{"id": "flag_abc", "resolved": False, "message": "...", "source": "compliance"}
# OR different structure if found
```

**How to check if a flag is unresolved:**
```python
# e.g., flag["resolved"] == False
# OR:  flag.get("is_resolved", True) == False
# OR:  other pattern found in code
```
```

---

### Step 2: Verify pause detection mechanism

**Tasks:**

- [ ] Search `src/agents/nodes/` directory for any directories named `review`, `flag-review`, `human_review`, or similar (confirm they do NOT exist per repo tree)
- [ ] Read `src/agents/graph/interrupts.py` (full file or search for `interrupt_before`)
- [ ] Document how pause is triggered (interrupt condition in graph builder? state flag checked by orchestrator? other?)
- [ ] Document what state condition triggers pause (e.g., `state.needs_review`, `state.flagged`, presence of unresolved flags, other)
- [ ] Confirm whether pause is orthogonal to routing (pause happens via interrupt, not by routing to a pause node)

**Required findings in `verification-findings.md`:**

```markdown
## Step 2: Pause Detection Mechanism

**Directories searched:** `src/agents/nodes/` (confirmed no `review/`, `flag-review/`, etc.)

**Pause route nodes exist?** NO (per repo tree and directory search)

**File inspected:** `src/agents/graph/interrupts.py` (lines X-Y)

**Pause mechanism:**
- Triggered by: (interrupt_before condition / state flag / other)
- Condition code location: (show relevant code snippet from interrupts.py or build.py)
- Example condition: (e.g., `interrupt_before=lambda state: state.get("flagged", False)`)

**State field/condition that triggers pause:**
```python
# e.g., state.flagged == True
# OR:  len(state.open_flags) > 0
# OR:  state.needs_review == True
# OR:  other verified condition
```

**Conclusion:** Pause is (interrupt-based / routing-based). Guard 1 will check (state condition / route name).
```

---

### Step 3: Verify orchestrator function structure

**Tasks:**

- [ ] Read `src/agents/nodes/orchestrator/node.py` (search for main function and model decision function)
- [ ] Document the main orchestrator function name (is it `run`, `decide`, `orchestrator_node`, other?)
- [ ] Document the model decision function name and signature (is it `_decide`, `decide_route`, other? what does it return?)
- [ ] Document step creation pattern: where are steps written? what dict structure?
- [ ] Document state field update pattern: how are `status`, `priority`, `confidence`, `flagged`, `summary` updated? (show code snippet)
- [ ] Document evidence structure: what keys exist in evidence dict?

**Required findings in `verification-findings.md`:**

```markdown
## Step 3: Orchestrator Structure from node.py

**File inspected:** `src/agents/nodes/orchestrator/node.py` (lines X-Y)

**Main orchestrator function:**
```python
def <verified_name>(state: State) -> dict:  # e.g., orchestrator_node
    # Show function signature line
```

**Model decision function:**
```python
def <verified_name>(state: State) -> dict:  # e.g., _decide
    # Show function signature line
```

**Model decision function returns:**
```python
{
  "steps": [...],  # list of step dicts
  "next": [...],   # list of node names
  "status": str,
  "summary": str,
  # Other fields found
}
```

**Step structure (show example from code):**
```python
{
  "title": str,
  "result": str,
  "evidence": {
    "attempts": int,
    # Other evidence keys found
  },
  # Other step fields: timestamp, etc.
}
```

**State field update pattern (show code snippet):**
```python
# How status, priority, confidence, flagged, summary are computed/updated
# e.g., return {"status": _compute_status(...), "priority": ..., ...}
```
```

---

### Step 4: Verify state update pattern

**Tasks:**

- [ ] Confirm how LangGraph nodes return state updates in this codebase (dict with updated fields? other?)
- [ ] Document field names for `next`, `status`, `summary`, `steps`, `priority`, `confidence`, `flagged`
- [ ] Verify how `steps` field accumulates (list append each hop? single step overwrite? other?)
- [ ] Verify `summary` behavior (overwrite? append? other?)

**Required findings in `verification-findings.md`:**

```markdown
## Step 4: State Update Pattern

**State update return type:** (dict with updated fields / other)

**Field names verified:**
- next: (list[str] / other)
- status: (str / other)
- summary: (str / other)
- steps: (list[dict] / other)
- priority: (str / other)
- confidence: (str / other)
- flagged: (bool / other)

**Steps field behavior:**
```python
# Append each hop: return {"steps": [new_step]} and reducer appends
# OR: Single step overwrite: return {"steps": new_step}
# OR: other pattern observed
```

**Summary field behavior:**
```python
# Overwrite: return {"summary": "new summary"}
# OR: Append: reducer appends to existing
# OR: other pattern
```
```

---

### Step 5: Document findings and adjust implementation plan

**Tasks:**

- [ ] Create `stories/03-agent/verification-findings.md` with all findings from Steps 1-4 documented
- [ ] Based on findings, write adjusted implementation notes:
  - Actual flag field name and ID extraction pattern
  - Actual pause detection pattern (state condition check, not route matching)
  - Actual model decision function name
  - Actual step structure and state update pattern
- [ ] List any blocking issues (e.g., if flag schema cannot support unique IDs, Guard 2 cannot be implemented as designed)

**Required section in `verification-findings.md`:**

```markdown
## Step 5: Implementation Adjustments

**Flag extraction pattern (adjusted from Step 1):**
```python
current_flags = state.<verified_flag_field_name>
flag_ids = {flag.get("<verified_id_field>") for flag in current_flags if "<verified_id_field>" in flag}
```

**Unresolved flag check (adjusted from Step 1):**
```python
unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
```

**Pause detection (adjusted from Step 2):**
```python
# Guard 1 checks:
if <verified_pause_condition>:
    # e.g., if state.flagged or len(state.open_flags) > 0
    # Call model decision function for summary
```

**Model decision function call (adjusted from Step 3):**
```python
decide_result = <verified_function_name>(state)
# e.g., _decide(state)
```

**Forced hop step creation (adjusted from Step 3):**
```python
step = {
    "title": f"Next: {route}",
    "result": "forced: only one valid route",
    "evidence": {
        "attempts": 0,
        # Other verified evidence fields
    },
    # Other verified step fields
}
```

**State update return (adjusted from Step 4):**
```python
return {
    "steps": [step],  # or other verified pattern
    "next": route,
    "status": <computed>,
    "flagged": <computed>,
    "priority": <computed>,
    "confidence": <computed>,
    "summary": state.<verified_summary_field>  # unchanged
}
```

**Blocking issues:** (none / list any issues that prevent implementation as designed)
```

---

**BLOCKING RULE:** The developer MUST NOT proceed to File Plan or Implementation until `verification-findings.md` exists with all five steps completed. Code review WILL REJECT any PR that skips verification or uses placeholder values from this LLD.

---

## File Plan

### Modified Files

#### `src/agents/nodes/orchestrator/node.py`

**Current state (from repo tree):**

- File exists.
- Contains orchestrator node logic.
- Calls a model decision function.
- Writes steps to state.

**Changes (ALL field names, function names, and conditions below are PLACEHOLDERS—replace with verified values from `verification-findings.md`):**

1. **Import `forced_route`:**

   ```python
   from src.agents.nodes.orchestrator.route_rules import forced_route
   ```

2. **Extract flag IDs before decision:**

   ```python
   # VERIFIED VALUES REQUIRED FROM Step 1:
   current_flags = state.<verified_flag_field_name>
   flag_ids = {flag.get("<verified_id_field>") for flag in current_flags if "<verified_id_field>" in flag}
   
   state_updates = {}
   if flag_ids:
       state_updates["orchestrator_seen_flag_ids"] = flag_ids
   ```

3. **Call `forced_route(state)` before model decision:**

   ```python
   route = forced_route(state)
   ```

4. **Implement model work guards (VERIFIED CONDITIONS REQUIRED FROM Steps 1 and 2):**

   ```python
   if route is not None:
       # Guard 1: Pause detection (VERIFIED CONDITION FROM Step 2)
       # Example (ADJUST to verified condition):
       if <verified_pause_condition>:  # e.g., state.flagged or len(state.open_flags) > 0
           decide_result = <verified_model_function_name>(state)  # FROM Step 3
           decide_result.update(state_updates)
           return decide_result
       
       # Guard 2: Unseen flags (VERIFIED SCHEMA FROM Step 1)
       unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
       unseen_flags = [f for f in unresolved_flags if f.get("<verified_id_field>") not in state.orchestrator_seen_flag_ids]
       if unseen_flags:
           decide_result = <verified_model_function_name>(state)
           decide_result.update(state_updates)
           return decide_result
       
       # No model work, write forced hop
       forced_result = _write_forced_hop(state, route)
       forced_result.update(state_updates)
       return forced_result
   
   # No forced route, call model
   decide_result = <verified_model_function_name>(state)
   decide_result.update(state_updates)
   return decide_result
   ```

5. **Implement `_write_forced_hop` (VERIFIED STRUCTURE FROM Steps 3 and 4):**

   ```python
   def _write_forced_hop(state: State, route: list[str]) -> dict:
       # VERIFIED STEP STRUCTURE FROM Step 3:
       step = {
           "title": f"Next: {route}",
           "result": "forced: only one valid route",
           "evidence": {
               "attempts": 0,
               # VERIFIED other evidence fields FROM Step 3
           },
           # VERIFIED other step fields FROM Step 3
       }
       
       # VERIFIED STATE COMPUTATION FROM Step 3:
       status = _compute_status(state, route)
       flagged = _compute_flagged(state, route)
       priority = _compute_priority(state, route)
       confidence = _compute_confidence(state, route)
       summary = state.<verified_summary_field>  # FROM Step 4
       
       # VERIFIED RETURN PATTERN FROM Step 4:
       return {
           "steps": [step],  # or other verified pattern
           "next": route,
           "status": status,
           "flagged": flagged,
           "priority": priority,
           "confidence": confidence,
           "summary": summary
       }
   ```

6. **Implement helper functions for state computation:**

   ```python
   def _compute_status(state: State, route: list[str]) -> str:
       # VERIFIED TERMINAL NODE FROM Story 2:
       if route == [<verified_terminal_node_name>]:  # e.g., ["__end__"]
           return "complete"
       return state.<verified_status_field> or "in_progress"
   
   def _compute_flagged(state: State, route: list[str]) -> bool:
       # VERIFIED FLAG FIELD FROM Step 1:
       current_flags = state.<verified_flag_field_name>
       unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
       return len(unresolved_flags) > 0
   
   def _compute_priority(state: State, route: list[str]) -> str:
       return state.<verified_priority_field> or "normal"
   
   def _compute_confidence(state: State, route: list[str]) -> str:
       return "high"  # forced hops are deterministic
   ```

**CRITICAL NOTE:** Every `<verified_*>` placeholder above MUST be replaced with actual values from `verification-findings.md` before implementation. Code that uses these placeholders will NOT compile or will fail at runtime.

---

## Interfaces

### `forced_route(state: State) -> list[str] | None`

**Module:** `src/agents/nodes/orchestrator/route_rules.py` (from Story 2)

**Contract:** Returns single valid route or `None`.

**Usage:** Called before model decision to check if route is deterministic.

---

### Model Decision Function (name verified in Step 3)

**Module:** `src/agents/nodes/orchestrator/node.py` (existing function)

**Signature (VERIFIED IN Step 3):**

```python
def <verified_name>(state: State) -> dict:
    # Name could be _decide, decide_route, or other
```

**Usage:** Called when model work is required.

---

### `_write_forced_hop(state: State, route: list[str]) -> dict`

**Module:** `src/agents/nodes/orchestrator/node.py` (new function)

**Signature:**

```python
def _write_forced_hop(state: State, route: list[str]) -> dict:
    """
    Writes a forced hop step and updates state fields without calling the model.
    
    Args:
        state: Current graph state
        route: Forced route (single valid candidate from forced_route)
    
    Returns:
        dict: State update with step, next, status, flagged, priority, confidence, summary
    """
```

**Contract:**

- Step format matches model hop steps (verified structure from Step 3)
- State field updates match model hop behavior (verified pattern from Step 3)
- `summary` is unchanged (carried forward from state)

---

## Algorithm Notes

### Decision Flow

1. Extract flag IDs using verified schema → add to `orchestrator_seen_flag_ids`
2. Call `forced_route(state)`
3. If `None`, call model (existing behavior)
4. If route returned:
   - Check Guard 1 (pause required via verified condition)
   - Check Guard 2 (unseen flags via verified schema)
   - If either guard passes, call model
   - Otherwise, write forced hop

---

### Model Work Guards

**Guard 1: Pause detection (VERIFIED IN Step 2)**

**Condition:** Pause is required (detected by verified state condition from `interrupts.py`, NOT by checking route names).

**Implementation pattern (ADJUST to verified condition):**

```python
# Example patterns (use verified condition from Step 2):
if state.flagged:
    # Call model for summary
if len(state.open_flags) > 0:
    # Call model for summary
if state.needs_review:
    # Call model for summary
```

**Note:** Guard 1 does NOT check `route == ["review"]` because no separate pause node exists (verified in Step 2).

---

**Guard 2: Unseen flags (VERIFIED IN Step 1)**

**Condition:** An unresolved flag exists whose ID is not in `orchestrator_seen_flag_ids`.

**Implementation pattern (ADJUST to verified schema):**

```python
current_flags = state.<verified_flag_field>
unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
unseen_flags = [f for f in unresolved_flags if f.get("<verified_id_field>") not in state.orchestrator_seen_flag_ids]
if unseen_flags:
    # Call model to merge
```

---

## Edge Cases

### Edge Case 1: Pause required on forced route (model work required)

**Scenario:** `forced_route` returns a route, but pause condition is met (verified from Step 2).

**Handling:** Guard 1 triggers → call model decision function for summary.

**Test:** AC4

---

### Edge Case 2: Unseen flag on forced route (model work required)

**Scenario:** `forced_route` returns a route, but an unseen flag exists (verified schema from Step 1).

**Handling:** Guard 2 triggers → call model decision function for merge.

**Test:** AC5

---

### Edge Case 3: Flag schema different than assumed

**Scenario:** After Step 1 verification, flags are stored in `state.active_flags` with ID field `flag_id`, not `state.flags` with `id`.

**Handling:** All flag references adjusted to verified schema before implementation.

**Mitigation:** Step 1 verification prevents runtime discovery.

---

### Edge Case 4: Pause is interrupt-based (no route nodes)

**Scenario:** After Step 2 verification, pause is triggered by `interrupt_before`, not by routing.

**Handling:** Guard 1 checks verified state condition (e.g., `state.flagged`), not route name.

**Mitigation:** Step 2 verification reveals mechanism before implementation.

---

### Edge Case 5: Flag missing ID field

**Scenario:** A flag has no ID field (malformed state).

**Handling:** Flag extraction skips it: `{f.get("id") for f in flags if "id" in f}` (adjusted to verified field name).

**Assumption:** Flags always have IDs (enforced elsewhere). Log warning if encountered.

---

## Tests

### Test File: `tests/test_orchestrator_forced.py`

**Setup:**

- Mock `forced_route` to return canned routes
- Mock model decision function (verified name from Step 3)
- Use verified field names from Steps 1 and 4
- No live model calls

**CRITICAL:** All test code below uses PLACEHOLDERS. Replace with verified values from `verification-findings.md`.

---

#### Test 1: Forced route, no model work → model not called, `attempts: 0`

```python
from unittest.mock import patch
from src.agents.graph.states import State
from src.agents.nodes.orchestrator.node import <verified_main_function_name>  # FROM Step 3

def test_orchestrator_skips_model_on_forced():
    # VERIFIED FIELD NAMES FROM Step 1 and Step 4:
    state = State(
        <verified_flag_field_name>=[],  # e.g., flags=[]
        orchestrator_seen_flag_ids=set(),
        <verified_status_field>="in_progress",  # e.g., status="in_progress"
        <verified_summary_field>="Previous summary"  # e.g., summary="Previous summary"
    )
    
    with patch("src.agents.nodes.orchestrator.node.forced_route") as mock_forced:
        with patch("src.agents.nodes.orchestrator.node.<verified_model_function_name>") as mock_decide:  # FROM Step 3
            mock_forced.return_value = ["planner"]  # VERIFIED node name from Story 2
            
            result = <verified_main_function_name>(state)  # FROM Step 3
            
            mock_decide.assert_not_called()
            
            # VERIFIED step structure FROM Step 3:
            assert result["steps"][0]["title"] == "Next: ['planner']"
            assert result["steps"][0]["result"] == "forced: only one valid route"
            assert result["steps"][0]["evidence"]["attempts"] == 0
            assert result["next"] == ["planner"]
```

---

#### Test 2: Pause required → model called

```python
def test_orchestrator_calls_model_on_pause():
    # VERIFIED pause condition FROM Step 2 (e.g., state.flagged = True):
    state = State(
        <verified_flag_field_name>=[{"<verified_id_field>": "flag1", "<verified_resolved_field>": False}],
        orchestrator_seen_flag_ids=set(),
        <verified_status_field>="in_progress",
        <verified_summary_field>="Previous summary",
        flagged=True  # EXAMPLE pause trigger (adjust to verified condition)
    )
    
    with patch("src.agents.nodes.orchestrator.node.forced_route") as mock_forced:
        with patch("src.agents.nodes.orchestrator.node.<verified_model_function_name>") as mock_decide:
            mock_forced.return_value = [<verified_next_node>]  # Any route
            mock_decide.return_value = {
                "steps": [{"title": "Pause", "result": "Review needed", "evidence": {"attempts": 1}}],
                "next": [<verified_next_node>],
                "summary": "Summary from model"
            }
            
            result = <verified_main_function_name>(state)
            
            mock_decide.assert_called_once()  # Model SHOULD be called
            assert result["steps"][0]["evidence"]["attempts"] == 1
```

**Note:** Adjust pause condition (`flagged=True`) to verified condition from Step 2.

---

#### Test 3: Unseen flag → model called

```python
def test_orchestrator_calls_model_on_unseen_flag():
    # VERIFIED flag schema FROM Step 1:
    state = State(
        <verified_flag_field_name>=[
            {"<verified_id_field>": "flag1", "<verified_resolved_field>": False}
        ],
        orchestrator_seen_flag_ids=set(),  # flag1 is unseen
        <verified_status_field>="in_progress",
        <verified_summary_field>="Previous summary"
    )
    
    with patch("src.agents.nodes.orchestrator.node.forced_route") as mock_forced:
        with patch("src.agents.nodes.orchestrator.node.<verified_model_function_name>") as mock_decide:
            mock_forced.return_value = ["__end__"]  # VERIFIED terminal node from Story 2
            mock_decide.return_value = {
                "steps": [{"title": "Merge", "result": "Flags merged", "evidence": {"attempts": 1}}],
                "next": ["__end__"],
                "summary": "Merged summary"
            }
            
            result = <verified_main_function_name>(state)
            
            mock_decide.assert_called_once()  # Model SHOULD be called
```

---

#### Test 4: Flag IDs extracted before decision

```python
def test_orchestrator_updates_seen_flag_ids():
    state = State(
        <verified_flag_field_name>=[
            {"<verified_id_field>": "flag1", "<verified_resolved_field>": False},
            {"<verified_id_field>": "flag2", "<verified_resolved_field>": True}
        ],
        orchestrator_seen_flag_ids=set(),
        <verified_status_field>="in_progress",
        <verified_summary_field>="Summary"
    )
    
    with patch("src.agents.nodes.orchestrator.node.forced_route", return_value=None):
        with patch("src.agents.nodes.orchestrator.node.<verified_model_function_name>") as mock_decide:
            mock_decide.return_value = {"steps": [], "next": ["planner"]}
            
            result = <verified_main_function_name>(state)
            
            # Verify flag IDs were added to state update
            assert "orchestrator_seen_flag_ids" in result
            assert result["orchestrator_seen_flag_ids"] == {"flag1", "flag2"}
```

---

#### Test 5: Forced hop state fields match model hop

```python
def test_forced_hop_state_fields():
    state = State(
        <verified_flag_field_name>=[],
        orchestrator_seen_flag_ids=set(),
        <verified_status_field>="in_progress",
        <verified_summary_field>="Previous summary",
        <verified_priority_field>="normal",
        <verified_confidence_field>="medium"
    )
    
    with patch("src.agents.nodes.orchestrator.node.forced_route", return_value=["planner"]):
        result = <verified_main_function_name>(state)
        
        # VERIFIED field names FROM Step 4:
        assert result["status"] in ["in_progress", "complete"]  # Depends on route
        assert "flagged" in result
        assert "priority" in result
        assert "confidence" in result
        assert result["summary"] == "Previous summary"  # Unchanged
```

---

#### Test 6: Forced hop evidence structure

```python
def test_forced_hop_evidence_complete():
    state = State(
        <verified_flag_field_name>=[],
        orchestrator_seen_flag_ids=set(),
        <verified_status_field>="in_progress",
        <verified_summary_field>="Summary"
    )
    
    with patch("src.agents.nodes.orchestrator.node.forced_route", return_value=["planner"]):
        result = <verified_main_function_name>(state)
        
        evidence = result["steps"][0]["evidence"]
        assert evidence["attempts"] == 0
        # VERIFIED other evidence fields FROM Step 3 (if any)
```

---

## Implementation Checklist

**BLOCKING PRE-IMPLEMENTATION:**

- [ ] Complete Step 1: Verify flag schema in `states.py`
- [ ] Complete Step 2: Verify pause detection in `interrupts.py`
- [ ] Complete Step 3: Verify orchestrator function structure in `node.py`
