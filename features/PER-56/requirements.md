# Requirements — PER-56

**made from:** ticket  
**source:** Mary (business analyst)

---

## Context

The agent orchestrator currently executes one LLM call, one tool round (if any), then a final synthesis call with `tools=None`. Multi-step queries (e.g., "Who is EMP-001's manager and what are that manager's sales orders?") where the second lookup depends on the first result cannot be answered.

---

## Functional requirements

### FR-1: Bounded multi-step tool loop

**Given** the orchestrator handles a query  
**When** the LLM response is `tool_use`  
**Then** the orchestrator validates and executes each tool call as today (unknown-tool and missing-required-field `ValueError`s, `id` uppercasing, 10s timeout), appends the assistant tool-use message and the user tool-result message to the conversation, and calls `llm_client.infer(messages, tools=tools)` again.

**And When** the LLM response is `text` (non-empty)  
**Then** the orchestrator returns the text and stops.

**And When** the orchestrator completes a tool round without reaching a text answer  
**Then** it repeats FR-1 until either a text answer is received or the step limit (FR-2) is reached.

---

### FR-2: Step limit enforcement

**Given** the orchestrator is configured with `max_steps: int` (default 5)  
**When** the orchestrator completes `max_steps` tool rounds without receiving a text answer  
**Then** it makes one final `llm_client.infer(messages, tools=None)` call asking for an answer with the context accumulated so far.

**And When** that final call returns non-empty text  
**Then** the orchestrator returns the text.

**And When** that final call does not return non-empty text (e.g., returns `tool_use` or empty text)  
**Then** the orchestrator raises `RuntimeError("agent exceeded max_steps without an answer")`.

---

### FR-3: Step limit configuration

**Given** `run_server` reads the environment variable `AGENT_MAX_STEPS`  
**When** the value is a valid positive integer  
**Then** `run_server` passes it to `Orchestrator(max_steps=<value>)`.

**And When** the value is invalid (non-integer, non-positive, or empty)  
**Then** `run_server` logs a warning and uses the default value `5`.

---

### FR-4: Step logging

**Given** the orchestrator enters a tool round  
**When** a tool round completes  
**Then** the orchestrator logs at INFO level: the step number and the tool names called in that round.

---

### FR-5: HTTP behavior unchanged

**Given** the `/query` endpoint receives a request  
**When** the orchestrator raises `ValueError` (unknown tool, missing required field)  
**Then** the endpoint returns HTTP 400.

**And When** the orchestrator raises any other error (including `RuntimeError` from FR-2)  
**Then** the endpoint returns HTTP 500.

**And When** the orchestrator returns text successfully  
**Then** the endpoint returns HTTP 200 with the text in the response body.

---

## Non-functional requirements

### NFR-1: No changes to MCP client or server

The MCP client (`mcp_client.py`) and the MCP server remain unchanged. Tool execution logic (validation, timeout, result structure) is preserved exactly as today.

---

### NFR-2: Backward compatibility

**Given** existing queries that produce a direct text answer without tool use  
**When** those queries are sent to the updated orchestrator  
**Then** the behavior is identical to today: one LLM call, no tool calls, text returned.

---

## Acceptance criteria

### AC-1: Direct text answer

**Given** the LLM returns text on the first call  
**When** `Orchestrator.handle_query` is invoked  
**Then** the orchestrator makes exactly one `infer` call, no tool calls, and returns the text.

---

### AC-2: Two-step tool chain

**Given** the LLM returns `tool_use` on the first call and `text` on the second  
**When** `Orchestrator.handle_query` is invoked  
**Then** the orchestrator makes two `infer` calls, executes one tool round after the first call, appends the tool-use message and tool-result message to the conversation, and returns the text from the second call.

**And** the MCP client is called exactly once, with the tools from the first `tool_use` response, in the order returned by the LLM.

**And** the second `infer` call's `messages` argument includes the first tool-use assistant message and the first tool-result user message.

---

### AC-3: Step limit with answer

**Given** the orchestrator is configured with `max_steps=2` and the LLM returns `tool_use` on the first two calls and `text` on the third (final `tools=None`) call  
**When** `Orchestrator.handle_query` is invoked  
**Then** the orchestrator executes exactly two tool rounds, then makes one `infer(..., tools=None)` call, and returns the text from that final call.

---

### AC-4: Step limit without answer

**Given** the orchestrator is configured with `max_steps=2` and the LLM returns `tool_use` on every call (including the final `tools=None` call)  
**When** `Orchestrator.handle_query` is invoked  
**Then** the orchestrator executes exactly two tool rounds, makes one final `infer(..., tools=None)` call, and raises `RuntimeError("agent exceeded max_steps without an answer")`.

---

### AC-5: Unknown tool on step 2

**Given** the LLM returns `tool_use` with a valid tool on the first call and `tool_use` with an unknown tool on the second call  
**When** `Orchestrator.handle_query` is invoked  
**Then** the orchestrator executes the first tool round, then raises `ValueError` (unknown tool) on the second round before making a third LLM call.

---

### AC-6: Environment variable parsing

**Given** `run_server` reads `AGENT_MAX_STEPS`  
**When** the value is a valid positive integer (e.g., `3`)  
**Then** `Orchestrator` is instantiated with `max_steps=3`.

**And When** the value is invalid (e.g., `"foo"`, `-1`, `""`)  
**Then** a warning is logged and `Orchestrator` is instantiated with `max_steps=5`.

---

### AC-7: Existing synthesis-loop test removed

**Given** existing tests assert the old "synthesis loop detected" `RuntimeError`  
**When** those tests are updated or removed  
**Then** all tests pass with `python -m pytest -q`.

---

### AC-8: README updated

**Given** the README lists environment variables  
**When** the README is updated  
**Then** `AGENT_MAX_STEPS` is added to the environment-variables table with a description: "Maximum tool-use steps before forcing a final answer (default: 5)."

**And** one sentence describes the multi-step tool-use capability (e.g., "The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.").

---

## Open questions

None.

---

## Verification approach

- Unit tests (`tests/test_orchestrator.py`) with mocked LLM and MCP clients cover AC-1 through AC-5.
- Unit tests (`tests/test_http_server.py`) cover AC-6.
- Manual review confirms AC-7 (old test removed/updated) and AC-8 (README updated).
