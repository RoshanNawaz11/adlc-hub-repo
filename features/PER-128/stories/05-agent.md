# LLD — Story 05-agent: Usage collection in ModelGateway direct calls

**Story:** Usage collection in ModelGateway direct calls  
**Epic:** PER-128 (Skip orchestrator model calls on deterministic hops and track token usage)  
**Repo:** `agent`  
**Version:** 1.0  
**Status:** Draft

---

## Overview

Modify `src/agents/model/gateway.py` to support usage tracking for direct calls to `ModelGateway.complete` that bypass `run_skill` or the MCP tool loop. When a caller provides a `node_name`, the gateway extracts usage metadata from the model response and returns it alongside the response, allowing the caller to include it in their state update. If `node_name` is not provided, usage is logged only (existing behavior).

This story handles the **direct call path** to `ModelGateway.complete` — calls that bypass `run_skill` and the MCP tool loop (covered in Story 4). It ensures comprehensive usage tracking across all model invocation patterns.

**Integration note:** Story 4 established the pattern where calling functions extract usage from responses and include it in their returned state update dicts. This story extends that pattern to `ModelGateway.complete`, allowing direct callers to opt into usage tracking by providing a `node_name`.

---

## Acceptance Criteria

### AC1: `ModelGateway.complete` accepts optional `node_name` parameter

**Given** a caller invokes `ModelGateway.complete`  
**When** the caller provides a `node_name` parameter  
**Then** the method accepts it without error and includes usage data in the return value.

---

### AC2: When `node_name` provided, usage is returned alongside response

**Given** `ModelGateway.complete` is called with `node_name`  
**When** the model response is received  
**Then** the method returns `(response, usage_data)` where `usage_data` is `{node_name: {calls: 1, input_tokens: ..., output_tokens: ..., cache_creation_input_tokens: ..., cache_read_input_tokens: ...}}`, allowing the caller to include it in their state update.

---

### AC3: When `node_name` not provided, usage is logged only (no usage return)

**Given** `ModelGateway.complete` is called without `node_name` (or with `node_name=None`)  
**When** the model response is received  
**Then** usage data is logged (existing behavior) and the method returns only the response object (not a tuple).

---

### AC4: Returned usage matches response fields exactly

**Given** a model response contains usage metadata  
**When** usage is extracted and returned  
**Then** the returned dict includes all four token fields with values matching the response.

---

## File Plan

### Modified Files

#### `src/agents/model/gateway.py`

**Current state (inferred from repo tree):**

- Contains `ModelGateway` class with `complete` method.
- `complete` calls the model (likely Claude via Anthropic SDK) and returns a response object.
- Likely logs token usage per call (existing behavior from ticket context).
- Response object has a `usage` attribute with token counts.

**Changes:**

1. **Add `node_name` parameter to `complete` signature:**
   - Make it optional: `node_name: str | None = None`.

2. **Extract usage after model call:**
   - After receiving the model response, extract `usage` metadata (same pattern as Story 4).
   - If the response has no `usage` attribute, skip usage extraction (existing behavior for mocked responses or non-standard models).

3. **Return usage when `node_name` is provided:**
   - If `node_name` is not `None`, return `(response, usage_data)` where `usage_data` is `{node_name: {calls: 1, input_tokens: ..., ...}}`.
   - If `node_name` is `None`, return only `response` (existing behavior, backward compatible).

4. **Preserve existing logging:**
   - Keep the existing usage logging (if any) for observability, regardless of whether usage is returned.

**Concrete implementation sketch:**

