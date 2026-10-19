# High-Level Design: Serve MCP GraphQL tools over FastAPI + FastMCP HTTP (replace stdio)

**Repo:** `mcp` (adlc-mcp-server)  
**Epic/Ticket:** PER-19  
**Altitude:** Story  

---

## 1. Overview

**Purpose:**  
Replace the stdio-based MCP server transport with FastAPI + FastMCP HTTP, so that the four existing GraphQL tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) can be invoked over HTTP by multiple clients without spawning per-client stdio processes.

**Scope:**  
- Transport layer change (stdio → FastAPI + FastMCP HTTP).
- Keep all four tool names, GraphQL queries, and JSON response shapes unchanged.
- Add a health/readiness endpoint.
- Update README to document HTTP as the primary setup path.
- Maintain test coverage under the new transport.

**Non-goals (this story):**  
- Authentication/authorization (document as follow-up unless org policy requires it now).
- GraphQL schema changes.
- New tools or domain logic.

---

## 2. Design Paradigm

**Layered MCP-over-HTTP**  
- **Transport:** FastAPI + FastMCP SSE endpoint.
- **Tool layer:** Existing MCP tool definitions (no change to signatures or behavior).
- **Data layer:** GraphQL client (same queries as stdio version).

**Key invariants:**  
- Tool names and JSON schemas are **read-only** (no divergence from stdio payloads).
- GraphQL endpoint URL remains configurable via `GRAPHQL_ENDPOINT` (no hardcoding).
- Health check is **non-invasive** (does not require GraphQL to be up).

---

## 3. Components and Responsibilities

### 3.1 FastAPI Application (`main.py` or `server.py`)

**Responsibilities:**  
- Serve FastMCP SSE endpoint at `/sse` (or configured path).
- Expose health/readiness endpoint at `/health`.
- Read `GRAPHQL_ENDPOINT` from environment at startup.

**Key behaviors:**  
- Health endpoint returns `{"status": "ok"}` (200) when process is up; does not validate GraphQL connectivity.
- FastMCP SSE endpoint delegates tool calls to the MCP tool layer.

**Concrete files (new or updated):**  
- `/home/goos/projects/adlc-mcp-server/server.py` (new) — FastAPI app with `/sse` and `/health`.
- `/home/goos/projects/adlc-mcp-server/main.py` (update if exists, or new entrypoint) — runs Uvicorn server.

---

### 3.2 MCP Tool Definitions

**Responsibilities:**  
- Expose the four tools via FastMCP's `@mcp.tool()` decorator or equivalent.
- Delegate GraphQL queries to the GraphQL client component.
- Return success payloads as-is, or `{"error": "..."}` on failure.

**Key behaviors:**  
- Tools:
  - `list_sales_order` → returns list of sales order objects.
  - `list_employees` → returns list of employee objects.
  - `get_sales_order_by_id(id)` → returns single sales order or error.
  - `get_employee_by_id(id)` → returns single employee or error.
- Error shape: `{"error": "<message>"}` when GraphQL call fails or returns error.

**Concrete files (updated):**  
- `/home/goos/projects/adlc-mcp-server/tools.py` (update if exists, or new) — tool definitions wrapping GraphQL calls.

---

### 3.3 GraphQL Client

**Responsibilities:**  
- Execute the same GraphQL queries as the stdio version.
- Read `GRAPHQL_ENDPOINT` from environment.
- Return structured JSON payloads or raise exceptions on error.

**Key behaviors:**  
- GraphQL down or error → raise exception, caught by tool layer and returned as `{"error": "..."}`.
- No change to query syntax or variables from stdio version.

**Concrete files (updated):**  
- `/home/goos/projects/adlc-mcp-server/graphql_client.py` (update if exists, or new) — shared GraphQL query executor.

---

### 3.4 Test Suite

**Responsibilities:**  
- Cover all four tools via HTTP client or in-process FastMCP test client.
- Verify health endpoint.
- Mock GraphQL responses or use a test GraphQL endpoint.

**Key behaviors:**  
- Tools return expected shapes (success and error cases).
- Health endpoint returns 200 + `{"status": "ok"}`.
- No regression in tool names or payloads.

