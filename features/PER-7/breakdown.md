# breakdown.md

**Epic**: [agent] Agent which can directly answer or via MCP tools  
**Date**: 2025-01-21  
**Status**: Draft

---

## Overview

This breakdown slices the agent epic into implementation stories. Each story is a session-sized implementation step sequenced to reach the epic's Done state. Stories are ordered to establish infrastructure first, then core agent logic, then integration and testing.

**Build Order Rationale**:
1. Establish HTTP server skeleton and basic routing (foundation)
2. Integrate LLM client for direct answers (validates LLM connectivity)
3. Add MCP client for tool discovery and invocation (validates MCP stdio)
4. Implement orchestrator with tool-calling logic including ID normalization and argument validation (core agent behavior)
5. Add health check endpoint (robustness)
6. Verify with tests (validation)

---

## Story List

### Story 1: HTTP Server Skeleton
**ID**: `1.1`  
**Title**: HTTP server with `/query` endpoint scaffold  
**Type**: story  
**Prerequisites**: None

**Description**:
Create a Flask HTTP server exposing `POST /query` that accepts JSON body `{"query": "<text>"}` and returns `{"answer": "<placeholder>"}` with HTTP 200. Implement basic request validation (400 for malformed JSON or missing `query` field) and error handling (500 for unhandled exceptions). No LLM or MCP integration yet; orchestrator is stubbed to return a hardcoded answer.

**Covers**: AC1 (HTTP endpoint structure), partial AD-5 (error handling skeleton)

**Verification Approach**:
1. Valid POST request to `/query` → returns 200 with JSON `{"answer": "<text>"}`
2. Malformed JSON to `/query` → returns 400
3. Valid JSON without `query` field → returns 400
4. Internal error (mock exception in stub) → returns 500

**Tests**: 4 unit tests in `tests/test_http_server.py` (valid request, malformed JSON, missing field, internal error)

**Allowed Paths**:
- `app/agent/http_server.py` (new)
- `app/agent/__init__.py` (new)
- `requirements.txt` (add Flask)
- `tests/test_http_server.py` (new, 4 tests)

---

### Story 2: LLM Client for Direct Answers
**ID**: `1.2`  
**Title**: LLM client integration with direct answer path  
**Type**: story  
**Prerequisites**: `1.1`

**Description**:
Implement LLM client (`app/agent/llm_client.py`) that calls Anthropic Messages API with user query and returns parsed response. Update orchestrator to invoke LLM client for queries and return direct text answers (no tool-calling yet). Configure via `ANTHROPIC_*` env vars. Handle LLM API errors (raise exceptions caught by HTTP server → 500).

**Covers**: AC2 (direct answer without tool), partial AD-1 (LLM integration), AD-5 (LLM error handling)

**Verification Approach**:
1. Query `"hello"` → LLM returns direct answer, HTTP 200 with answer text
2. Mock LLM API error → HTTP 500
3. LLM client uses env vars `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`
4. Orchestrator direct answer path (no tool invocation for queries requiring no data)

**Tests**: 3 unit tests in `tests/test_llm_client.py` (direct answer parsing, API error propagation, env var validation) + 1 orchestrator test in `tests/test_orchestrator.py` (test 1: direct answer path)

**Allowed Paths**:
- `app/agent/llm_client.py` (new)
- `app/agent/orchestrator.py` (new, stub from Story 1 replaced)
- `requirements.txt` (add `anthropic` SDK)
- `tests/test_llm_client.py` (new, 3 tests)
- `tests/test_orchestrator.py` (new, 1 test: direct answer path)

---

### Story 3: MCP Client and Tool Discovery
**ID**: `1.3`  
**Title**: MCP stdio client with tool list fetching  
**Type**: story  
**Prerequisites**: `1.2`

**Description**:
Implement MCP client (`app/agent/mcp_client.py`) that spawns `python -m mcp.server` subprocess on agent startup, performs MCP JSON-RPC handshake over stdin/stdout, and fetches available tool list. Subprocess configured with `GRAPHQL_ENDPOINT` in environment. Expose tool schemas for orchestrator. Implement graceful shutdown (terminate subprocess). Implement tool invocation method with 10s timeout. Verify all four tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) are discovered.

**Covers**: AC4 (all four MCP tools available), AD-2 (MCP via stdio subprocess), partial AD-5 (MCP startup error handling and timeout)

**Verification Approach**:
1. MCP client starts subprocess → handshake succeeds → returns tool list with all four tools
2. Mock MCP server crash during handshake → raises exception (logged, agent fails to start)
3. Graceful shutdown → subprocess terminates cleanly
4. Mock MCP request timeout (>10s) → raises `TimeoutError`

**Tests**: 4 unit tests in `tests/test_mcp_client.py` (successful handshake with tool discovery, handshake failure, graceful shutdown, tool call timeout)