```python
# In src/agents/model/gateway.py

from typing import Optional, Union
import logging

logger = logging.getLogger(__name__)

class ModelGateway:
    def complete(
        self,
        messages,
        tools=None,
        node_name: Optional[str] = None,
        **kwargs
    ) -> Union[object, tuple[object, dict]]:
        """
        Calls the model with the given messages and optional tools.
        Returns a response object with content and usage metadata.
        
        Args:
            messages: List of message dicts (role, content)
            tools: Optional list of tool definitions
            node_name: Optional node name for usage tracking (if None, returns response only)
            **kwargs: Additional arguments for the model call (temperature, max_tokens, etc.)
        
        Returns:
            If node_name is None:
                Response object with content, usage, and tool_calls
            If node_name is provided:
                tuple: (response, usage_data)
                    response: Response object with content, usage, and tool_calls
                    usage_data: dict with {node_name: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}
        
        Note:
            If calling from run_skill or run_tools, do not pass node_name — those functions
            extract usage themselves. Only pass node_name for direct calls that need usage tracking.
        """
        # Existing model call logic
        response = self._call_model(messages, tools, **kwargs)
        
        # Extract and log usage metadata
        if hasattr(response, 'usage'):
            usage = response.usage
            
            # Log usage (existing behavior)
            logger.info(
                f"Model call usage: input={usage.input_tokens}, "
                f"output={usage.output_tokens}, "
                f"cache_creation={getattr(usage, 'cache_creation_input_tokens', 0)}, "
                f"cache_read={getattr(usage, 'cache_read_input_tokens', 0)}"
            )
            
            # Return usage data if node_name provided
            if node_name is not None:
                usage_data = self._extract_usage(response, node_name)
                return (response, usage_data)
        
        return response
    
    def _extract_usage(self, response, node_name: str) -> dict:
        """
        Extracts usage from model response and formats for state update.
        
        Args:
            response: Model response object (with usage attribute)
            node_name: Name of the node making the call
        
        Returns:
            dict: {node_name: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}
        """
        if not hasattr(response, 'usage'):
            return {node_name: {
                "calls": 1,
                "input_tokens": 0,
                "output_tokens": 0,
                "cache_creation_input_tokens": 0,
                "cache_read_input_tokens": 0
            }}
        
        return {
            node_name: {
                "calls": 1,
                "input_tokens": response.usage.input_tokens,
                "output_tokens": response.usage.output_tokens,
                "cache_creation_input_tokens": getattr(response.usage, 'cache_creation_input_tokens', 0),
                "cache_read_input_tokens": getattr(response.usage, 'cache_read_input_tokens', 0)
            }
        }
    
    def _call_model(self, messages, tools, **kwargs):
        """
        Internal method that makes the actual model call.
        (Existing implementation unchanged)
        """
        # ... existing model call logic ...
        pass
```

**Notes:**

- The exact SDK call (`anthropic.messages.create` or similar) is in `_call_model` — unchanged by this story.
- Return type changes from `object` to `Union[object, tuple[object, dict]]` depending on whether `node_name` is provided.
- The `_extract_usage` helper can be extracted to the shared utility module (`src/agents/utils/usage.py` from Story 4) to avoid duplication.

---

#### Calling Nodes (e.g., custom nodes that call `gateway.complete` directly)

**Current state (inferred):**

- Direct callers invoke `gateway.complete(messages, tools)` and receive a response object.
- They return a state update dict to LangGraph.

**Changes:**

1. **Unpack usage when `node_name` is provided:**
   - When calling `complete` with `node_name`, unpack the returned tuple: `(response, usage_data) = gateway.complete(..., node_name="custom_node")`.

2. **Include usage in state update dict:**
   - Add `usage` to the returned dict: `{"node_result": response.content, "usage": usage_data}`.
   - LangGraph merges this dict into `State` using the `usage` reducer (from Story 1), which sums the counts into `State.usage[node_name]`.

**Concrete implementation sketch:**

```python
# In a custom node that calls gateway.complete directly

def custom_node(state):
    """
    Runs a direct model call and returns updated state fields.
    """
    from src.agents.model.gateway import ModelGateway
    
    gateway = ModelGateway()
    
    # Call complete with node_name to get usage data
    response, usage_data = gateway.complete(
        messages=[{"role": "user", "content": state.query}],
        node_name="custom_node"
    )
    
    # Build state update dict with usage
    return {
        "custom_node_result": response.content,
        "usage": usage_data,  # LangGraph merges into State.usage
        "steps": [{"title": "Custom node completed", "result": response.content}]
    }
```

**Notes:**

- Nodes that don't provide `node_name` receive only the response object (existing behavior, backward compatible).
- `run_skill` and `run_tools` (from Story 4) must NOT pass `node_name` to `gateway.complete` — they extract usage themselves to avoid double-dispatch.

---

### New Files

None. All changes are in `src/agents/model/gateway.py` and calling nodes.

---

