# Tech Spec — Story 01-agent

**ticket:** 01-agent  
**title:** Implement bounded multi-step tool loop in Orchestrator  
**story points:** 8  
**made from:** PER-56 requirements + HLD + breakdown  
**source:** designer (LLD)

---

## Acceptance criteria

From requirements.md and breakdown.md:

1. **Direct text answer:** One `infer` call, no tool calls, text returned (existing behavior preserved).
2. **Two-step tool chain:** `tool_use` → `tool_use` → `text` returns the text; MCP called twice in order; second `infer` call's `messages` includes first tool result.
3. **Step limit with answer:** `max_steps=2`, two tool rounds, final `infer(..., tools=None)` returns text → text returned.
4. **Step limit without answer:** `max_steps=2`, all calls return `tool_use` → `RuntimeError` raised.
5. **Unknown tool on step 2:** First call valid, second call unknown tool → `ValueError` raised before third LLM call.

---

## File plan

### Files to modify

1. **`app/agent/orchestrator.py`**
   - **Lines affected:** ~40–50
   - **Changes:**
     - Add `max_steps: int = 5` parameter to `__init__`, store as `self.max_steps`.
     - Add `import logging` at top if not present; get logger: `logger = logging.getLogger(__name__)`.
     - Replace single tool-execution block in `handle_query` with bounded loop (pseudocode below).
     - Remove old `tools=None` synthesis call and "synthesis loop detected" `RuntimeError` guard.
     - Add step counter (`step = 0`), increment after each `tool_use` response.
     - Add `logger.info(f"Step {step}: executed tools {tool_names}")` after tool execution.
     - Add final `llm_client.infer(messages, tools=None)` call when `step == max_steps`.
     - Raise `RuntimeError("agent exceeded max_steps without an answer")` if final call does not return non-empty text.

2. **`tests/test_orchestrator.py`**
   - **Lines affected:** ~100–150
   - **Changes:**
     - Add 5 new test cases (AC 1–5 above).
     - Remove or update existing test that asserts "synthesis loop detected" error.
     - Use `unittest.mock` or `pytest-mock` to mock `llm_client.infer` and `mcp_client.execute_tools`.

### Files to read (context only, no changes)

- **`app/llm_client.py`** — existing `infer(messages, tools)` signature and return types.
- **`app/mcp_client.py`** — existing tool-execution interface (unchanged).

---

## Interfaces

### Orchestrator constructor signature (modified)

```python
class Orchestrator:
    def __init__(
        self,
        llm_client: LLMClient,
        mcp_client: MCPClient,
        tools: list[dict],
        max_steps: int = 5  # NEW PARAMETER
    ):
        self.llm_client = llm_client
        self.mcp_client = mcp_client
        self.tools = tools
        self.max_steps = max_steps  # NEW FIELD
```

---

### Loop control flow (handle_query algorithm)

**Input:** `query: str`  
**Output:** `str` (text answer)  
**Raises:** `ValueError` (unknown tool, missing required field), `RuntimeError` (step limit exceeded without answer)

**Algorithm:**

```python
def handle_query(self, query: str) -> str:
    messages = [{"role": "user", "content": query}]
    step = 0
    
    while step < self.max_steps:
        response = self.llm_client.infer(messages, tools=self.tools)
        
        if response["type"] == "text":
            text = response["content"]
            if not text.strip():
                raise ValueError("LLM returned empty text")
            return text
        
        if response["type"] == "tool_use":
            tool_calls = response["content"]  # list of tool_use dicts
            
            # EXISTING VALIDATION LOGIC (unchanged)
            for call in tool_calls:
                if call["name"] not in {t["name"] for t in self.tools}:
                    raise ValueError(f"unknown tool {call['name']}")
                # check required fields per tool schema (existing logic)
                # uppercase call["id"] (existing logic)
            
            # EXISTING EXECUTION LOGIC (unchanged, 10s timeout per tool)
            results = self.mcp_client.execute_tools(tool_calls)
            
            # APPEND MESSAGES (existing pattern, now in loop)
            assistant_msg = {"role": "assistant", "content": tool_calls}
            user_msg = {"role": "user", "content": results}
            messages.append(assistant_msg)
            messages.append(user_msg)
            
            # NEW: increment step and log
            step += 1
            tool_names = [call["name"] for call in tool_calls]
            logger.info(f"Step {step}: executed tools {tool_names}")
            continue
    
    # step == max_steps: final attempt with tools=None
    response = self.llm_client.infer(messages, tools=None)
    
    if response["type"] == "text":
        text = response["content"]
        if text.strip():
            return text
    
    raise RuntimeError("agent exceeded max_steps without an answer")
```

