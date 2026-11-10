# LLD — Story 04-agent: Usage collection in run_skill and MCP tool loop

**Story:** Usage collection in run_skill and MCP tool loop  
**Epic:** PER-128 (Skip orchestrator model calls on deterministic hops and track token usage)  
**Repo:** `agent`  
**Version:** 1.1  
**Status:** Draft

---

## Overview

Extract token usage from every model call in `src/agents/skills/client.py` (`run_skill`, including JSON retries) and `src/agents/tools/bindings.py` (MCP tool loop, including final no-tools call). After each call, the calling node returns a state update dict containing `{usage: {node_name: {calls: 1, input_tokens: ..., output_tokens: ..., cache_creation_input_tokens: ..., cache_read_input_tokens: ...}}}`. LangGraph merges this dict into `State` using the `usage` reducer (from Story 1), which sums the counts into `State.usage[node_name]`.

This story implements the first integration points for usage telemetry, capturing calls in the skill execution and MCP tool loop paths. Story 5 will handle direct `ModelGateway.complete` calls, and Story 6 will aggregate and report totals.

**State update pattern (LangGraph):** Nodes return a dict of updated fields, e.g., `{"usage": {...}, "steps": [...]}`. LangGraph merges each field using its registered reducer. The `usage` reducer (from Story 1) is additive, so multiple updates for the same node accumulate counts.

**CRITICAL CONSISTENCY REQUIREMENT:** This story modifies `run_skill`'s return type from `dict` to `tuple[dict, dict]`. Every node in `src/agents/nodes/` that calls `run_skill` or `run_tools` MUST be updated in the same PR to unpack the tuple and include usage in state updates. Failure to update all calling nodes causes runtime crashes when those nodes execute.

---

## Acceptance Criteria

### AC1: Every model call in `run_skill` returns usage in state update

**Given** `run_skill` is invoked with a skill and node name  
**When** the function makes a model call via `ModelGateway.complete`  
**Then** usage data (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) is extracted from the response and returned as the second element of the tuple with `calls: 1`.

---

### AC2: JSON retries return usage for each attempt

**Given** `run_skill` makes a model call that returns invalid JSON  
**When** the function retries with the error message  
**Then** both the initial call and the retry accumulate usage in the returned tuple, and when a calling node includes this in its state update, `State.usage[node_name]` aggregates counts from both calls.

---

### AC3: Every MCP tool loop round returns usage in state update

**Given** the MCP tool loop in `bindings.py` makes multiple rounds of tool calls  
**When** each round completes (including the final no-tools call)  
**Then** usage data is extracted from each model response and accumulated in the returned tuple.

---

### AC4: Returned usage matches response fields exactly

**Given** a model response contains `usage` metadata  
**When** usage is extracted and returned  
**Then** the returned dict includes all four token fields (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) with values matching the response.

---

### AC5: All calling nodes updated to unpack usage tuple

**Given** `run_skill` or `run_tools` return type changes to `tuple[dict, dict]`  
**When** all nodes in `src/agents/nodes/` that call these functions are identified  
**Then** each calling node is updated to unpack `(result, usage_data) = run_skill(...)` and include `{"usage": {node_name: usage_data}}` in its returned state update dict.

**Verification:** Search `src/agents/nodes/` for all `run_skill` and `run_tools` calls, document calling nodes, verify each is updated. Integration test exercises all identified nodes and verifies no `TypeError: tuple indices must be integers` crashes occur.

---

## File Plan

### Modified Files

#### `src/agents/skills/client.py`

**Current state (inferred from repo tree):**

- Contains `run_skill(skill, ...)` function that calls `ModelGateway.complete`.
- Likely handles JSON parsing and retries on invalid JSON.
- Returns skill execution result (the calling node wraps this in a state update dict).

**Changes:**

1. **Extract usage after each `ModelGateway.complete` call:**
   - After the initial call, extract `usage` from the response: `response.usage` (verify exact structure in `gateway.py`).
   - Store usage data in a local variable.

2. **Extract usage after JSON retry calls:**
   - If the response is invalid JSON and `run_skill` retries, extract usage from the retry response and accumulate it with the initial call's usage.

3. **Return usage alongside skill result:**
   - Change return type from `dict` to `tuple[dict, dict]`.
   - Return `(result, usage_data)` where `usage_data` is `{calls: total_calls, input_tokens: total_input, ...}`.
   - The calling node includes this in its state update: `{"planner_result": result, "usage": {node_name: usage_data}}`.

