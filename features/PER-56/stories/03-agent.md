# Tech Spec — Story 03-agent

**ticket:** 03-agent  
**title:** Update orchestrator tests for multi-step behavior  
**story points:** 5  
**made from:** PER-56 requirements + HLD + breakdown + review_feedback.md  
**source:** designer (LLD)

---

## Overview

Expand `tests/test_orchestrator.py` with test cases covering the bounded loop, step limit, and multi-step tool chains implemented in Story 1. Remove or update the existing test that asserts the old "synthesis loop detected" error, since that guard no longer exists. Ensure all tests pass with `python -m pytest -q`.

This story implements test coverage for AC-1 through AC-5 and AC-7 from requirements.md. It depends on Story 1 (which added the bounded loop logic to `Orchestrator.handle_query`). The tests verify the loop behavior, step limit enforcement, message-list evolution, and error handling.

**Change from prior version:** The HTTP endpoint is `/query_given` (not `/query`), but this story does not modify HTTP-layer tests—those are in Story 2. This story focuses solely on orchestrator unit tests.

---

## Stack

- **Language:** Python 3.11+
- **Framework:** None (synchronous request handler)
- **Test framework:** `pytest`
- **Mocking:** `unittest.mock` (or `pytest-mock` if installed)
- **LLM client:** `llm_client.infer(messages, tools)` (mocked in tests)
- **MCP client:** `mcp_client.execute_tools(tool_calls)` (mocked in tests)

---

## Acceptance criteria

From requirements.md and breakdown.md, this story covers:

- **AC-1:** Direct text answer: one `infer` call, no tool calls, text returned (existing behavior still passes).
- **AC-2:** Two-step tool chain: `tool_use` → `tool_use` → `text` returns the text; MCP called twice in order; second `infer` call's `messages` includes first tool result.
- **AC-3:** Step limit with answer: `max_steps=2`, two tool rounds, final `infer(..., tools=None)` returns text → text returned.
- **AC-4:** Step limit without answer: `max_steps=2`, all calls return `tool_use` → `RuntimeError` raised.
- **AC-5:** Unknown tool on step 2 still raises `ValueError`.
- **AC-7:** Old synthesis-loop test removed or updated; all tests pass.

These ACs map to 6 test cases in `tests/test_orchestrator.py`.

---

## Files to create / modify

### Files to modify

1. **`tests/test_orchestrator.py`**
   - **Change:** Add 5 new test cases for AC-1 through AC-5 (see Tests section below).
   - **Change:** Remove or update existing test that asserts "synthesis loop detected" error (AC-7).
   - **Lines affected:** ~120–150 lines (5 new test cases + cleanup of old test).

### Files to read (context only)

- **`app/agent/orchestrator.py`** (Story 1 added the bounded loop; tests verify its behavior).
- **`app/llm_client.py`** (existing `infer` method signature; mocked in tests).
- **`app/mcp_client.py`** (existing `execute_tools` method signature; mocked in tests).
- **`tests/test_orchestrator.py`** (existing test structure, mocking patterns).

---

## Interfaces

### Test helper: Mock LLM client

**Mock `llm_client.infer` to return canned responses:**

```python
from unittest.mock import Mock

llm_client = Mock()
llm_client.infer.side_effect = [
    {"type": "text", "content": "Hello, world!"},
    # or
    {"type": "tool_use", "content": [{"id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]},
    # ...
]
```

**Return value schema:**

- Text response: `{"type": "text", "content": "<text>"}`
- Tool-use response: `{"type": "tool_use", "content": [<list of tool_use dicts>]}`

**Tool-use dict schema:**

```python
{
    "id": "toolu_01",
    "name": "get_employee",
    "input": {"employee_id": "EMP-001"}
}
```

---

### Test helper: Mock MCP client

**Mock `mcp_client.execute_tools` to return canned results:**

```python
mcp_client = Mock()
mcp_client.execute_tools.return_value = [
    {"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}
]
```

**Return value schema:**

- List of tool results: `[{"type": "tool_result", "tool_use_id": "<id>", "content": "<JSON string>"}]`

