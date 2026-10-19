# Story 01-agent LLD

**Repo**: agent  
**Type**: story  
**Epic**: [agent] Agent which can directly answer or via MCP tools  
**Status**: Draft

---

## 1. Acceptance Criteria

*(Inherited from requirements.md and breakdown.md — no additions, no changes)*

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
**Then** 21 unit tests with mocked LLM/MCP pass (6 HTTP server tests, 3 LLM client tests, 4 MCP client tests, 8 orchestrator tests) AND 1 integration test with mocked services passes AND a documented manual verification script successfully demonstrates AC1–AC5 against real PER-5/PER-6 services

---

## 2. File Plan

All paths are relative to `/home/goos/projects/adlc-agents`.

### 2.1 New Files

| Path | Purpose | Depends On |
|------|---------|------------|
| `app/agent/__init__.py` | Package marker for agent module | — |
| `app/agent/http_server.py` | Flask HTTP server with `/query` and `/health` endpoints | `orchestrator.py` |
| `app/agent/orchestrator.py` | Agent orchestrator coordinating LLM and MCP | `llm_client.py`, `mcp_client.py` |
| `app/agent/llm_client.py` | Anthropic LLM client for inference and tool-calling | — |
| `app/agent/mcp_client.py` | MCP stdio client for subprocess management and tool invocation | — |
| `tests/test_http_server.py` | Unit tests for HTTP server (request validation, error handling, health check) | `app/agent/http_server.py` |
| `tests/test_orchestrator.py` | Unit tests for orchestrator (direct answer, tool-calling, ID normalization, argument validation, synthesis loop detection) | `app/agent/orchestrator.py` |
| `tests/test_llm_client.py` | Unit tests for LLM client (API calls, parsing, error handling) | `app/agent/llm_client.py` |
| `tests/test_mcp_client.py` | Unit tests for MCP client (subprocess lifecycle, tool discovery, invocation, timeout) | `app/agent/mcp_client.py` |
| `tests/test_integration.py` | Integration test with mocked PER-5/PER-6 services | All `app/agent/*.py` |
| `tests/manual_verification.md` | Manual verification script documentation | — |

### 2.2 Modified Files

| Path | Change | Reason |
|------|--------|--------|
| `README.md` | Add quickstart, environment variables, test instructions | Document agent usage and setup |
| `requirements.txt` | Add `flask==3.1.0`, `anthropic==0.18.0` | New dependencies for HTTP server and LLM client |

---

## 3. Interfaces

### 3.1 HTTP Server (`app/agent/http_server.py`)

**Module Exports**:
```python
def create_app(orchestrator: Orchestrator) -> Flask:
    """Create and configure Flask app with /query and /health endpoints."""
    ...

def run_server(host: str = "0.0.0.0", port: int = 5000) -> None:
    """Start the HTTP server (blocking)."""
    ...
```

**Endpoint Contracts**:

**POST /query**
- Request: `{"query": "<string>"}`
- Success Response (200): `{"answer": "<string>"}`
- Client Error (400): `{"error": "<validation message>"}` (malformed JSON or missing `query`)
- Server Error (500): `{"error": "internal error"}` (LLM/MCP/orchestrator exception, logged)

**GET /health**
- Success Response (200): `{"status": "ok", "mcp_alive": true}`
- Degraded Response (503): `{"status": "degraded", "mcp_alive": false}` (MCP subprocess crashed)

**Dependencies**: `Orchestrator` (injected at app creation)

---

### 3.2 Orchestrator (`app/agent/orchestrator.py`)

**Class Interface**:
```python
class Orchestrator:
    def __init__(self, llm_client: LLMClient, mcp_client: MCPClient):
        """Initialize orchestrator with LLM and MCP clients."""
        ...

    def handle_query(self, query: str) -> str:
        """
        Process user query via LLM and optionally MCP tools.
        Returns final answer string.
        Raises ValueError on argument validation failures (caught by HTTP server → 400).
        Raises RuntimeError on synthesis loop detection (caught by HTTP server → 500).
        Raises exceptions on LLM/MCP failures (caught by HTTP server → 500).
        """
        ...
```

