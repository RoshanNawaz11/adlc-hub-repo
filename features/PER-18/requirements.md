# Requirements

## Story: Serve MCP GraphQL tools over FastAPI + FastMCP HTTP (replace stdio)

### Acceptance Criteria

**AC1: Tool availability over HTTP**

**Given** the MCP server is running in the new mode  
**When** a client connects via FastMCP HTTP (not stdio)  
**Then** these tools remain available and behave as today:
- `list_sales_order`
- `list_employees`
- `get_sales_order_by_id`
- `get_employee_by_id`

---

**AC2: Tool response shape preservation**

**Given** a valid `GRAPHQL_ENDPOINT` is configured  
**When** a tool is invoked over HTTP  
**Then** results use the same success/error JSON shapes as the current stdio server:
- List/get payloads for successful calls
- `{"error": "..."}` for errors

---

**AC3: Health/readiness check**

**Given** the server process is started  
**When** an operator checks health/readiness on the HTTP app  
**Then** they can tell the process is up without calling a GraphQL tool

---

**AC4: HTTP setup documentation**

**Given** the `adlc-mcp-server` README  
**When** a developer follows setup instructions  
**Then** they can:
- Run the server with FastAPI + FastMCP over HTTP
- Point a client at that URL
- (HTTP is the documented primary path)

---

**AC5: Test coverage maintained**

**Given** automated tests for tools/client exist  
**When** CI or local tests run  
**Then** tool behavior is still covered (HTTP or in-process FastMCP test client)

---

### Exception Criteria

**Exception 1: GraphQL service unavailable**

**Given** GraphQL is down or returns errors  
**When** a tool is invoked  
**Then**:
- Tools return the existing error JSON shape
- The server process stays up

---

**Exception 2: Legacy stdio client**

**Given** a client is still configured for stdio  
**When** a developer consults the README  
**Then** README documents migration to HTTP (prefer HTTP-only once callers can switch)

---

**Exception 3: Authentication out of scope**

**Given** v1 scope  
**When** auth requirements are considered  
**Then** auth is out of scope unless org policy already requires it (document as follow-up)

---

### Tests

**Test 1: Server starts and `/health` returns 200 + `{"status": "ok"}`**  
- **Verifies:** AC3 (health check)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_server.py`

**Test 2: `/sse` FastMCP SSE handshake succeeds**  
- **Verifies:** AC1 (FastMCP endpoint exists)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_server.py`

**Test 3: Server fails to start if `GRAPHQL_ENDPOINT` is missing**  
- **Verifies:** Exception 1 (startup validation)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_server.py`

**Test 4: GraphQL client executes query successfully**  
- **Verifies:** AC2 (GraphQL query execution)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py`

**Test 5: GraphQL client raises exception on network error**  
- **Verifies:** Exception 1 (GraphQL unreachable)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py`

**Test 6: GraphQL client raises exception on GraphQL error response**  
- **Verifies:** Exception 1 (GraphQL errors field)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py`

**Test 7: GraphQL client raises exception on HTTP error**  
- **Verifies:** Exception 1 (HTTP status error)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_graphql_client.py`

**Test 8: `list_sales_order` returns success payload**  
- **Verifies:** AC1 (tool availability), AC2 (response shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_tools.py`

**Test 9: `list_employees` returns success payload**  
- **Verifies:** AC1 (tool availability), AC2 (response shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_tools.py`

**Test 10: `get_sales_order_by_id` returns success payload**  
- **Verifies:** AC1 (tool availability), AC2 (response shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_tools.py`

**Test 11: `get_employee_by_id` returns success payload**  
- **Verifies:** AC1 (tool availability), AC2 (response shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_tools.py`

**Test 12: `get_sales_order_by_id` returns error for missing `id`**  
- **Verifies:** AC2 (error shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_tools.py`

**Test 13: Tool returns error when GraphQL client raises exception**  
- **Verifies:** Exception 1 (error JSON shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_tools.py`

**Test 14: Integration test for `list_sales_order` over HTTP**  
- **Verifies:** AC1 (HTTP invocation), AC2 (response shape), AC5 (test coverage)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

**Test 15: Integration test for `list_employees` over HTTP**  
- **Verifies:** AC1 (HTTP invocation), AC2 (response shape), AC5 (test coverage)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

**Test 16: Integration test for `get_sales_order_by_id` over HTTP**  
- **Verifies:** AC1 (HTTP invocation), AC2 (response shape), AC5 (test coverage)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

**Test 17: Integration test for `get_employee_by_id` over HTTP**  
- **Verifies:** AC1 (HTTP invocation), AC2 (response shape), AC5 (test coverage)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

**Test 18: Integration test for GraphQL error handling**  
- **Verifies:** Exception 1 (server stays up, error JSON)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

**Test 19: Integration test for health endpoint**  
- **Verifies:** AC3 (health check)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

**Test 20: Integration test for invalid tool input**  
- **Verifies:** AC2 (error shape)
- **Path:** `/home/goos/projects/adlc-mcp-server/tests/test_integration.py`

---

### Definition of Done

- [ ] `mcp` runs on FastAPI + FastMCP HTTP
- [ ] README primary path is HTTP
- [ ] All four tools work through HTTP against GraphQL
- [ ] Tests green (20 automated tests pass)
- [ ] No regression in tool names/payloads
- [ ] Clients know the MCP URL to use instead of stdio
