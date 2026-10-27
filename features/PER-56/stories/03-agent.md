# Tech Spec — Story 03-agent

**ticket:** 03-agent  
**title:** Update orchestrator tests for multi-step behavior  
**story points:** 5  
**made from:** PER-56 requirements + HLD + breakdown  
**source:** designer (LLD)

---

## Acceptance criteria

From requirements.md and breakdown.md:

1. **Direct text answer (AC-1):** One `infer` call, no tool calls, text returned (existing behavior preserved).
2. **Two-step tool chain (AC-2):** `tool_use` → `tool_use` → `text` returns the text; MCP called twice in order; second `infer` call's `messages` includes first tool result.
3. **Step limit with answer (AC-3):** `max_steps=2`, two tool rounds, final `infer(..., tools=None)` returns text → text returned.
4. **Step limit without answer (AC-4):** `max_steps=2`, all calls return `tool_use` → `RuntimeError` raised.
5. **Unknown tool on step 2 (AC-5):** First call valid, second call unknown tool → `ValueError` raised before third LLM call.
6. **Old test removed (AC-7):** Existing test asserting "synthesis loop detected" is removed or updated; all tests pass.

---

## File plan

### Files to modify

1. **`tests/test_orchestrator.py`**
   - **Lines affected:** ~120–150
   - **Changes:**
     - Add 5 new test cases (Tests 1–5 below) covering multi-step loop behavior.
     - Find and remove (or update) existing test that asserts `RuntimeError("... synthesis loop detected")` (Test 6 below).
     - Use `unittest.mock` or `pytest-mock` to mock `llm_client.infer` and `mcp_client.execute_tools`.
     - Canned response dicts: `{"type": "text", "content": "..."}` or `{"type": "tool_use", "content": [{"id": "...", "name": "...", "input": {...}}]}`.

### Files to read (context only, no changes)

- **`app/agent/orchestrator.py`** — verify loop logic (added in Story 01-agent) to understand message-list structure and step counter behavior.
- **`tests/test_orchestrator.py`** — existing test structure, mocking patterns, and the old synthesis-loop test to remove.

---

## Interfaces

### Mock LLM client response format

**Text response:**

```python
{"type": "text", "content": "Hello, world!"}
```

**Tool-use response:**

```python
{
    "type": "tool_use",
    "content": [
        {
            "type": "tool_use",
            "id": "toolu_01",
            "name": "get_employee",
            "input": {"employee_id": "EMP-001"}
        }
    ]
}
```

**Notes:**
- `content` for `tool_use` is a **list** of tool-call dicts (even if only one tool).
- Each tool-call dict has `type`, `id`, `name`, `input` fields.

---

### Mock MCP client response format

**Tool-result response:**

```python
[
    {
        "type": "tool_result",
        "tool_use_id": "toolu_01",
        "content": "{\"manager_id\": \"EMP-042\"}"
    }
]
```

**Notes:**
- Return value is a **list** of tool-result dicts (matches the tool-call list length).
- Each tool-result dict has `type`, `tool_use_id`, `content` fields.
- `content` is a JSON string (as returned by the MCP server).

---

### Orchestrator message-list structure

After one tool round (step 1):

```python
[
    {"role": "user", "content": "Who is EMP-001's manager?"},
    {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {...}}]},
    {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "..."}]}
]
```

After two tool rounds (step 2):

```python
[
    {"role": "user", "content": "Who is EMP-001's manager?"},
    {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {...}}]},
    {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "..."}]},
    {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_02", "name": "get_sales_orders", "input": {...}}]},
    {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_02", "content": "..."}]}
]
```

**Key points:**
- Message list grows by 2 entries per tool round (assistant tool-use + user tool-result).
- Second `infer` call (after one tool round) receives a 3-entry message list.
- Third `infer` call (after two tool rounds) receives a 5-entry message list.

---

## Algorithm notes

### Step counter semantics (from Story 01-agent)

- `step` starts at 0.
- Incremented **after** each `tool_use` response (after tools execute).
- A direct text answer (no tools) never increments `step` → stays 0.
- A `max_steps=2` query executes at most 2 tool rounds (steps 1 and 2), then final call.

### Final fallback call (from Story 01-agent)

- Always called when `step == max_steps`, even if last tool round looked promising.
- Uses `tools=None` to signal "answer now or never."
- If final call returns non-empty text → return it.
- If final call returns `tool_use` or empty text → raise `RuntimeError("agent exceeded max_steps without an answer")`.

### Error propagation (from Story 01-agent)

- `ValueError` (unknown tool, missing field) → raised during validation, interrupts loop → HTTP 400.
- `RuntimeError` (step limit exceeded) → raised after final fallback fails → HTTP 500.
- Empty text → existing check raises `ValueError("LLM returned empty text")` → HTTP 400.

