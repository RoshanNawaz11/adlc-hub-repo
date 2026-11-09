# Story 01-agent LLD

**Story:** State schema for usage tracking and seen flags

**Repo:** `app`

**Epic:** PER-119

---

## Acceptance Criteria

From requirements and breakdown only:

1. **AC-1 (REQ-2 AC-2.2):** `State` has `usage` field: `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`
2. **AC-2 (REQ-2 AC-2.4):** A run saved before this change loads and resumes successfully (backward compatibility: missing `usage` field does not break loading)
3. **AC-3 (REQ-3 AC-3.4):** The orchestrator tracks which flags it has seen in a new state field

---

## HLD Binding Outcomes

From `hld-per-119.md`:

- **AD-1:** `State.usage` is `dict[str, dict[str, int]]` where outer key is node name, inner keys are `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`. `State.orchestrator_seen_flags` is `set[str]` (flag IDs). Runs saved before this change load as if those fields were `{}` and `set()` respectively.
- **AD-2:** The reducer for `usage` deep-merges per node: if `state.usage[node]` exists, add incoming counts to existing counts; otherwise, set `state.usage[node] = incoming_counts`. The reducer for `orchestrator_seen_flags` is set union: `state.orchestrator_seen_flags |= incoming_flags`.
- **AD-6:** Backward compatibility (initializing missing `usage` and `orchestrator_seen_flags` fields to `{}` and `set()` respectively) is implemented in the state deserialization logic. Common patterns: Pydantic model with `@model_validator(mode='before')`, dataclass with custom `__post_init__`, or manual deserialization function. The implementation must locate the state deserialization code and add the initialization logic there. Initialization must run during deserialization (when loading a saved run), before the state is passed to reducers or used by nodes.

---

## File Plan

**Note:** Repo tree for `app` is not provided. The story specifies `src/agents/state.py` as the likely location; implementation must verify the actual path in the repo and use it.

### Modified Files

#### `src/agents/state.py` (or actual state definition location)

**Changes:**

1. **Add new fields to `State` class/schema:**
   - `usage: dict[str, dict[str, int]] = field(default_factory=dict)` — initialized to `{}` by default
   - `orchestrator_seen_flags: set[str] = field(default_factory=set)` — initialized to `set()` by default