**Concrete files (updated):**  
- `/home/goos/projects/adlc-mcp-server/tests/test_server.py` (new) — server and health endpoint tests.
- `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py` (new) — GraphQL client unit tests.
- `/home/goos/projects/adlc-mcp-server/tests/test_tools.py` (new) — tool behavior unit tests.
- `/home/goos/projects/adlc-mcp-server/tests/test_integration.py` (new) — end-to-end integration tests.

---

### 3.5 README and Documentation

**Responsibilities:**  
- Document HTTP setup as the primary path (Uvicorn command, environment variables).
- Document migration from stdio to HTTP for existing clients.
- Note authentication as a follow-up (unless org policy requires it now).

**Key behaviors:**  
- Developer can copy-paste setup steps to run the server locally.
- README lists the MCP URL format (e.g., `http://localhost:8000/sse`).

**Concrete files (updated):**  
- `/home/goos/projects/adlc-mcp-server/README.md` (update) — HTTP setup, environment variables, tool list, migration notes.

---

## 4. Data Contracts and APIs

### 4.1 FastAPI Endpoints

**`GET /health`**  
- **Success (200):**
  ```json
  {"status": "ok"}
  ```
- **No failure case** (if process is down, no response).

**`/sse` (FastMCP SSE endpoint)**  
- **Protocol:** Server-Sent Events, FastMCP handshake.
- **Tool invocations:** JSON-RPC or FastMCP protocol messages.
- **Tool responses:** Same JSON shapes as stdio version (see 4.2).

---

### 4.2 Tool Contracts (unchanged from stdio)

**`list_sales_order`**  
- **Input:** none.
- **Success output:** Array of sales order objects (shape per existing GraphQL query).
- **Error output:** `{"error": "..."}`

**`list_employees`**  
- **Input:** none.
- **Success output:** Array of employee objects (shape per existing GraphQL query).
- **Error output:** `{"error": "..."}`

**`get_sales_order_by_id`**  
- **Input:** `{"id": "<string>"}`.
- **Success output:** Single sales order object (shape per existing GraphQL query).
- **Error output:** `{"error": "..."}`

**`get_employee_by_id`**  
- **Input:** `{"id": "<string>"}`.
- **Success output:** Single employee object (shape per existing GraphQL query).
- **Error output:** `{"error": "..."}`

---

### 4.3 Environment Variables

- `GRAPHQL_ENDPOINT` (required): URL of the GraphQL service (e.g., `https://example.com/graphql`).
- `PORT` (optional): HTTP server port (default `8000`).
- `HOST` (optional): HTTP server host (default `0.0.0.0`).

---

## 5. Architecture Decisions

**AD-1: Use FastAPI + FastMCP for HTTP transport**  
**Binds:** Web framework is FastAPI; MCP-over-HTTP is FastMCP SSE.  
**Prevents:** Divergence in transport layer (no custom SSE or WebSocket implementation).  
**Rule:** All MCP tool calls over HTTP go through the FastMCP SSE endpoint; no parallel REST API for tools.  
**Status:** [ADOPTED] (task specifies FastAPI + FastMCP).

---

**AD-2: Preserve tool names and JSON shapes**  
**Binds:** The four tool names (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) and their success/error payloads are read-only.  
**Prevents:** Breaking changes for existing clients during migration from stdio to HTTP.  
**Rule:** Any change to tool signatures or JSON structure is a breaking change requiring a new AC.  
**Status:** [ADOPTED] (AC2 requires same shapes; AC5 requires no regression).

---

**AD-3: Health endpoint is non-invasive**  
**Binds:** `/health` returns 200 + `{"status": "ok"}` when the process is up; does not check GraphQL connectivity.  
**Prevents:** Health check failures due to transient GraphQL issues.  
**Rule:** Health endpoint logic must not invoke GraphQL or any downstream service.  
**Status:** [ADOPTED] (AC3 requires health check without calling a GraphQL tool).

---

