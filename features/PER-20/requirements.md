# Requirements: Integrate Agent with MCP server

## Overview

Replace the agent's stdio MCP subprocess with a FastMCP HTTP client pointed at the shared MCP server. The agent keeps its existing behavior—LLM chooses whether to answer directly or call MCP tools, then synthesizes tool results into the final answer—while switching from Flask + stdio to FastAPI + HTTP transport.

## References

- Epic: (not provided)
- Ticket: Integrate Agent with MCP server
- Related: [PER-18](https://linear.app/personalgs/issue/PER-18/serve-mcp-graphql-tools-over-fastapi-fastmcp-http-replace-stdio) MCP server over FastAPI

## Acceptance Criteria

### AC-1: Direct LLM answers (no tool call)

**Given** the agent is running  
**When** a caller sends a question that does not need data (a greeting or a help request)  
**Then** the LLM answers directly and no MCP tool is called

### AC-2: Tool calls over FastMCP HTTP

**Given** `MCP_URL` points at the FastAPI and FastMCP server  
**When** the LLM requests employee or sales-order data  
**Then** the agent calls that tool over FastMCP HTTP (not stdio) and returns the synthesized answer  
**And** the tools remain `list_sales_order`, `list_employees`, `get_sales_order_by_id`, and `get_employee_by_id`

### AC-3: MCP error handling

**Given** a tool returns `{"error": "..."}` or the MCP server is unreachable  
**When** the question needed that tool  
**Then** the agent returns a clear error and stays up  
**And** it does not start a stdio MCP process

### AC-4: Health check

**Given** the agent process is started  
**When** an operator checks health on the FastAPI app  
**Then** they can see the process is up and whether the MCP HTTP endpoint is reachable, without calling a tool

### AC-5: Setup documentation

**Given** the `adlc-agents` README  
**When** a developer follows setup  
**Then** they run the agent with FastAPI, set `MCP_URL` to the FastMCP SSE URL, set the LLM env, and `POST` a question  
**And** HTTP MCP is the documented primary path  
**And** Flask and stdio are not documented

## Exception Criteria

- **LLM or gateway down**: The request fails with an error and the process stays up.
- **MCP HTTP server down or tool error**: The agent surfaces that failure and does not fall back to stdio.
- **ID normalization**: An `id` argument is still normalized the way it is today (`emp-001` becomes `EMP-001`) before the HTTP tool call.
- **Auth out of scope**: Auth on the agent or MCP URL is out of scope for v1 unless org policy already requires it.

## Verification

Test coverage must include:

1. Direct answer (no tool call)
2. One tool call over HTTP with successful result
3. MCP server unreachable
4. Tool returns `{"error": "..."}`
5. Health endpoint reports process up and MCP endpoint reachable

## Definition of Done

- The `agent` serves its query API on FastAPI.
- MCP access is a FastMCP HTTP client.
- The LLM is still called for direct answers and for synthesis after tool use.
- README primary path is FastAPI plus `MCP_URL`.
- Tests cover direct answer, one tool call over HTTP, MCP or tool error, and health.
- No stdio MCP subprocess remains on that path.