**Concrete implementation sketch:**

```python
# In src/agents/skills/client.py

def run_skill(skill, input_data, **kwargs) -> tuple[dict, dict]:
    """
    Executes a skill by calling the model gateway.
    Returns (skill_result, usage_data) where usage_data aggregates token counts across all calls.
    
    Args:
        skill: Skill definition (from loader)
        input_data: Input to the skill
        **kwargs: Additional arguments for ModelGateway.complete
    
    Returns:
        tuple: (skill_result, usage_data)
            skill_result: Parsed skill execution result
            usage_data: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    """
    from src.agents.model.gateway import ModelGateway
    
    gateway = ModelGateway()
    
    # Accumulator for usage across all calls (initial + retries)
    usage_accumulator = {
        "calls": 0,
        "input_tokens": 0,
        "output_tokens": 0,
        "cache_creation_input_tokens": 0,
        "cache_read_input_tokens": 0
    }
    
    # Initial model call
    response = gateway.complete(
        messages=[...],  # build from skill and input_data
        **kwargs
    )
    
    # Extract and accumulate usage
    _accumulate_usage(response, usage_accumulator)
    
    # Parse response
    try:
        result = _parse_json(response.content)
        return (result, usage_accumulator)
    except JSONDecodeError as e:
        # JSON retry
        retry_response = gateway.complete(
            messages=[..., {"role": "user", "content": f"Invalid JSON: {e}"}],
            **kwargs
        )
        
        # Extract and accumulate usage for retry
        _accumulate_usage(retry_response, usage_accumulator)
        
        result = _parse_json(retry_response.content)
        return (result, usage_accumulator)

def _accumulate_usage(response, accumulator):
    """
    Extracts usage from model response and adds counts to accumulator.
    
    Args:
        response: Model response object (from ModelGateway.complete)
        accumulator: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    """
    if not hasattr(response, 'usage'):
        return  # No usage metadata in response
    
    accumulator["calls"] += 1
    accumulator["input_tokens"] += getattr(response.usage, "input_tokens", 0)
    accumulator["output_tokens"] += getattr(response.usage, "output_tokens", 0)
    accumulator["cache_creation_input_tokens"] += getattr(response.usage, "cache_creation_input_tokens", 0)
    accumulator["cache_read_input_tokens"] += getattr(response.usage, "cache_read_input_tokens", 0)
```

**Notes:**

- The exact response structure (`response.usage.input_tokens` vs `response["usage"]["input_tokens"]`) depends on `ModelGateway.complete`'s return type. Verify by inspecting `src/agents/model/gateway.py`.
- The calling node (e.g., `src/agents/nodes/planner/node.py`) wraps the returned `(result, usage_data)` into a state update dict: `{"planner_result": result, "usage": {"planner": usage_data}}`.

---

#### `src/agents/tools/bindings.py`

**Current state (inferred from repo tree):**

- Contains MCP tool loop logic (likely a `run_tools` or similar function).
- Makes multiple rounds of model calls when tools are invoked.
- Final call has no tools (model decides to stop).

**Changes:**

1. **Extract usage after each tool loop round:**
   - After each `ModelGateway.complete` call (tool invocation round), extract `usage` from the response and accumulate counts.

2. **Extract usage after final no-tools call:**
   - The final call (when the model decides not to invoke tools) also accumulates usage.

3. **Return usage alongside final response:**
   - Change return type to `tuple[object, dict]`.
   - Return `(final_response, usage_data)` where `usage_data` aggregates all rounds.
   - The calling node includes this in its state update: `{"node_result": final_response, "usage": {node_name: usage_data}}`.

**Concrete implementation sketch:**