2. **Add reducer for `usage`:**
   - Deep-merge per node: for each node in incoming update's `usage`, add counts to existing `state.usage[node]` or initialize if node not present
   - Counts to merge: `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
   - Missing counts in incoming update default to `0` (via `.get(count_key, 0)`)

3. **Add reducer for `orchestrator_seen_flags`:**
   - Set union: `state.orchestrator_seen_flags |= incoming_flags`
   - If `incoming_flags` is missing from the update, leave `state.orchestrator_seen_flags` unchanged

4. **Backward compatibility in state deserialization:**
   - Locate the state deserialization code (where a saved run is loaded from storage). Common patterns:
     - **Pydantic model:** Add `@model_validator(mode='before')` to the `State` class that initializes missing fields before validation
     - **Dataclass:** Add initialization logic in `__post_init__` that checks for missing fields and sets defaults
     - **Manual deserialization function:** Add initialization logic to the function that deserializes state from storage (e.g., `load_run_state(thread_id)`)
   - Initialize missing fields:
     ```python
     if "usage" not in loaded_state:
         loaded_state["usage"] = {}
     if "orchestrator_seen_flags" not in loaded_state:
         loaded_state["orchestrator_seen_flags"] = set()
     ```
   - This initialization must run **before** the state is passed to reducers or used by nodes

**Algorithm notes:**

- **Usage reducer logic (per node):**
  ```python
  # In the reducer that processes state updates (exact location depends on existing reducer system):
  
  def reduce_usage(current_usage: dict, incoming_update: dict) -> dict:
      """
      Deep-merge usage counts per node.
      
      Args:
          current_usage: state.usage (dict[node_name, dict[count_key, int]])
          incoming_update: update payload with {"usage": {node_name: counts_dict}}
      
      Returns:
          Updated usage dict
      """
      updated_usage = current_usage.copy()  # shallow copy; we'll update node dicts in place
      
      incoming_usage = incoming_update.get("usage", {})
      
      for node_name, incoming_counts in incoming_usage.items():
          if node_name not in updated_usage:
              # Initialize node with all counts at 0
              updated_usage[node_name] = {
                  "calls": 0,
                  "input_tokens": 0,
                  "output_tokens": 0,
                  "cache_creation_input_tokens": 0,
                  "cache_read_input_tokens": 0
              }
          
          # Add incoming counts to existing counts (missing counts default to 0)
          for count_key in ["calls", "input_tokens", "output_tokens", 
                            "cache_creation_input_tokens", "cache_read_input_tokens"]:
              updated_usage[node_name][count_key] += incoming_counts.get(count_key, 0)
      
      return updated_usage
  ```

- **Seen flags reducer logic:**
  ```python
  def reduce_orchestrator_seen_flags(current_flags: set, incoming_update: dict) -> set:
      """
      Set union for orchestrator seen flags.
      
      Args:
          current_flags: state.orchestrator_seen_flags (set[str])
          incoming_update: update payload with {"orchestrator_seen_flags": set[str]}
      
      Returns:
          Updated seen flags set
      """
      incoming_flags = incoming_update.get("orchestrator_seen_flags", set())
      return current_flags | incoming_flags
  ```

- **Backward compatibility (deserialization) — Pydantic example:**
  ```python
  from pydantic import BaseModel, field_validator, model_validator
  from typing import Dict, Set
  
  class State(BaseModel):
      usage: Dict[str, Dict[str, int]] = {}
      orchestrator_seen_flags: Set[str] = set()
      # ... other fields
      
      @model_validator(mode='before')
      @classmethod
      def initialize_missing_fields(cls, data: dict) -> dict:
          """Initialize missing fields for backward compatibility."""
          if "usage" not in data:
              data["usage"] = {}
          if "orchestrator_seen_flags" not in data:
              data["orchestrator_seen_flags"] = set()
          return data
  ```

- **Backward compatibility (deserialization) — Dataclass example:**
  ```python
  from dataclasses import dataclass, field
  from typing import Dict, Set
  
  @dataclass
  class State:
      usage: Dict[str, Dict[str, int]] = field(default_factory=dict)
      orchestrator_seen_flags: Set[str] = field(default_factory=set)
      # ... other fields
      
      def __post_init__(self):
          """Initialize missing fields for backward compatibility."""
          # When loaded from storage, fields may be None instead of missing
          if self.usage is None:
              self.usage = {}
          if self.orchestrator_seen_flags is None:
              self.orchestrator_seen_flags = set()
  ```

- **Backward compatibility (deserialization) — Manual loader example:**
  ```python
  def load_run_state(thread_id: str) -> State:
      """Load a run state from storage, initializing missing fields."""
      # Load serialized state from storage (e.g., database, file, cache)
      serialized_state = load_from_storage(thread_id)
      
      # Initialize missing fields for backward compatibility
      if "usage" not in serialized_state:
          serialized_state["usage"] = {}
      if "orchestrator_seen_flags" not in serialized_state:
          serialized_state["orchestrator_seen_flags"] = set()
      
      # Deserialize into State object
      state = State(**serialized_state)
      return state
  ```

- **Implementation guidance:** The developer must:
  1. Locate the `State` class/schema definition (likely `src/agents/state.py` or similar)
  2. Add the two new fields (`usage`, `orchestrator_seen_flags`) with default factories
  3. Locate the reducer registration mechanism (e.g., a reducer registry, state update handler)
  4. Add the two reducers (`reduce_usage`, `reduce_orchestrator_seen_flags`) to the mechanism
  5. Locate the state deserialization code (where saved runs are loaded from storage)
  6. Add backward compatibility initialization to the deserialization code (before validation/construction)

---

## Interfaces

### State Schema (output)

**`State` fields added:**

```python
usage: dict[str, dict[str, int]]
# Outer key: node name (e.g., "orchestrator", "planner", "agent-analyst")
# Inner dict: {
#   "calls": int,
#   "input_tokens": int,
#   "output_tokens": int,
#   "cache_creation_input_tokens": int,
#   "cache_read_input_tokens": int
# }