---

### Test assertions

**Common assertions across all test cases:**

1. **Call count:** `assert llm_client.infer.call_count == <expected>`
2. **MCP call count:** `assert mcp_client.execute_tools.call_count == <expected>`
3. **Return value:** `assert result == "<expected text>"`
4. **Exception raised:** `with pytest.raises(ValueError, match="<expected message>"):`
5. **Messages argument:** `assert llm_client.infer.call_args_list[1][0][0] == [<expected messages>]` (verify message-list evolution)

---

## Algorithm notes

### Loop invariant

At the start of each iteration, `messages` contains the full conversation history: the original user query, plus zero or more pairs of (assistant tool-use message, user tool-result message). The loop terminates when:

1. The LLM returns `text` (non-empty) → return immediately.
2. `step == max_steps` → exit loop, make final `tools=None` call, return text or raise `RuntimeError`.

### Step counter semantics

- `step` starts at 0.
- A direct text answer (first call returns text) never increments `step`.
- Each tool round (LLM returns `tool_use`, tools execute successfully) increments `step` by 1.
- Multiple tool calls in one LLM response (e.g., two tools in the `content` list) still count as one step.
- A `max_steps=2` allows two tool rounds (steps 1 and 2), then the final fallback call.

### Message-list verification

For Test 2 (two-step chain), verify that the second `infer` call receives a `messages` list with exactly 3 entries:

1. `{"role": "user", "content": "<original query>"}`
2. `{"role": "assistant", "content": [<tool_use dict>]}`
3. `{"role": "user", "content": [<tool_result dict>]}`

Extract the argument using `llm_client.infer.call_args_list[1][0][0]` (second call, first positional arg).

---

## Edge cases / failure modes

### Edge case 1: Empty text answer on final call

**Scenario:** LLM returns `{"type": "text", "content": ""}` on final `tools=None` call.

**Behavior:** Existing empty-text check raises `ValueError("LLM returned empty text")` → HTTP 400. **Not a RuntimeError**, because empty text is a validation failure, not a step-limit failure.

**Test:** Add sub-case to Test 4 where final call returns empty text → assert `ValueError`.

---

### Edge case 2: Step limit = 1

**Scenario:** `max_steps=1`, LLM returns `tool_use` on first call, text on second.

**Behavior:** One tool round executes, then final `tools=None` call returns text → success.

**Test:** Variant of Test 3 with `max_steps=1`.

---

### Edge case 3: Multiple tool calls in one step

**Scenario:** LLM returns `tool_use` with multiple tools in the `content` list on one call.

**Behavior:** Existing logic (unchanged) executes all tools in order, returns all results in one user message, increments step by 1 (not by tool count). Log line shows all tool names: `"Step 1: executed tools ['tool_a', 'tool_b']"`.

**Test:** Variant of Test 2 where first call returns two tools → assert one step increment, one log line with both tool names.

---

## Risks

### RISK-1: Mock setup complexity

**Likelihood:** Medium (mocking multiple LLM calls with side effects can be error-prone)  
**Impact:** Low (test failures are caught early in development)  
**Mitigation (implemented):** Use `side_effect` for sequential mock returns; verify call counts and arguments explicitly.  
**Residual risk:** Tests may be brittle if mock setup is incorrect. Acceptance: careful review of test setup catches issues.

---

### RISK-2: Test coverage gaps

**Likelihood:** Low (all ACs are covered by 6 test cases)  
**Impact:** Medium (missing edge case could slip into production)  
**Mitigation (implemented):** 6 test cases cover all ACs plus 3 edge cases. Manual review of test plan confirms coverage.  
**Residual risk:** Unanticipated edge cases may exist. Acceptance: standard risk for unit testing; integration tests in later stories catch additional issues.

---

## Dependencies

- **Story 1:** `Orchestrator.handle_query` loop logic (implemented in Story 1; tests verify its behavior).
- **Existing:** `llm_client.infer` and `mcp_client.execute_tools` (mocked in tests; real implementations unchanged).
- **Test framework:** `pytest` (existing; no new dependencies).

