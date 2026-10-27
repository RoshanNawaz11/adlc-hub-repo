# Tech Spec — Story 01-agent

**ticket:** 01-agent  
**title:** Implement bounded multi-step tool loop in Orchestrator  
**story points:** 5  
**made from:** PER-56 requirements + HLD + breakdown + review_feedback.md  
**source:** designer (LLD)

---

## Overview

Replace the single tool-execution block in `Orchestrator.handle_query` with a bounded loop that allows up to `max_steps` tool rounds. The LLM can call tools, see results, and call more tools until it produces a text answer or reaches the step limit. When the limit is reached, make one final `tools=None` call; if that does not return text, raise `RuntimeError`. Preserve all existing tool-validation and execution logic (unknown-tool and missing-field `ValueError`s, `id` uppercasing, 10s timeout). Log each step at INFO level with the step number and tool names called.

This story implements the core loop logic (FR-1, FR-2, FR-4) without environment variable configuration (Story 2) or README documentation (Story 4). The `max_steps` parameter is hardcoded to `5` in `__init__` for now; Story 2 will wire it to `AGENT_MAX_STEPS`.

**Change from prior version:** The HTTP endpoint is `/query_given` (not `/query`), but this story does not modify the HTTP layer—that happens in Story 2. This story focuses solely on `orchestrator.py` loop logic.

---

## Stack

- **Language:** Python 3.11+
- **Framework:** None (synchronous request handler)
- **LLM client:** `llm_client.infer(messages, tools)` (existing, unchanged)
- **MCP client:** `mcp_client` (existing, unchanged)
- **Test framework:** `pytest`
- **Logging:** `logging` module (existing)

---

## Acceptance criteria

From requirements.md and breakdown.md, this story covers:

- **AC-1:** Direct text answer: one `infer` call, no tool calls, text returned (existing behavior still passes).
- **AC-2:** Two-step tool chain: `tool_use` → `tool_use` → `text` returns the text; MCP called twice in order; second `infer` call's `messages` includes first tool result.
- **AC-3:** Step limit with answer: `max_steps=2`, two tool rounds, final `infer(..., tools=None)` returns text → text returned.
- **AC-4:** Step limit without answer: `max_steps=2`, all calls return `tool_use` → `RuntimeError` raised.
- **AC-5:** Unknown tool on step 2 still raises `ValueError`.

These ACs map to 5 unit tests in `tests/test_orchestrator.py`.

---

## Schema / API contracts

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

**Change:** Add `max_steps: int = 5` parameter and store it as an instance field. Hardcoded default `5` for this story; Story 2 will pass a parsed env var value.

---

### Loop control flow (Orchestrator.handle_query)

**Input:** `query: str` (user question)  
**Output:** `str` (text answer)  
**Raises:** `ValueError` (unknown tool, missing required field) → HTTP 400, `RuntimeError` (step limit exceeded without answer) → HTTP 500

**Pseudocode:**

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

- `step` starts at 0; incremented after each `tool_use` response (after execution completes).
- Direct text answer (first call) → `step` stays 0, returns immediately.
- `tools=self.tools` on every loop iteration; only the final fallback uses `tools=None`.
- Message list grows: `[user query] → [user, assistant tool_use, user tool_result] → [user, asst, user, asst, user] → ...`
- Existing validation/execution logic is **reused verbatim**, now inside the loop.
- Logging at `logger.info` after tools execute, one line per step.
- Remove old `tools=None` synthesis call and "synthesis loop detected" guard (no longer needed).

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

**Step 3 (LLM call):** returns text `"EMP-001's manager is EMP-042, who has one sales order: SO-123."` → return this text.

---

### Error messages

- **Unknown tool (existing):** `ValueError: unknown tool <name>` → HTTP 400
- **Missing required field (existing):** `ValueError: missing required field <field> for tool <name>` → HTTP 400
- **Step limit exceeded (new):** `RuntimeError("agent exceeded max_steps without an answer")` → HTTP 500
- **Empty text (existing):** `ValueError("LLM returned empty text")` → HTTP 400

