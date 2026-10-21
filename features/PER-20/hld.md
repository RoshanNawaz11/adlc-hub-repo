# HLD: Integrate Agent with MCP server

## Overview

Replace the agent's stdio MCP subprocess with a FastMCP HTTP client pointed at the shared MCP server running on FastAPI. The agent preserves its existing LLM-driven behavior—direct answers when no data is needed, MCP tool calls when data is required, synthesis of tool results into final answers—while switching transport from Flask + stdio to FastAPI + HTTP.

## Components

### Agent Service (`agent` repo)

**Responsibilities:**
- Serve query API on FastAPI
- Route incoming questions to LLM client
- Invoke MCP tools over FastMCP HTTP when LLM requests data
- Synthesize tool results into final answers
- Expose health endpoint reporting process and MCP connectivity status

**Dependencies:**
- LLM gateway (Anthropic or existing gateway client)
- MCP server over FastMCP HTTP at `MCP_URL`

**Does not:**
- Spawn stdio MCP subprocess
- Make GraphQL calls directly
- Change tool names or payloads

### FastMCP HTTP Client (new component in `agent`)

**Responsibilities:**
- Connect to MCP server at `MCP_URL` (e.g. `http://127.0.0.1:8000/sse`)
- Call tools `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`
- Normalize `id` arguments (`emp-001` → `EMP-001`) before HTTP call
- Return tool results or `{"error": "..."}` to agent orchestrator

**Does not:**
- Fall back to stdio on failure
- Own tool schema or GraphQL queries (those remain on MCP server per PER-18)

### LLM Client (existing)

**Responsibilities:**
- Decide whether to answer directly or request MCP tool
- Synthesize tool results into natural language answers

**No changes** to this component.

### MCP Server (external, `mcp` repo, PER-18)

**Responsibilities:**
- Serve four tools over FastMCP HTTP/SSE
- Execute GraphQL queries against TriNet backend
- Return employee or sales-order data, or `{"error": "..."}`

**Interface:**
- Tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`
- Transport: FastMCP HTTP (SSE)
- No stdio interface

## Contracts

### Agent Query API (FastAPI)

**Endpoint:** `POST /query` (exact path deferred to implementation)

**Request:**
```json
{
  "question": "string"
}
```

**Response (success):**
```json
{
  "answer": "string"
}
```

**Response (error):**
```json
{
  "error": "string"
}
```

**Behavior:**
- AC-1: When `question` is a greeting or help request, LLM answers directly, no MCP call.
- AC-2: When LLM requests data, agent calls MCP tool over HTTP, synthesizes answer.
- AC-3: When MCP unreachable or tool returns error, agent returns clear error, stays up.

### Health Endpoint (FastAPI)

**Endpoint:** `GET /health` (exact path deferred to implementation)

**Response:**
```json
{
  "status": "up",
  "mcp_reachable": true
}
```

**Behavior:**
- AC-4: Reports process up and MCP HTTP endpoint reachable without calling a tool.
- `mcp_reachable` may be a simple connectivity check (e.g. HEAD request or cached probe result).

### FastMCP HTTP Client → MCP Server

**Transport:** FastMCP HTTP (SSE) at `MCP_URL`

**Tool calls:**
- `list_sales_order` → returns list of sales orders
- `list_employees` → returns list of employees
- `get_sales_order_by_id(id: string)` → returns single sales order or `{"error": "..."}`
- `get_employee_by_id(id: string)` → returns single employee or `{"error": "..."}`

**ID normalization:**
- Before HTTP call, normalize `id` argument: `emp-001` → `EMP-001` (existing logic preserved).

**Error contract:**
- Tool may return `{"error": "..."}` on failure.
- MCP server unreachable: HTTP client surfaces connection error.
- Agent does not fall back to stdio (AC-3).

### Environment Configuration

**Required:**
- `MCP_URL`: FastMCP SSE URL (e.g. `http://127.0.0.1:8000/sse`)
- LLM credentials (existing env vars for Anthropic or gateway)

**Optional:**
- Auth credentials (out of scope for v1 unless org policy requires)