No new external dependencies.

---

## Implementation notes

- **Mocking strategy:** Use `unittest.mock.Mock` to mock `llm_client` and `mcp_client`. Use `side_effect` for sequential return values (e.g., first call returns tool_use, second call returns text).
- **Message-list verification:** For Test 2, extract the `messages` argument from the second `infer` call using `llm_client.infer.call_args_list[1][0][0]` and assert its length and structure.
- **Logging verification:** Mock `logger.info` to verify step logging. Example:
  ```python
  with patch('app.agent.orchestrator.logger.info') as mock_log:
      orchestrator.handle_query("Query")
      mock_log.assert_called_with("Step 1: executed tools ['tool_a']")
  ```
- **Old test cleanup:** Search `tests/test_orchestrator.py` for "synthesis loop" or "synthesis loop detected" to find the old test. Remove it entirely.
- **Test organization:** Group tests by AC or behavior (e.g., "direct text", "multi-step", "step limit", "errors"). Use descriptive test method names.

---

## Verification checklist

- [ ] All unit tests in `tests/test_orchestrator.py` pass (Tests 1–6 + edge cases).
- [ ] `python -m pytest tests/test_orchestrator.py -q` exits 0 (all orchestrator tests pass).
- [ ] Test 1 (AC-1): Direct text answer → 1 `infer` call, 0 MCP calls, text returned.
- [ ] Test 2 (AC-2): Two-step chain → 2 `infer` calls, 1 MCP call, second call's `messages` has 3 entries, text returned.
- [ ] Test 3 (AC-3): Step limit with answer → 3 `infer` calls (2 with tools, 1 with `tools=None`), 2 MCP calls, text returned.
- [ ] Test 4 (AC-4): Step limit without answer → 3 `infer` calls, 2 MCP calls, `RuntimeError` raised.
- [ ] Test 5 (AC-5): Unknown tool on step 2 → 2 `infer` calls, 1 MCP call, `ValueError` raised.
- [ ] Test 6 (AC-7): Old synthesis-loop test removed; no references to "synthesis loop detected" in test file.
- [ ] Edge case tests pass (empty final answer, step limit = 1, multiple tools in one step).
- [ ] Manual review: test coverage aligns with requirements.md ACs.

---

## Out of scope for this story

- **HTTP layer tests:** Story 2 adds `tests/test_http_server.py` for env var parsing and endpoint path. This story focuses on orchestrator unit tests only.
- **README documentation:** Story 4 updates `README.md` with `AGENT_MAX_STEPS` and capability description. This story focuses on test code only.
- **Integration tests:** Out of scope; these are unit tests with mocked dependencies.
- **Performance tests:** Out of scope; functional correctness is the focus.

---

## Tests

### Test 1: Direct text answer (AC-1)

**File:** `tests/test_orchestrator.py`

**Test method:** `test_direct_text_answer`

**Setup:**
- Mock `llm_client.infer` to return `{"type": "text", "content": "Hello, world!"}` on first call.
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[]`.

**Execute:**
- Call `orchestrator.handle_query("Say hello")`.

**Assert:**
- `llm_client.infer` called exactly once.
- `mcp_client.execute_tools` called zero times.
- Return value is `"Hello, world!"`.
- No log lines emitted (step counter stays 0; can verify by mocking `logger.info` and asserting not called).

**Code snippet:**

```python
def test_direct_text_answer():
    llm_client = Mock()
    llm_client.infer.return_value = {"type": "text", "content": "Hello, world!"}
    
    mcp_client = Mock()
    
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=[],
        max_steps=5
    )
    
    result = orchestrator.handle_query("Say hello")
    
    assert llm_client.infer.call_count == 1
    assert mcp_client.execute_tools.call_count == 0
    assert result == "Hello, world!"
