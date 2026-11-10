# LLD — Story 01-agent: State schema for usage tracking and seen-flags

**Story:** State schema for usage tracking and seen-flags  
**Epic:** PER-128 (Skip orchestrator model calls on deterministic hops and track token usage)  
**Repo:** `agent`  
**Version:** 1.1  
**Status:** Draft

---

## Overview

Add two new fields to `State` in `src/agents/graph/states.py`:

1. **`usage: dict[str, dict[str, int]]`** — aggregates token counts per node across all model calls (skill invocations, MCP tool loops, direct gateway calls).
2. **`orchestrator_seen_flag_ids: set[str]`** — tracks which flags the orchestrator has already processed, to avoid redundant model calls for flag merging.

Implement **additive reducer** for `usage` (incoming counts sum into existing counts per node, missing keys initialize to zero) and **union reducer** for `orchestrator_seen_flag_ids` (incoming set unions into existing set).

Handle **backward compatibility**: runs saved before this change (checkpoints without `usage` field) must load and resume without error, with `usage` initialized as an empty dict.

This story is **foundational** — later stories depend on these fields for forced routing logic (Story 2, 3) and usage collection (Story 4, 5, 6).

**CRITICAL PRE-IMPLEMENTATION REQUIREMENT:** Before writing any code, the developer MUST inspect `src/agents/graph/states.py` and document the actual flag schema in `stories/01-agent/verification-findings.md`. Story 3 will read this findings document for flag extraction logic. No code may be written until verification is complete.

---

## Acceptance Criteria

### AC1: `State.usage` schema and reducer

