# Breakdown: PER-19 · Serve MCP GraphQL tools over FastAPI + FastMCP HTTP (replace stdio)

**Repo:** `mcp` (adlc-mcp-server)  
**Epic:** PER-19  
**Root:** `/home/goos/projects/adlc-mcp-server/`

---

## Overview

This epic migrates the MCP server from stdio transport to FastAPI + FastMCP HTTP. The four existing GraphQL tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) remain unchanged in behavior and JSON shape; only the transport layer is replaced. Health/readiness endpoints are added, and the README is updated to document HTTP as the primary setup path.

**Execution order:** Stories are sequenced to deliver incremental value and validate the transport change early. Tests are written alongside each component to prevent regressions.

---

## Child Stories

### Story 1: Set up FastAPI app with health endpoint and FastMCP SSE stub

**ID:** 1  
**Repo:** `mcp`  
**Type:** Story  
**Description:**  
Create a minimal FastAPI app with a `/health` endpoint and a stubbed `/sse` FastMCP endpoint. The health endpoint returns `{"status": "ok"}` when the process is up. The `/sse` endpoint initializes FastMCP SSE but returns no tools yet. Validate the app starts via Uvicorn and the health endpoint is reachable.

**Covers:** AC3 (health check), partial AC4 (HTTP setup), partial AC1 (FastMCP SSE endpoint exists).

**Dependencies:** None (opening story).

**Verification Approach:**  
- **Test 1 (from requirements Test 1):** Server starts and `/health` returns 200 + `{"status": "ok"}` — verifies AC3.
- **Test 2 (from requirements Test 2):** `/sse` FastMCP SSE handshake succeeds — verifies partial AC1.
- **Test 3 (from requirements Test 3):** Server fails to start if `GRAPHQL_ENDPOINT` is missing — verifies Exception 1 (startup validation).

**Allowed Paths:**  
- `/home/goos/projects/adlc-mcp-server/server.py` (new) — FastAPI app with `/health` and `/sse`.
- `/home/goos/projects/adlc-mcp-server/main.py` (new or update) — Uvicorn entrypoint.
- `/home/goos/projects/adlc-mcp-server/tests/test_server.py` (new) — Integration tests for `/health` and `/sse` stub (Tests 1–3).
- `/home/goos/projects/adlc-mcp-server/requirements.txt` or `pyproject.toml` (update) — Add `fastapi`, `uvicorn[standard]`, `fastmcp`, `pytest`, `pytest-asyncio`, `httpx`.

**Acceptance Criteria:**  
1. **AC1 (partial - FastMCP endpoint exists):** Given a client connects to `/sse`, then the FastMCP SSE handshake succeeds (no tools invoked yet).
2. **AC2 (not covered in this story):** N/A.
3. **AC3 (health check):** Given the server process is started, when an operator checks `/health`, then they receive 200 + `{"status": "ok"}` without calling a GraphQL tool.
4. **AC4 (partial - HTTP setup):** Given a developer runs `uvicorn server:app`, then the server starts and listens on the configured host/port.
5. **AC5 (not covered in this story):** N/A.
6. **Exception 1 (partial - startup validation):** Given `GRAPHQL_ENDPOINT` is missing, when the server starts, then it logs an error and exits.
7. **Exception 2 (not covered in this story):** N/A.
8. **Exception 3 (not covered in this story):** N/A.

**Notes:**  
- Do not implement GraphQL client or tools yet; focus on the FastAPI skeleton.
- `GRAPHQL_ENDPOINT` is read from environment but not validated (validation added in Story 2).
- README update deferred to Story 5.

---

### Story 2: Implement GraphQL client with error handling

**ID:** 2  
**Repo:** `mcp`  
**Type:** Story  
**Description:**  
Create a shared GraphQL client that reads `GRAPHQL_ENDPOINT` from environment, executes the same queries as the stdio version, and raises exceptions on network/GraphQL errors. Validate the client at startup: fail fast if `GRAPHQL_ENDPOINT` is missing or malformed.

**Covers:** Partial AC2 (error shape), Exception 1 (GraphQL errors), partial risk mitigation (config validation).

