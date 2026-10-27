# Tech Spec — PER-59

**ticket:** PER-59  
**title:** Support multi-step tool use in the agent orchestrator  
**story points:** (inherited from breakdown)  
**made from:** PER-56 requirements + HLD + breakdown + review_feedback.md  
**source:** designer (LLD)

---

## Overview

This tech stub covers the full PER-56 feature: replace the single tool round in `Orchestrator.handle_query` with a bounded multi-step loop, enabling the LLM to call tools, observe results, and call additional tools until it produces a text answer or hits a configurable step limit (`max_steps`). The loop preserves existing tool-execution logic (validation, timeout, error codes) while removing the old `tools=None` synthesis call and its "synthesis loop detected" guard. When the step limit is reached, one final `tools=None` call attempts to force an answer; failure raises `RuntimeError`.

**Change from prior version:** The HTTP endpoint is now `/query_given` (was `/query`). All contracts, specifications, route handlers, test fixtures, and documentation reference `/query_given`.

---

## Stack

- **Language:** Python 3.11+
- **Framework:** None (synchronous request handler)
- **LLM client:** `llm_client.infer(messages, tools)` (existing, unchanged)
- **MCP client:** `mcp_client` (existing, unchanged)
- **HTTP server:** `run_server` (existing, parses env vars and instantiates `Orchestrator`; route handler updated to `/query_given`)
- **Test framework:** `pytest`
- **Logging:** `logging` module (existing)

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

**Change:** Add `max_steps: int = 5` parameter and store it as an instance field.

---

### Environment variable (new)

**Name:** `AGENT_MAX_STEPS`  
**Type:** string (parsed as positive integer in `run_server`)  
**Default:** `"5"` (when absent or invalid)  
**Valid values:** `"1"`, `"2"`, ..., `"99"`, etc. (any string that parses as a positive int)  
**Invalid values:** `"0"`, `"-1"`, `"foo"`, `"3.5"` → log warning, use default `5`  
**Absent (`""`):** Use default `5`, no warning (env var not set is normal)

Parsing logic in `run_server`:

```python
try:
    raw = os.environ.get("AGENT_MAX_STEPS", "")
    if raw:
        max_steps = int(raw)
        if max_steps <= 0:
            raise ValueError("AGENT_MAX_STEPS must be positive")
    else:
        max_steps = 5  # default when env var is absent
except (ValueError, TypeError):
    logger.warning("Invalid AGENT_MAX_STEPS, using default 5")
    max_steps = 5
```

---

### HTTP endpoint contract (changed)

**Endpoint:** `POST /query_given`  
**Request body:** `{"query": "<user question>"}`  
**Response (success):** `200 OK`, body: `{"answer": "<text answer>"}`  
**Response (validation error):** `400 Bad Request`, body: `{"error": "<ValueError message>"}`  
**Response (other error):** `500 Internal Server Error`, body: `{"error": "<RuntimeError or other exception message>"}`

**Change from prior version:** Endpoint path is `/query_given` (was `/query`). Route handler registration uses `@app.route("/query_given", methods=["POST"])`. Client examples, test requests, and README documentation all reference `/query_given`. No `/query` route exists.

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
- **Invalid AGENT_MAX_STEPS (new):** `logger.warning("Invalid AGENT_MAX_STEPS, using default 5")` (logged, not raised)

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

2. **`app/run_server.py`**
   - **Change:** Add `import os` and `import logging` at top (if not already present).
   - **Change:** Parse `AGENT_MAX_STEPS` from `os.environ` before instantiating `Orchestrator` (see parsing logic above).
   - **Change:** Pass `max_steps=<parsed_value>` to `Orchestrator(...)` constructor.
   - **Change:** Update HTTP route handler from `@app.route("/query", methods=["POST"])` to `@app.route("/query_given", methods=["POST"])`.
   - **Lines affected:** ~10–15 lines (env var parsing + logging + constructor call update + route handler path change).

3. **`tests/test_orchestrator.py`**
   - **Change:** Add new test cases for AC-1 through AC-5 (direct text answer, two-step chain, step limit with/without answer, unknown tool on step 2).
   - **Change:** Update or remove existing test that asserts "synthesis loop detected" error (AC-7).
   - **Lines affected:** ~100–150 lines (5 new test cases + cleanup of old test).