## Interfaces

### `ModelGateway.complete` (modified)

**Module:** `src/agents/model/gateway.py`

**Signature (before changes):**

```python
def complete(self, messages, tools=None, **kwargs):
    """
    Calls the model with the given messages and optional tools.
    Returns a response object with content and usage metadata.
    """
```

**Signature (after changes):**

```python
def complete(
    self,
    messages,
    tools=None,
    node_name: Optional[str] = None,
    **kwargs
) -> Union[object, tuple[object, dict]]:
    """
    Calls the model with the given messages and optional tools.
    Returns a response object with content and usage metadata.
    
    Args:
        messages: List of message dicts (role, content)
        tools: Optional list of tool definitions
        node_name: Optional node name for usage tracking (if None, returns response only)
        **kwargs: Additional arguments for the model call (temperature, max_tokens, etc.)
    
    Returns:
        If node_name is None:
            Response object with content, usage, and tool_calls
        If node_name is provided:
            tuple: (response, usage_data)
                response: Response object
                usage_data: dict with {node_name: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}
    """
```

**Changes:**

- Add `node_name: Optional[str] = None` parameter.
- Return type changes from `object` to `Union[object, tuple[object, dict]]`.
- Extract usage after model call.
- If `node_name` is provided, return `(response, usage_data)`.
- If `node_name` is `None`, return only `response` (existing behavior).

**Backward compatibility:**

- Existing calls without `node_name` work unchanged (return only response object).
- No breaking changes for existing callers (`run_skill`, MCP tool loop, etc.).

**Contract:**

- When `node_name` is provided, caller must unpack the tuple: `(response, usage_data) = gateway.complete(..., node_name="node")`.
- Usage is extracted after every model call (no retries at this level — retries are in `run_skill`).
- If response has no `usage` metadata, returned `usage_data` has all fields set to 0.

---

### `_extract_usage` (new helper method)

**Module:** `src/agents/model/gateway.py`

**Signature:**

```python
def _extract_usage(self, response, node_name: str) -> dict:
    """
    Extracts usage from model response and formats for state update.
    
    Args:
        response: Model response object (with usage attribute)
        node_name: Name of the node making the call
    
    Returns:
        dict: {node_name: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}
    """
```

**Behavior:**

1. Check if `response` has a `usage` attribute.
2. Extract `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`.
3. Build usage dict: `{node_name: {calls: 1, input_tokens: ..., output_tokens: ..., cache_creation_input_tokens: ..., cache_read_input_tokens: ...}}`.
4. Return the dict.

**Contract:**

- If `response.usage` is missing, return a dict with all counts set to 0 (no error).
- Token fields default to `0` if missing in `response.usage` (via `getattr(..., 0)`).
- The `calls` field is always `1` (each call extracts separately; the reducer sums them).

---

### Response Object Structure

**Assumed structure (verify in existing `gateway.py` implementation):**

```python
response.content  # str: Model's text response
response.usage.input_tokens  # int
response.usage.output_tokens  # int
response.usage.cache_creation_input_tokens  # int (may be missing in some responses)
response.usage.cache_read_input_tokens  # int (may be missing in some responses)
response.tool_calls  # list: Tool invocations (empty if no tools)
```

**Handling missing fields:**

- Use `getattr(response.usage, 'cache_creation_input_tokens', 0)` to default missing fields to `0`.

---

## Algorithm Notes

### Direct Call Path

**Context:** Some nodes or utilities may call `ModelGateway.complete` directly, without going through `run_skill` or the MCP tool loop (covered in Story 4).

**Examples of direct calls:**

- A custom node that doesn't use the skill framework.
- A utility function that needs a one-off model call.
- Test code or scripts that invoke the gateway directly.

**Flow:**

1. Caller invokes `gateway.complete(messages, tools, node_name="custom_node")`.
2. `complete` calls the model via `_call_model`.
3. Model responds with `usage` metadata.
4. `complete` logs usage (existing behavior).
5. `complete` extracts usage via `_extract_usage(response, node_name)`.
6. `complete` returns `(response, usage_data)`.
7. Caller includes `usage_data` in their state update dict: `{"usage": usage_data}`.
8. LangGraph merges the dict into `State` using the `usage` reducer (from Story 1), which sums the counts into `State.usage[node_name]`.