**AD-4: GraphQL errors are returned as `{"error": "..."}`**  
**Binds:** When a GraphQL call fails (network, GraphQL error response, timeout), tools return `{"error": "<message>"}`.  
**Prevents:** Uncaught exceptions bubbling to the client or process crash.  
**Rule:** Tool layer must catch all GraphQL client exceptions and map them to the error JSON shape.  
**Status:** [ADOPTED] (AC2, Exception 1).

---

**AD-5: README documents HTTP as primary path**  
**Binds:** Setup instructions lead with HTTP (Uvicorn command); stdio is legacy/deprecated.  
**Prevents:** Ambiguity about the recommended deployment mode.  
**Rule:** README must include a one-command HTTP server start and the MCP URL format.  
**Status:** [ADOPTED] (AC4).

---

**AD-6: Authentication is deferred**  
**Binds:** No authentication/authorization logic in v1 unless org policy requires it.  
**Prevents:** Scope creep.  
**Rule:** Document authentication as a follow-up; if org policy requires it, add before merge.  
**Status:** [ADOPTED] (Exception 3).

---

## 6. Edge Cases and Error Handling

### 6.1 GraphQL service down or errors (Exception 1)
- **Scenario:** `GRAPHQL_ENDPOINT` is unreachable or returns GraphQL errors.
- **Behavior:** Tools return `{"error": "..."}` with a descriptive message; server process stays up.
- **Implementation:** GraphQL client raises an exception; tool layer catches and formats as error JSON.

### 6.2 Missing `GRAPHQL_ENDPOINT` environment variable
- **Scenario:** Server starts without `GRAPHQL_ENDPOINT` set.
- **Behavior:** Server fails to start (FastAPI startup validation raises an error).
- **Implementation:** Check `GRAPHQL_ENDPOINT` at startup; log clear error and exit if missing.

### 6.3 Legacy stdio client (Exception 2)
- **Scenario:** A client is still configured to run the MCP server via stdio.
- **Behavior:** README documents migration to HTTP; stdio support may be removed in a future version.
- **Implementation:** README includes a "Migration" section with before/after examples.

### 6.4 Invalid tool arguments
- **Scenario:** `get_sales_order_by_id` called without `id` or with malformed `id`.
- **Behavior:** Return `{"error": "Invalid or missing id"}` (same as stdio version).
- **Implementation:** Tool layer validates input before calling GraphQL.

---

## 7. Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| **FastMCP SSE protocol drift** | Clients cannot connect or invoke tools | Low | Pin FastMCP version; test with known clients before release. |
| **GraphQL endpoint config mismatch** | Tools fail with cryptic errors | Medium | Validate `GRAPHQL_ENDPOINT` at startup; log clear error if missing or unreachable. |
| **Test coverage gap during transport change** | Regressions in tool behavior | Medium | Run existing stdio tests in parallel during transition; migrate to HTTP client tests incrementally. |
| **Unclear migration path for stdio clients** | Clients continue using stdio, or fail to switch | Medium | README includes side-by-side stdio vs. HTTP examples; document the MCP URL format. |
| **Health check confusion** | Operators assume health=ok means GraphQL is up | Low | Document that `/health` only checks process liveness; add a separate `/readiness` endpoint if GraphQL liveness is needed. |

---

## 8. Testing Strategy

### 8.1 Unit Tests
- **GraphQL client:** Mock GraphQL responses (success, error, timeout).
- **Tool layer:** Call each tool with valid/invalid inputs; assert correct JSON shapes.

### 8.2 Integration Tests
- **FastAPI app:** Start server, call `/health`, assert 200 + `{"status": "ok"}`.
- **Tools over HTTP:** Use FastMCP test client or HTTP client to invoke each tool; assert payloads match stdio version.
- **GraphQL error handling:** Mock GraphQL error responses; assert tools return `{"error": "..."}`.

### 8.3 End-to-End Tests
- Run server against a test GraphQL endpoint; invoke tools; assert results.