**Algorithm Outline** (see §4 for detail):
1. Construct messages: system prompt (role definition + tool schemas from MCP client) + user query.
2. Call `llm_client.infer(messages, tools=mcp_tools)` → parse response.
3. If direct text response: return answer.
4. If tool calls: for each call, validate arguments against tool schema (raise `ValueError` on missing required field or unknown tool), normalize IDs (`.upper()`), invoke `mcp_client.call_tool(name, args)`, collect results.
5. Append tool results to messages, call `llm_client.infer(messages)` again (synthesis step, no tools).
6. If synthesis response requests tools again: raise `RuntimeError` (synthesis loop detected).
7. Return synthesized answer.
8. Limit to 2 LLM calls total (initial + synthesis).

**Dependencies**: `LLMClient`, `MCPClient`

---

### 3.3 LLM Client (`app/agent/llm_client.py`)

**Class Interface**:
```python
class LLMClient:
    def __init__(self, api_key: str, base_url: str = "https://api.anthropic.com"):
        """Initialize Anthropic client with API credentials."""
        ...

    def infer(self, messages: list[dict], tools: list[dict] | None = None) -> dict:
        """
        Call Anthropic Messages API.
        Args:
            messages: Conversation history (system, user, assistant, tool_result).
            tools: Tool schemas in Anthropic format (optional).
        Returns:
            Parsed response:
                {"type": "text", "content": "<answer>"}
                OR
                {"type": "tool_use", "calls": [{"id": "<id>", "name": "<name>", "input": {...}}, ...]}
        Raises:
            Exception on API errors (network, auth, rate limit).
        """
        ...
```

**Tool Schema Format** (Anthropic Messages API):
```json
{
  "name": "get_employee_by_id",
  "description": "Fetch employee details by ID",
  "input_schema": {
    "type": "object",
    "properties": {
      "id": {"type": "string", "description": "Employee ID (e.g., EMP-001)"}
    },
    "required": ["id"]
  }
}
```

**Dependencies**: `anthropic` SDK (from `requirements.txt`)

---

### 3.4 MCP Client (`app/agent/mcp_client.py`)

**Class Interface**:
```python
class MCPClient:
    def __init__(self, server_command: str, working_dir: str, env: dict[str, str]):
        """
        Initialize MCP client.
        Args:
            server_command: Command to start MCP server (e.g., "python -m mcp.server").
            working_dir: Working directory for subprocess.
            env: Environment variables for subprocess (must include GRAPHQL_ENDPOINT).
        """
        ...

    def start(self) -> None:
        """
        Spawn MCP subprocess, perform JSON-RPC handshake, fetch tool list.
        Raises exception on handshake failure (agent fails to start).
        """
        ...

    def get_tools(self) -> list[dict]:
        """
        Return list of available tool schemas (Anthropic format).
        Must be called after start().
        """
        ...

    def call_tool(self, name: str, arguments: dict, timeout: int = 10) -> dict:
        """
        Invoke MCP tool via JSON-RPC over stdio.
        Args:
            name: Tool name (e.g., "get_employee_by_id").
            arguments: Tool input (e.g., {"id": "EMP-001"}).
            timeout: Request timeout in seconds.
        Returns:
            Tool result dict (content varies by tool).
        Raises:
            TimeoutError if request exceeds timeout.
            RuntimeError on MCP protocol error or tool execution error.
        """
        ...

    def is_alive(self) -> bool:
        """Check if MCP subprocess is still running."""
        ...

    def shutdown(self) -> None:
        """Gracefully terminate MCP subprocess (send EOF, wait 5s, SIGKILL if needed)."""
        ...
```

**JSON-RPC Protocol Example**:

*Request (to subprocess stdin):*
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_employee_by_id",
    "arguments": {"id": "EMP-001"}
  }
}
```

*Response (from subprocess stdout):*
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"name\": \"Alice\", \"id\": \"EMP-001\"}"
      }
    ]
  }
}
```

**Dependencies**: `subprocess` (stdlib), `json` (stdlib), `select` (stdlib, POSIX timeout enforcement)

---

## 4. Algorithm Details

### 4.1 Orchestrator Query Handling

**Function**: `Orchestrator.handle_query(query: str) -> str`

**Steps**:
1. **Construct Initial Messages**:
   ```python
   system_prompt = {
       "role": "system",
       "content": (
           "You are a helpful assistant with access to MCP tools for employee and sales order data. "
           "Use tools when the user asks for specific data. Answer directly for greetings or help requests."
       )
   }
   messages = [
       system_prompt,
       {"role": "user", "content": query}
   ]
   ```