4. **`tests/test_http_server.py`** (or `tests/test_run_server.py` if separate)
   - **Change:** Add new test cases for `AGENT_MAX_STEPS` parsing (valid value, invalid values).
   - **Change:** Update test fixtures that send requests to `/query` endpoint to use `/query_given` instead.
   - **Lines affected:** ~30–50 lines (2 env var test cases + endpoint path updates in existing tests).

5. **`README.md`**
   - **Change:** Add `AGENT_MAX_STEPS` to environment-variables table (or create table if it does not exist).
   - **Change:** Add one-sentence capability description: "The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit."
   - **Change:** Update any API endpoint documentation or example requests from `/query` to `/query_given`.
   - **Lines affected:** ~5–10 lines (table row + sentence + endpoint path updates).

### Files to read (context only)

- **`app/agent/orchestrator.py`** (existing implementation of tool validation, execution, message appending).
- **`app/llm_client.py`** (existing `infer` method signature and return types).
- **`app/mcp_client.py`** (existing tool-execution interface; unchanged).
- **`tests/test_orchestrator.py`** (existing test structure, mocking patterns).

---

## Risks

### RISK-1: LLM never converges to a text answer

**Likelihood:** Medium (model may loop indefinitely on certain queries)  
**Impact:** High (user gets no answer, wasted API calls)  
**Mitigation (implemented):** `max_steps` hard limit + final `tools=None` call forces the model to attempt an answer. If it still fails, `RuntimeError` with clear message tells the user the query could not be answered.  
**Residual risk:** The final answer may be low-quality ("I don't have enough information") vs a crisp "exceeded step limit" user-facing message. Acceptance: low-quality answer is better than a crash; the step-limit error is for unrecoverable cases.

---

### RISK-2: Step limit too low for complex queries

**Likelihood:** Low (default 5 is generous for typical two- or three-hop queries)  
**Impact:** Medium (query fails when it could have succeeded with one more step)  
**Mitigation (implemented):** `AGENT_MAX_STEPS` env var allows tuning per deployment. README documents the parameter.  
**Residual risk:** Users may not discover the env var. Acceptance: default 5 is a reasonable balance; production deployments can tune after observing real query patterns.

---

### RISK-3: Tool execution timeout unchanged (10s per tool)

**Likelihood:** Low (existing 10s timeout has been sufficient)  
**Impact:** Low (multi-step query may hit timeout on one tool, failing the whole query)  
**Mitigation (deferred):** Timeout is per-tool, not per-query. A five-step query can run up to 50s of tool time (plus LLM latency). If this becomes a problem, future work can add a per-query timeout or make the per-tool timeout configurable.  
**Residual risk:** Accepted; no change needed for this feature.

---

### RISK-4: Message list grows unbounded in pathological cases

**Likelihood:** Very Low (bounded by `max_steps`)  
**Impact:** Low (memory usage, LLM context window)  
**Mitigation (implicit):** Each step adds exactly two messages (assistant + user). Max message count = `1 (user query) + 2 * max_steps + 1 (final call) = 2 * max_steps + 2`. At default `max_steps=5`, that's 12 messages. Well within typical context windows (100k+ tokens).  
**Residual risk:** Accepted; no explicit truncation needed.

---

### RISK-5: Endpoint path change breaks existing clients

**Likelihood:** High (if existing clients are hardcoded to `/query`)  
**Impact:** High (client requests fail with 404)  
**Mitigation (documentation):** README updated with new endpoint path `/query_given`. All test fixtures updated. API documentation updated (if exists).  
**Residual risk:** Clients not updated will break. Acceptance: this is a breaking change per review feedback; clients must be updated to the new endpoint. Consider adding a deprecation notice or redirect from `/query` to `/query_given` in a follow-up story if backward compatibility is required.

---

## Tests

All tests follow the same structure and assertions as the per-story specs in `stories/01-agent.md`, `stories/02-agent.md`, `stories/03-agent.md`, and `stories/04-agent.md`. The test count and verification approach are as defined in `breakdown.md`:

### Unit tests (`tests/test_orchestrator.py`)

Mock `llm_client.infer` and `mcp_client.execute_tools` to return canned responses. Assert on call counts, message-list contents, return values, and exceptions.