---

## Tests

### Test structure

All tests in `tests/test_orchestrator.py`.  
Mock `llm_client.infer` and `mcp_client.execute_tools` using `unittest.mock.patch` or `pytest-mock`.  
Each test sets up canned responses, calls `orchestrator.handle_query(...)`, and asserts on call counts, arguments, and return value/exception.

---

### Test 1: Direct text answer (AC-1)

**Setup:**
- Mock `llm_client.infer` to return `{"type": "text", "content": "Hello, world!"}` on first call.
- Mock `mcp_client.execute_tools` (never called, but mock it to verify call count is 0).
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[{"name": "dummy_tool"}]`.

**Execute:**
- Call `result = orchestrator.handle_query("Say hello")`.

**Assert:**
- `llm_client.infer` called exactly **1 time**.
  - First call args: `messages=[{"role": "user", "content": "Say hello"}]`, `tools=[{"name": "dummy_tool"}]`.
- `mcp_client.execute_tools` called **0 times**.
- `result == "Hello, world!"`.

**Implementation:**

```python
@patch("app.agent.orchestrator.mcp_client.execute_tools")
@patch("app.agent.orchestrator.llm_client.infer")
def test_direct_text_answer(mock_infer, mock_execute_tools):
    # Setup
    mock_infer.return_value = {"type": "text", "content": "Hello, world!"}
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=[{"name": "dummy_tool"}],
        max_steps=5
    )
    
    # Execute
    result = orchestrator.handle_query("Say hello")
    
    # Assert
    assert mock_infer.call_count == 1
    assert mock_infer.call_args[1]["messages"] == [{"role": "user", "content": "Say hello"}]
    assert mock_infer.call_args[1]["tools"] == [{"name": "dummy_tool"}]
    assert mock_execute_tools.call_count == 0
    assert result == "Hello, world!"
