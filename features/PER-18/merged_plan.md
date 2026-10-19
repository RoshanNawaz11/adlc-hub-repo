# Merged plan — PER-18

Execution order (1 ticket(s)):

1. `PER-19`

---

## Ticket `PER-19`

# Tech Stub: PER-19

**Ticket:** PER-19  
**Repo:** `mcp` (adlc-mcp-server)  
**Title:** Serve MCP GraphQL tools over FastAPI + FastMCP HTTP (replace stdio)

---

## Stack

**Language & Runtime:**  
- Python 3.11+ (inferred from repo context: Python-based MCP server)

**Web Framework:**  
- FastAPI (per ticket requirement)

**MCP Transport:**  
- FastMCP HTTP/SSE (per ticket requirement)

**GraphQL Client:**  
- `httpx` (async HTTP client, standard for FastAPI projects) or `gql` (GraphQL client library)

**Server:**  
- Uvicorn (ASGI server, standard for FastAPI)

**Testing:**  
- pytest (Python standard)
- httpx or FastMCP test client (for integration tests)

**Linting/Formatting:**  
- ruff or black/flake8 (standard Python tooling; verify repo conventions)

**Dependency Management:**  
- pip + `requirements.txt` or Poetry (verify existing setup in `/home/goos/projects/adlc-mcp-server/`)

---

## Schema & API Contracts

### Inbound (HTTP Endpoints)

**`GET /health`**  
- **Response (200):**
  ```json
  {"status": "ok"}
  ```
- **Purpose:** Process liveness check; does not validate GraphQL connectivity.

**`/sse` (FastMCP SSE Endpoint)**  
- **Protocol:** Server-Sent Events (SSE) via FastMCP.
- **Tool Invocations:** JSON-RPC or FastMCP protocol messages.
- **Tool Names (preserved from stdio):**
  - `list_sales_order`
  - `list_employees`
  - `get_sales_order_by_id`
  - `get_employee_by_id`
- **Tool Responses:** Same JSON shapes as stdio version (see Tool Contracts).

---

### Tool Contracts (unchanged from stdio)

**`list_sales_order`**  
- **Input:** none.
- **Success Output:**
  ```json
  [
    { "id": "...", "customer": "...", "total": 123.45, ... },
    ...
  ]
  ```
- **Error Output:**
  ```json
  {"error": "GraphQL endpoint unreachable"}
  ```

**`list_employees`**  
- **Input:** none.
- **Success Output:**
  ```json
  [
    { "id": "...", "name": "...", "role": "...", ... },
    ...
  ]
  ```
- **Error Output:**
  ```json
  {"error": "GraphQL query failed"}
  ```

**`get_sales_order_by_id`**  
- **Input:**
  ```json
  {"id": "SO-12345"}
  ```
- **Success Output:**
  ```json
  { "id": "SO-12345", "customer": "...", "total": 123.45, ... }
  ```
- **Error Output:**
  ```json
  {"error": "Invalid or missing id"}
  ```
  or
  ```json
  {"error": "Sales order not found"}
  ```

**`get_employee_by_id`**  
- **Input:**
  ```json
  {"id": "EMP-67890"}
  ```
- **Success Output:**
  ```json
  { "id": "EMP-67890", "name": "...", "role": "...", ... }
  ```
- **Error Output:**
  ```json
  {"error": "Invalid or missing id"}
  ```
  or
  ```json
  {"error": "Employee not found"}
  ```

---

### Outbound (GraphQL)

**Endpoint:**  
- Configured via `GRAPHQL_ENDPOINT` environment variable.
- Same queries as the stdio version (unchanged).

**Queries (examples, exact syntax per existing code):**
- `query ListSalesOrders { ... }`
- `query ListEmployees { ... }`
- `query GetSalesOrderById($id: ID!) { ... }`
- `query GetEmployeeById($id: ID!) { ... }`

**Error Handling:**  
- GraphQL network error or error response → raise exception → tool layer catches and returns `{"error": "..."}`.

---

## File Touch List

**Root:** `/home/goos/projects/adlc-mcp-server/`

### New Files

1. **`server.py`**  
   - FastAPI app: `/sse` endpoint (FastMCP SSE), `/health` endpoint.
   - Reads `GRAPHQL_ENDPOINT`, `HOST`, `PORT` from environment.

2. **`main.py`** (or update if exists)  
   - Uvicorn entrypoint: `uvicorn server:app --host 0.0.0.0 --port 8000`.