2. **Initial LLM Inference**:
   ```python
   tools = self.mcp_client.get_tools()
   response = self.llm_client.infer(messages, tools=tools)
   llm_call_count = 1
   ```

3. **Parse Response Type**:
   - If `response["type"] == "text"`: return `response["content"]` (direct answer path, AC2).
   - If `response["type"] == "tool_use"`: proceed to step 4.

4. **Execute Tool Calls** (AC3, AC4, AC5):
   ```python
   tool_results = []
   for call in response["calls"]:
       # Validate tool name against available tools
       tool_schema = next((t for t in tools if t["name"] == call["name"]), None)
       if not tool_schema:
           raise ValueError(f"LLM requested unknown tool: {call['name']}")
       
       # Validate required fields present in tool call input
       required = tool_schema["input_schema"].get("required", [])
       for field in required:
           if field not in call["input"]:
               raise ValueError(f"Missing required field '{field}' for tool '{call['name']}'")
       
       # Normalize IDs to uppercase (AC5, AD-3)
       normalized_input = {k: v.upper() if k == "id" else v for k, v in call["input"].items()}
       
       # Invoke tool with 10s timeout
       result = self.mcp_client.call_tool(call["name"], normalized_input, timeout=10)
       tool_results.append({
           "tool_use_id": call["id"],
           "content": json.dumps(result)
       })
   ```

5. **Append Tool Results to Messages**:
   ```python
   messages.append({"role": "assistant", "content": response["calls"]})  # LLM's tool call
   messages.append({"role": "user", "content": tool_results})  # Tool results
   ```

6. **Synthesis LLM Call** (AD-4):
   ```python
   synthesis_response = self.llm_client.infer(messages, tools=None)  # No tools on synthesis
   llm_call_count += 1
   
   if synthesis_response["type"] != "text":
       raise RuntimeError("LLM requested tools again on synthesis step (synthesis loop detected)")
   
   return synthesis_response["content"]
   ```

**Error Handling**:
- `ValueError` from validation → propagate to HTTP server → 400 response
- `RuntimeError` from synthesis loop → propagate to HTTP server → 500 response
- `TimeoutError` from MCP client → propagate → 500 response
- Any other exception → propagate → 500 response

**LLM Call Count Invariant**: Exactly 2 calls on tool path (initial + synthesis), exactly 1 call on direct answer path.

---

### 4.2 MCP Client Startup

**Function**: `MCPClient.start() -> None`

**Steps**:
1. **Spawn Subprocess**:
   ```python
   self.process = subprocess.Popen(
       self.server_command.split(),
       stdin=subprocess.PIPE,
       stdout=subprocess.PIPE,
       stderr=subprocess.PIPE,
       cwd=self.working_dir,
       env=self.env,
       text=True,
       bufsize=1
   )
   ```

2. **Send Handshake Request**:
   ```json
   {
     "jsonrpc": "2.0",
     "id": 0,
     "method": "initialize",
     "params": {
       "protocolVersion": "2024-11-05",
       "capabilities": {}
     }
   }
   ```
   Write to `self.process.stdin`, flush.

3. **Read Handshake Response** (with 5s timeout):
   ```python
   import select
   ready, _, _ = select.select([self.process.stdout], [], [], 5)
   if not ready:
       raise RuntimeError("MCP handshake timed out after 5s")
   
   line = self.process.stdout.readline()
   response = json.loads(line)
   if "error" in response:
       raise RuntimeError(f"MCP handshake failed: {response['error']}")
   ```