```python
# In src/agents/tools/bindings.py

def run_tools(initial_input, tools, **kwargs) -> tuple[object, dict]:
    """
    Runs the MCP tool loop: model invokes tools, processes results, repeats.
    Returns (final_response, usage_data) where usage_data aggregates token counts across all rounds.
    
    Args:
        initial_input: Initial messages/input for the model
        tools: Available tools for the model to invoke
        **kwargs: Additional arguments for ModelGateway.complete
    
    Returns:
        tuple: (final_response, usage_data)
            final_response: Final model response (after no more tools are invoked)
            usage_data: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    """
    from src.agents.model.gateway import ModelGateway
    
    gateway = ModelGateway()
    messages = initial_input.copy()
    
    # Accumulator for usage across all rounds
    usage_accumulator = {
        "calls": 0,
        "input_tokens": 0,
        "output_tokens": 0,
        "cache_creation_input_tokens": 0,
        "cache_read_input_tokens": 0
    }
    
    while True:
        # Model call (may invoke tools or finish)
        response = gateway.complete(
            messages=messages,
            tools=tools,
            **kwargs
        )
        
        # Extract and accumulate usage
        _accumulate_usage(response, usage_accumulator)
        
        # Check if model invoked tools
        if not response.tool_calls:
            # No tools invoked, loop ends (this is the final no-tools call)
            return (response, usage_accumulator)
        
        # Execute tools and append results to messages
        for tool_call in response.tool_calls:
            tool_result = _execute_tool(tool_call, tools)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": tool_result
            })
        
        # Continue loop (next round will make another model call)

def _accumulate_usage(response, accumulator):
    """
    Extracts usage from model response and adds counts to accumulator.
    Same implementation as in client.py (or import from shared module).
    """
    if not hasattr(response, 'usage'):
        return
    
    accumulator["calls"] += 1
    accumulator["input_tokens"] += getattr(response.usage, "input_tokens", 0)
    accumulator["output_tokens"] += getattr(response.usage, "output_tokens", 0)
    accumulator["cache_creation_input_tokens"] += getattr(response.usage, "cache_creation_input_tokens", 0)
    accumulator["cache_read_input_tokens"] += getattr(response.usage, "cache_read_input_tokens", 0)
```

**Notes:**

- The exact tool loop structure depends on the existing `bindings.py` implementation. Adjust the sketch to match.
- Consider extracting `_accumulate_usage` into a shared utility module (e.g., `src/agents/utils/usage.py`) to avoid duplication between `client.py` and `bindings.py`.

---

#### Calling Nodes (ALL nodes in `src/agents/nodes/` that call `run_skill` or `run_tools`)

**Identification required (MANDATORY PRE-IMPLEMENTATION):**

1. **Search for `run_skill` calls:** Run `grep -r "run_skill" src/agents/nodes/ --include="*.py"` to find all files that call `run_skill`.
2. **Search for `run_tools` calls:** Run `grep -r "run_tools" src/agents/nodes/ --include="*.py"` to find all files that call `run_tools`.
3. **Document findings:** Create a list of all calling node files with line numbers before implementing any changes.
4. **Update each identified node:** For every calling node, modify to unpack the tuple and include usage in state update.

**Verified calling nodes from repo tree (MUST be confirmed by grep):**

Based on repo tree structure, likely calling nodes include:
- `src/agents/nodes/planner/node.py` (confirmed: planner skill exists)
- `src/agents/nodes/orchestrator/node.py` (confirmed: orchestrator skill exists)
- `src/agents/nodes/compliance/node.py` (confirmed: compliance skill exists)
- `src/agents/nodes/payroll_calculation/node.py` (confirmed: payroll-calculation skill exists)
- `src/agents/nodes/validation/node.py` (confirmed: validation skill exists)
- `src/agents/nodes/output_validator/node.py` (confirmed: output-validator skill exists)
- `src/agents/nodes/execution/node.py` (may call run_tools — verify)

**Changes (apply to EVERY identified calling node):**

1. **Unpack usage from `run_skill` or `run_tools`:**
   - Change: `result = run_skill(...)` → `(result, usage_data) = run_skill(...)`
   - Change: `response = run_tools(...)` → `(response, usage_data) = run_tools(...)`

2. **Include usage in state update dict:**
   - Add `usage` to the returned dict: `{"node_result": result, "usage": {node_name: usage_data}}`.
   - LangGraph merges this dict into `State` using the `usage` reducer (from Story 1), which sums the counts into `State.usage[node_name]`.

**Concrete implementation sketch (example for planner):**

```python
# In src/agents/nodes/planner/node.py

def planner_node(state):
    """
    Runs the planner skill and returns updated state fields.
    """
    # Call run_skill (now returns (result, usage_data))
    result, usage_data = run_skill(
        skill=load_skill("planner"),
        input_data=state.service_order
    )
    
    # Build state update dict with usage
    return {
        "planner_result": result,
        "usage": {"planner": usage_data},  # LangGraph merges into State.usage
        "steps": [{"title": "Planner completed", "result": result}],
        "status": "planner_complete"
    }
```

**Similar changes required for:**