3. **`tools.py`** (new or update if exists)  
   - MCP tool definitions (`@mcp.tool()` or equivalent).
   - Wraps GraphQL client calls; returns success payloads or `{"error": "..."}`.

4. **`graphql_client.py`** (new or update if exists)  
   - Shared GraphQL query executor.
   - Reads `GRAPHQL_ENDPOINT` from environment.
   - Raises exceptions on network/GraphQL errors.

5. **`tests/test_server.py`**  
   - Integration tests for `/health` and `/sse` endpoints.

6. **`tests/test_tools.py`**  
   - Unit tests for tool behavior (valid/invalid inputs, GraphQL success/error mocking).

---

### Updated Files

1. **`README.md`**  
   - **Additions:**
     - HTTP setup as primary path (Uvicorn command, environment variables).
     - MCP URL format (e.g., `http://localhost:8000/sse`).
     - Migration notes from stdio to HTTP.
     - Authentication deferred (document as follow-up).
   - **Sections to add/update:**
     - "Setup" → HTTP server start.
     - "Environment Variables" → `GRAPHQL_ENDPOINT`, `PORT`, `HOST`.
     - "Migration from stdio" → before/after client config examples.

2. **`requirements.txt`** or **`pyproject.toml`** (if Poetry)  
   - **Add dependencies:**
     - `fastapi`
     - `uvicorn[standard]`
     - `fastmcp` (verify current stable version)
     - `httpx` (if not already present, for GraphQL client)
     - `pytest`, `pytest-asyncio` (if not already present)

3. **Existing tool/GraphQL code** (if any stdio-specific logic exists)  
   - Remove stdio-specific setup (e.g., `mcp.run()` for stdio).
   - Adapt to FastMCP HTTP initialization.

---

## Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| **FastMCP SSE protocol mismatch with clients** | Clients cannot connect or invoke tools | Low | Pin FastMCP version; test with known clients (e.g., Claude Desktop, custom MCP clients) before release. |
| **GraphQL endpoint URL misconfiguration** | All tools fail with unclear errors | Medium | Validate `GRAPHQL_ENDPOINT` at startup; log clear error and exit if missing or malformed. Add startup readiness check (optional `/readiness` endpoint that pings GraphQL). |
| **Test coverage gap during stdio→HTTP migration** | Regressions in tool behavior or JSON shapes | Medium | Run existing stdio tests (if any) in parallel during transition. Migrate to HTTP-based tests incrementally. Compare tool outputs byte-for-byte between stdio and HTTP modes. |
| **Unclear migration path for existing stdio clients** | Clients continue using deprecated stdio, or fail to switch correctly | Medium | README includes side-by-side stdio vs. HTTP config examples. Document the exact MCP URL format. Provide a "Quick Start" HTTP section at the top of README. |
| **Health check interpreted as GraphQL liveness** | Operators assume `/health=ok` means GraphQL is reachable; false confidence | Low | Document that `/health` only checks process liveness. Consider adding a separate `/readiness` endpoint that validates GraphQL connectivity (optional, not in v1 scope). |
| **Missing or invalid `GRAPHQL_ENDPOINT` at runtime** | Server starts but all tools fail | Medium | Fail fast at startup if `GRAPHQL_ENDPOINT` is missing. Log the configured endpoint on startup for debugging. |
| **Tool input validation gap** | Malformed `id` arguments cause uncaught exceptions | Low | Validate tool inputs (e.g., non-empty `id`) before calling GraphQL. Return `{"error": "Invalid or missing id"}` for malformed inputs. |
| **Authentication requirement emerges mid-development** | Scope creep or org policy blocks release | Low | Confirm with stakeholders before starting: is auth required for v1? If yes, add to scope and file touch list. If no, document as follow-up in README. |

---

## Open Questions

*(None at this time; all ACs and exceptions addressed in HLD.)*

---

## Notes

- **Stdio deprecation:** Once HTTP is stable, consider removing stdio support entirely in a future version. Document this intent in README.
- **FastMCP version:** Verify the current stable FastMCP version supports SSE and is compatible with FastAPI. Pin the version in `requirements.txt`.
- **GraphQL query preservation:** The exact GraphQL queries (field names, arguments, response shapes) must match the stdio version. Review existing code or stdio logs to ensure no drift.
- **Test strategy:** If no existing tests cover the tools, this is the opportunity to add them. Prioritize integration tests (tool invocation over HTTP) over unit tests (mocked GraphQL responses).

---

**End of Tech Stub**

---

# Story summaries

- `01-graphql`: Story 1: Low-Level Design

---

_Single-ticket pack: order is trivial (no multi-split)._