4. **Fetch Tool List**:
   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "method": "tools/list",
     "params": {}
   }
   ```
   Write, flush, read response (with 5s timeout), parse `result["tools"]` array.

5. **Validate Four Tools Present** (AC4):
   ```python
   expected_tools = {"list_sales_order", "list_employees", "get_sales_order_by_id", "get_employee_by_id"}
   actual_tools = {tool["name"] for tool in tools}
   if not expected_tools.issubset(actual_tools):
       raise RuntimeError(f"Missing required tools: {expected_tools - actual_tools}")
   ```

6. **Store Tools**:
   ```python
   self.tools = self._convert_to_anthropic_format(tools)
   ```

**Error Handling**:
- Subprocess fails to start → raise exception (agent fails to start, logged in `run_server`)
- Handshake timeout → raise `RuntimeError`
- Tool validation fails → raise `RuntimeError`

---

### 4.3 MCP Tool Invocation with Timeout

**Function**: `MCPClient.call_tool(name: str, arguments: dict, timeout: int) -> dict`

**Steps**:
1. **Build JSON-RPC Request**:
   ```python
   request_id = self._next_id()  # Internal counter for request tracking
   request = {
       "jsonrpc": "2.0",
       "id": request_id,
       "method": "tools/call",
       "params": {"name": name, "arguments": arguments}
   }
   ```

2. **Write to Subprocess with Timeout**:
   ```python
   import select
   self.process.stdin.write(json.dumps(request) + "\n")
   self.process.stdin.flush()
   
   ready, _, _ = select.select([self.process.stdout], [], [], timeout)
   if not ready:
       raise TimeoutError(f"MCP tool call '{name}' timed out after {timeout}s")
   
   line = self.process.stdout.readline()
   response = json.loads(line)
   ```

3. **Parse Response**:
   ```python
   if "error" in response:
       raise RuntimeError(f"MCP tool '{name}' returned error: {response['error']}")
   return response["result"]
   ```

**Notes**:
- `select.select` ensures timeout enforcement on POSIX systems.
- Windows alternative: use `threading.Timer` to interrupt blocking read after timeout.
- Tool result format varies by tool; orchestrator expects dict or parseable JSON string.

---

## 5. Environment Variables

Document in `README.md` (updated as part of this story):

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | Yes | — | Anthropic API key for LLM client |
| `ANTHROPIC_BASE_URL` | No | `https://api.anthropic.com` | Anthropic API base URL (override for gateway) |
| `GRAPHQL_ENDPOINT` | Yes | — | GraphQL service URL for MCP subprocess (e.g., `http://localhost:4000/graphql`) |
| `MCP_SERVER_COMMAND` | No | `python -m mcp.server` | Command to start MCP server |
| `MCP_SERVER_WORKING_DIR` | Yes | — | Working directory for MCP subprocess (PER-6 repo path) |
| `FLASK_HOST` | No | `0.0.0.0` | HTTP server bind host |
| `FLASK_PORT` | No | `5000` | HTTP server bind port |
| `LOG_LEVEL` | No | `INFO` | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

---

## 6. Tests

### 6.1 Unit Tests (21 total)

**`tests/test_http_server.py`** (6 tests):
1. `test_query_endpoint_valid_request`: POST `/query` with valid JSON `{"query": "hello"}` → 200 + `{"answer": "..."}`
2. `test_query_endpoint_malformed_json`: POST `/query` with malformed JSON `{invalid}` → 400 + `{"error": "..."}`
3. `test_query_endpoint_missing_query_field`: POST `/query` with `{}` → 400 + `{"error": "missing 'query' field"}`
4. `test_query_endpoint_internal_error`: Mock orchestrator raises exception → 500 + `{"error": "internal error"}` + logged trace
5. `test_health_endpoint_mcp_alive`: GET `/health` with MCP subprocess alive → 200 + `{"status": "ok", "mcp_alive": true}`
6. `test_health_endpoint_mcp_crashed`: GET `/health` with MCP subprocess dead → 503 + `{"status": "degraded", "mcp_alive": false}`

**`tests/test_llm_client.py`** (3 tests):
1. `test_infer_direct_answer`: Mock Anthropic API returns text response → parsed as `{"type": "text", "content": "Hello!"}`
2. `test_infer_tool_call`: Mock Anthropic API returns tool use → parsed as `{"type": "tool_use", "calls": [{"id": "...", "name": "get_employee_by_id", "input": {"id": "emp-001"}}]}`
3. `test_infer_api_error`: Mock Anthropic API raises exception → exception propagated

**`tests/test_mcp_client.py`** (4 tests):
1. `test_start_successful_handshake`: Mock subprocess returns valid handshake response + tool list with 4 tools → `get_tools()` returns all four tool schemas
2. `test_start_handshake_failure`: Mock subprocess returns error in handshake → raises `RuntimeError("MCP handshake failed: ...")`
3. `test_shutdown_graceful`: `shutdown()` sends EOF to stdin, waits 5s, verifies subprocess terminated (no SIGKILL needed)
4. `test_call_tool_timeout`: Mock subprocess does not write response within 10s → raises `TimeoutError("MCP tool call '...' timed out after 10s")`