**Key points:**

- `step` starts at 0; incremented **after** each `tool_use` response (after tools execute).
- Direct text answer (first call) → `step` stays 0, returns immediately.
- `tools=self.tools` on every loop iteration; only the final fallback uses `tools=None`.
- Message list grows: `[user query] → [user, assistant tool_use, user tool_result] → [user, asst, user, asst, user] → ...`
- Existing validation/execution logic is **reused verbatim**, now inside the loop.
- Logging at `logger.info` after tools execute, one line per step.

---

### Message-list evolution (example)

**Initial state (step 0):**

```json
[
  {"role": "user", "content": "Who is EMP-001's manager and what are that manager's sales orders?"}
]
```

**After step 1** (LLM returns `tool_use` with `get_employee`, MCP returns `{"manager_id": "EMP-042"}`):

```json
[
  {"role": "user", "content": "Who is EMP-001's manager and what are that manager's sales orders?"},
  {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]},
  {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]}
]
```

**After step 2** (LLM returns `tool_use` with `get_sales_orders`, MCP returns `[{"order_id": "SO-123"}]`):

```json
[
  {"role": "user", "content": "Who is EMP-001's manager and what are that manager's sales orders?"},
  {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]},
  {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01", "content": "{\"manager_id\": \"EMP-042\"}"}]},
  {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_02", "name": "get_sales_orders", "input": {"employee_id": "EMP-042"}}]},
  {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_02", "content": "[{\"order_id\": \"SO-123\"}]"}]}
]
```

**Step 3 (LLM call):** returns text `"EMP-001's manager is EMP-042, who has one sales order: SO-123."` → orchestrator returns this text.

---

## Algorithm notes

### Step counter semantics

- `step` starts at 0.
- Incremented **after** each `tool_use` response (after tools execute).
- A direct text answer (no tools) never increments `step` → stays 0.
- A `max_steps=2` query executes at most 2 tool rounds (steps 1 and 2), then final call.

### Final fallback call

- Always called when `step == max_steps`, even if last tool round looked promising.
- Uses `tools=None` to signal "answer now or never."
- If final call returns non-empty text → return it.
- If final call returns `tool_use` or empty text → raise `RuntimeError`.

### Logging format

- One `logger.info` line per step, **after** tools execute.
- Tool names extracted from `tool_calls` list: `[call["name"] for call in tool_calls]`.
- Format: `"Step 1: executed tools ['get_employee']"`.

### Error propagation

- `ValueError` (unknown tool, missing field) → raised during validation, interrupts loop → HTTP 400.
- `RuntimeError` (step limit exceeded) → raised after final fallback fails → HTTP 500.
- Empty text → existing check raises `ValueError("LLM returned empty text")` → HTTP 400.

---

## Tests

### Test structure

All tests in `tests/test_orchestrator.py`.  
Mock `llm_client.infer` and `mcp_client.execute_tools` using `unittest.mock` or `pytest-mock`.  
Canned response dicts: `{"type": "text", "content": "..."}` or `{"type": "tool_use", "content": [{"id": "...", "name": "...", "input": {...}}]}`.

---

### Test 1: Direct text answer (AC-1)