**Dependencies:**  
- `after = [1]` (Story 1: FastAPI app exists).

**Verification Approach:**  
- **Test 4 (from requirements Test 4):** GraphQL client executes query successfully — verifies partial AC2.
- **Test 5 (from requirements Test 5):** GraphQL client raises exception on network error — verifies Exception 1.
- **Test 6 (from requirements Test 6):** GraphQL client raises exception on GraphQL error response — verifies Exception 1.
- **Test 7 (from requirements Test 7):** GraphQL client raises exception on HTTP error — verifies Exception 1.

**Allowed Paths:**  
- `/home/goos/projects/adlc-mcp-server/graphql_client.py` (new) — GraphQL query executor.
- `/home/goos/projects/adlc-mcp-server/server.py` (update) — Validate `GRAPHQL_ENDPOINT` at startup.
- `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py` (new) — Unit tests for GraphQL client (Tests 4–7).

**Acceptance Criteria:**  
1. **AC1 (not covered in this story):** N/A.
2. **AC2 (partial - error handling):** Given GraphQL returns an error or is unreachable, when the client is called, then it raises an exception with a descriptive message.
3. **AC3 (not covered in this story):** N/A.
4. **AC4 (not covered in this story):** N/A.
5. **AC5 (not covered in this story):** N/A.
6. **Exception 1 (GraphQL down):** Given GraphQL is down or returns errors, when a query is executed, then the client raises an exception (not a JSON response; tool layer handles formatting).
7. **Exception 2 (not covered in this story):** N/A.
8. **Exception 3 (not covered in this story):** N/A.

**Notes:**  
- Use `httpx.AsyncClient` for async GraphQL queries (FastAPI is async).
- GraphQL query syntax must match the stdio version (review existing code or logs).
- Error messages should be clear: "GraphQL endpoint unreachable", "GraphQL returned error: <message>".

---

### Story 3: Implement the four MCP tools over FastMCP

**ID:** 3  
**Repo:** `mcp`  
**Type:** Story  
**Description:**  
Define the four tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) using FastMCP decorators. Each tool wraps a GraphQL client call, validates inputs, and returns success payloads or `{"error": "..."}` on failure. Wire the tools into the FastMCP SSE endpoint in `server.py`.

**Covers:** AC1 (tool availability), AC2 (response shapes), Exception 1 (error JSON shape), partial AC5 (tool tests).

**Dependencies:**  
- `after = [2]` (Story 2: GraphQL client exists).

**Verification Approach:**  
- **Test 8 (from requirements Test 8):** `list_sales_order` returns success payload — verifies AC1, AC2.
- **Test 9 (from requirements Test 9):** `list_employees` returns success payload — verifies AC1, AC2.
- **Test 10 (from requirements Test 10):** `get_sales_order_by_id` returns success payload — verifies AC1, AC2.
- **Test 11 (from requirements Test 11):** `get_employee_by_id` returns success payload — verifies AC1, AC2.
- **Test 12 (from requirements Test 12):** `get_sales_order_by_id` returns error for missing `id` — verifies AC2 (error shape).
- **Test 13 (from requirements Test 13):** Tool returns error when GraphQL client raises exception — verifies Exception 1.

**Allowed Paths:**  
- `/home/goos/projects/adlc-mcp-server/tools.py` (new) — MCP tool definitions.
- `/home/goos/projects/adlc-mcp-server/server.py` (update) — Wire tools into FastMCP SSE endpoint.
- `/home/goos/projects/adlc-mcp-server/tests/test_tools.py` (new) — Unit/integration tests for each tool (Tests 8–13).