---

## Files to create / modify

### Files to modify

1. **`app/agent/orchestrator.py`**
   - **Change:** Add `max_steps: int = 5` parameter to `__init__`, store as `self.max_steps`.
   - **Change:** Replace single tool-execution block in `handle_query` with the bounded loop (pseudocode above).
   - **Change:** Remove old `tools=None` synthesis call and "synthesis loop detected" `RuntimeError` guard (no longer needed).
   - **Change:** Add `import logging` at top (if not already present) and get logger: `logger = logging.getLogger(__name__)`.
   - **Change:** Add step counter (`step = 0`), increment after each `tool_use` round, log step + tool names at INFO.
   - **Change:** Add final `tools=None` fallback when `step == max_steps`, raise `RuntimeError` if no text answer.
   - **Lines affected:** ~30–50 lines (loop structure, step tracking, logging, final fallback).

2. **`tests/test_orchestrator.py`**
   - **Change:** Add 5 new test cases for AC-1 through AC-5 (see Tests section below).
   - **Change:** Remove or update existing test that asserts "synthesis loop detected" error (if it exists).
   - **Lines affected:** ~100–150 lines (5 new test cases + cleanup of old test).

### Files to read (context only)

- **`app/agent/orchestrator.py`** (existing implementation of tool validation, execution, message appending).
- **`app/llm_client.py`** (existing `infer` method signature and return types).
- **`app/mcp_client.py`** (existing tool-execution interface; unchanged).

---

## Interfaces

### Orchestrator.__init__ (modified)

**Before:**

```python
def __init__(
    self,
    llm_client: LLMClient,
    mcp_client: MCPClient,
    tools: list[dict]
):
    self.llm_client = llm_client
    self.mcp_client = mcp_client
    self.tools = tools
```

**After:**

```python
def __init__(
    self,
    llm_client: LLMClient,
    mcp_client: MCPClient,
    tools: list[dict],
    max_steps: int = 5  # NEW
):
    self.llm_client = llm_client
    self.mcp_client = mcp_client
    self.tools = tools
    self.max_steps = max_steps  # NEW
```

---

### Orchestrator.handle_query (modified)

**Signature unchanged:**

```python
def handle_query(self, query: str) -> str:
    ...
```

**Internal changes:**

- Replace single tool block with `while step < self.max_steps:` loop.
- Add `step = 0` at start, increment after each tool round.
- Log `f"Step {step}: executed tools {tool_names}"` after tool execution.
- Final `llm_client.infer(messages, tools=None)` call when loop exits.
- Raise `RuntimeError("agent exceeded max_steps without an answer")` if final call does not return non-empty text.
- Remove old `tools=None` synthesis call and "synthesis loop detected" guard.

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

### Logging format

One log line per step, after tools execute:

```
INFO: Step 1: executed tools ['get_employee']
INFO: Step 2: executed tools ['get_sales_orders']
```

Tool names extracted from `tool_calls` list: `[call["name"] for call in tool_calls]`.

### Final fallback logic

When `step == max_steps`, the loop exits and makes one final `llm_client.infer(messages, tools=None)` call. This signals the LLM to produce a text answer with the accumulated context, without calling more tools. If the final call returns:

- `text` (non-empty) → return it.
- `tool_use` or empty text → raise `RuntimeError("agent exceeded max_steps without an answer")`.

This matches FR-2 from requirements.md.

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

### Edge case 3: Tool execution timeout (existing)

**Scenario:** MCP tool takes >10s to execute.

**Behavior:** Existing timeout logic (unchanged) raises exception → HTTP 500. Loop does not continue.

**Test:** Not added (existing timeout tests already cover this; no new behavior).

---

### Edge case 4: Multiple tool calls in one step

**Scenario:** LLM returns `tool_use` with multiple tools in the `content` list on one call.