**Given** a state update contains `usage` for a node  
**When** the reducer processes the update  
**Then** the incoming counts (`calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) sum into the existing counts for that node, and missing keys initialize to zero.

**Schema:**

```python
usage: dict[str, dict[str, int]]
# Structure:
# {
#   "node_name": {
#     "calls": int,
#     "input_tokens": int,
#     "output_tokens": int,
#     "cache_creation_input_tokens": int,
#     "cache_read_input_tokens": int
#   }
# }
```

---

### AC2: `State.orchestrator_seen_flag_ids` schema and reducer

**Given** a state update contains `orchestrator_seen_flag_ids`  
**When** the reducer processes the update  
**Then** the incoming set of flag IDs unions into the existing set.

**Schema:**

```python
orchestrator_seen_flag_ids: set[str]
# Structure: {"flag_id_1", "flag_id_2", ...}
# Default: empty set
```

---

### AC3: Backward compatibility on checkpoint load

**Given** a checkpoint saved before `usage` field existed (field is absent)  
**When** the checkpoint is loaded  
**Then** the state initializes with `usage` as an empty dict `{}`, and the run resumes without error.

**Given** a checkpoint saved with `usage` field present  
**When** the checkpoint is loaded  
**Then** the state preserves the existing `usage` counts, and the run resumes normally.

---

### AC4: Flag schema documented in verification findings

**Given** the developer begins implementing Story 1  
**When** they inspect `src/agents/graph/states.py`  
**Then** they document the actual flag field name, dict structure (ID field name, resolved field name), and example flag dict in `stories/01-agent/verification-findings.md` before writing any code, so Story 3 can reference this document without re-deriving the schema.

---

## Verification Checklist (MANDATORY BLOCKING GATE)

**NO CODE MAY BE WRITTEN UNTIL THIS CHECKLIST IS COMPLETE AND DOCUMENTED IN `stories/01-agent/verification-findings.md`.**

### Step 1: Verify flag schema in `State`

**Tasks:**

- [ ] Read `src/agents/graph/states.py` (full file or search for class `State`)
- [ ] Document actual flag field name (search for "flag" in the file—is it `flags`, `open_flags`, `active_flags`, `flagged`, other?)
- [ ] Document flag storage type: list of dicts? dict keyed by ID? other?
- [ ] If list of dicts, document flag dict structure: what keys exist? (e.g., `id`/`flag_id`, `resolved`/`is_resolved`, `message`, other?)
- [ ] Provide example flag from code or docstring
- [ ] Document how to check if a flag is unresolved (e.g., `flag["resolved"] == False` or other pattern)

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

**Flag extraction pattern for orchestrator (for Story 3):**
```python
# Example pattern to extract flag IDs from state
current_flags = state.<verified_flag_field_name>
flag_ids = {flag.get("<verified_id_field>") for flag in current_flags if "<verified_id_field>" in flag}
```

**Unresolved flag check pattern (for Story 3):**
```python
# Example pattern to filter unresolved flags
unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
```
```

---

### Step 2: Verify reducer registration pattern

**Tasks:**

- [ ] Read existing field definitions in `src/agents/graph/states.py` to determine reducer registration pattern
- [ ] Document pattern: `Annotated[type, reducer_func]`, `__reduce_key__` dict, or other?
- [ ] Show example from existing field (copy relevant code snippet)

**Required findings in `verification-findings.md`:**

```markdown
## Step 2: Reducer Registration Pattern

**File inspected:** `src/agents/graph/states.py` (lines X-Y)

**Reducer registration pattern:** (Annotated / __reduce_key__ dict / other)

**Example from existing field:**
```python
# Copy relevant code snippet showing how an existing field registers its reducer
# e.g., existing_field: Annotated[type, reducer_func] = field(default_factory=...)
```

**Pattern to use for `usage` and `orchestrator_seen_flag_ids`:**
```python
# Based on observed pattern above
```
```

---

### Step 3: Verify checkpoint loading mechanism

**Tasks:**

- [ ] Determine where checkpoint loading happens: `src/agents/graph/states.py`, `src/agents/runtime/checkpointer.py`, or other file?
- [ ] Document how missing fields are handled on load
- [ ] Identify where migration logic should be added

**Required findings in `verification-findings.md`:**

```markdown
## Step 3: Checkpoint Loading Mechanism

**Checkpoint loading location:** (`src/agents/graph/states.py` / `src/agents/runtime/checkpointer.py` / other)

**How missing fields are handled:**
```python
# Copy relevant code snippet showing existing migration logic (if any)
# Or: "No explicit migration logic found — fields default via field(default_factory=...)"
```

**Where to add migration for `usage` and `orchestrator_seen_flag_ids`:**
- Option 1: In `State.__post_init__` (if checkpoint dict is passed to State constructor)
- Option 2: In checkpointer's load function (if separate load logic exists)
- Option 3: No migration needed (LangGraph handles missing fields via defaults)
```

---

### Step 4: Confirm field names don't conflict

**Tasks:**

- [ ] Search `src/agents/graph/states.py` for existing fields named `usage` or `orchestrator_seen_flag_ids`
- [ ] Confirm no naming conflicts

**Required findings in `verification-findings.md`:**

```markdown
## Step 4: Field Name Conflicts

**Search results for "usage":** (FOUND / NOT FOUND)
- If FOUND: describe conflict and propose alternative name

**Search results for "orchestrator_seen_flag_ids":** (FOUND / NOT FOUND)
- If FOUND: describe conflict and propose alternative name

**Conclusion:** Field names `usage` and `orchestrator_seen_flag_ids` are (SAFE / CONFLICTING).
```

---

### Step 5: Document implementation plan

**Tasks:**

- [ ] Based on findings from Steps 1-4, write concrete implementation notes
- [ ] Adjust any placeholder code in this LLD to match verified patterns

**Required section in `verification-findings.md`:**

```markdown
## Step 5: Implementation Decisions

**Reducer registration approach (from Step 2):**
```python
# Exact pattern to use for usage field
usage: Annotated[dict[str, dict[str, int]], _reduce_usage] = field(default_factory=dict)

# Exact pattern to use for orchestrator_seen_flag_ids field
orchestrator_seen_flag_ids: Annotated[set[str], _reduce_orchestrator_seen_flag_ids] = field(default_factory=set)
```

**Migration approach (from Step 3):**
- Location: (State.__post_init__ / checkpointer.py / no migration needed)
- Code:
```python
# If migration needed, show exact location and code
# If not needed, state: "No migration code required — LangGraph defaults handle missing fields"
```

**Flag extraction pattern for Story 3 (from Step 1):**
```python
# Concrete pattern Story 3 will use (not placeholder)
current_flags = state.<verified_flag_field_name>
flag_ids = {flag.get("<verified_id_field>") for flag in current_flags if "<verified_id_field>" in flag}
unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
```

**Confirmed safe field names:**
- `usage` (no conflict)
- `orchestrator_seen_flag_ids` (no conflict)
```

---

**BLOCKING RULE:** The developer MUST NOT proceed to File Plan or Implementation until `stories/01-agent/verification-findings.md` exists with all five steps completed. Code review WILL REJECT any PR that skips verification or uses placeholder values from this LLD without verifying them against actual code.

---

## File Plan

### Modified Files

#### `src/agents/graph/states.py`

**Changes:**

1. Add `usage: dict[str, dict[str, int]]` field to `State` dataclass.
2. Add `orchestrator_seen_flag_ids: set[str]` field to `State` dataclass.
3. Implement `_reduce_usage` reducer function:
   - For each node in the incoming update, sum `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` into the existing counts for that node.
   - If a key is missing in the existing counts, initialize it to zero before summing.
4. Implement `_reduce_orchestrator_seen_flag_ids` reducer function:
   - Union the incoming set into the existing set.
5. Register reducers in the `State` class using pattern verified in Step 2.
6. Add migration logic if required per Step 3 findings (may be in this file or in `src/agents/runtime/checkpointer.py`).

**Implementation sketch (ADJUST AFTER VERIFICATION):**

```python
from typing import Annotated
from dataclasses import field

def _reduce_usage(
    existing: dict[str, dict[str, int]],
    incoming: dict[str, dict[str, int]]
) -> dict[str, dict[str, int]]:
    """Sum incoming token counts into existing counts per node."""
    result = existing.copy()
    for node, counts in incoming.items():
        if node not in result:
            result[node] = {
                "calls": 0,
                "input_tokens": 0,
                "output_tokens": 0,
                "cache_creation_input_tokens": 0,
                "cache_read_input_tokens": 0
            }
        for key in ["calls", "input_tokens", "output_tokens", 
                    "cache_creation_input_tokens", "cache_read_input_tokens"]:
            result[node][key] = result[node].get(key, 0) + counts.get(key, 0)
    return result

def _reduce_orchestrator_seen_flag_ids(
    existing: set[str],
    incoming: set[str]
) -> set[str]:
    """Union incoming flag IDs into existing set."""
    return existing | incoming

@dataclass
class State:
    # ... existing fields ...
    
    # ADJUST PATTERN BASED ON STEP 2 VERIFICATION:
    usage: Annotated[
        dict[str, dict[str, int]],
        _reduce_usage
    ] = field(default_factory=dict)
    
    orchestrator_seen_flag_ids: Annotated[
        set[str],
        _reduce_orchestrator_seen_flag_ids
    ] = field(default_factory=set)
    
    # ... rest of State definition ...
```

**Migration logic (location determined by Step 3 verification):**

```python
# Example if migration is in checkpointer.py:
def load_checkpoint(checkpoint_dict: dict) -> State:
    # Ensure 'usage' exists
    if "usage" not in checkpoint_dict:
        checkpoint_dict["usage"] = {}
    # Ensure 'orchestrator_seen_flag_ids' exists
    if "orchestrator_seen_flag_ids" not in checkpoint_dict:
        checkpoint_dict["orchestrator_seen_flag_ids"] = set()
    return State(**checkpoint_dict)

# Example if migration is in State.__post_init__:
@dataclass
class State:
    # ... fields ...
    
    def __post_init__(self):
        # Handle missing fields from old checkpoints
        if not hasattr(self, 'usage'):
            self.usage = {}
        if not hasattr(self, 'orchestrator_seen_flag_ids'):
            self.orchestrator_seen_flag_ids = set()
```

**CRITICAL NOTE:** The exact implementation pattern MUST match verification findings. Do not use the sketches above verbatim — adjust based on Steps 2 and 3 findings.

---

## Interfaces

### `_reduce_usage`

**Signature:**

```python
def _reduce_usage(
    existing: dict[str, dict[str, int]],
    incoming: dict[str, dict[str, int]]
) -> dict[str, dict[str, int]]
```

**Behavior:**

- For each `node` in `incoming`:
  - If `node` not in `existing`, initialize `existing[node]` with all keys set to zero.
  - For each key (`calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`):
    - Sum `incoming[node][key]` (default 0 if missing) into `existing[node][key]` (default 0 if missing).
- Return the updated `existing` dict.

**Contract:**

- Incoming counts always **add** to existing counts (never replace).
- Missing keys in either `existing` or `incoming` default to zero.
- The reducer is **commutative** (order of updates doesn't matter) and **associative** (multiple updates can be batched).

---

### `_reduce_orchestrator_seen_flag_ids`

**Signature:**

```python
def _reduce_orchestrator_seen_flag_ids(
    existing: set[str],
    incoming: set[str]
) -> set[str]
```

**Behavior:**

- Return `existing | incoming` (set union).

**Contract:**

- The reducer is **idempotent** (applying the same update twice has no effect) and **commutative**.
- Empty incoming set is a no-op.

---

### Checkpoint Migration

**Location:** Determined by Step 3 verification (`src/agents/runtime/checkpointer.py` or `State.__post_init__` or not needed).

**Behavior:**

- On checkpoint load, if `usage` key is absent in the checkpoint dict, initialize it to `{}` before constructing `State`.
- On checkpoint load, if `orchestrator_seen_flag_ids` key is absent, initialize it to `set()` before constructing `State`.

**Contract:**

- Old checkpoints (pre-PER-128) load without error.
- New runs start with `usage = {}` and `orchestrator_seen_flag_ids = set()`.

---

## Algorithm Notes

### Usage Accumulation

**Context:** Model calls happen at three sites (per AD-4):

1. `run_skill` in `src/agents/skills/client.py` (including JSON retries)
2. MCP tool loop in `src/agents/tools/bindings.py` (multiple rounds + final no-tools call)
3. Direct `ModelGateway.complete` in `src/agents/model/gateway.py` (when not wrapped by above)

**Flow:**

1. Call site extracts usage from model response: `{input_tokens: ..., output_tokens: ..., cache_creation_input_tokens: ..., cache_read_input_tokens: ...}`.
2. Dispatch state update: `{usage: {node_name: {calls: 1, input_tokens: ..., output_tokens: ..., cache_creation_input_tokens: ..., cache_read_input_tokens: ...}}}`.
3. `_reduce_usage` sums the new counts into `State.usage[node_name]`.

**Invariants:**

- Each model call increments `calls` by 1.
- Token counts are non-negative integers.
- Retries and tool loop rounds all contribute to the same node's totals.

---

### Seen Flags Tracking

**Context:** The orchestrator's model merges duplicate flags. To avoid redundant merges, track which flags the orchestrator has already seen (per AD-3).

**Flow:**

1. Before calling `_decide`, the orchestrator extracts flag IDs from the current state using the pattern documented in Step 1 verification findings.
2. Dispatch state update: `{orchestrator_seen_flag_ids: {flag_id_1, flag_id_2, ...}}`.
3. `_reduce_orchestrator_seen_flag_ids` unions the new IDs into `State.orchestrator_seen_flag_ids`.
4. When deciding whether to force a route (Story 3), check if any unresolved flag's ID is **not** in `orchestrator_seen_flag_ids`. If so, model work is required.

**Flag extraction pattern (from Step 1 findings):**

The exact pattern is documented in `stories/01-agent/verification-findings.md` Step 1. Story 3 will read this document for the concrete implementation.

**Invariants:**

- Flag IDs are strings (assumed unique per flag).
- Once a flag ID is in `orchestrator_seen_flag_ids`, it stays there for the run's lifetime.
- Resolved flags may be removed from the state's flag field, but their IDs remain in `orchestrator_seen_flag_ids` (no harm — just a slightly larger set).

---

## Edge Cases

### Edge Case 1: Reducer receives update with missing keys

**Scenario:** Incoming `usage` update for a node has `calls` and `input_tokens`, but no `output_tokens`.

**Handling:**

- `_reduce_usage` initializes missing keys to zero: `incoming[node].get("output_tokens", 0)` → 0.
- Existing counts for that key are unchanged.

**Test:** Unit test verifies missing keys default to zero.

---

### Edge Case 2: Reducer receives update for a new node

**Scenario:** First usage update for a node (node not yet in `existing`).

**Handling:**

- `_reduce_usage` initializes `existing[node]` with all keys set to zero, then sums incoming counts.

**Test:** Unit test verifies first update creates node entry with correct counts.

---

### Edge Case 3: Multiple updates for the same node in one reducer call

**Scenario:** LangGraph batches multiple state updates; reducer receives `incoming` with counts for the same node from two different calls.

**Handling:**

- If LangGraph calls the reducer once per update (typical), this doesn't happen.
- If it batches, the reducer still sums correctly (each `incoming` is processed in sequence).

**Assumption:** LangGraph reducers are called once per state update dispatch. This is the standard LangGraph behavior. If your implementation differs, the reducer logic is still correct (additive).

---

### Edge Case 4: Checkpoint load without `usage` field (backward compatibility)

**Scenario:** A checkpoint saved before this change has no `usage` key.

**Handling:**

- Migration logic (location per Step 3 findings) detects missing `usage` and initializes it to `{}` before or during `State` construction.
- The run resumes with an empty `usage` dict and starts tracking from that point forward.

**Test:** Unit test loads a checkpoint dict without `usage`, verifies `state.usage == {}` and run continues.

---

### Edge Case 5: Checkpoint load with `usage` field (normal case)

**Scenario:** A checkpoint saved after this change has `usage` with existing counts.

**Handling:**

- Checkpoint load preserves `usage` as-is.
- Subsequent updates sum into the existing counts.

**Test:** Unit test loads a checkpoint with `usage = {"orchestrator": {"calls": 5, "input_tokens": 1000, ...}}`, dispatches a new update, verifies counts sum correctly.

---

### Edge Case 6: Empty set union (seen-flags)

**Scenario:** Incoming `orchestrator_seen_flag_ids` is an empty set.

**Handling:**

- `_reduce_orchestrator_seen_flag_ids` returns `existing | set()` → `existing` (no change).

**Test:** Unit test verifies empty incoming set is a no-op.

---

### Edge Case 7: Duplicate flag IDs (seen-flags)

**Scenario:** The same flag ID appears in multiple updates.

**Handling:**

- Set union is idempotent: `{a, b} | {b, c}` → `{a, b, c}`.
- Duplicate IDs have no effect.

**Test:** Unit test verifies duplicate IDs don't cause errors or extra entries.

---

### Edge Case 8: Flag schema differs from HLD assumptions

**Scenario:** After Step 1 verification, flags are stored in `state.active_flags` with ID field `flag_id`, not `state.flags` with `id`.

**Handling:**

- All references to flags in this story and downstream stories use the verified schema from Step 1 findings.
- No hardcoded assumptions; everything adapts to verification results.

**Mitigation:**

- Step 1 verification prevents runtime discovery of schema mismatches.
- Story 3 reads the findings document and uses the verified extraction pattern.

**Test:**

- Tests use the verified flag schema from Step 1 findings (not assumed schema).

---

## Tests

### Test File: `tests/test_usage_reducer.py`

#### Test 1: `usage` reducer sums counts for a single node across multiple calls

```python
def test_usage_reducer_single_node_multiple_calls():
    reducer = _reduce_usage
    existing = {
        "orchestrator": {
            "calls": 2,
            "input_tokens": 500,
            "output_tokens": 100,
            "cache_creation_input_tokens": 0,
            "cache_read_input_tokens": 400
        }
    }
    incoming = {
        "orchestrator": {
            "calls": 1,
            "input_tokens": 300,
            "output_tokens": 50,
            "cache_creation_input_tokens": 0,
            "cache_read_input_tokens": 200
        }
    }
    result = reducer(existing, incoming)
    assert result["orchestrator"]["calls"] == 3
    assert result["orchestrator"]["input_tokens"] == 800
    assert result["orchestrator"]["output_tokens"] == 150
    assert result["orchestrator"]["cache_creation_input_tokens"] == 0
    assert result["orchestrator"]["cache_read_input_tokens"] == 600
```

---

#### Test 2: `usage` reducer sums counts for multiple nodes

```python
def test_usage_reducer_multiple_nodes():
    reducer = _reduce_usage
    existing = {
        "orchestrator": {"calls": 2, "input_tokens": 500, "output_tokens": 100, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 400}
    }
    incoming = {
        "planner": {"calls": 1, "input_tokens": 200, "output_tokens": 30, "cache_creation_input_tokens": 50, "cache_read_input_tokens": 150},
        "orchestrator": {"calls": 1, "input_tokens": 100, "output_tokens": 20, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 80}
    }
    result = reducer(existing, incoming)
    assert result["orchestrator"]["calls"] == 3
    assert result["orchestrator"]["input_tokens"] == 600
    assert result["planner"]["calls"] == 1
    assert result["planner"]["input_tokens"] == 200
```

---

#### Test 3: `usage` reducer initializes missing keys to zero

```python
def test_usage_reducer_missing_keys_initialize_to_zero():
    reducer = _reduce_usage
    existing = {
        "orchestrator": {"calls": 1, "input_tokens": 100}  # missing other keys
    }
    incoming = {
        "orchestrator": {"output_tokens": 50}  # missing other keys
    }
    result = reducer(existing, incoming)
    assert result["orchestrator"]["calls"] == 1  # unchanged (incoming has no calls)
    assert result["orchestrator"]["input_tokens"] == 100  # unchanged
    assert result["orchestrator"]["output_tokens"] == 50  # added from incoming
    assert result["orchestrator"]["cache_creation_input_tokens"] == 0  # initialized
    assert result["orchestrator"]["cache_read_input_tokens"] == 0  # initialized
```

---

### Test File: `tests/test_seen_flags_reducer.py`

#### Test 4: `orchestrator_seen_flag_ids` reducer unions incoming set into empty set

```python
def test_seen_flags_reducer_empty_initial_set():
    reducer = _reduce_orchestrator_seen_flag_ids
    existing = set()
    incoming = {"flag_1", "flag_2"}
    result = reducer(existing, incoming)
    assert result == {"flag_1", "flag_2"}
```

---

#### Test 5: `orchestrator_seen_flag_ids` reducer unions incoming set into existing set

```python
def test_seen_flags_reducer_existing_set_extended():
    reducer = _reduce_orchestrator_seen_flag_ids
    existing = {"flag_1", "flag_2"}
    incoming = {"flag_2", "flag_3"}
    result = reducer(existing, incoming)
    assert result == {"flag_1", "flag_2", "flag_3"}
```

---

### Test File: `tests/test_state_migration.py`

#### Test 6: Load checkpoint without `usage` field → state initializes with empty dict

```python
def test_load_checkpoint_without_usage():
    # Mock checkpoint dict (pre-PER-128)
    checkpoint_dict = {
        "service_order": {"id": "SO-1003"},
        "status": "in_progress"
        # no 'usage' key
    }
    # Simulate load logic (adjust based on Step 3 findings)
    if "usage" not in checkpoint_dict:
        checkpoint_dict["usage"] = {}
    if "orchestrator_seen_flag_ids" not in checkpoint_dict:
        checkpoint_dict["orchestrator_seen_flag_ids"] = set()
    
    state = State(**checkpoint_dict)
    assert state.usage == {}
    assert state.orchestrator_seen_flag_ids == set()
```

---

#### Test 7: Load checkpoint with `usage` field → state preserves existing counts

```python
def test_load_checkpoint_with_usage():
    checkpoint_dict = {
        "service_order": {"id": "SO-1003"},
        "status": "in_progress",
        "usage": {
            "orchestrator": {
                "calls": 5,
                "input_tokens": 1000,
                "output_tokens": 200,
                "cache_creation_input_tokens": 0,
                "cache_read_input_tokens": 800
            }
        },
        "orchestrator_seen_flag_ids": {"flag_1"}
    }
    state = State(**checkpoint_dict)
    assert state.usage["orchestrator"]["calls"] == 5
    assert state.usage["orchestrator"]["input_tokens"] == 1000
    assert state.orchestrator_seen_flag_ids == {"flag_1"}
```

---

## Implementation Checklist

**BLOCKING PRE-IMPLEMENTATION:**

- [ ] Complete Step 1: Verify flag schema in `src/agents/graph/states.py`
- [ ] Complete Step 2: Verify reducer registration pattern
- [ ] Complete Step 3: Verify checkpoint loading mechanism
- [ ] Complete Step 4: Confirm no field name conflicts
- [ ] Complete Step 5: Document implementation decisions in `stories/01-agent/verification-findings.md`
- [ ] **BLOCK: Code review rejects PR if verification-findings.md is missing or incomplete**

**Implementation (after verification complete):**

- [ ] Add `usage: dict[str, dict[str, int]]` field to `State` in `src/agents/graph/states.py` using verified pattern from Step 2
- [ ] Add `orchestrator_seen_flag_ids: set[str]` field to `State` using verified pattern from Step 2
- [ ] Implement `_reduce_usage` reducer function
- [ ] Implement `_reduce_orchestrator_seen_flag_ids` reducer function
- [ ] Register reducers using verified pattern from Step 2
- [ ] Add migration logic per Step 3 findings (if needed)
- [ ] Write unit tests in `tests/test_usage_reducer.py` (Tests 1–3)
- [ ] Write unit tests in `tests/test_seen_flags_reducer.py` (Tests 4–5)
- [ ] Write unit tests in `tests/test_state_migration.py` (Tests 6–7) using verified migration pattern from Step 3
- [ ] Verify all 7 tests pass (`uv run pytest tests/test_usage_reducer.py tests/test_seen_flags_reducer.py tests/test_state_migration.py -v`)
- [ ] Verify existing tests pass (`uv run pytest -q`)

---

## Dependencies

**Depends on:** (none — this is the foundational story)

**Blocks:**

- Story 2 (Forced route logic) — needs `State.usage` schema to exist (though not used yet)
- Story 3 (Orchestrator guards) — needs `State.orchestrator_seen_flag_ids` for unseen flag checks, and reads flag extraction pattern from this story's verification findings
- Story 4 (Usage collection in run_skill/MCP) — needs `usage` reducer to aggregate counts
- Story 5 (Usage collection in gateway) — needs `usage` reducer
- Story 6 (Logging and API) — needs `State.usage` to report totals

---

## Notes

- **Reducer pattern:** Exact pattern determined by Step 2 verification. Do not assume `Annotated[type, reducer_func]` — verify from existing code.
- **Checkpoint location:** Determined by Step 3 verification. Migration logic may be in `State.__post_init__`, `checkpointer.py`, or not needed at all if LangGraph handles defaults automatically.
- **Default values:** `field(default_factory=dict)` and `field(default_factory=set)` ensure new runs start with empty `usage` and