**`tests/test_orchestrator.py`** (8 tests):
1. `test_direct_answer_path`: Mock LLM returns `{"type": "text", "content": "Hello!"}` → orchestrator returns `"Hello!"`, no MCP call, 1 LLM call
2. `test_tool_call_success`: Mock LLM returns tool call `get_employee_by_id({"id": "emp-001"})`, mock MCP returns `{"name": "Alice", "id": "EMP-001"}`, mock LLM synthesis returns `"The name is Alice."` → orchestrator returns synthesized answer, 2 LLM calls, 1 MCP call
3. `test_tool_call_mcp_error`: Mock LLM returns tool call, mock MCP raises `RuntimeError("tool error")` → orchestrator propagates exception
4. `test_llm_call_count_limit`: Verify exactly 2 LLM calls on tool path (initial + synthesis), 1 call on direct path
5. `test_id_normalization`: Mock LLM returns tool call with `{"id": "emp-001"}` → verify MCP receives `{"id": "EMP-001"}` (normalized to uppercase)
6. `test_argument_validation_missing_field`: Mock LLM returns tool call missing required field `id` → raises `ValueError("Missing required field 'id' for tool 'get_employee_by_id'")`
7. `test_argument_validation_unknown_tool`: Mock LLM returns unknown tool name `"nonexistent_tool"` → raises `ValueError("LLM requested unknown tool: nonexistent_tool")`
8. `test_synthesis_loop_detection`: Mock LLM returns tool call on synthesis step (instead of text) → raises `RuntimeError("LLM requested tools again on synthesis step (synthesis loop detected)")`

**Total Unit Tests**: 6 + 3 + 4 + 8 = 21

---

### 6.2 Integration Test (1 test)

**`tests/test_integration.py`**:
1. `test_end_to_end_with_mocked_services`:
   - **Setup**: Create mocked LLM client (returns canned tool call for query containing `"emp-001"`, returns canned synthesis text `"The name is Alice."`), create mocked MCP client (returns `{"name": "Alice", "id": "EMP-001"}` for `get_employee_by_id`), start Flask app with these clients.
   - **Execute**: POST `/query` with body `{"query": "what is the name of employee with emp-001"}`.
   - **Assert**:
     - Response status 200
     - Response body `{"answer": "The name is Alice."}`
     - LLM client called exactly 2 times (initial + synthesis)
     - MCP client called exactly 1 time with `name="get_employee_by_id"` and `arguments={"id": "EMP-001"}` (normalized from `emp-001`)
   - **Teardown**: Stop Flask app.

**Notes**:
- Mocked services avoid dependency on running PER-5/PER-6.
- Real services tested via manual verification script.

---

### 6.3 Manual Verification Script (1 artifact)

**`tests/manual_verification.md`**:

**Purpose**: Document steps to verify agent against real PER-5 GraphQL service and PER-6 MCP server (satisfies AC6 manual verification requirement).

**Content Outline**:

#### Prerequisites
- PER-5 GraphQL service running at `http://localhost:4000/graphql`
- PER-6 MCP server repository cloned at `/path/to/per6/repo`
- PER-6 venv activated (contains `mcp` package)
- Anthropic API key obtained

#### Setup
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
export GRAPHQL_ENDPOINT="http://localhost:4000/graphql"
export MCP_SERVER_WORKING_DIR="/path/to/per6/repo"
export LOG_LEVEL="DEBUG"  # Optional: verbose logging