**Behavior:** Existing logic (unchanged) executes all tools in order, returns all results in one user message, increments step by 1 (not by tool count). Log line shows all tool names: `"Step 1: executed tools ['tool_a', 'tool_b']"`.

**Test:** Variant of Test 2 where first call returns two tools → assert one step increment, one log line with both tool names.

---

## Risks

### RISK-1: LLM never converges to text answer

**Mitigation (implemented):** `max_steps` hard limit + final `tools=None` call forces answer. If it still fails, `RuntimeError` with clear message.

**Residual risk:** Final answer may be low-quality ("I don't have enough information"). Accepted: better than crash.

---

### RISK-2: Step limit too low for complex queries

**Mitigation (deferred to Story 2):** `AGENT_MAX_STEPS` env var allows tuning. Default 5 is reasonable. This story hardcodes `max_steps=5`; Story 2 will add env var parsing.

**Residual risk (for this story):** No runtime tuning. Accepted: Story 2 adds env var support.

---

### RISK-3: Message list grows large

**Mitigation (implicit):** Bounded by `max_steps`. At `max_steps=5`, max 12 messages (1 user + 2 * 5 + 1 final). Well within typical context windows.

**Residual risk:** None; no action needed.

---

## Dependencies

- **Existing:** `llm_client.infer(messages, tools)` (unchanged).
- **Existing:** `mcp_client.execute_tools(tool_calls)` (unchanged).
- **Existing:** `Orchestrator` constructor and `handle_query` method (modified in this story).
- **Story 2 dependency:** Env var parsing (`AGENT_MAX_STEPS`) not implemented yet; this story hardcodes `max_steps=5`.
- **Story 4 dependency:** README documentation not updated yet; this story focuses on code changes only.

No new external dependencies.

---

## Implementation notes

- **Reuse existing logic:** Tool validation, execution, message appending, error handling are **unchanged**. Only the control flow (loop vs single call) is new.
- **Step counter semantics:** `step` starts at 0, increments after `tool_use` response (after tools execute). Direct text answer never increments `step`. A `max_steps=2` query executes at most 2 tool rounds (steps 1 and 2), then final call.
- **Logging:** One `logger.info` line per step, after tools execute. Tool names extracted from `tool_calls` list: `[call["name"] for call in tool_calls]`.
- **Final fallback:** Always called when `step == max_steps`, even if last tool round looked promising. The final call uses `tools=None` to signal "answer now or never."
- **Error propagation:** `ValueError` and `RuntimeError` both propagate to HTTP layer unchanged (400 and 500 respectively). No new HTTP error codes.
- **HTTP layer unchanged:** This story does not modify `run_server.py` or HTTP routes. The endpoint path change to `/query_given` happens in Story 2.

---

## Verification checklist

- [ ] All unit tests in `tests/test_orchestrator.py` pass (Tests 1–5).
- [ ] Old synthesis-loop test removed (if it existed).
- [ ] `python -m pytest tests/test_orchestrator.py -q` exits 0 (all orchestrator tests pass).
- [ ] Manual review: existing tool validation and execution logic is reused verbatim (no changes to MCP client, no changes to validation rules).
- [ ] Manual review: step counter and logging work as specified (one log line per step, tool names extracted correctly).

---

## Out of scope for this story

- **Environment variable parsing:** Story 2 will add `AGENT_MAX_STEPS` parsing in `run_server.py`. This story hardcodes `max_steps=5`.
- **HTTP endpoint path change:** Story 2 will update the route handler from `/query` to `/query_given`. This story does not modify `run_server.py`.
- **README documentation:** Story 4 will update `README.md` with `AGENT_MAX_STEPS` and capability description. This story focuses on code changes only.
- **HTTP layer tests:** Story 2 will add `tests/test_http_server.py` for env var parsing. This story adds only orchestrator unit tests.

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
- No log lines emitted (step counter stays 0).

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
- One INFO log line emitted: `"Step 1: executed tools ['get_employee']"`.

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
