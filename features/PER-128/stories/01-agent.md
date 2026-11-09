# Story 01-agent LLD

**Ticket:** PER-128-1  
**Epic:** PER-128  
**Repo:** agent  
**Title:** Add token usage tracking to State and model gateways

---

## Acceptance Criteria

From `requirements.md` AC1 and AC4 (this story's scope):

1. **AC1.1:** Every model call records `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` in `State.usage` under the current node's key.
2. **AC1.2:** Multiple calls for the same node add counts to existing ones (not replace).
3. **AC1.3:** A run saved before this change (no `usage` field) loads and resumes without error.
4. **AC4.1:** New tests verify usage accumulates correctly across calls (including JSON retries, MCP tool rounds).
5. **AC4.2:** Test verifies a run loads without error when `usage` is missing.
6. **AC4.3:** All existing tests pass unchanged after the changes (`uv run pytest -q`).

---

## HLD Outcomes (Binding Constraints)

From `hld.md`:

- **AD-1 (State schema extensions):** `State` gains `usage: Dict[str, UsageStats]` with an **additive reducer** (sum incoming counts into existing, never replace). Missing `usage` in old runs treated as `{}`.
- **AD-4 (Usage recording contract):** Every model call records four token counts from API response. Each node update carries a `UsageStats` delta with `calls: 1` and token fields. Scope: `run_skill` (including JSON retries), MCP tool loop (per round + final no-tools call), `ModelGateway.complete`.

---

## File Plan

**Note:** Repo tree not provided. All paths below are placeholders that must be verified against the actual codebase before implementation. Developer must locate the real file locations by scanning the agent repo.

### Files to Create

1. **UsageStats dataclass definition**
   - Location TBD (likely `src/agents/usage_stats.py` or inline in state module)
   - Purpose: Define `UsageStats` with four token fields, `calls` counter, and `__add__` method

2. **`tests/test_usage_tracking.py`**
   - Purpose: New test file for usage tracking behavior
   - Content: 5 tests per Verification Approach below

### Files to Modify

3. **State schema module** (location TBD, e.g. `src/agents/state.py`)
   - Add `usage: Dict[str, UsageStats]` field to `State` schema
   - Implement additive reducer: `_reduce_usage(existing, delta) -> Dict[str, UsageStats]`

4. **State loader module** (location TBD, e.g. `src/agents/state_loader.py`)
   - Handle missing `usage` field: default to `{}` when loading old runs

5. **Skill client module** (location TBD, e.g. `src/agents/skills/client.py`)
   - Modify `run_skill`: extract token counts from each completion response (including JSON retry calls)
   - Return `UsageStats` delta with `calls: 1`

6. **Tool bindings module** (location TBD, e.g. `src/agents/tools/bindings.py`)
   - Modify MCP tool loop: extract token counts after each round (including final no-tools call)
   - Return `UsageStats` delta per round

7. **Model gateway module** (location TBD, e.g. `src/agents/model/gateway.py`)
   - Modify `ModelGateway.complete`: extract token counts from direct completion responses
   - Return `UsageStats` delta

8. **Node update logic** (location TBD, each node in `src/agents/nodes/`)
   - Include `UsageStats` delta from model calls in each node's state update payload
   - Apply pattern to all nodes that call model gateways

---

## Interfaces and Data Structures

### UsageStats Dataclass

```python
# Location TBD (e.g. src/agents/usage_stats.py or inline in state module)
from dataclasses import dataclass

@dataclass
class UsageStats:
    """Token usage counts for model API calls."""
    calls: int = 0
    input_tokens: int = 0
    output_tokens: int = 0
    cache_creation_input_tokens: int = 0
    cache_read_input_tokens: int = 0
    
    def __add__(self, other: 'UsageStats') -> 'UsageStats':
        """
        Additive merge for reducer.
        Allows: result[node] = existing[node] + delta[node]
        """
        return UsageStats(
            calls=self.calls + other.calls,
            input_tokens=self.input_tokens + other.input_tokens,
            output_tokens=self.output_tokens + other.output_tokens,
            cache_creation_input_tokens=self.cache_creation_input_tokens + other.cache_creation_input_tokens,
            cache_read_input_tokens=self.cache_read_input_tokens + other.cache_read_input_tokens,
        )
```

### State Extension

```python
# State schema module (location TBD)
from typing import Dict
# Import UsageStats from wherever it's defined

# Add to State schema (TypedDict or dataclass)
usage: Dict[str, UsageStats] = field(default_factory=dict)
```

### State Reducer

```python
# State schema module (location TBD)
def _reduce_usage(
    existing: Dict[str, UsageStats],
    delta: Dict[str, UsageStats]
) -> Dict[str, UsageStats]:
    """
    Additive merge: sum incoming counts into existing per node.
    Uses UsageStats.__add__ to handle field-by-field addition.
    If node not in existing, initialize with delta (adding to zero-initialized UsageStats).
    """
    result = existing.copy()
    for node, new_stats in delta.items():
        result[node] = result.get(node, UsageStats()) + new_stats
    return result
```

**Note:** The reducer uses `UsageStats.__add__` to perform the addition. The `__add__` method handles all field-by-field sums, so the reducer only loops over nodes and assigns the result.

### API Response Extraction

```python
# Helper function (location TBD, likely in each model gateway module)
def extract_usage_from_response(response) -> UsageStats:
    """
    Extract token counts from Anthropic API response.
    
    Expected response.usage structure:
    {
        "input_tokens": int,
        "output_tokens": int,
        "cache_creation_input_tokens": int,
        "cache_read_input_tokens": int
    }
    """
    usage = response.get("usage", {})
    return UsageStats(
        calls=1,
        input_tokens=usage.get("input_tokens", 0),
        output_tokens=usage.get("output_tokens", 0),
        cache_creation_input_tokens=usage.get("cache_creation_input_tokens", 0),
        cache_read_input_tokens=usage.get("cache_read_input_tokens", 0),
    )
```

---

## Algorithm Notes

### 1. State Loader Migration Logic

```python
# State loader module (location TBD)
def load_state(serialized_state: dict) -> State:
    """
    Load state from persistence, handling missing fields.
    
    Pre-change runs have no 'usage' field.
    Default to empty dict to allow execution to continue.
    """
    if "usage" not in serialized_state:
        serialized_state["usage"] = {}
    
    # Continue with normal deserialization...
    return State(**serialized_state)
```

### 2. run_skill Usage Recording

```python
# Skill client module (location TBD, pseudocode)
def run_skill(skill_name, input_data, node_name):
    """
    Call skill, handle JSON retries, record usage per attempt.
    Uses UsageStats.__add__ to accumulate across retries.
    """
    total_usage = UsageStats()
    
    for attempt in range(max_retries):
        response = call_skill_api(skill_name, input_data)
        
        # Extract and accumulate usage (including retries)
        attempt_usage = extract_usage_from_response(response)
        total_usage = total_usage + attempt_usage
        
        if is_valid_json(response.content):
            break
    
    # Return result + total usage delta
    return {
        "result": response.content,
        "usage_delta": {node_name: total_usage}
    }
```

### 3. MCP Tool Loop Usage Recording

```python
# Tool bindings module (location TBD, pseudocode)
def run_tool_loop(initial_prompt, node_name):
    """
    Run tool loop until no tools remain, recording usage per round.
    Uses UsageStats.__add__ to accumulate across rounds.
    """
    total_usage = UsageStats()
    messages = [initial_prompt]
    
    while True:
        response = call_model_with_tools(messages)
        
        # Record this round's usage
        round_usage = extract_usage_from_response(response)
        total_usage = total_usage + round_usage
        
        if no_tool_calls(response):
            # Final no-tools call also recorded above
            break
        
        tool_results = execute_tools(response.tool_calls)
        messages.append(tool_results)
    
    return {
        "result": response.content,
        "usage_delta": {node_name: total_usage}
    }
```

### 4. Node Update Pattern

```python
# Node modules (location TBD, generic pattern for all nodes)
def run_node(state: State) -> StateUpdate:
    """
    Execute node logic, record usage from any model calls.
    """
    # Make model call(s)
    result = run_skill("skill_name", input_data, node_name="my_node")
    # OR: run_tool_loop(...) OR: ModelGateway.complete(...)
    
    # Include usage delta in state update
    return StateUpdate(
        node_result=result["result"],
        usage=result["usage_delta"],  # Reducer will merge using __add__
        # ... other fields
    )
```

### 5. Reducer Application

```python
# State module (location TBD, state update application)
def apply_update(state: State, update: StateUpdate) -> State:
    """
    Apply update to state, merging usage additively.
    Uses _reduce_usage which delegates to UsageStats.__add__.
    """
    new_usage = _reduce_usage(state.usage, update.usage)
    
    return State(
        **state.__dict__,
        usage=new_usage,
        # ... other updated fields
    )
```

---

## Tests

### Test File: `tests/test_usage_tracking.py`

5 tests matching the Verification Approach from `breakdown.md` exactly:

#### Test 1: Usage Accumulates Across Two Calls for Same Node

```python
def test_usage_accumulation_same_node(fake_gateway):
    """
    Given a node makes two model calls
    When usage is recorded for both
    Then state.usage[node].calls == 2 and tokens are summed
    
    Verifies AC1.2 (additive merge, not replace)
    """
    state = State(usage={})
    
    # First call
    delta1 = {"orchestrator": UsageStats(calls=1, input_tokens=100, output_tokens=50, cache_creation_input_tokens=10, cache_read_input_tokens=5)}
    state = apply_update(state, StateUpdate(usage=delta1))
    
    # Second call
    delta2 = {"orchestrator": UsageStats(calls=1, input_tokens=200, output_tokens=75, cache_creation_input_tokens=20, cache_read_input_tokens=10)}
    state = apply_update(state, StateUpdate(usage=delta2))
    
    assert state.usage["orchestrator"].calls == 2
    assert state.usage["orchestrator"].input_tokens == 300
    assert state.usage["orchestrator"].output_tokens == 125
    assert state.usage["orchestrator"].cache_creation_input_tokens == 30
    assert state.usage["orchestrator"].cache_read_input_tokens == 15
```

#### Test 2: JSON Retry in run_skill Contributes to Usage

```python
def test_json_retry_counted(fake_gateway):
    """
    Given run_skill makes initial call that returns invalid JSON
    When it retries and succeeds
    Then both calls contribute to usage (calls == 2, tokens summed)
    
    Verifies AC4.1 (JSON retries counted)
    """
    fake_gateway.responses = [
        {"content": "not json", "usage": {"input_tokens": 50, "output_tokens": 10, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 5}},
        {"content": '{"valid": "json"}', "usage": {"input_tokens": 60, "output_tokens": 20, "cache_creation_input_tokens": 5, "cache_read_input_tokens": 10}},
    ]
    
    result = run_skill("test_skill", {"input": "data"}, "test_node")
    
    assert result["usage_delta"]["test_node"].calls == 2
    assert result["usage_delta"]["test_node"].input_tokens == 110
    assert result["usage_delta"]["test_node"].output_tokens == 30
    assert result["usage_delta"]["test_node"].cache_creation_input_tokens == 5
    assert result["usage_delta"]["test_node"].cache_read_input_tokens == 15
```

#### Test 3: MCP Tool Loop's Three Rounds Accumulate

```python
def test_mcp_tool_loop_rounds(fake_gateway):
    """
    Given MCP tool loop makes three rounds (two tool calls, one final no-tools)
    When usage is recorded
    Then calls == 3 and tokens sum across all rounds
    
    Verifies AC4.1 (MCP tool rounds counted)
    """
    fake_gateway.responses = [
        {"tool_calls": [{"name": "tool1"}], "usage": {"input_tokens": 100, "output_tokens": 20, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 10}},
        {"tool_calls": [{"name": "tool2"}], "usage": {"input_tokens": 150, "output_tokens": 30, "cache_creation_input_tokens": 5, "cache_read_input_tokens": 15}},
        {"tool_calls": [], "content": "done", "usage": {"input_tokens": 200, "output_tokens": 40, "cache_creation_input_tokens": 10, "cache_read_input_tokens": 20}},
    ]
    
    result = run_tool_loop("initial prompt", "tool_node")
    
    assert result["usage_delta"]["tool_node"].calls == 3
    assert result["usage_delta"]["tool_node"].input_tokens == 450
    assert result["usage_delta"]["tool_node"].output_tokens == 90
    assert result["usage_delta"]["tool_node"].cache_creation_input_tokens == 15
    assert result["usage_delta"]["tool_node"].cache_read_input_tokens == 45
```

#### Test 4: State Loads Without Error When usage Missing

```python
def test_state_migration_no_usage_field():
    """
    Given a serialized state from before this change (no 'usage' field)
    When the state is loaded
    Then it loads successfully and state.usage == {}
    
    Verifies AC1.3 and AC4.2 (migration works)
    
    Note: Developer must identify all required State schema fields before 
    writing this test. The old_state_data dict must include those fields 
    to avoid schema validation errors before the loader's migration check runs.
    Placeholder below shows run_id and status; real implementation must 
    include all required fields discovered from State schema scan.
    """
    # Developer: Replace with actual required State fields from schema scan
    old_state_data = {
        "run_id": "test-123",
        "status": "running",
        # Add all other required State fields here (e.g. flags, plan, etc.)
        # 'usage' field intentionally missing to test migration
    }
    
    state = load_state(old_state_data)
    
    assert state.usage == {}
    assert state.run_id == "test-123"
    assert state.status == "running"
```

#### Test 5: UsageStats Correctly Extracts from Mocked API Response

```python
def test_usage_extraction_from_api_response():
    """
    Given a mocked Anthropic API response with all four token fields
    When extract_usage_from_response is called
    Then UsageStats is correctly populated
    
    Verifies AC1.1 (all four token fields extracted)
    """
    mock_response = {
        "content": "response text",
        "usage": {
            "input_tokens": 123,
            "output_tokens": 456,
            "cache_creation_input_tokens": 78,
            "cache_read_input_tokens": 90,
        }
    }
    
    stats = extract_usage_from_response(mock_response)
    
    assert stats.calls == 1
    assert stats.input_tokens == 123
    assert stats.output_tokens == 456
    assert stats.cache_creation_input_tokens == 78
    assert stats.cache_read_input_tokens == 90
```

### Existing Tests

All existing tests in the agent repo must pass unchanged after adding `usage` field (verified via `uv run pytest -q`).

**Regression risk mitigation:**
- `usage` field defaults to `{}` in State schema (no impact on tests that don't inspect it)
- State loader handles missing `usage` gracefully (old test fixtures continue to load)
- Reducer only processes `usage` delta if present in update (non-usage updates unaffected)

**Verifies AC4.3**

---

## Out of Scope

- Logging or returning usage totals at run end (PER-128-3: `run_query` aggregation)
- `seen_flags` field (PER-128-3: orchestrator integration)
- Forced routing logic (PER-128-2 and PER-128-3)
- API response format (`GET /runs/{thread_id}` extension in PER-128-3)
- Changing models, `policy.yaml`, or prompt content

---

## Open Questions

**Must resolve before implementation:**

1. **What are all required fields in the State schema?**  
   - Test 4 depends on this: must include all required fields except `usage` to test migration correctly
   - Developer must scan State definition (location TBD) to identify required fields
   - Without this, Test 4 will fail with schema validation errors instead of testing the loader's migration logic

2. **Where do the model gateway modules live?**  
   - Developer must locate `run_skill`, MCP tool loop, and `ModelGateway.complete` in the codebase
   - Paths listed above are placeholders derived from ticket context
   - Cannot implement usage extraction until actual call sites are found

3. **How does the state reducer work today?**  
   - Developer must examine existing reducer pattern to understand how to add `_reduce_usage` without breaking existing field merges
   - Location TBD (likely in state module or update application logic)

---

**End of Story 01-agent LLD**