**Setup:**
- Mock `llm_client.infer` to return `{"type": "text", "content": "Hello, world!"}` on first call.
- Instantiate `Orchestrator` with `max_steps=5`.

**Execute:**
- Call `orchestrator.handle_query("Say hello")`.

**Assert:**
- `llm_client.infer` called exactly once.
- `mcp_client.execute_tools` called zero times.
- Return value is `"Hello, world!"`.
- No log lines emitted (step counter stays 0).

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
- Instantiate `Orchestrator` with `max_steps=5`, `tools=[{"name": "get_employee", ...}]`.

**Execute:**
- Call `orchestrator.handle_query("Who is EMP-001's manager?")`.

**Assert:**
- `llm_client.infer` called exactly twice.
- `mcp_client.execute_tools` called exactly once, with `[{"id": "toolu_01", "name": "get_employee", "input": {"employee_id": "EMP-001"}}]`.
- Second `infer` call's `messages` argument has 3 entries:
  1. `{"role": "user", "content": "Who is EMP-001's manager?"}`
  2. `{"role": "assistant", "content": [{"type": "tool_use", ...}]}`
  3. `{"role": "user", "content": [{"type": "tool_result", ...}]}`
- Return value is `"EMP-001's manager is EMP-042."`.
- One INFO log line emitted: `"Step 1: executed tools ['get_employee']"`.

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
- Call `orchestrator.handle_query("Complex query")`.

**Assert:**
- `llm_client.infer` called exactly three times.
- First call args: `tools=[...]` (tools available).
- Second call args: `tools=[...]` (tools still available).
- Third call args: `tools=None` (final fallback).
- `mcp_client.execute_tools` called exactly twice.
- Return value is `"Final answer."`.
- Two INFO log lines emitted: `"Step 1: executed tools ['tool_a']"`, `"Step 2: executed tools ['tool_b']"`.

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
- `llm_client.infer` called exactly three times.
- `mcp_client.execute_tools` called exactly twice.
- `RuntimeError` raised with message `"agent exceeded max_steps without an answer"`.
- Two INFO log lines emitted before exception.

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
- `llm_client.infer` called exactly twice (second call triggers validation error **before** third call).
- `mcp_client.execute_tools` called exactly once (only first tool round completes).
- `ValueError` raised with message `"unknown tool unknown_tool"`.
- One INFO log line emitted: `"Step 1: executed tools ['tool_a']"`.

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

### Test 6: Remove old synthesis-loop test (cleanup)

**Action:**
- Find existing test case that asserts `RuntimeError("... synthesis loop detected")`.
- Remove test case entirely (old behavior no longer exists).

**Assert:**
- `python -m pytest tests/test_orchestrator.py -q` exits 0 (no failures).

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

- [ ] Add `max_steps: int = 5` parameter to `Orchestrator.__init__`, store as `self.max_steps`.
- [ ] Add `import logging` and `logger = logging.getLogger(__name__)` at top of `orchestrator.py`.
- [ ] Replace single tool-execution block in `handle_query` with `while step < self.max_steps:` loop.
- [ ] Add step counter (`step = 0`), increment after each `tool_use` response.
- [ ] Add `logger.info(f"Step {step}: executed tools {tool_names}")` after tool execution.
- [ ] Add final `llm_client.infer(messages, tools=None)` call when `step == max_steps`.
- [ ] Raise `RuntimeError("agent exceeded max_steps without an answer")` if final call does not return non-empty text.
- [ ] Remove old `tools=None` synthesis call and "synthesis loop detected" guard.
- [ ] Add Test 1 (direct text answer).
- [ ] Add Test 2 (two-step tool chain).
- [ ] Add Test 3 (step limit with answer).
- [ ] Add Test 4 (step limit without answer).
- [ ] Add Test 5 (unknown tool on step 2).
- [ ] Remove or update Test 6 (old synthesis-loop test).
- [ ] Run `python -m pytest tests/test_orchestrator.py -q` and confirm all tests pass.