**Acceptance Criteria:**  
1. **AC1 (tool availability over HTTP):** Given the MCP server is running, when a client connects via FastMCP HTTP, then all four tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) are listed and callable.
2. **AC2 (tool response shape preservation):** Given a valid `GRAPHQL_ENDPOINT`, when a tool is invoked over HTTP, then results use the same success/error JSON shapes as the stdio version (list/get payloads or `{"error": "..."}`).
3. **AC3 (not covered in this story):** N/A.
4. **AC4 (not covered in this story):** N/A.
5. **AC5 (partial - tool tests):** Given automated tests for tools, when tests run, then tool behavior is covered.
6. **Exception 1 (error shape):** Given GraphQL is down or returns errors, when a tool is invoked, then it returns `{"error": "..."}` and the server process stays up.
7. **Exception 2 (not covered in this story):** N/A.
8. **Exception 3 (not covered in this story):** N/A.

**Notes:**  
- Tool names must match stdio version exactly: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`.
- Input validation: `get_*_by_id` tools require non-empty `id`.
- Use FastMCP test client or `httpx` to invoke tools in tests (no real GraphQL calls).

---

### Story 4: Add end-to-end integration tests

**ID:** 4  
**Repo:** `mcp`  
**Type:** Story  
**Description:**  
Write integration tests that start the FastAPI server, connect via FastMCP SSE, invoke all four tools with valid/invalid inputs, and verify the JSON responses match the stdio version. Mock the GraphQL endpoint to control responses. Validate health endpoint in the same test suite.

**Covers:** AC5 (test coverage), partial AC1 (tools work over HTTP), partial AC2 (payloads match).

**Dependencies:**  
- `after = [3]` (Story 3: tools implemented).

**Verification Approach:**  
- **Test 14 (from requirements Test 14):** Integration test for `list_sales_order` over HTTP — verifies AC1, AC2, AC5.
- **Test 15 (from requirements Test 15):** Integration test for `list_employees` over HTTP — verifies AC1, AC2, AC5.
- **Test 16 (from requirements Test 16):** Integration test for `get_sales_order_by_id` over HTTP — verifies AC1, AC2, AC5.
- **Test 17 (from requirements Test 17):** Integration test for `get_employee_by_id` over HTTP — verifies AC1, AC2, AC5.
- **Test 18 (from requirements Test 18):** Integration test for GraphQL error handling — verifies Exception 1.
- **Test 19 (from requirements Test 19):** Integration test for health endpoint — verifies AC3.
- **Test 20 (from requirements Test 20):** Integration test for invalid tool input — verifies AC2 (error shape).

**Allowed Paths:**  
- `/home/goos/projects/adlc-mcp-server/tests/test_integration.py` (new) — End-to-end tests (Tests 14–20).

**Acceptance Criteria:**  
1. **AC1 (partial - HTTP invocation):** Given integration tests, when tools are invoked over HTTP, then they return expected results.
2. **AC2 (partial - payload validation):** Given integration tests, when tool responses are compared to stdio baseline, then no divergence is found.
3. **AC3 (health check validation):** Given the integration test suite, when `/health` is tested, then it returns 200 + `{"status": "ok"}`.
4. **AC4 (not covered in this story):** N/A.
5. **AC5 (test coverage maintained):** Given automated tests for tools/client exist, when CI or local tests run, then tool behavior is covered via HTTP or in-process FastMCP test client.
6. **Exception 1 (error JSON validation):** Given integration tests, when GraphQL errors occur, then tools return `{"error": "..."}`.
7. **Exception 2 (not covered in this story):** N/A.
8. **Exception 3 (not covered in this story):** N/A.

**Notes:**  
- Use pytest fixtures to start/stop the server in test mode.
- Mock GraphQL responses with `httpx-mock` or similar.
- If stdio baseline is unavailable, document expected payloads in test comments.

---

### Story 5: Update README with HTTP setup and migration guide

**ID:** 5  
**Repo:** `mcp`  
**Type:** Story  
**Description:**  
Update the README to document HTTP as the primary setup path. Include Uvicorn command, environment variables, MCP URL format, and a migration section for existing stdio clients. Note authentication as a follow-up.

**Covers:** AC4 (HTTP setup documentation), Exception 2 (stdio migration), Exception 3 (auth deferred).

**Dependencies:**  
- `after = [3]` (Story 3: tools implemented; server is usable).

**Verification Approach:**  
- **Manual verification:** Follow README setup steps on a clean machine/container; server starts and tools are callable.
- **Checklist:** README includes:
  1. Environment variables (`GRAPHQL_ENDPOINT`, `PORT`, `HOST`).
  2. Uvicorn command (`uvicorn server:app --host 0.0.0.0 --port 8000`).
  3. MCP URL format (`http://localhost:8000/sse`).
  4. Migration section with before/after stdio vs. HTTP client config examples.
  5. Authentication noted as follow-up.