## Data Flow

### Direct Answer (AC-1)

1. Client sends `POST /query` with greeting or help question.
2. Agent routes to LLM client.
3. LLM determines no data needed, generates answer.
4. Agent returns `{"answer": "..."}`.

### Tool Call (AC-2)

1. Client sends `POST /query` with data question (e.g. "list sales orders").
2. Agent routes to LLM client.
3. LLM requests MCP tool (e.g. `list_sales_order`).
4. Agent calls FastMCP HTTP client.
5. FastMCP HTTP client normalizes any `id` argument, calls MCP server at `MCP_URL`.
6. MCP server returns data or `{"error": "..."}`.
7. Agent routes result to LLM client for synthesis.
8. LLM synthesizes natural language answer.
9. Agent returns `{"answer": "..."}`.

### Error Handling (AC-3)

**MCP server unreachable:**
1. FastMCP HTTP client fails to connect.
2. Agent returns `{"error": "MCP server unreachable"}`.
3. Process stays up.

**Tool returns error:**
1. MCP server returns `{"error": "employee not found"}`.
2. Agent routes to LLM for synthesis or returns error directly.
3. Process stays up.

**LLM or gateway down (exception):**
1. LLM client fails.
2. Agent returns `{"error": "LLM unavailable"}`.
3. Process stays up.

### Health Check (AC-4)

1. Operator sends `GET /health`.
2. Agent checks process status (always up if responding).
3. Agent probes MCP HTTP endpoint (e.g. HEAD request or cached connectivity status).
4. Returns `{"status": "up", "mcp_reachable": true/false}`.

## Risks

### R-1: MCP server unavailable

**Likelihood:** Medium  
**Impact:** High (agent cannot answer data questions)  
**Mitigation:**
- AC-3 requires clear error and no stdio fallback.
- Health endpoint (AC-4) surfaces MCP connectivity before first query fails.
- Operational monitoring on health endpoint alerts when `mcp_reachable: false`.

### R-2: FastMCP HTTP transport mismatch

**Likelihood:** Low  
**Impact:** High (integration breaks)  
**Mitigation:**
- PER-18 defines MCP server FastMCP HTTP interface; this ticket consumes it.
- Integration test (verification item 2) validates one tool call over HTTP.
- Defer detailed transport contract to PER-18 implementation; this ticket assumes SSE at `MCP_URL`.

### R-3: ID normalization regression

**Likelihood:** Low  
**Impact:** Medium (wrong records returned)  
**Mitigation:**
- Exception criterion explicitly preserves existing normalization (`emp-001` → `EMP-001`).
- Verification item 2 includes ID normalization in test case.

### R-4: LLM decision divergence

**Likelihood:** Low  
**Impact:** Medium (unexpected tool calls or direct answers)  
**Mitigation:**
- "Does not change" constraint: LLM choice logic unchanged.
- Verification items 1 and 2 test direct answer and tool call paths.

## Architecture Decision Records

### ADR-1: FastAPI replaces Flask

**Context:** Agent currently serves on Flask; FastAPI is PER-18 MCP server stack.

**Decision:** Migrate agent to FastAPI for consistency and async support.

**Consequences:**
- Pros: Async-native for HTTP MCP client; stack consistency with MCP server.
- Cons: Migration effort; existing Flask knowledge may not transfer.

**Alternatives considered:**
- Keep Flask: would require sync HTTP client or thread pool; diverges from MCP server stack.

### ADR-2: FastMCP HTTP over stdio

**Context:** Agent currently spawns stdio MCP subprocess; PER-18 serves MCP over HTTP.

**Decision:** Replace stdio with FastMCP HTTP client.

**Consequences:**
- Pros: Shared MCP server; no per-request subprocess; scales horizontally.
- Cons: Network hop; MCP server becomes critical dependency (R-1).

**Alternatives considered:**
- Keep stdio: requires per-agent MCP subprocess; cannot share MCP server.

### ADR-3: No stdio fallback

**Context:** MCP server may be unreachable (R-1).

**Decision:** Return clear error; do not fall back to stdio (AC-3).

