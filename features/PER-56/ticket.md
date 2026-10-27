---
key: PER-56
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Support multi-step tool use in the agent orchestrator
---

## Context

`Orchestrator.handle_query` (`app/agent/orchestrator.py`) runs one LLM call, executes at most one

round of tool calls, then makes a final "synthesis" call with `tools=None`. If the model asks for

tools again, it raises `RuntimeError("... synthesis loop detected")`.

Questions whose second lookup depends on the first result

(e.g. "Who is EMP-001's manager and what are that manager's sales orders?") cannot be answered.

## Goal

Replace the single tool round with a bounded agent loop: the model can call tools, see the results,

and call more tools until it produces a text answer or hits a step limit.

## Requirements

* Loop: call `llm_client.infer(messages, tools=tools)`.
  * If the response is `text`, return it (non-empty check as today).
  * If the response is `tool_use`, validate and run each call exactly as today (unknown-tool and

    missing-required-field `ValueError`s, `id` uppercasing, 10s timeout), append the assistant
    tool-use message and the user tool-result message, and loop.
* Tools stay available on every step (no more `tools=None` synthesis call).
* Step limit: new constructor arg `max_steps: int = 5` on `Orchestrator`. One step = one LLM call

  that returns `tool_use`. When the limit is reached, make one final `infer(messages, tools=None)`

  call asking for an answer with what it has; if that is not non-empty text, raise

  `RuntimeError("agent exceeded max_steps without an answer")`.
* `run_server` reads `AGENT_MAX_STEPS` (positive int, default 5; invalid or empty → default with a

  warning log) and passes it to `Orchestrator`.
* Log each step at INFO: step number and the tool names called.
* `/query` HTTP behaviour and status codes are unchanged (`ValueError` → 400, other errors → 500).
* No changes to `mcp_client.py` or the MCP server.

## Acceptance criteria

* `tests/test_orchestrator.py`, with mocked LLM and MCP clients:
  * Direct text answer: one `infer` call, no tool calls (existing behaviour still passes).
  * Two-step chain: tool_use → tool_use → text returns the text; MCP called twice, in order;

    the second step's messages include the first tool result.
  * Step limit: with `max_steps=2` and a model that always returns tool_use, exactly 2 tool

    rounds run, then one `infer(..., tools=None)`; its text is returned.
  * Step limit with no answer: that final call returns tool_use → `RuntimeError` mentioning `max_steps`.
  * Unknown tool on step 2 still raises `ValueError`.
* `tests/test_http_server.py`: `AGENT_MAX_STEPS` parsing (valid value, invalid → default 5).
* Update or remove existing tests that assert the old "synthesis loop detected" error.
* All tests pass with `python -m pytest -q`.
* README: add `AGENT_MAX_STEPS` to the environment-variables table and one sentence describing

  multi-step tool use.

## Out of scope

* Conversation memory across `/query` requests.
* Streaming responses.
* Any MCP or GraphQL repo changes.