cd /home/goos/projects/adlc-agents
python -m app.agent.http_server
```

#### Test Cases

**Test 1: AC1/AC2 — Direct Answer (No Tool)**
```bash
curl -X POST http://localhost:5000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "hello"}'
```
**Expected Output**:
```json
{"answer": "Hello! How can I help you today?"}
```
*(Exact wording may vary; verify it's a greeting, not a tool call)*

---

**Test 2: AC3/AC5 — Tool Call with ID Normalization**
```bash
curl -X POST http://localhost:5000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "what is the name of employee with emp-001"}'
```
**Expected Output**:
```json
{"answer": "The employee's name is Alice."}
```
*(Verify answer contains name from GraphQL data; check logs for ID normalization `emp-001` → `EMP-001`)*

---

**Test 3: AC4 — List Employees Tool**
```bash
curl -X POST http://localhost:5000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "list all employees"}'
```
**Expected Output**:
```json
{"answer": "The employees are: Alice, Bob, Charlie."}
```
*(Verify answer lists employee names from GraphQL data)*

---

**Test 4: AC4 — Get Sales Order Tool**
```bash
curl -X POST http://localhost:5000/query \
  -H "Content-Type: application/json" \
  -d '{"query": "get sales order ord-001"}'
```
**Expected Output**:
```json
{"answer": "Order ORD-001: Total $1500, Customer: Acme Corp, Status: Shipped"}
```
*(Verify answer contains order details from GraphQL data)*

---

**Test 5: Health Check**
```bash
curl http://localhost:5000/health
```
**Expected Output**:
```json
{"status": "ok", "mcp_alive": true}
```

---

#### Troubleshooting

**MCP subprocess not starting**:
- Verify `MCP_SERVER_WORKING_DIR` points to PER-6 repo root
- Verify PER-6 venv is activated and `python -m mcp.server` runs manually
- Check agent logs for subprocess stderr output

**GraphQL connection refused**:
- Verify PER-5 service is running: `curl http://localhost:4000/graphql`
- Check `GRAPHQL_ENDPOINT` env var matches PER-5 service URL

**LLM API errors**:
- Verify `ANTHROPIC_API_KEY` is valid
- Check network access to `https://api.anthropic.com` (or custom `ANTHROPIC_BASE_URL`)
- Review agent logs for API error details

**Incorrect answers**:
- Enable `LOG_LEVEL=DEBUG` to see full LLM prompts/responses and tool calls
- Verify GraphQL mock data in PER-5 matches expected values
- Check MCP tool schemas match expected format

---

#### Expected Timing
- Direct answer queries (Test 1): < 2s
- Tool call queries (Tests 2–4): < 5s per query (LLM inference + MCP round-trip)

---

## 7. Dependencies

### 7.1 Python Packages

Add to `requirements.txt`:
```
flask==3.1.0
anthropic==0.18.0
```

*(Verify latest stable versions on PyPI: Flask 3.1.x and anthropic 0.18.x current as of 2025-01-21)*

**Notes**:
- No `mcp` package dependency in agent repo; MCP protocol implemented directly via subprocess and JSON-RPC.
- `pytest>=8.0.0` assumed present in test dependencies (standard for Python testing).

### 7.2 External Services

- **PER-5 GraphQL Service**: Must be reachable at `GRAPHQL_ENDPOINT` (user-provided, not part of this repo).
- **PER-6 MCP Server**: Runnable via `python -m mcp.server` from its venv (user-provided, PER-6 repo).

---

## 8. Constraints and Decisions

*(Inherited from HLD; no new decisions at LLD level)*

- **AD-1**: LLM tool-calling (not keyword routing) — enforced in orchestrator algorithm (§4.1 step 2–4).
- **AD-2**: MCP via stdio subprocess — enforced in MCP client implementation (§4.2 step 1).
- **AD-3**: ID normalization to uppercase — enforced in orchestrator tool call execution (§4.1 step 4: `v.upper() if k == "id"`).
- **AD-4**: Multi-step inference (max 2 LLM calls) — enforced in orchestrator (§4.1 step 6 synthesis loop detection).
- **AD-5**: Comprehensive error handling — enforced across all modules:
  - Validation errors (`ValueError`) → HTTP 400
  - Internal errors (LLM/MCP exceptions, synthesis loop `RuntimeError`, timeout) → HTTP 500
  - All exceptions logged with full trace

---

## 9. Implementation Details

### 9.1 Module Structure

**`app/agent/__init__.py`**:
```python
"""Agent package providing HTTP query interface with LLM and MCP tool integration."""
from .http_server import create_app, run_server
from .orchestrator import Orchestrator
from .llm_client import LLMClient
from .mcp_client import MCPClient

__all__ = ["create_app", "run_server", "Orchestrator", "LLMClient", "MCPClient"]
```

**`app/agent/http_server.py`**:
- Flask app factory pattern: `create_app(orchestrator)` returns