1. **Direct text answer (AC-1):** One `infer` call, no tool calls, text returned.
2. **Two-step tool chain (AC-2):** `tool_use` → `tool_use` → `text` returns the text; MCP called twice in order; second `infer` call's `messages` includes first tool result.
3. **Step limit with answer (AC-3):** `max_steps=2`, two tool rounds, final `infer(..., tools=None)` returns text → text returned.
4. **Step limit without answer (AC-4):** `max_steps=2`, all calls return `tool_use` → `RuntimeError` raised.
5. **Unknown tool on step 2 (AC-5):** First call valid, second call unknown tool → `ValueError` raised before third LLM call.
6. **Remove old synthesis-loop test (AC-7):** Existing test asserting "synthesis loop detected" is removed or updated; all tests pass.

**Total:** 6 test cases in `tests/test_orchestrator.py`.

---

### Unit tests (`tests/test_http_server.py`)

Mock `os.environ` to test `AGENT_MAX_STEPS` parsing. Update existing tests that send requests to `/query` to use `/query_given`.

7. **Valid `AGENT_MAX_STEPS` (AC-6):** Value `"3"` → `Orchestrator` instantiated with `max_steps=3`, no warning.
8. **Invalid `AGENT_MAX_STEPS` (AC-6):** Values `"foo"`, `"-1"`, `"3.5"` → `Orchestrator` instantiated with `max_steps=5`, warning logged. Value `""` (absent) → `max_steps=5`, no warning.
9. **Endpoint path updated:** Existing HTTP tests that POST to `/query` are updated to POST to `/query_given`.

**Total:** 2 env var test cases + endpoint path updates in existing tests.

---

### Manual verification (AC-8)

10. **README environment-variables table:** Includes `AGENT_MAX_STEPS` row with description and default `5`.
11. **README capability description:** Includes one sentence describing multi-step tool use.
12. **README endpoint documentation:** All endpoint references updated from `/query` to `/query_given`.

**Total:** 3 manual checks.

---

## Verification summary

| Category | Automated tests | Manual checks |
|----------|-----------------|---------------|
| Orchestrator loop logic | 6 unit tests | 0 |
| Env var parsing | 2 unit tests | 0 |
| HTTP endpoint path | Updates to existing tests | 0 |
| README documentation | 0 | 3 manual checks |
| **Total** | **8 unit tests + existing test updates** | **3 manual checks** |

All tests must pass with `python -m pytest -q` before the feature is considered complete.

---

## Acceptance criteria (mapped to tests)

- **AC-1 (direct text):** Test 1
- **AC-2 (two-step chain):** Test 2
- **AC-3 (step limit with answer):** Test 3
- **AC-4 (step limit without answer):** Test 4
- **AC-5 (unknown tool on step 2):** Test 5
- **AC-6 (AGENT_MAX_STEPS parsing):** Tests 7–8
- **AC-7 (old test removed):** Test 6
- **AC-8 (README updated):** Manual checks 10–12

---

## Implementation notes

- **Reuse existing logic:** Tool validation, execution, message appending, error handling are **unchanged**. Only the control flow (loop vs single call) and endpoint path are new.
- **Step counter semantics:** `step` starts at 0, increments after `tool_use` response (after tools execute). Direct text answer never increments `step`. A `max_steps=2` query executes at most 2 tool rounds (steps 1 and 2), then final call.
- **Logging:** One `logger.info` line per step, after tools execute. Tool names extracted from `tool_calls` list: `[call["name"] for call in tool_calls]`.
- **Final fallback:** Always called when `step == max_steps`, even if last tool round looked promising. The final call uses `tools=None` to signal "answer now or never."
- **Error propagation:** `ValueError` and `RuntimeError` both propagate to HTTP layer unchanged (400 and 500 respectively). No new HTTP error codes.
- **Endpoint path:** Route handler updated to `/query_given`. All test requests updated. README updated.

---

## Verification checklist

- [ ] All unit tests in `tests/test_orchestrator.py` pass (Tests 1–6).
- [ ] All unit tests in `tests/test_http_server.py` pass (Tests 7–8).
- [ ] Existing HTTP tests updated to use `/query_given` endpoint.
- [ ] `python -m pytest -q` exits 0 (all tests pass).
- [ ] README updated with `AGENT_MAX_STEPS` row and capability sentence.
- [ ] README endpoint documentation updated from `/query` to `/query_given`.
- [ ] Manual review: HTTP status codes unchanged (400 for `ValueError`, 500 for others, 200 for success).
- [ ] Manual review: no changes to `mcp_client.py`.