```

---

### Test 2: Two-step tool chain (AC-2)

**File:** `tests/test_orchestrator.py`

**Test method:** `test_two_step_tool_chain`

**Setup:**
- Mock `llm_client.infer` to return:
  1. First call: `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]}`
  2. Second call: `{"type": "text", "content": "EMP-001's manager is EMP-042."}`
- Mock `mcp_client.execute_tools` to return `[{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]` on first call.
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[{"name": "get_employee"}]`.

**Execute:**
- Call `orchestrator.handle_query("Who is EMP-001's manager?")`.

**Assert:**
- `llm_client.infer` called exactly twice.
- `mcp_client.execute_tools` called exactly once, with `[{"id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]`.
- Second `infer` call's `messages` argument has 3 entries:
  1. `{"role": "user", "content": "Who is EMP-001's manager?"}`
  2. `{"role": "assistant", "content": [{"id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]}`
  3. `{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]}`
- Return value is `"EMP-001's manager is EMP-042."`.
- One INFO log line emitted: `"Step 1: executed tools ['get_employee']"` (verify by mocking `logger.info`).

**Code snippet:**

```python
def test_two_step_tool_chain():
    llm_client = Mock()
    llm_client.infer.side_effect = [
        {"type": "tool_use", "content": [{"id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]},
        {"type": "text", "content": "EMP-001's manager is EMP-042."}
    ]
    
    mcp_client = Mock()
    mcp_client.execute_tools.return_value = [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}
    ]
    
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=[{"name": "get_employee"}],
        max_steps=5
    )
    
    result = orchestrator.handle_query("Who is EMP-001's manager?")
    
    assert llm_client.infer.call_count == 2
    assert mcp_client.execute_tools.call_count == 1
    
    # Verify second infer call's messages argument
    second_call_messages = llm_client.infer.call_args_list[1][0][0]
    assert len(second_call_messages) == 3
    assert second_call_messages[0]["role"] == "user"
    assert second_call_messages[0]["content"] == "Who is EMP-001's manager?"
    assert second_call_messages[1]["role"] == "assistant"
    assert second_call_messages[2]["role"] == "user"
    
    assert result == "EMP-001's manager is EMP-042."
```

---

### Test 3: Step limit with answer (AC-3)

**File:** `tests/test_orchestrator.py`

**Test method:** `test_step_limit_with_answer`

**Setup:**
- Mock `llm_client.infer` to return:
  1. First call: `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]}`
  2. Second call: `{"type": "tool_use", "content": [{"id": "toolu_02", "name": "tool_b", "input": {}}]}`
  3. Third call (final `tools=None`): `{"type": "text", "content": "Final answer."}`
- Mock `mcp_client.execute_tools` to return dummy results on first two calls.
- Instantiate `Orchestrator` with `max_steps=2`, `tools=[{"name": "tool_a"}, {"name": "tool_b"}]`.

**Execute:**
- Call `orchestrator.handle_query("Complex query")`.

**Assert:**
- `llm_client.infer` called exactly three times:
  1. With `tools=[...]` (first call).
  2. With `tools=[...]` (second call).
  3. With `tools=None` (final call).
- `mcp_client.execute_tools` called exactly twice.
- Return value is `"Final answer."`.
- Two INFO log lines emitted: `"Step 1: executed tools ['tool_a']"`, `"Step 2: executed tools ['tool_b']"`.

**Code snippet:**

```python
def test_step_limit_with_answer():
    llm_client = Mock()
    llm_client.infer.side_effect = [
        {"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]},
        {"type": "tool_use", "content": [{"id": "toolu_02", "name": "tool_b", "input": {}}]},
        {"type": "text", "content": "Final answer."}
    ]
    
    mcp_client = Mock()
    mcp_client.execute_tools.return_value = [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}
    ]
    
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=[{"name": "tool_a"}, {"name": "tool_b"}],
        max_steps=2
    )
    
    result = orchestrator.handle_query("Complex query")
    
    assert llm_client.infer.call_count == 3
    assert mcp_client.execute_tools.call_count == 2
    
    # Verify final call used tools=None
    final_call_kwargs = llm_client.infer.call_args_list[2][1]
    assert final_call_kwargs["tools"] is None
    
    assert result == "Final answer."
```

---

### Test 4: Step limit without answer (AC-4)

**File:** `tests/test_orchestrator.py`

**Test method:** `test_step_limit_without_answer`

**Setup:**
- Mock `llm_client.infer` to return `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]}` on all three calls (including final `tools=None` call).
- Mock `mcp_client.execute_tools` to return dummy results.
- Instantiate `Orchestrator` with `max_steps=2`, `tools=[{"name": "tool_a"}]`.

**Execute:**
- Call `orchestrator.handle_query("Impossible query")`.

**Assert:**
- `llm_client.infer` called exactly three times.
- `mcp_client.execute_tools` called exactly twice.
- `RuntimeError` raised with message `"agent exceeded max_steps without an answer"`.
- Two INFO log lines emitted before exception.

**Code snippet:**

```python
def test_step_limit_without_answer():
    llm_client = Mock()
    llm_client.infer.return_value = {
        "type": "tool_use",
        "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]
    }
    
    mcp_client = Mock()
    mcp_client.execute_tools.return_value = [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}
    ]
    
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=[{"name": "tool_a"}],
        max_steps=2
    )
    
    with pytest.raises(RuntimeError, match="agent exceeded max_steps without an answer"):
        orchestrator.handle_query("Impossible query")
    
    assert llm_client.infer.call_count == 3
    assert mcp_client.execute_tools.call_count == 2
```

---

### Test 5: Unknown tool on step 2 (AC-5)

**File:** `tests/test_orchestrator.py`

**Test method:** `test_unknown_tool_on_step_2`

**Setup:**
- Mock `llm_client.infer` to return:
  1. First call: `{"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]}`
  2. Second call: `{"type": "tool_use", "content": [{"id": "toolu_02", "name": "unknown_tool", "input": {}}]}`
- Mock `mcp_client.execute_tools` to return dummy result on first call.
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[{"name": "tool_a"}]`.

**Execute:**
- Call `orchestrator.handle_query("Query")`.

**Assert:**
- `llm_client.infer` called exactly twice (second call triggers validation error before third call).
- `mcp_client.execute_tools` called exactly once (only first tool round completes).
- `ValueError` raised with message `"unknown tool unknown_tool"`.
- One INFO log line emitted: `"Step 1: executed tools ['tool_a']"`.

**Code snippet:**

```python
def test_unknown_tool_on_step_2():
    llm_client = Mock()
    llm_client.infer.side_effect = [
        {"type": "tool_use", "content": [{"id": "toolu_01", "name": "tool_a", "input": {}}]},
        {"type": "tool_use", "content": [{"id": "toolu_02", "name": "unknown_tool", "input": {}}]}
    ]
    
    mcp_client = Mock()
    mcp_client.execute_tools.return_value = [
        {"type": "tool_result", "tool_use_id": "toolu_01", "content": "{}"}
    ]
    
    orchestrator = Orchestrator(
        llm_client=llm_client,
        mcp_client=mcp_client,
        tools=[{"name": "tool_a"}],
        max_steps=5
    )
    
    with pytest.raises(ValueError, match="unknown tool unknown_tool"):
        orchestrator.handle_query("Query")
    
    assert llm_client.infer.call_count == 2
    assert mcp_client.execute_tools.call_count == 1
```

---

### Test 6: Remove old synthesis-loop test (AC-7)

**File:** `tests/test_orchestrator.py`

**Test method:** Find and remove (or update) existing test case

**Action:**
- Find existing test case (if any) that asserts `RuntimeError("... synthesis loop detected")`.
- Remove test case entirely (old behavior no longer exists).
- If no such test exists, add a comment noting that the old behavior was removed.

**Assert:**
- `python -m pytest tests/test_orchestrator.py -q` exits 0 (all tests pass).

**Code snippet:**

```python
# OLD TEST (to be removed):
# def test_synthesis_loop_detected():
#     # This test is no longer valid; synthesis loop guard removed in Story 1
#     pass

# NEW: If the old test existed, delete it. Otherwise, no action needed.
```