### 8.4 Coverage Targets
- **AC1 (Tool availability over HTTP):** Integration test per tool (4 tools × HTTP invocation = 4 tests in test_integration.py: Tests 14–17).
- **AC2 (Tool response shape preservation):** Unit tests for tool success/error shapes (6 tests in test_tools.py: Tests 8–13) + integration tests comparing HTTP responses to stdio baseline (4 tests in test_integration.py: Tests 14–17, plus Test 20 for error shape).
- **AC3 (Health/readiness check):** Integration test for `/health` endpoint (Test 1 in test_server.py, Test 19 in test_integration.py).
- **AC4 (HTTP setup documentation):** Manual verification of README setup steps (documented in breakdown Story 5; not an automated test).
- **AC5 (Test coverage maintained):** All 20 automated tests (Tests 1–20 across test_server.py, test_graphql_client.py, test_tools.py, test_integration.py).
- **Exception 1 (GraphQL service unavailable):** GraphQL client error handling tests (Tests 5–7 in test_graphql_client.py) + tool error handling (Test 13 in test_tools.py, Test 18 in test_integration.py).
- **Exception 2 (Legacy stdio client):** README migration section (manual verification in Story 5).
- **Exception 3 (Authentication out of scope):** README follow-up note (manual verification in Story 5).

**Total test count:** 20 automated tests (per requirements.md Tests section and breakdown Story verification approaches).

---

## 9. Dependencies and Constraints

### 9.1 External Dependencies
- **GraphQL service:** Must remain at the same endpoint and schema as the stdio version.
- **FastMCP library:** Version must support SSE transport (verify current stable version).
- **FastAPI + Uvicorn:** Standard Python async web stack.

### 9.2 Constraints
- **No schema changes:** GraphQL queries remain unchanged (inherited from stdio version).
- **No new tools:** Only the four existing tools are in scope.
- **No authentication in v1:** Unless org policy requires it (then add before merge).

---

## 10. Open Questions

*(None at this time; all ACs and exceptions are addressed.)*

---

## 11. File Tree (Concrete Paths)

**Updated files:**
- `/home/goos/projects/adlc-mcp-server/README.md` — HTTP setup, migration notes.
- `/home/goos/projects/adlc-mcp-server/tools.py` (if exists, or new) — MCP tool definitions.
- `/home/goos/projects/adlc-mcp-server/graphql_client.py` (if exists, or new) — GraphQL query executor.

**New files:**
- `/home/goos/projects/adlc-mcp-server/server.py` — FastAPI app with `/sse` and `/health`.
- `/home/goos/projects/adlc-mcp-server/main.py` (or update if exists) — Uvicorn entrypoint.
- `/home/goos/projects/adlc-mcp-server/tests/test_server.py` — FastAPI endpoint tests (Tests 1–3).
- `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py` — GraphQL client unit tests (Tests 4–7).
- `/home/goos/projects/adlc-mcp-server/tests/test_tools.py` — Tool behavior unit tests (Tests 8–13).
- `/home/goos/projects/adlc-mcp-server/tests/test_integration.py` — End-to-end integration tests (Tests 14–20).

---

## 12. Acceptance Criteria Traceability

| AC / Exception | Design Coverage |
|---|---|
| **AC1: Tool availability over HTTP** | Section 3.2 (tools), 4.2 (tool contracts), 8.4 (integration tests 14–17). |
| **AC2: Tool response shape preservation** | AD-2, Section 4.2 (same JSON shapes), 6.4 (error shape), 8.4 (unit tests 8–13, integration tests 14–17, 20). |
| **AC3: Health/readiness check** | Section 3.1 (`/health` endpoint), AD-3, 8.4 (tests 1, 19). |
| **AC4: HTTP setup documentation** | Section 3.5 (README), AD-5, 8.4 (manual verification in Story 5). |
| **AC5: Test coverage maintained** | Section 3.4 (test suite), 8.1–8.3 (test strategy), 8.4 (20 automated tests). |
| **Exception 1: GraphQL service unavailable** | AD-4, Section 6.1 (GraphQL error handling), 8.4 (tests 5–7, 13, 18). |
| **Exception 2: Legacy stdio client** | Section 3.5 (README migration section), 8.4 (manual verification in Story 5). |
| **Exception 3: Authentication out of scope** | AD-6 (auth deferred), 8.4 (README note in Story 5). |

---

**End of HLD**