**Consequences:**
- Pros: Simpler code; forces operational visibility of MCP availability.
- Cons: Agent cannot answer data questions when MCP down.

**Alternatives considered:**
- Stdio fallback: adds complexity; undermines shared MCP server goal; deferred for v2 if needed.

### ADR-4: Health endpoint for MCP connectivity

**Context:** Operators need to know if agent can reach MCP before queries fail.

**Decision:** Add `/health` endpoint reporting process up and MCP reachable (AC-4).

**Consequences:**
- Pros: Proactive operational visibility; supports readiness probes.
- Cons: Extra endpoint; MCP reachability check may add latency (use cached probe or HEAD request).

**Alternatives considered:**
- Omit health check: operators discover MCP unavailability only on query failure.

### ADR-5: ID normalization preserved in HTTP client

**Context:** Existing agent normalizes IDs before tool call (`emp-001` → `EMP-001`).

**Decision:** FastMCP HTTP client applies same normalization before HTTP call.

**Consequences:**
- Pros: Behavior unchanged; no GraphQL or MCP server changes needed.
- Cons: Normalization logic duplicated (agent and possibly MCP server).

**Alternatives considered:**
- Move normalization to MCP server: breaks "Does not change" constraint; deferred.

## Open Questions

_None. All behavioral contracts and error paths are specified._

## Implementation Notes

### Agent (`agent` repo)

**New files:**
- FastAPI app entry point (replaces Flask app)
- FastMCP HTTP client module
- Health endpoint handler

**Modified files:**
- README.md: document FastAPI setup, `MCP_URL` env var, primary path (AC-5)
- Orchestrator/router: route to LLM client, invoke FastMCP HTTP client on tool request
- Tests: add coverage per verification items 1-5

**Removed:**
- Flask app
- stdio MCP subprocess logic

**No changes outside `agent` repo.** MCP server (`mcp` repo) and GraphQL (`graphql` repo) are external dependencies.

### Test Coverage (Verification)

1. **Direct answer (AC-1):** Greeting question → LLM answers directly, no MCP call logged.
2. **Tool call over HTTP (AC-2):** Data question → LLM requests tool → FastMCP HTTP client calls `MCP_URL` → synthesized answer returned. Verify ID normalization.
3. **MCP unreachable (AC-3):** Mock `MCP_URL` unreachable → agent returns `{"error": "MCP server unreachable"}`, stays up.
4. **Tool error (AC-3):** Mock tool returns `{"error": "not found"}` → agent surfaces error, stays up.
5. **Health endpoint (AC-4):** `GET /health` → `{"status": "up", "mcp_reachable": true}` when MCP up, `false` when down.

### README Primary Path (AC-5)

```markdown
## Setup

1. Install dependencies: `pip install -r requirements.txt`
2. Set environment variables:
   - `MCP_URL`: FastMCP SSE URL (e.g. `http://127.0.0.1:8000/sse`)
   - LLM credentials (see [LLM setup])
3. Run: `uvicorn agent.main:app --host 0.0.0.0 --port 8001`
4. Query: `POST http://localhost:8001/query` with `{"question": "list sales orders"}`
```

Flask and stdio paths removed from README.

## Definition of Done Checklist

- [ ] Agent serves query API on FastAPI
- [ ] MCP access via FastMCP HTTP client (no stdio subprocess)
- [ ] LLM called for direct answers and synthesis after tool use
- [ ] README documents FastAPI + `MCP_URL` as primary path
- [ ] Tests cover:
  - [ ] Direct answer (no tool call)
  - [ ] One tool call over HTTP
  - [ ] MCP server unreachable
  - [ ] Tool returns error
  - [ ] Health endpoint
- [ ] No stdio MCP subprocess on FastAPI path
- [ ] AC-1 verified: direct answers when no data needed
- [ ] AC-2 verified: tools called over HTTP, four tool names unchanged
- [ ] AC-3 verified: clear error on MCP failure, no stdio fallback
- [ ] AC-4 verified: health endpoint reports process and MCP status
- [ ] AC-5 verified: README primary path is FastAPI + `MCP_URL`