```

---

### Test 2: Two-step tool chain (AC-2)

**Setup:**
- Mock `llm_client.infer` to return:
  1. First call: `{"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]}`
  2. Second call: `{"type": "text", "content": "EMP-001's manager is EMP-042."}`
- Mock `mcp_client.execute_tools` to return `[{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]` on first call.
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[{"name": "get_employee", "input_schema": {...}}]`.

**Execute:**
- Call `result = orchestrator.handle_query("Who is EMP-001's manager?")`.

**Assert:**
- `llm_client.infer` called exactly **2 times**.
  - First call args: `messages=[{"role": "user", "content": "Who is EMP-001's manager?"}]`, `tools=[...]`.
  - Second call args: `messages` has **3 entries**:
    1. `{"role": "user", "content": "Who is EMP-001's manager?"}`
    2. `{"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]}`
    3. `{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]}`
  - Second call args: `tools=[...]` (tools still available).
- `mcp_client.execute_tools` called exactly **1 time**.
  - Call args: `tool_calls=[{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]`.
- `result == "EMP-001's manager is EMP-042."`.

**Implementation:**

```python
@patch("app.agent.orchestrator.mcp_client.execute_tools")
@patch("app.agent.orchestrator.llm_client.infer")
def test_two_step_tool_chain(mock_infer, mock_execute_tools):
    # Setup
    mock_infer.side_effect = [
        {"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]},
        {"type": "text", "content": "EMP-001's manager is EMP-042."}
    ]
    mock_execute_tools.return_value = [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]
    
    tools = [{"name": "get_employee", "input_schema": {"type": "object", "properties": {"employee_id": {"type": "string"}}}}]
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=tools,
        max_steps=5
    )
    
    # Execute
    result = orchestrator.handle_query("Who is EMP-001's manager?")
    
    # Assert
    assert mock_infer.call_count == 2
    
    # First call
    assert mock_infer.call_args_list[0][1]["messages"] == [{"role": "user", "content": "Who is EMP-001's manager?"}]
    assert mock_infer.call_args_list[0][1]["tools"] == tools
    
    # Second call
    second_call_messages = mock_infer.call_args_list[1][1]["messages"]
    assert len(second_call_messages) == 3
    assert second_call_messages[0] == {"role": "user", "content": "Who is EMP-001's manager?"}
    assert second_call_messages[1]["role"] == "assistant"
    assert second_call_messages[1]["content"][0]["name"] == "get_employee"
    assert second_call_messages[2]["role"] == "user"
    assert second_call_messages[2]["content"][0]["tool_use_id"] == "toolu_01"
    assert mock_infer.call_args_list[1][1]["tools"] == tools
    
    # MCP call
    assert mock_execute_tools.call_count == 1
    assert mock_execute_tools.call_args[0][0][0]["name"] == "get_employee"
    
    assert result == "EMP-001's manager is EMP-042."
```

---

### Test 3: Step limit with answer (AC-3)

**Setup:**
- Mock `llm_client.infer` to return:
  1. First call: `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]}`
  2. Second call: `{"type": "tool_use", "content": [{"id": "toolu_02", "name": "tool_b", "input": {}}]}`
  3. Third call (final `tools=None`): `{"type": "text", "content": "Final answer."}`
- Mock `mcp_client.execute_tools` to return dummy results: `[{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}]` on first call, `[{"type": "tool_result", "tool_use_id": "toolu_02", "content": "{}"}]` on second call.
- Instantiate `Orchestrator` with `max_steps=2`, `tools=[{"name": "tool_a"}, {"name": "tool_b"}]`.

**Execute:**
- Call `result = orchestrator.handle_query("Complex query")`.

**Assert:**
- `llm_client.infer` called exactly **3 times**.
  - First call args: `tools=[...]` (tools available).
  - Second call args: `tools=[...]` (tools still available).
  - Third call args: `tools=None` (final fallback).
- `mcp_client.execute_tools` called exactly **2 times**.
- `result == "Final answer."`.

**Implementation:**

```python
@patch("app.agent.orchestrator.mcp_client.execute_tools")
@patch("app.agent.orchestrator.llm_client.infer")
def test_step_limit_with_answer(mock_infer, mock_execute_tools):
    # Setup
    mock_infer.side_effect = [
        {"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_01", "name": "tool_a", "input": {}}]},
        {"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_02", "name": "tool_b", "input": {}}]},
        {"type": "text", "content": "Final answer."}
    ]
    mock_execute_tools.side_effect = [
        [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}],
        [{"type": "tool_result", "tool_use_id": "toolu_02", "content": "{}"}]
    ]
    
    tools = [{"name": "tool_a"}, {"name": "tool_b"}]
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=tools,
        max_steps=2
    )
    
    # Execute
    result = orchestrator.handle_query("Complex query")
    
    # Assert
    assert mock_infer.call_count == 3
    
    # First two calls: tools available
    assert mock_infer.call_args_list[0][1]["tools"] == tools
    assert mock_infer.call_args_list[1][1]["tools"] == tools
    
    # Third call: tools=None
    assert mock_infer.call_args_list[2][1]["tools"] is None
    
    assert mock_execute_tools.call_count == 2
    assert result == "Final answer."
```

---

### Test 4: Step limit without answer (AC-4)

**Setup:**
- Mock `llm_client.infer` to return `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]}` on **all three calls** (including final `tools=None` call).
- Mock `mcp_client.execute_tools` to return dummy results: `[{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}]` on both calls.
- Instantiate `Orchestrator` with `max_steps=2`, `tools=[{"name": "tool_a"}]`.

**Execute:**
- Call `orchestrator.handle_query("Impossible query")` and catch exception.

**Assert:**
- `llm_client.infer` called exactly **3 times**.
- `mcp_client.execute_tools` called exactly **2 times**.
- `RuntimeError` raised with message `"agent exceeded max_steps without an answer"`.

**Implementation:**

```python
@patch("app.agent.orchestrator.mcp_client.execute_tools")
@patch("app.agent.orchestrator.llm_client.infer")
def test_step_limit_without_answer(mock_infer, mock_execute_tools):
    # Setup
    mock_infer.return_value = {"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_01", "name": "tool_a", "input": {}}]}
    mock_execute_tools.return_value = [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}]
    
    tools = [{"name": "tool_a"}]
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=tools,
        max_steps=2
    )
    
    # Execute and assert exception
    with pytest.raises(RuntimeError, match="agent exceeded max_steps without an answer"):
        orchestrator.handle_query("Impossible query")
    
    # Assert call counts
    assert mock_infer.call_count == 3
    assert mock_execute_tools.call_count == 2
```

---

### Test 5: Unknown tool on step 2 (AC-5)

**Setup:**
- Mock `llm_client.infer` to return:
  1. First call: `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]}`
  2. Second call: `{"type": "tool_use", "content": [{"id": "toolu_02", "name": "unknown_tool", "input": {}}]}`
- Mock `mcp_client.execute_tools` to return dummy result: `[{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}]` on first call.
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[{"name": "tool_a"}]`.

**Execute:**
- Call `orchestrator.handle_query("Query")` and catch exception.

**Assert:**
- `llm_client.infer` called exactly **2 times** (second call triggers validation error **before** third call).
- `mcp_client.execute_tools` called exactly **1 time** (only first tool round completes).
- `ValueError` raised with message `"unknown tool unknown_tool"`.

**Implementation:**

```python
@patch("app.agent.orchestrator.mcp_client.execute_tools")
@patch("app.agent.orchestrator.llm_client.infer")
def test_unknown_tool_on_step_2(mock_infer, mock_execute_tools):
    # Setup
    mock_infer.side_effect = [
        {"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_01", "name": "tool_a", "input": {}}]},
        {"type": "tool_use", "content": [{"type": "tool_use", "id": "toolu_02", "name": "unknown_tool", "input": {}}]}
    ]
    mock_execute_tools.return_value = [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}]
    
    tools = [{"name": "tool_a"}]
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=tools,
        max_steps=5
    )
    
    # Execute and assert exception
    with pytest.raises(ValueError, match="unknown tool unknown_tool"):
        orchestrator.handle_query("Query")
    
    # Assert call counts
    assert mock_infer.call_count == 2  # Second call triggers error before third call
    assert mock_execute_tools.call_count == 1  # Only first tool round completes
