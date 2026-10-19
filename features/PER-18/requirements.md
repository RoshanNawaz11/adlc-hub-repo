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

### Definition of Done

- [ ] `mcp` runs on FastAPI + FastMCP HTTP
- [ ] README primary path is HTTP
- [ ] All four tools work through HTTP against GraphQL
- [ ] Tests green
- [ ] No regression in tool names/payloads
- [ ] Clients know the MCP URL to use instead of stdio