- `src/agents/nodes/orchestrator/node.py` (if it calls `run_skill` for orchestrator skill)
- `src/agents/nodes/compliance/node.py` (calls `run_skill` for compliance skill)
- `src/agents/nodes/payroll_calculation/node.py` (calls `run_skill` for payroll-calculation skill)
- `src/agents/nodes/validation/node.py` (calls `run_skill` for validation skill)
- `src/agents/nodes/output_validator/node.py` (calls `run_skill` for output-validator skill)
- `src/agents/nodes/execution/node.py` (if it calls `run_tools` or `run_skill`)

**CRITICAL:** Each node must use its correct node name in the usage dict key:
- Planner: `{"usage": {"planner": usage_data}}`
- Orchestrator: `{"usage": {"orchestrator": usage_data}}`
- Compliance: `{"usage": {"compliance": usage_data}}`
- etc.

**Notes:**

- If a node makes multiple `run_skill` calls (e.g., retries or fallbacks), it must accumulate the usage data before returning:
  ```python
  result1, usage1 = run_skill(skill1, data)
  result2, usage2 = run_skill(skill2, data)
  
  total_usage = {
      "calls": usage1["calls"] + usage2["calls"],
      "input_tokens": usage1["input_tokens"] + usage2["input_tokens"],
      "output_tokens": usage1["output_tokens"] + usage2["output_tokens"],
      "cache_creation_input_tokens": usage1["cache_creation_input_tokens"] + usage2["cache_creation_input_tokens"],
      "cache_read_input_tokens": usage1["cache_read_input_tokens"] + usage2["cache_read_input_tokens"]
  }
  return {"node_result": result2, "usage": {node_name: total_usage}}
  ```

---

### New Files

#### `src/agents/utils/usage.py` (optional shared utility)

**Purpose:** Centralize usage extraction and accumulation logic to avoid duplication between `client.py` and `bindings.py`.

**Content:**

```python
# src/agents/utils/usage.py

def accumulate_usage(response, accumulator):
    """
    Extracts usage metadata from a model response and adds counts to accumulator.
    
    Args:
        response: Model response object (from ModelGateway.complete)
        accumulator: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    
    Returns:
        None (modifies accumulator in place)
    """
    if not hasattr(response, 'usage'):
        return  # No usage metadata in response
    
    accumulator["calls"] += 1
    accumulator["input_tokens"] += getattr(response.usage, "input_tokens", 0)
    accumulator["output_tokens"] += getattr(response.usage, "output_tokens", 0)
    accumulator["cache_creation_input_tokens"] += getattr(response.usage, "cache_creation_input_tokens", 0)
    accumulator["cache_read_input_tokens"] += getattr(response.usage, "cache_read_input_tokens", 0)
```

**Usage:**

```python
# In client.py and bindings.py
from src.agents.utils.usage import accumulate_usage

# After each model call:
accumulate_usage(response, usage_accumulator)
```

---

## Interfaces

### `run_skill` (modified)

**Module:** `src/agents/skills/client.py`

**Signature (before changes):**

```python
def run_skill(skill, input_data, **kwargs) -> dict:
    """
    Executes a skill by calling the model gateway.
    Returns the parsed skill result.
    """
```

**Signature (after changes):**

```python
def run_skill(skill, input_data, **kwargs) -> tuple[dict, dict]:
    """
    Executes a skill by calling the model gateway.
    Returns (skill_result, usage_data).
    
    Args:
        skill: Skill definition (from loader)
        input_data: Input to the skill
        **kwargs: Additional arguments for ModelGateway.complete
    
    Returns:
        tuple: (skill_result, usage_data)
            skill_result: Parsed skill execution result (dict)
            usage_data: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    """
```

**Changes:**

- Return type changes from `dict` to `tuple[dict, dict]`.
- Extract usage after each `ModelGateway.complete` call (initial and retries).
- Accumulate usage across all calls in a local dict.
- Return `(skill_result, usage_data)`.

**Contract:**

- Every model call (initial and retries) contributes to `usage_data`.
- Token counts are summed across all calls.
- If a model call has no `usage` metadata, that call contributes `0` to all fields (no error).
- **BREAKING CHANGE:** All calling code must be updated to unpack the tuple.

---

### `run_tools` (modified or new)

**Module:** `src/agents/tools/bindings.py`

**Signature (after changes):**

