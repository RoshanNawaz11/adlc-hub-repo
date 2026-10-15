# requirements.md

## Overview
An HTTP agent service that accepts natural-language questions via a `/query` endpoint and returns answers by either responding directly or invoking MCP tools over stdio.

## Acceptance Criteria

### AC1: HTTP Endpoint
**Given** the agent service is running  
**When** a POST request is sent to `/query` with body `{"query": "<natural language>"}`  
**Then** the service returns HTTP 200 with JSON body `{"answer": "<text>"}`

### AC2: Direct Answer (No Tool)
**Given** the agent receives a question requiring no data lookup (e.g., "hello", "help")  
**When** the LLM determines no MCP tool is needed  
**Then** the agent returns a direct text answer without invoking any tool

### AC3: Tool Call for Employee Lookup
**Given** the agent receives query `"what is the name of employee with emp-001"`  
**When** the LLM determines data lookup is required  
**Then** the agent invokes `get_employee_by_id` via MCP stdio and returns the employee name from GraphQL data

### AC4: All Four MCP Tools Available
**Given** the MCP server (PER-6) is running over stdio  
**When** the agent initializes  
**Then** the agent can successfully list and invoke all four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`

### AC5: ID Normalization
**Given** the agent receives a query with lowercase ID like `"emp-001"`  
**When** the GraphQL mock uses uppercase form `"EMP-001"`  
**Then** the agent normalizes the ID before calling the tool and returns correct data

### AC6: Basic Tests
**Given** the implementation is complete  
**When** tests are run  
**Then** 21 unit tests with mocked LLM/MCP pass AND 1 integration test with mocked services passes AND a documented manual verification script successfully demonstrates AC1–AC5 against real PER-5/PER-6 services

**Test Breakdown** (21 unit tests total):
- **HTTP server tests** (`tests/test_http_server.py`): 6 tests
  1. Valid request to `/query` returns 200 with JSON answer
  2. Malformed JSON to `/query` returns 400
  3. Missing `query` field returns 400
  4. Internal error returns 500
  5. Health check with MCP alive returns 200
  6. Health check with MCP crashed returns 503
- **LLM client tests** (`tests/test_llm_client.py`): 3 tests
  1. Direct answer parsing from Anthropic API
  2. Tool call parsing from Anthropic API
  3. API error propagation
- **MCP client tests** (`tests/test_mcp_client.py`): 4 tests
  1. Successful handshake and tool discovery
  2. Handshake failure handling
  3. Graceful shutdown
  4. Tool call timeout after 10s
- **Orchestrator tests** (`tests/test_orchestrator.py`): 8 tests
  1. Direct answer path (no tool invocation)
  2. Tool call success with employee lookup and synthesis
  3. MCP tool error propagation
  4. LLM call count verification (exactly 2 on tool path)
  5. ID normalization (`emp-001` → `EMP-001`)
  6. Argument validation: missing required field raises `ValueError`
  7. Argument validation: unknown tool name raises `ValueError`
  8. Synthesis loop detection (tool request on synthesis step raises `RuntimeError`)

**Integration test** (`tests/test_integration.py`): 1 test
- End-to-end query with mocked LLM/MCP services verifies HTTP 200, synthesized answer, 2 LLM calls, 1 MCP call with normalized ID

**Manual verification** (`tests/manual_verification.md`):
- Documented script with 5 test cases against real PER-5/PER-6 services:
  1. Direct answer query (`"hello"`)
  2. Employee lookup with ID normalization (`"what is the name of employee with emp-001"`)
  3. List employees (`"list all employees"`)
  4. Get sales order (`"get sales order ord-001"`)
  5. Health check endpoint (`GET /health`)

## Dependencies
- **PER-5**: GraphQL mock service reachable at `GRAPHQL_ENDPOINT` (e.g., `http://localhost:4000/graphql`)
- **PER-6**: MCP server running via `python -m mcp.server` (stdio) with the four required tools

## Environment Variables
- `ANTHROPIC_API_KEY`: LLM API key (required)
- `ANTHROPIC_BASE_URL`: LLM API base URL (optional, defaults to `https://api.anthropic.com`)
- `GRAPHQL_ENDPOINT`: URL for the GraphQL service used by MCP child process (required)
- `MCP_SERVER_COMMAND`: Command to start MCP server (optional, defaults to `python -m mcp.server`)
- `MCP_SERVER_WORKING_DIR`: Working directory for MCP subprocess (required, path to PER-6 repo)
- `FLASK_HOST`: HTTP server bind host (optional, defaults to `0.0.0.0`)
- `FLASK_PORT`: HTTP server bind port (optional, defaults to `5000`)
- `LOG_LEVEL`: Logging level (optional, defaults to `INFO`)

## Implementation Files
All paths relative to `/home/goos/projects/adlc-agents`:

### New Files
- `app/agent/__init__.py` — package marker
- `app/agent/http_server.py` — Flask server with `/query` and `/health` endpoints
- `app/agent/orchestrator.py` — agent orchestrator coordinating LLM and MCP
- `app/agent/llm_client.py` — Anthropic LLM client
- `app/agent/mcp_client.py` — MCP stdio client
- `tests/test_http_server.py` — 6 unit tests (valid request, malformed JSON, missing field, internal error, health check alive, health check crashed)
- `tests/test_orchestrator.py` — 8 unit tests (direct answer, tool call success, MCP error, LLM call count, ID normalization, argument validation missing field, argument validation unknown tool, synthesis loop detection)
- `tests/test_llm_client.py` — 3 unit tests (direct answer parsing, tool call parsing, API error)
- `tests/test_mcp_client.py` — 4 unit tests (successful handshake, handshake failure, graceful shutdown, tool call timeout)
- `tests/test_integration.py` — 1 integration test (end-to-end with mocked services)
- `tests/manual_verification.md` — manual verification script documentation

### Modified Files
- `README.md` — add quickstart, environment variables, test instructions
- `requirements.txt` — add `flask==3.1.0`, `anthropic==0.18.0`

## Out of Scope
- Authentication
- Modifying PER-5 GraphQL schemas
- Changing PER-6 MCP transport (no FastMCP, no HTTP MCP server)
- Implementing GraphQL or MCP tool business logic beyond client usage