**Allowed Paths**:
- `app/agent/mcp_client.py` (new)
- `tests/test_mcp_client.py` (new, 4 tests)

---

### Story 4: Orchestrator with Tool Calling
**ID**: `1.4`  
**Title**: Orchestrator tool-calling logic and multi-step inference  
**Type**: story  
**Prerequisites**: `1.3`

**Description**:
Update orchestrator to send user query + MCP tool schemas to LLM client. Parse LLM response: if direct text, return as-is; if tool calls, validate arguments against MCP tool schemas (raise `ValueError` on missing required fields or unknown tool names), normalize IDs (`.upper()` on all `id` fields), invoke MCP client for each tool, collect results, re-prompt LLM with tool results to synthesize final answer. Limit to 2 LLM calls (initial + synthesis) per AD-4. Detect synthesis loop (LLM requests tools on second call) and raise `RuntimeError`. Handle MCP tool errors (propagate exceptions → 500).

**Covers**: AC3 (tool call for employee lookup), AC5 (ID normalization), AD-1 (LLM tool calling), AD-3 (ID normalization), AD-4 (multi-step inference), AD-5 (argument validation, synthesis loop detection, tool error handling)

**Verification Approach**:
1. Query `"what is the name of employee with emp-001"` (mocked LLM returns tool call with `{"id": "emp-001"}`, orchestrator normalizes to `{"id": "EMP-001"}`, mocked MCP returns `{"name": "Alice"}`, mocked LLM synthesis returns `"The name is Alice."`) → HTTP 200 with synthesized answer (test 2: tool-call success with ID normalization and synthesis)
2. Mock MCP tool error → HTTP 500 with logged trace (test 3: MCP error propagation)
3. Verify exactly 2 LLM calls for tool-call path (initial + synthesis), 1 call for direct answer path (test 4: LLM call count verification)
4. Verify ID normalization: input `"emp-001"` → MCP receives `"EMP-001"` (test 5: ID normalization verification)
5. Mock LLM returns tool call with missing required field → raises `ValueError` (caught by HTTP server → 400) (test 6: argument validation missing field)
6. Mock LLM returns unknown tool name → raises `ValueError` (caught by HTTP server → 400) (test 7: argument validation unknown tool)
7. Mock LLM requests tools on synthesis step → raises `RuntimeError` (caught by HTTP server → 500) (test 8: synthesis loop detection)

**Tests**: 7 new orchestrator tests in `tests/test_orchestrator.py` (tests 2–8 as listed above), for a total of 8 orchestrator tests including Story 1.2's test 1 (direct answer path)

**Allowed Paths**:
- `app/agent/orchestrator.py` (update)
- `app/agent/mcp_client.py` (use existing `call_tool` method)
- `tests/test_orchestrator.py` (update, +7 tests for total of 8)

---

### Story 5: Health Check Endpoint
**ID**: `1.5`  
**Title**: Health check endpoint for MCP subprocess monitoring  
**Type**: story  
**Prerequisites**: `1.4`

**Description**:
Add health check endpoint `GET /health` to HTTP server that validates MCP subprocess is alive via MCP client's `is_alive()` method. Return 200 with `{"status": "ok", "mcp_alive": true}` when subprocess is running, or 503 with `{"status": "degraded", "mcp_alive": false}` when crashed.

**Covers**: §7 risk mitigation (health check)

**Verification Approach**:
1. `GET /health` with MCP subprocess alive → 200 with `{"status": "ok", "mcp_alive": true}`
2. `GET /health` with subprocess crashed → 503 with `{"status": "degraded", "mcp_alive": false}`

**Tests**: 2 unit tests in `tests/test_http_server.py` (health check alive, health check crashed), for a total of 6 HTTP server tests

**Allowed Paths**:
- `app/agent/http_server.py` (add `/health` endpoint)
- `tests/test_http_server.py` (update, +2 tests for total of 6)

---

### Story 6: Integration Tests and Manual Verification
**ID**: `1.6`  
**Title**: Tests and manual verification script  
**Type**: story  
**Prerequisites**: `1.5`

**Description**:
Add integration test that starts agent with mocked LLM/MCP services and verifies end-to-end flow (query with tool call → synthesized answer, verifying HTTP 200, synthesized answer content, 2 LLM calls, 1 MCP call with normalized ID). Create manual verification script (`tests/manual_verification.md`) documenting steps to test against real PER-5 GraphQL service and PER-6 MCP server: setup instructions, test queries (`"hello"`, `"name of emp-001"`, `"list employees"`, `"get sales order ord-001"`, `GET /health`), expected outputs, troubleshooting steps, and timing expectations. Update `README.md` with quickstart, environment variables, and test instructions.

**Covers**: AC6 (basic tests), §8 testing strategy (integration + manual paths)