**Fallback (no usage return):**

- If `node_name` is `None`, `complete` returns only `response` (existing behavior).
- Usage is still logged, so the call is tracked in logs but not in `State.usage`.

---

### Integration with Story 4

**Story 4 modified `run_skill` and `run_tools` to extract usage:**

- Those functions call `ModelGateway.complete` and extract usage themselves from the response.
- With this story's changes, they **must NOT pass `node_name`** to `complete` to avoid double-dispatch.
- Direct callers (those bypassing `run_skill`/`run_tools`) pass `node_name` to opt into usage tracking.

**Pattern:**

- **Story 4 callers:** Call `complete()` without `node_name`, extract usage from `response.usage` themselves.
- **Direct callers:** Call `complete(node_name="node")`, receive usage in the return tuple.

---

## Edge Cases

### Edge Case 1: Response has no `usage` metadata

**Scenario:** `ModelGateway.complete` returns a response without a `usage` attribute (e.g., a mocked response in tests, or a non-standard model).

**Handling:**

- `_extract_usage` checks `hasattr(response, 'usage')`.
- If `False`, return a dict with all counts set to 0.
- No error is raised; usage tracking gracefully degrades.

**Test:** Verify `_extract_usage` returns all-zeros when `response.usage` is missing.

---

### Edge Case 2: `node_name` is `None`

**Scenario:** `complete` is called without `node_name` (existing usage pattern).

**Handling:**

- `complete` returns only `response` (not a tuple).
- Usage is logged but not returned.

**Test:** Verify backward compatibility — calls without `node_name` work as before.

---

### Edge Case 3: Token fields are missing in `response.usage`

**Scenario:** `response.usage` exists but has only `input_tokens` and `output_tokens` (no cache fields).

**Handling:**

- Use `getattr(response.usage, 'cache_creation_input_tokens', 0)` to default missing fields to `0`.
- Returned `usage_data` includes all four fields, with missing ones set to `0`.

**Test:** Verify `_extract_usage` handles partial usage metadata (some fields missing).

---

### Edge Case 4: Caller provides `node_name` but doesn't unpack tuple

**Scenario:** A caller passes `node_name` but assigns the result to a single variable: `response = gateway.complete(..., node_name="node")`.

**Handling:**

- `response` is now a tuple `(response_obj, usage_data)`, not a response object.
- Downstream code expecting `response.content` will fail (tuple has no `content` attribute).
- **This is a caller error, not a gateway error.**

**Mitigation:**

- Document in the docstring: "If `node_name` is provided, unpack the returned tuple: `(response, usage_data) = gateway.complete(..., node_name='node')`."
- No runtime check in the gateway (callers are responsible for unpacking).

**Test:** Not tested in gateway tests (caller responsibility), but integration tests verify correct usage.

---

### Edge Case 5: Caller from Story 4 accidentally passes `node_name`

**Scenario:** `run_skill` or `run_tools` (from Story 4) passes `node_name` to `gateway.complete`.

**Handling:**

- Usage is returned in the tuple, but `run_skill`/`run_tools` also extract usage themselves.
- Both extracted usage dicts are included in the state update, causing double-counting.
- **This is incorrect behavior.**

**Mitigation:**

- **Story 4 callers (`run_skill`, `run_tools`) must NOT pass `node_name` to `gateway.complete`.**
- Document this in the `complete` docstring: "If calling from `run_skill` or `run_tools`, do not pass `node_name` — those functions extract usage themselves."
- Code review ensures Story 4 implementations don't pass `node_name`.

**Test:** Integration test verifies usage is not double-counted when `run_skill` calls `gateway.complete` (no `node_name` passed).

---

### Edge Case 6: Calling node doesn't have access to state update mechanism

**Scenario:** A utility function calls `gateway.complete` but can't return a state update dict (not part of the LangGraph node flow).

**Handling:**

- The utility simply doesn't pass `node_name` (or passes `None`).
- Usage is logged but not returned.
- No error; the call completes normally.

**Test:** Verify `complete` works without `node_name` (backward compatibility).

---

## Tests

### Test File: `tests/test_gateway_usage.py`

**Setup:**

