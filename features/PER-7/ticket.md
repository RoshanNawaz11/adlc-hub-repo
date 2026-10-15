---
key: PER-7
type: story
status: todo
blocks: []
blocked_by: []
labels:
- Feature
- repo:agent
repos:
- agent
summary: '[agent] Agent: Agent which can directly answer or via MCP tools'
---

## Goal

Build an HTTP agent that accepts natural-language questions on a single `/query` endpoint.

The agent uses an LLM to decide whether to answer directly or call MCP tools, then returns a useful answer.

MCP tools come from [PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service) over **stdio** (no FastMCP / no [PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service) changes).

## Dependencies

* [PER-5](https://linear.app/personalgs/issue/PER-5/graphql-graphql-service-mock-schemas-and-data-for-salesforce-hr-prod) GraphQL mock service must be reachable (e.g. `GRAPHQL_ENDPOINT=http://localhost:4000/graphql`).
* [PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service) MCP server stays **stdio**; agent launches/connects as an MCP stdio client.

## Must use MCP tools (via stdio)

1. `list_sales_order`
2. `list_employees`
3. `get_sales_order_by_id`
4. `get_employee_by_id`

## Behavior

* Expose `POST /query` with body `{"query": "<natural language>"}` and response `{"answer": "<text>"}`.
* LLM decides:
  * **Direct answer** when no tool is needed (e.g. greetings, help text, clarifying questions).
  * **Tool call(s)** when data from MCP is required (e.g. "what is the name of employee with emp-001" → `get_employee_by_id`).
* Normalize IDs when needed (e.g. `emp-001` → `EMP-001` if mocks use that form).
* Do **not** use keyword-only routing as the primary design; LLM tool-calling is required.

## Out of scope

* Authentication
* Changing [PER-5](https://linear.app/personalgs/issue/PER-5/graphql-graphql-service-mock-schemas-and-data-for-salesforce-hr-prod) GraphQL schemas
* Changing [PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service) MCP transport (no FastMCP / no HTTP MCP server)
* Implementing GraphQL or MCP tool business logic beyond client usage

## Acceptance

* Agent HTTP service with `POST /query` works for natural-language questions.
* Agent can invoke all four MCP tools over **stdio** and return useful answers.
* Example: query `what is the name of employee with emp-001` returns the employee name from MCP/GraphQL data (not an error).
* Questions that need no tools can be answered directly without a tool call.
* Basic tests (unit with mocked LLM/MCP and/or a documented manual verification script).

## Env (documented in story)

* LLM: existing Anthropic gateway / `ANTHROPIC_*` (same pattern as ADLC).
* `GRAPHQL_ENDPOINT` for the MCP child process.
* MCP server command: [PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service) `python -m mcp.server` (stdio) from the mcp repo venv.