**Allowed Paths:**  
- `/home/goos/projects/adlc-mcp-server/README.md` (update) — HTTP setup, environment variables, migration notes.

**Acceptance Criteria:**  
1. **AC1 (not covered in this story):** N/A.
2. **AC2 (not covered in this story):** N/A.
3. **AC3 (not covered in this story):** N/A.
4. **AC4 (HTTP setup documentation):** Given the `adlc-mcp-server` README, when a developer follows setup instructions, then they can run the server with FastAPI + FastMCP over HTTP and point a client at that URL (HTTP is the documented primary path).
5. **AC5 (not covered in this story):** N/A.
6. **Exception 1 (not covered in this story):** N/A.
7. **Exception 2 (stdio migration):** Given a client is still configured for stdio, when a developer consults the README, then they find migration guidance (prefer HTTP-only once callers can switch).
8. **Exception 3 (auth deferred):** Given v1 scope, when auth is considered, then it is documented as a follow-up unless org policy already requires it.

**Notes:**  
- "Setup" section should be at the top of README (quick start).
- Migration section: show stdio config (if applicable) vs. HTTP config side-by-side.
- Link to FastMCP documentation for client-side setup.

---

## Execution Order Summary

```
Story 1 (FastAPI skeleton + health)
  ↓
Story 2 (GraphQL client + validation)
  ↓
Story 3 (Four MCP tools)
  ↓
Story 4 (Integration tests)
  ↓
Story 5 (README update)
```

**Opening story:** Story 1 (no prerequisites).  
**Closing story:** Story 5 (all ACs verified, documentation complete).

---

## Acceptance Criteria Traceability

| AC / Exception | Covered in Story |
|---|---|
| **AC1:** Tools available over HTTP | Story 1 (partial - SSE endpoint), Story 3 (full implementation), Story 4 (integration tests) |
| **AC2:** Response shapes preserved | Story 2 (partial - error handling), Story 3 (tool logic), Story 4 (validation) |
| **AC3:** Health check | Story 1 (implementation + test), Story 4 (integration test) |
| **AC4:** HTTP setup documentation | Story 5 (README update) |
| **AC5:** Test coverage maintained | Story 3 (unit tests), Story 4 (integration tests) |
| **Exception 1:** GraphQL errors → JSON error shape | Story 1 (startup validation), Story 2 (GraphQL client), Story 3 (tool error handling), Story 4 (integration test) |
| **Exception 2:** stdio migration guidance | Story 5 (README migration section) |
| **Exception 3:** Auth deferred | Story 5 (README note) |

---

## Dependency Graph

```
1 (FastAPI skeleton)
  ↓
2 (GraphQL client) ← depends on 1
  ↓
3 (Tools) ← depends on 2
  ↓
4 (Integration tests) ← depends on 3
  ↓
5 (README) ← depends on 3 (server is usable)
```

---

## Notes

- **Story sizing:** Each story is session-sized for a build agent (1–2 hours estimated).
- **Test count:** Total 20 automated tests across Stories 1–4 (Story 1: Tests 1–3, Story 2: Tests 4–7, Story 3: Tests 8–13, Story 4: Tests 14–20); manual verification in Story 5.
- **Concrete paths:** All file paths use `/home/goos/projects/adlc-mcp-server/` per repo context.
- **No new ACs:** All acceptance criteria are lifted verbatim from requirements.md; no edge cases added.
- **GraphQL query preservation:** Queries must match stdio version; this is verified in Story 2 (client) and Story 3 (tools).
- **Test coverage alignment:** Each test number (1–20) corresponds exactly to the test in requirements.md Tests section, preserving test IDs and verification mappings.

---

**End of Breakdown**