```python
def run_tools(initial_input, tools, **kwargs) -> tuple[object, dict]:
    """
    Runs the MCP tool loop: model invokes tools, processes results, repeats.
    Returns (final_response, usage_data).
    
    Args:
        initial_input: Initial messages/input for the model
        tools: Available tools for the model to invoke
        **kwargs: Additional arguments for ModelGateway.complete
    
    Returns:
        tuple: (final_response, usage_data)
            final_response: Final model response (after no more tools are invoked)
            usage_data: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    """
```

**Changes:**

- Return type changes to `tuple[object, dict]`.
- Extract usage after each tool loop round (including final no-tools call).
- Accumulate usage across all rounds in a local dict.
- Return `(final_response, usage_data)`.

**Contract:**

- Every model call in the loop (tool invocation rounds + final call) contributes to `usage_data`.
- Token counts are summed across all rounds.
- **BREAKING CHANGE:** All calling code must be updated to unpack the tuple.

---

### `accumulate_usage` (new utility function)

**Module:** `src/agents/utils/usage.py` (or inline in `client.py` and `bindings.py`)

**Signature:**

```python
def accumulate_usage(response, accumulator: dict) -> None:
    """
    Extracts usage metadata from a model response and adds counts to accumulator.
    
    Args:
        response: Model response object (from ModelGateway.complete)
        accumulator: dict with {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}
    
    Returns:
        None (modifies accumulator in place)
    """
```

**Behavior:**

1. Check if `response` has a `usage` attribute (or key).
2. Extract `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`.
3. Add counts to `accumulator`:
   - `accumulator["calls"] += 1`
   - `accumulator["input_tokens"] += response.usage.input_tokens` (default 0 if missing)
   - Same for other fields.

**Contract:**

- If `response.usage` is missing, the function is a no-op (no change to accumulator).
- Missing token fields default to `0` (via `getattr(..., 0)`).
- The function modifies `accumulator` in place (no return value).

---

### Node State Update Pattern

**Pattern (LangGraph):**

Nodes return a dict with updated state fields. LangGraph merges each field using its registered reducer. For `usage`, the reducer (from Story 1) sums incoming counts into existing counts per node.

**Example node return:**

```python
return {
    "planner_result": result,
    "usage": {"planner": {"calls": 2, "input_tokens": 1000, "output_tokens": 200, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 1000}},
    "steps": [{"title": "Planner completed", "result": result}],
    "status": "planner_complete"
}
```

**LangGraph merge:**

- `usage` field is merged by the `usage` reducer (additive).
- If `State.usage` already has `{"planner": {"calls": 1, "input_tokens": 500, ...}}`, the new counts sum: `{"calls": 3, "input_tokens": 1500, ...}`.

**Contract:**