orchestrator_seen_flags: set[str]
# Set of flag IDs (strings) the orchestrator has seen
```

**Example state after updates:**

```python
{
  "usage": {
    "orchestrator": {
      "calls": 3,
      "input_tokens": 4500,
      "output_tokens": 150,
      "cache_creation_input_tokens": 1200,
      "cache_read_input_tokens": 800
    },
    "planner": {
      "calls": 1,
      "input_tokens": 2000,
      "output_tokens": 100,
      "cache_creation_input_tokens": 0,
      "cache_read_input_tokens": 500
    }
  },
  "orchestrator_seen_flags": {"flag-1", "flag-2", "flag-3"}
}
```

### State Update (input to reducers)

**Usage update payload:**

```python
{
  "usage": {
    "orchestrator": {
      "calls": 1,
      "input_tokens": 1000,
      "output_tokens": 50,
      "cache_creation_input_tokens": 500,
      "cache_read_input_tokens": 200
    }
  }
}
```

**Seen flags update payload:**

```python
{
  "orchestrator_seen_flags": {"flag-1", "flag-2"}
}
```

---

## Tests

From breakdown Verification Approach (4 tests, exact count matching breakdown):

### Test 1: Usage accumulates per node across updates

**Mapped from:** Breakdown verification item 1

**Setup:**
- Initialize state with `usage = {}`
- Send first update: `{"usage": {"orchestrator": {"calls": 1, "input_tokens": 1000, "output_tokens": 50, "cache_creation_input_tokens": 500, "cache_read_input_tokens": 200}}}`
- Apply reducer: `state.usage = reduce_usage(state.usage, first_update)`
- Send second update: `{"usage": {"orchestrator": {"calls": 1, "input_tokens": 500, "output_tokens": 25, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 100}}}`
- Apply reducer: `state.usage = reduce_usage(state.usage, second_update)`

**Expected outcome:**
- `state.usage["orchestrator"]` equals `{"calls": 2, "input_tokens": 1500, "output_tokens": 75, "cache_creation_input_tokens": 500, "cache_read_input_tokens": 300}`

**Verification:**
- AC-1: State has `usage` field with correct structure
- Reducer deep-merges per node (adds incoming counts to existing counts)
- Missing counts in incoming update default to 0 (second update has `cache_creation_input_tokens: 0`, which is added correctly)

---

### Test 2: Orchestrator seen flags accumulate across updates

**Mapped from:** Breakdown verification item 2

**Setup:**
- Initialize state with `orchestrator_seen_flags = set()`
- Send first update: `{"orchestrator_seen_flags": {"flag-1", "flag-2"}}`
- Apply reducer: `state.orchestrator_seen_flags = reduce_orchestrator_seen_flags(state.orchestrator_seen_flags, first_update)`
- Send second update: `{"orchestrator_seen_flags": {"flag-2", "flag-3"}}`
- Apply reducer: `state.orchestrator_seen_flags = reduce_orchestrator_seen_flags(state.orchestrator_seen_flags, second_update)`

**Expected outcome:**
- `state.orchestrator_seen_flags` equals `{"flag-1", "flag-2", "flag-3"}`

**Verification:**
- AC-3: State tracks which flags orchestrator has seen
- Reducer performs set union (no duplicates, all IDs accumulated)
- Duplicate flag ID `"flag-2"` appears in both updates but only once in final set

---

### Test 3: Run saved without `usage` field loads successfully

**Mapped from:** Breakdown verification item 3

**Setup:**
- Create a saved run state (serialized dict) with no `usage` field:
  ```python
  serialized_state = {
      "thread_id": "test-123",
      "status": "paused",
      "orchestrator_seen_flags": {"flag-1"}
      # "usage" is missing
  }
  ```
- Serialize to storage format (JSON, pickle, or whatever the codebase uses)
- Load the run state using the state deserialization logic (e.g., `State.model_validate(serialized_state)` for Pydantic, or `State(**serialized_state)` for dataclass, or `load_run_state(thread_id)` for manual loader)

**Expected outcome:**
- State loads without error (no `KeyError`, no validation error)
- `state.usage` is initialized to `{}`
- Other fields (`thread_id`, `status`, `orchestrator_seen_flags`) load correctly

**Verification:**
- AC-2: Backward compatibility — missing `usage` field does not break loading
- State can resume: send an update with `usage` and verify reducer works correctly (no crash, counts accumulate from `{}`)

---

### Test 4: Run saved without `orchestrator_seen_flags` loads successfully

**Mapped from:** Breakdown verification item 4

**Setup:**
- Create a saved run state (serialized dict) with no `orchestrator_seen_flags` field:
  ```python
  serialized_state = {
      "thread_id": "test-456",
      "status": "paused",
      "usage": {"orchestrator": {"calls": 1, "input_tokens": 1000, "output_tokens": 50, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 0}}
      # "orchestrator_seen_flags" is missing
  }
  ```
- Serialize to storage format
- Load the run state using the state deserialization logic

**Expected outcome:**
- State loads without error
- `state.orchestrator_seen_flags` is initialized to `set()`
- Other fields (`thread_id`, `status`, `usage`) load correctly

**Verification:**
- AC-2: Backward compatibility — missing field does not break loading
- AC-3: State field for orchestrator seen flags exists and works
- State can resume: send an update with `orchestrator_seen_flags` and verify reducer works correctly (no crash, IDs accumulate from `set()`)

---

## Notes

- **Repo tree not provided:** Implementation must verify the actual path of the state definition file in the `app` repo. The story specifies `src/agents/state.py` as the likely location.
- **Reducer registration mechanism unknown:** The story does not specify where reducers are registered or defined. Implementation must locate the existing reducer registration mechanism (e.g., a reducer registry in a state manager class, a state update handler function that dispatches to reducer functions, or a framework-specific pattern like Redux-style reducers). Add the two new reducers (`reduce_usage`, `reduce_orchestrator_seen_flags`) to this mechanism.
- **State deserialization pattern unknown:** The story does not specify the exact pattern used for state deserialization. Common patterns are documented in the algorithm notes (Pydantic `@model_validator`, dataclass `__post_init__`, manual loader function). Implementation must locate the existing deserialization code and add backward compatibility initialization there, before the state is passed to reducers or used by nodes.
- **Backward compatibility is mandatory:** Tests 3 and 4 verify that runs saved before this change (missing `usage` or `orchestrator_seen_flags` fields) load and resume successfully. This is a hard requirement from AC-2 (REQ-2 AC-2.4). The initialization logic must run during deserialization, not in the reducer (reducers receive already-constructed state objects).
- **Missing count keys default to 0:** The usage reducer uses `.get(count_key, 0)` when adding incoming counts to existing counts. This ensures that if an incoming update is missing a count key (e.g., `cache_creation_input_tokens` is absent from the update payload), it defaults to 0 instead of crashing.
- **No live runs in tests:** All tests use fake state objects (in-memory dicts or objects) and mock deserialization logic. No live model calls, no live runs, no external storage.
- **Existing tests unchanged:** After this story, `uv run pytest -q` must pass, and existing tests must pass unchanged. If existing tests construct `State` objects directly (e.g., `State(thread_id="123", ...)`), they may need minor updates to include the new fields or rely on default factories (which are already present in the schema).
- **Test count matches breakdown exactly:** 4 tests as specified in breakdown Verification Approach for story 01-agent. No additional tests beyond those agreed in the breakdown.