**Verification Approach**:
1. Integration test with mocked services passes (covers AC1–AC5 verification: HTTP 200 response, synthesized answer, 2 LLM calls on tool path, 1 MCP call with normalized ID `EMP-001`)
2. Manual script executed against real services returns expected outputs for all 5 test queries (hello, employee lookup with ID normalization, list employees, get sales order, health check)
3. Documentation includes env setup, commands, expected JSON responses, troubleshooting, and timing expectations

**Tests**: 1 integration test in `tests/test_integration.py` + manual verification script in `tests/manual_verification.md`

**Allowed Paths**:
- `tests/test_integration.py` (new, 1 test)
- `tests/manual_verification.md` (new)
- `README.md` (update with quickstart and test instructions)

---

## Dependencies

- **External**: PER-5 GraphQL service must be reachable at `GRAPHQL_ENDPOINT` (provided by user)
- **External**: PER-6 MCP server runnable via `python -m mcp.server` (PER-6 repo venv, provided by user)
- **Between Stories**: Linear sequence (each story depends on prior story)

---

## Repo Assignment

All stories target the existing `app` Python codebase under `/home/goos/projects/adlc-agents/app/agent/`. New files are created as specified in Allowed Paths. Existing `README.md` is updated in Story 6.

---

## Acceptance Criteria Summary

| AC | Covered in Stories |
|----|-------------------|
| AC1: HTTP endpoint returns 200 + JSON | 1.1 (scaffold), 1.2–1.6 (full impl) |
| AC2: Direct answer without tool | 1.2 (impl), 1.4 (regression check via test 1) |
| AC3: Tool call for employee lookup | 1.4 (impl with tool calling and ID normalization, test 2) |
| AC4: All four MCP tools available | 1.3 (tool discovery and invocation) |
| AC5: ID normalization | 1.4 (impl, test 5) |
| AC6: Basic tests | 1.1–1.5 (21 unit tests), 1.6 (1 integration test + manual script) |

---

## Story Verification Test Count

| Story | Test Count | Type | Test File(s) |
|-------|-----------|------|-------------|
| 1.1 | 4 | Unit (HTTP responses) | `tests/test_http_server.py` (valid request, malformed JSON, missing field, internal error) |
| 1.2 | 3 + 1 | Unit (LLM client + orchestrator) | `tests/test_llm_client.py` (3: direct answer parsing, API error, env validation), `tests/test_orchestrator.py` (1: test 1 direct answer path) |
| 1.3 | 4 | Unit (MCP client) | `tests/test_mcp_client.py` (4: successful handshake with tool discovery, handshake failure, graceful shutdown, tool call timeout) |
| 1.4 | 7 | Unit (orchestrator) | `tests/test_orchestrator.py` (+7 for total of 8: test 2 tool-call success with ID normalization and synthesis, test 3 MCP error propagation, test 4 LLM call count verification, test 5 ID normalization verification, test 6 argument validation missing field, test 7 argument validation unknown tool, test 8 synthesis loop detection) |
| 1.5 | 2 | Unit (health check) | `tests/test_http_server.py` (+2 for total of 6: health check alive, health check crashed) |
| 1.6 | 1 + manual | Integration + manual script | `tests/test_integration.py` (1: end-to-end with mocked services), `tests/manual_verification.md` (manual verification script) |
| **Total** | **21 unit tests + 1 integration test + 1 manual script** | | **Unit test breakdown**: `test_http_server.py` (6), `test_llm_client.py` (3), `test_mcp_client.py` (4), `test_orchestrator.py` (8). **Total**: 6 + 3 + 4 + 8 = 21 unit tests, matching requirements.md AC6. |

**Alignment Note**: This breakdown delivers exactly 21 unit tests (6 HTTP server + 3 LLM client + 4 MCP client + 8 orchestrator) as specified in requirements.md AC6, plus 1 integration test and 1 manual verification script. Story 1.2 delivers orchestrator test 1 (direct answer path). Story 1.3 implements the MCP client's `call_tool` method with 10s timeout (test 4 in `tests/test_mcp_client.py`). Story 1.4 implements all orchestrator tool-calling logic and delivers tests 2–8 in `tests/test_orchestrator.py`: test 2 (tool-call success with ID normalization and synthesis), test 3 (MCP error propagation), test 4 (LLM call count verification: 2 on tool path, 1 on direct path), test 5 (ID normalization verification: `emp-001` → `EMP-001`), test 6 (argument validation missing field raises `ValueError`), test 7 (argument validation unknown tool raises `ValueError`), test 8 (synthesis loop detection raises `RuntimeError`). Story 1.5 adds 2 health check tests to `test_http_server.py` (total 6). All test counts are concrete, explicitly numbered, and aligned with change_requests.md exact test ownership table. No placeholders or deferrals remain.

---

**End of Breakdown**