- Mock the model call in `ModelGateway._call_model` to return canned responses with usage metadata.
- Use real `ModelGateway.complete` (no need to test the model SDK itself, just the usage extraction logic).

---

#### Test 1: `complete` called with `node_name` → usage returned in tuple

```python
from unittest.mock import MagicMock, patch
from src.agents.model.gateway import ModelGateway

def test_complete_with_node_name_returns_usage():
    # Mock model response with usage
    mock_response = MagicMock()
    mock_response.content = "Test response"
    mock_response.usage = MagicMock(
        input_tokens=600,
        output_tokens=120,
        cache_creation_input_tokens=80,
        cache_read_input_tokens=520
    )
    mock_response.tool_calls = []
    
    gateway = ModelGateway()
    
    with patch.object(gateway, '_call_model', return_value=mock_response):
        response, usage_data = gateway.complete(
            messages=[{"role": "user", "content": "test"}],
            node_name="custom_node"
        )
    
    # Verify response is returned
    assert response.content == "Test response"
    
    # Verify usage data matches response fields
    assert usage_data == {
        "custom_node": {
            "calls": 1,
            "input_tokens": 600,
            "output_tokens": 120,
            "cache_creation_input_tokens": 80,
            "cache_read_input_tokens": 520
        }
    }
```

---

#### Test 2: `complete` called without `node_name` → usage logged, only response returned

```python
def test_complete_without_node_name_returns_response_only():
    mock_response = MagicMock()
    mock_response.content = "Test response"
    mock_response.usage = MagicMock(
        input_tokens=600,
        output_tokens=120,
        cache_creation_input_tokens=80,
        cache_read_input_tokens=520
    )
    mock_response.tool_calls = []
    
    gateway = ModelGateway()
    
    with patch.object(gateway, '_call_model', return_value=mock_response):
        with patch('src.agents.model.gateway.logger') as mock_logger:
            result = gateway.complete(
                messages=[{"role": "user", "content": "test"}],
                # No node_name
            )
    
    # Verify usage was logged (existing behavior)
    mock_logger.info.assert_called_once()
    log_message = mock_logger.info.call_args[0][0]
    assert "input=600" in log_message
    assert "output=120" in log_message
    
    # Verify only response returned (not a tuple)
    assert result is mock_response
    assert result.content == "Test response"
```

---

#### Test 3: Verify returned usage matches response fields exactly

```python
def test_returned_usage_matches_response_fields():
    mock_response = MagicMock()
    mock_response.content = "Test response"
    mock_response.usage = MagicMock(
        input_tokens=1500,
        output_tokens=300,
        cache_creation_input_tokens=200,
        cache_read_input_tokens=1300
    )
    mock_response.tool_calls = []
    
    gateway = ModelGateway()
    
    with patch.object(gateway, '_call_model', return_value=mock_response):
        response, usage_data = gateway.complete(
            messages=[{"role": "user", "content": "test"}],
            node_name="test_node"
        )
    
    usage = usage_data["test_node"]
    
    # Verify all fields match exactly
    assert usage["calls"] == 1
    assert usage["input_tokens"] == 1500
    assert usage["output_tokens"] == 300
    assert usage["cache_creation_input_tokens"] == 200
    assert usage["cache_read_input_tokens"] == 1300
```

---

#### Test 4: Response missing `usage` → returns all-zeros

```python
def test_complete_response_missing_usage_returns_zeros():
    mock_response = MagicMock()
    mock_response.content = "Test response"
    # No usage attribute
    del mock_response.usage
    mock_response.tool_calls = []
    
    gateway = ModelGateway()
    
    with patch.object(gateway, '_call_model', return_value=mock_response):
        response, usage_data = gateway.complete(
            messages=[{"role": "user", "content": "test"}],
            node_name="test_node"
        )
    
    # Verify all fields are zero (no error raised)
    usage = usage_data["test_node"]
    assert usage["calls"] == 1
    assert usage["input_tokens"] == 0
    assert usage["output_tokens"] == 0
    assert usage["cache_creation_input_tokens"] == 0
    assert usage["cache_read_input_tokens"] == 0
```

---

#### Test 5: Partial usage metadata (missing cache fields) → defaults to 0