```

---

### Test 6: Remove old synthesis-loop test (AC-7)

**Action:**
- Find existing test case that asserts `RuntimeError("... synthesis loop detected")` in `tests/test_orchestrator.py`.
- **Remove the test method entirely** (the old guard no longer exists in the code, so the test is obsolete).
- Alternatively, if the test structure is valuable, **replace it with Test 4 above** (step limit without answer, which is the new equivalent error).

**Verification:**
- Run `python -m pytest tests/test_orchestrator.py -q` and confirm:
  - No test failures.
  - No test named `test_synthesis_loop_detected` (or similar) remains.

**Implementation note:**

Search for patterns like:

```python
def test_synthesis_loop_detected(...):
    ...
    with pytest.raises(RuntimeError, match="synthesis loop detected"):
        ...
```

Remove the entire test method. If there's a comment explaining why the test existed, replace it with a comment pointing to Test 4:

```python
# Old synthesis-loop guard removed in Story 01-agent.
# Step-limit behavior now tested in test_step_limit_without_answer.
```

---

## Implementation checklist

- [ ] Add Test 1 (direct text answer) to `tests/test_orchestrator.py`.
- [ ] Add Test 2 (two-step tool chain) to `tests/test_orchestrator.py`.
- [ ] Add Test 3 (step limit with answer) to `tests/test_orchestrator.py`.
- [ ] Add Test 4 (step limit without answer) to `tests/test_orchestrator.py`.
- [ ] Add Test 5 (unknown tool on step 2) to `tests/test_orchestrator.py`.
- [ ] Find and remove existing test asserting "synthesis loop detected" (Test 6).
- [ ] Run `python -m pytest tests/test_orchestrator.py -q` and confirm all tests pass (exit code 0).
- [ ] Verify no test named `test_synthesis_loop_detected` (or similar) remains in the file.

---

## Notes

### Dependency on Story 01-agent

This story depends on Story 01-agent having implemented the bounded loop logic in `Orchestrator.handle_query`. If Story 01-agent is not complete, these tests will fail (loop logic does not exist).

### Mock framework

- Use `unittest.mock.patch` (Python standard library) or `pytest-mock` (if pytest is the test framework).
- Patch `llm_client.infer` and `mcp_client.execute_tools` at the module level where `Orchestrator` imports them (e.g., `app.agent.orchestrator.llm_client.infer`).
- Use `side_effect` for multi-call mocks (e.g., Test 2, Test 3) and `return_value` for single-call mocks (e.g., Test 1).

### Assertion patterns

- **Call count:** `assert mock_obj.call_count == N`
- **Call args (single call):** `assert mock_obj.call_args[1]["param_name"] == expected_value`
- **Call args (multi-call):** `assert mock_obj.call_args_list[i][1]["param_name"] == expected_value`
- **Exception:** `with pytest.raises(ExceptionType, match="message pattern"):`
- **Return value:** `assert result == expected_value`

### Message-list assertions

For Test 2 (two-step chain), the second `infer` call's `messages` argument must have **exactly 3 entries** in the correct order. Use `len(messages) == 3` and per-entry checks (`messages[0]["role"] == "user"`, etc.) to verify structure.

---

## Verification checklist

- [ ] All 6 tests in `tests/test_orchestrator.py` pass (Tests 1–6).
- [ ] `python -m pytest tests/test_orchestrator.py -q` exits 0 (no failures).
- [ ] No test named `test_synthesis_loop_detected` (or similar) remains in the file.
- [ ] Manual review: Test 2 asserts on second `infer` call's `messages` structure (3 entries).
- [ ] Manual review: Test 5 asserts `ValueError` raised **before** third LLM call (only 2 `infer` calls).