- Nodes return `usage` keyed by node name: `{"usage": {node_name: {calls, input_tokens, ...}}}`.
- The `usage` reducer handles the summing (nodes don't need to read existing state to add counts).

---

## Algorithm Notes

### Usage Accumulation Flow (within `run_skill` or `run_tools`)

**Goal:** Sum token counts across all model calls made by a single invocation of `run_skill` or `run_tools`.

**Flow:**

1. Initialize accumulator: `{calls: 0, input_tokens: 0, output_tokens: 0, cache_creation_input_tokens: 0, cache_read_input_tokens: 0}`.
2. For each `ModelGateway.complete` call:
   - Receive `response` object.
   - Call `accumulate_usage(response, accumulator)`:
     - Extract `response.usage.input_tokens` (default 0 if missing).
     - Add to `accumulator["input_tokens"]`.
     - Same for other fields.
     - Increment `accumulator["calls"]` by 1.
3. Return `(result, accumulator)` to the calling node.

**Invariants:**

- `accumulator["calls"]` equals the number of model calls (initial + retries, or tool rounds + final).
- All token counts are non-negative integers.

---

### State Merge Flow (LangGraph)

**Goal:** Merge node's returned `usage` dict into `State.usage`.

**Flow:**

1. Node returns: `{"usage": {node_name: {"calls": 2, "input_tokens": 1000, ...}}}`.
2. LangGraph calls the `usage` reducer (from Story 1) with:
   - `existing = State.usage` (e.g., `{node_name: {"calls": 1, "input_tokens": 500, ...}}`)
   - `incoming = {node_name: {"calls": 2, "input_tokens": 1000, ...}}`
3. Reducer sums: `State.usage[node_name] = {"calls": 3, "input_tokens": 1500, ...}`.
4. State is updated.

**Invariants:**

- The reducer (from Story 1) is additive and handles missing keys (initializes to 0).
- Multiple nodes can update `State.usage` independently; counts accumulate correctly.

---

### JSON Retry Flow

**Context:** `run_skill` calls the model, gets invalid JSON, retries with error message.

**Flow:**

1. Initial call: `response1 = gateway.complete(...)`.
2. Accumulate usage from `response1`: `accumulator["calls"] = 1, accumulator["input_tokens"] = 500, ...`.
3. Parse `response1.content` → `JSONDecodeError`.
4. Retry call: `response2 = gateway.complete(..., messages + error_message)`.
5. Accumulate usage from `response2`: `accumulator["calls"] = 2, accumulator["input_tokens"] = 800, ...` (500 + 300).
6. Return `(result, accumulator)`.

**Result:**

- Node returns `{"usage": {node_name: {"calls": 2, "input_tokens": 800, ...}}}`.
- LangGraph merges into `State.usage[node_name]`.

---

### MCP Tool Loop Flow

**Context:** `run_tools` makes multiple rounds of model calls (tool invocations + final no-tools call).

**Flow:**

1. Initial call: `response1 = gateway.complete(..., tools=tools)`.
2. Accumulate usage: `accumulator["calls"] = 1, accumulator["input_tokens"] = 500, ...`.
3. Check `response1.tool_calls`:
   - If empty, return `(response1, accumulator)` (final no-tools call).
   - If not empty, execute tools, append results to messages, loop.
4. Next round: `response2 = gateway.complete(..., messages + tool_results)`.
5. Accumulate usage: `accumulator["calls"] = 2, accumulator["input_tokens"] = 800, ...`.
6. Repeat until `response.tool_calls` is empty.

**Result:**

- Node returns `{"usage": {node_name: {"calls": 3, "input_tokens": 1000, ...}}}` (if 3 rounds total).

---

## Edge Cases

### Edge Case 1: Response has no `usage` metadata

**Scenario:** `ModelGateway.complete` returns a response without a `usage` attribute (e.g., a mock response in tests).

**Handling:**

- `accumulate_usage` checks `hasattr(response, 'usage')`.
- If `False`, the function returns early (no change to accumulator).
- No error is raised; usage tracking is skipped for that call.

**Test:** Verify `accumulate_usage` is a no-op when `response.usage` is missing.

---

### Edge Case 2: Token fields are missing in `response.usage`

**Scenario:** `response.usage` exists but has only `input_tokens` and `output_tokens` (no cache fields).

**Handling:**

- Use `getattr(response.usage, "cache_creation_input_tokens", 0)` to default missing fields to `0`.
- Accumulator adds `0` for missing fields (no error).

**Test:** Verify `accumulate_usage` handles partial usage metadata (some fields missing).

---

### Edge Case 3: Multiple retries in `run_skill`

**Scenario:** `run_skill` retries JSON parsing 3 times before giving up.

**Handling:**

- Each retry accumulates usage.
- `accumulator["calls"]` = 3 (one per retry).
- Token counts sum across all retries.

**Test:** `test_usage_json_retry` verifies this (AC2).

---

### Edge Case 4: MCP tool loop with many rounds

**Scenario:** The model invokes tools 5 times before finishing.

**Handling:**

- Each round (5 tool invocations + 1 final no-tools call = 6 total) accumulates usage.
- `accumulator["calls"]` = 6.
- Token counts sum across all rounds.

**Test:** `test_usage_mcp_tool_loop` verifies this (AC3).

---

### Edge Case 5: Node makes multiple `run_skill` calls

**Scenario:** A node calls `run_skill` twice (e.g., primary skill + fallback skill).

**Handling:**

- First call returns `(result1, usage1)`.
- Second call returns `(result2, usage2)`.
- Node must sum `usage1` and `usage2` before returning:
  ```python
  total_usage = {
      "calls": usage1["calls"] + usage2["calls"],
      "input_tokens": usage1["input_tokens"] + usage2["input_tokens"],
      "output_tokens": usage1["output_tokens"] + usage2["output_tokens"],
      "cache_creation_input_tokens": usage1["cache_creation_input_tokens"] + usage2["cache_creation_input_tokens"],
      "cache_read_input_tokens": usage1["cache_read_input_tokens"] + usage2["cache_read_input_tokens"]
  }
  return {"node_result": result2, "usage": {node_name: total_usage}}
  ```

**Test:** Verify a node that makes multiple `run_skill` calls returns summed usage.

---

### Edge Case 6: Calling node doesn't unpack usage (BREAKING CHANGE MITIGATION)

**Scenario