```python
def test_returned_usage_defaults_missing_fields():
    mock_response = MagicMock()
    mock_response.content = "Test response"
    mock_response.usage = MagicMock(
        input_tokens=800,
        output_tokens=150
        # Missing cache_creation_input_tokens and cache_read_input_tokens
    )
    # Simulate missing attributes by deleting them
    del mock_response.usage.cache_creation_input_tokens
    del mock_response.usage.cache_read_input_tokens
    mock_response.tool_calls = []
    
    gateway = ModelGateway()
    
    with patch.object(gateway, '_call_model', return_value=mock_response):
        response, usage_data = gateway.complete(
            messages=[{"role": "user", "content": "test"}],
            node_name="test_node"
        )
    
    usage = usage_data["test_node"]
    
    # Verify missing fields default to 0
    assert usage["input_tokens"] == 800
    assert usage["output_tokens"] == 150
    assert usage["cache_creation_input_tokens"] == 0
    assert usage["cache_read_input_tokens"] == 0
```

---

## Implementation Checklist

- [ ] Modify `ModelGateway.complete` signature in `src/agents/model/gateway.py`:
  - [ ] Add `node_name: Optional[str] = None` parameter
  - [ ] Change return type annotation to `Union[object, tuple[object, dict]]`
- [ ] Implement `_extract_usage` helper method in `ModelGateway`:
  - [ ] Check if `response.usage` exists
  - [ ] Extract `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
  - [ ] Build usage dict with `calls: 1`
  - [ ] Return dict with all fields (default missing fields to 0)
- [ ] Modify `complete` method to return usage when `node_name` provided:
  - [ ] After model call, check if `node_name` is not None
  - [ ] If yes, call `_extract_usage` and return `(response, usage_data)`
  - [ ] If no, return only `response` (existing behavior)
- [ ] Preserve existing usage logging (add to logs regardless of `node_name`)
- [ ] Add docstring note: "If calling from `run_skill` or `run_tools`, do not pass `node_name`"
- [ ] Write unit tests in `tests/test_gateway_usage.py` (5 tests):
  - [ ] Test 1: `node_name` provided → tuple returned with usage
  - [ ] Test 2: `node_name` not provided → response only returned
  - [ ] Test 3: returned usage matches response fields
  - [ ] Test 4: response missing `usage` → all-zeros returned
  - [ ] Test 5: partial usage metadata → missing fields default to 0
- [ ] Run tests: `uv run pytest tests/test_gateway_usage.py -v`
- [ ] Verify all 5 tests pass
- [ ] Verify existing tests pass: `uv run pytest -q`

---

## Dependencies

**Depends on:**

- Story 1 (State schema) — `State.usage` must exist with additive reducer
- Story 4 (Usage in run_skill/MCP) — establishes the usage extraction pattern

**Does not depend on:**

- Story 2, 3 (Forced routing) — usage tracking is independent of routing logic

**Blocks:**

- Story 6 (Logging and API) — needs usage data to be populated (Story 4, 5 complete the collection)

**Integration:**

- This story completes the usage collection feature (Stories 1, 4, 5 together cover all model call sites).
- Story 6 will aggregate and report the collected data.

---

## Notes

- **Backward compatibility:** Existing calls to `gateway.complete` without `node_name` work unchanged (return only response).
- **No double dispatch:** `run_skill` and `run_tools` (from Story 4) must NOT pass `node_name` to `gateway.complete` — they extract usage themselves. This story's changes are for **direct calls only** (calls that bypass `run_skill`/`run_tools`).
- **Shared utility:** The `_extract_usage` logic is similar to Story 4's utility function. Consider extracting to `src/agents/utils/usage.py` to avoid duplication, but inline implementation is acceptable for this story.
- **Response structure:** Verify the exact structure of `response.usage` by inspecting `src/agents/model/gateway.py` and the Anthropic SDK documentation. Adjust `getattr` calls if field names differ.
- **Logging:** Preserve existing usage logging (the `logger.info` call) for observability, regardless of whether usage is returned. Both logs and state updates are valuable.
- **Caller responsibility:** Direct callers must unpack the tuple when `node_name` is provided: `(response, usage_data) = gateway.complete(..., node_name='node')`. The gateway doesn't validate this at runtime.

---

**End of LLD — Story 05-agent**
