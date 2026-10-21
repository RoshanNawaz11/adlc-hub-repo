# LLD: Story 03-agent — Implement FastMCP HTTP client and integrate tool calls

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 03-agent  
**After:** Story 02-agent (LLM client and orchestrator must exist)

---

## Acceptance Criteria

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

### AC-4: Health check (MCP reachable when up)

**Given** the agent process is started  
**When** an operator checks health on the FastAPI app  
**Then** they can see the process is up and whether the MCP HTTP endpoint is reachable, without calling a tool

---

## File Plan

### New Files

- `agent/mcp_client.py` — FastMCP HTTP client; calls four MCP tools over HTTP at `MCP_URL`
- `tests/test_tool_call.py` — AC-2: tool call over HTTP with successful result
- `tests/test_id_normalization.py` — AC-2: ID normalization (`emp-001` → `EMP-001`)
- `tests/test_mcp_unreachable.py` — AC-3: MCP server unreachable
- `tests/test_tool_error.py` — AC-3: tool returns `{"error": "..."}`
- `tests/test_health_mcp_up.py` — AC-4: health endpoint when MCP server up

### Modified Files

- `agent/orchestrator.py` — Replace `mcp_stub.call_tool` with `mcp_client.call_tool`
- `agent/health.py` — Update `_probe_mcp` to use same connectivity check as `mcp_client` (if needed)
- `requirements.txt` — Ensure `httpx` is listed (added in Story 1; verify presence)

### Removed Files

- `agent/mcp_stub.py` — Replaced by `mcp_client.py`
- Any Flask app entry point (e.g. `agent/flask_app.py`, if still present) — remove at this stage
- Any stdio MCP subprocess logic (e.g. `agent/stdio_mcp.py`, if still present) — remove at this stage

### No changes outside `agent` repo

---

## Interfaces

### FastMCP HTTP Client Interface (`agent/mcp_client.py`)

**Function:** `async def call_tool(tool_name: str, tool_args: dict) -> dict`

**Input:**
- `tool_name` (string): One of `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`
- `tool_args` (dict): Tool arguments (e.g. `{"id": "emp-001"}`)

**Output:**
```json
{
  "result": <tool result data>
}
```
or
```json
{
  "error": "string"
}
```

**Behavior:**
- Normalize `id` argument if present: `emp-001` → `EMP-001` (case-insensitive prefix, uppercase).
- Call MCP tool over FastMCP HTTP at `MCP_URL` (e.g. `http://127.0.0.1:8000/sse`).
- On success, return tool result wrapped in `{"result": ...}`.
- On tool error (MCP returns `{"error": "..."}`), return `{"error": "..."}`.
- On MCP server unreachable (connection error, timeout), return `{"error": "MCP server unreachable"}`.
- Do NOT fall back to stdio subprocess (AC-3).

**Environment:**
- `MCP_URL` (required): FastMCP SSE URL (e.g. `http://127.0.0.1:8000/sse`)

**Transport:**
- FastMCP HTTP (SSE) over `httpx` async client
- Timeout: 10 seconds (fail fast if MCP server slow or down)

---

### Updated Orchestrator Interface (`agent/orchestrator.py`)

**Changes:**
- Replace `from agent.mcp_stub import call_tool` with `from agent.mcp_client import call_tool`
- No other logic changes; orchestrator already routes tool requests to `call_tool` and passes result to LLM for synthesis

**Pseudocode diff:**
```diff
- from agent.mcp_stub import call_tool
+ from agent.mcp_client import call_tool
```

---

### Updated Health Endpoint Interface (`agent/health.py`)

**Changes:**
- Ensure `_probe_mcp` uses same connectivity check as `mcp_client` (HEAD request or similar)
- If already compatible (Story 1 implementation), no changes needed

**Behavior (unchanged from Story 1):**
- `GET /health` → `{"status": "up", "mcp_reachable": true}` when MCP server up
- `mcp_reachable: false` when `MCP_URL` unset or MCP server unreachable

---

## Algorithm Notes

### `agent/mcp_client.py` (New)

**Responsibilities:**
- Call four MCP tools over FastMCP HTTP at `MCP_URL`.
- Normalize `id` arguments before HTTP call.
- Handle connection errors (MCP unreachable) and tool errors (MCP returns `{"error": "..."}`).
- No stdio fallback.

**Pseudocode:**
```python
import os
import httpx
import re

_MCP_URL = os.getenv("MCP_URL", "")

async def call_tool(tool_name: str, tool_args: dict) -> dict:
    """
    Call MCP tool over FastMCP HTTP.
    
    Args:
        tool_name: One of list_sales_order, list_employees, 
                   get_sales_order_by_id, get_employee_by_id
        tool_args: Tool arguments (e.g. {"id": "emp-001"})
    
    Returns:
        {"result": <data>} on success
        {"error": "..."} on failure
    """
    if not _MCP_URL:
        return {"error": "MCP_URL not configured"}
    
    # Normalize ID argument if present (AC-2: emp-001 → EMP-001)
    normalized_args = _normalize_id(tool_args)
    
    # Build FastMCP HTTP request payload
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
            "name": tool_name,
            "arguments": normalized_args
        },
        "id": 1
    }
    
    try:
        async with httpx.AsyncClient(timeout=10.0) as client:
            response = await client.post(_MCP_URL, json=payload)
            response.raise_for_status()
            
            data = response.json()
            
            # FastMCP response structure: {"result": {...}} or {"error": {...}}
            if "error" in data:
                return {"error": data["error"].get("message", "Tool error")}
            
            return {"result": data.get("result", {})}
    
    except httpx.ConnectError:
        return {"error": "MCP server unreachable"}
    except httpx.TimeoutException:
        return {"error": "MCP server timeout"}
    except httpx.HTTPStatusError as e:
        return {"error": f"MCP server error: {e.response.status_code}"}
    except Exception as e:
        return {"error": f"MCP client error: {str(e)}"}

def _normalize_id(args: dict) -> dict:
    """
    Normalize 'id' argument: emp-001 → EMP-001, so-123 → SO-123.
    
    Preserves existing logic: case-insensitive prefix, uppercase result.
    """
    if "id" not in args:
        return args
    
    id_value = args["id"]
    
    # Pattern: <prefix>-<digits> where prefix is case-insensitive
    # Examples: emp-001, EMP-001, so-123, SO-123
    match = re.match(r'^([a-z]+)-(\d+)$', id_value, re.IGNORECASE)
    if match:
        prefix, digits = match.groups()
        normalized_id = f"{prefix.upper()}-{digits}"
        return {**args, "id": normalized_id}
    
    # No match: return as-is (e.g. already uppercase, or different format)
    return args
```

**Notes:**
- **FastMCP payload structure:** JSON-RPC 2.0 format assumed per PER-18 MCP server contract. Adjust if actual PER-18 implementation differs (verify during integration test).
- **ID normalization:** Regex pattern `^([a-z]+)-(\d+)$` matches `emp-001`, `EMP-001`, `so-123`, `SO-123`. Uppercases prefix. Preserves existing logic (exception criterion).
- **Error handling:** Distinguishes connection errors (unreachable), timeouts, HTTP errors (5xx), and tool errors (MCP returns `{"error": "..."}`). All return `{"error": "..."}` to orchestrator.
- **Timeout:** 10 seconds (fail fast per AC-3).
- **No stdio fallback:** No subprocess logic anywhere in this file.

---

### `agent/orchestrator.py` (Modified)

**Changes:**
- Replace import: `from agent.mcp_stub import call_tool` → `from agent.mcp_client import call_tool`
- No other logic changes

**Updated Pseudocode:**
```python
from agent.llm_client import ask_llm
from agent.mcp_client import call_tool  # CHANGED: was mcp_stub

async def process_question(question: str) -> dict:
    try:
        # Step 1: Ask LLM (no tool result yet)
        llm_response = await ask_llm(question)
        
        if llm_response["type"] == "direct_answer":
            # AC-1: Direct answer, no MCP call
            return {"answer": llm_response["answer"]}
        
        elif llm_response["type"] == "tool_request":
            # AC-2: Route to FastMCP HTTP client
            tool_name = llm_response["tool_name"]
            tool_args = llm_response["tool_args"]
            
            tool_result = await call_tool(tool_name, tool_args)
            
            # AC-3: If tool_result has error, LLM synthesizes error message
            # (or return error directly; design choice)
            synthesized = await ask_llm(question, tool_result=tool_result)
            return {"answer": synthesized["answer"]}
    
    except Exception as e:
        # AC-3: LLM unavailable or unexpected error
        return {"error": "LLM unavailable"}
```

**Notes:**
- Tool result may be `{"result": {...}}` or `{"error": "..."}`. LLM client (`ask_llm`) handles both in synthesis.
- If tool returns `{"error": "..."}`, synthesized answer might be "I'm sorry, I couldn't retrieve that data. The system reported: <error>."

---

### `agent/health.py` (Modified or unchanged)

**Changes (if needed):**
- If Story 1 `_probe_mcp` already uses HEAD request to `MCP_URL`, no changes needed.
- If Story 1 used socket connect, optionally align with `mcp_client.py` approach (HEAD request).

**Verification:**
- Story 1 implementation used `httpx.Client(timeout=2.0).head(MCP_URL)`.
- `mcp_client.py` uses `httpx.AsyncClient(timeout=10.0).post(MCP_URL, ...)`.
- Health probe timeout (2s) is faster than tool call timeout (10s); this is intentional (fail fast on health check).
- No changes needed unless Story 1 implementation differs.

**Pseudocode (unchanged from Story 1, for reference):**
```python
import os
import httpx
from datetime import datetime, timedelta

_probe_cache = {"reachable": False, "timestamp": None}
_CACHE_TTL_SECONDS = 10

def check_health() -> dict:
    mcp_url = os.getenv("MCP_URL", "")
    
    if not mcp_url:
        return {"status": "up", "mcp_reachable": False}
    
    # Check cache
    now = datetime.now()
    if _probe_cache["timestamp"] and (now - _probe_cache["timestamp"]) < timedelta(seconds=_CACHE_TTL_SECONDS):
        return {"status": "up", "mcp_reachable": _probe_cache["reachable"]}
    
    # Probe MCP endpoint (lightweight HEAD request)
    reachable = _probe_mcp(mcp_url)
    _probe_cache["reachable"] = reachable
    _probe_cache["timestamp"] = now
    
    return {"status": "up", "mcp_reachable": reachable}

def _probe_mcp(url: str) -> bool:
    try:
        with httpx.Client(timeout=2.0) as client:
            response = client.head(url)
            return response.status_code < 500
    except Exception:
        return False
```

---

## Tests

### `tests/test_tool_call.py`

**Coverage:** AC-2 (tool call over HTTP with successful result)

**Test 1: Successful tool call over HTTP**

**Given:** `MCP_URL` is set and MCP server is up (mocked)  
**When:** `POST /query` with `{"question": "list sales orders"}`  
**Then:** LLM requests `list_sales_order` → FastMCP HTTP client calls `MCP_URL` → MCP returns sales order list → LLM synthesizes answer → agent returns `{"answer": "Here are the sales orders: ..."}`

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock
import httpx

def test_tool_call_success(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    # Mock httpx POST to return successful tool result
    mock_response = httpx.Response(
        status_code=200,
        json={
            "jsonrpc": "2.0",
            "result": [{"id": "SO-001", "amount": 100}],
            "id": 1
        }
    )
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("httpx.AsyncClient.post", new_callable=AsyncMock) as mock_http:
        
        # LLM requests tool, then synthesizes result
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "list_sales_order", "tool_args": {}},
            {"type": "direct_answer", "answer": "Here are the sales orders: SO-001 ($100)"}
        ]
        
        mock_http.return_value = mock_response
        
        response = client.post("/query", json={"question": "list sales orders"})
        
        assert response.status_code == 200
        assert "sales orders" in response.json()["answer"].lower()
        
        # Verify HTTP POST called with correct payload
        mock_http.assert_called_once()
        call_args = mock_http.call_args
        assert call_args[0][0] == "http://127.0.0.1:8000/sse"
        payload = call_args[1]["json"]
        assert payload["method"] == "tools/call"
        assert payload["params"]["name"] == "list_sales_order"
```

**Notes:**
- Mock `httpx.AsyncClient.post` to return FastMCP success response.
- Verify payload structure matches JSON-RPC 2.0 format.

---

### `tests/test_id_normalization.py`

**Coverage:** AC-2 (ID normalization: `emp-001` → `EMP-001`)

**Test 1: ID normalization before HTTP call**

**Given:** `MCP_URL` is set and MCP server is up (mocked)  
**When:** `POST /query` with `{"question": "get employee emp-001"}`  
**Then:** LLM requests `get_employee_by_id(id="emp-001")` → FastMCP HTTP client normalizes to `EMP-001` → HTTP call includes `{"id": "EMP-001"}` → correct employee returned

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock
import httpx

def test_id_normalization(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    # Mock httpx POST to return employee data
    mock_response = httpx.Response(
        status_code=200,
        json={
            "jsonrpc": "2.0",
            "result": {"id": "EMP-001", "name": "Alice"},
            "id": 1
        }
    )
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("httpx.AsyncClient.post", new_callable=AsyncMock) as mock_http:
        
        # LLM requests tool with lowercase id
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_employee_by_id", "tool_args": {"id": "emp-001"}},
            {"type": "direct_answer", "answer": "Employee EMP-001 is Alice"}
        ]
        
        mock_http.return_value = mock_response
        
        response = client.post("/query", json={"question": "get employee emp-001"})
        
        assert response.status_code == 200
        assert "Alice" in response.json()["answer"]
        
        # Verify ID normalized to EMP-001 in HTTP payload
        payload = mock_http.call_args[1]["json"]
        assert payload["params"]["arguments"]["id"] == "EMP-001"
```

**Notes:**
- Verifies `_normalize_id` logic works before HTTP call.
- Exception criterion: preserves existing normalization behavior.

---

### `tests/test_mcp_unreachable.py`

**Coverage:** AC-3 (MCP server unreachable)

**Test 1: MCP server unreachable**

**Given:** `MCP_URL` is set to unreachable endpoint or httpx raises `ConnectError`  
**When:** `POST /query` with data question  
**Then:** Returns `{"answer": "...MCP server unreachable..."}` (synthesized error), process stays up, no stdio subprocess

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock
import httpx

def test_mcp_unreachable(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:9999/sse")
    
    client = TestClient(app)
    
    # Mock httpx POST to raise ConnectError
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("httpx.AsyncClient.post", new_callable=AsyncMock) as mock_http:
        
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "list_sales_order", "tool_args": {}},
            {"type": "direct_answer", "answer": "I'm sorry, the MCP server is unreachable."}
        ]
        
        mock_http.side_effect = httpx.ConnectError("Connection refused")
        
        response = client.post("/query", json={"question": "list sales orders"})
        
        assert response.status_code == 200
        assert "unreachable" in response.json()["answer"].lower()
        
        # Verify no subprocess spawned (implicit: no stdio logic exists)
```

**Notes:**
- `mcp_client.call_tool` returns `{"error": "MCP server unreachable"}`.
- LLM synthesizes error message.
- Process stays up (no crash).

---

### `tests/test_tool_error.py`

**Coverage:** AC-3 (tool returns `{"error": "..."}`)

**Test 1: Tool returns error**

**Given:** MCP server returns `{"error": {"message": "Employee not found"}}`  
**When:** `POST /query` with `{"question": "get employee emp-999"}`  
**Then:** Returns `{"answer": "...Employee not found..."}` (synthesized error), process stays up

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock
import httpx

def test_tool_error(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    # Mock httpx POST to return tool error
    mock_response = httpx.Response(
        status_code=200,
        json={
            "jsonrpc": "2.0",
            "error": {"message": "Employee not found"},
            "id": 1
        }
    )
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("httpx.AsyncClient.post", new_callable=AsyncMock) as mock_http:
        
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_employee_by_id", "tool_args": {"id": "EMP-999"}},
            {"type": "direct_answer", "answer": "I'm sorry, the employee was not found."}
        ]
        
        mock_http.return_value = mock_response
        
        response = client.post("/query", json={"question": "get employee emp-999"})
        
        assert response.status_code == 200
        assert "not found" in response.json()["answer"].lower()
```

**Notes:**
- `mcp_client.call_tool` returns `{"error": "Employee not found"}`.
- LLM synthesizes error into natural language.

---

### `tests/test_health_mcp_up.py`

**Coverage:** AC-4 (health endpoint when MCP server up)

**Test 1: Health endpoint reports MCP reachable when up**

**Given:** `MCP_URL` is set and MCP server is up (mocked)  
**When:** `GET /health`  
**Then:** Returns `{"status": "up", "mcp_reachable": true}`

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch
import httpx

def test_health_mcp_up(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    # Mock httpx HEAD to return 200 OK
    with patch("httpx.Client.head") as mock_head:
        mock_head.return_value = httpx.Response(status_code=200)
        
        response = client.get("/health")
        
        assert response.status_code == 200
        assert response.json() == {"status": "up", "mcp_reachable": True}
```

**Notes:**
- Verifies health endpoint correctly reports MCP reachability when server is up.
- Story 1 already tested `mcp_reachable: false` when unreachable; this test covers the `true` case.

---

## Verification Summary

**Total test count:** 5 tests (per breakdown Verification Approach)

| Test | File | Coverage |
|---|---|---|
| Successful tool call over HTTP | `tests/test_tool_call.py` | AC-2 (tool call over HTTP, synthesized answer) |
| ID normalization before HTTP call | `tests/test_id_normalization.py` | AC-2 (ID normalization: `emp-001` → `EMP-001`) |
| MCP server unreachable | `tests/test_mcp_unreachable.py` | AC-3 (MCP unreachable, clear error, no stdio) |
| Tool returns error | `tests/test_tool_error.py` | AC-3 (tool error, synthesized error, process stays up) |
| Health endpoint when MCP up | `tests/test_health_mcp_up.py` | AC-4 (health reports `mcp_reachable: true`) |

**Run tests:**
```bash
pytest tests/test_tool_call.py tests/test_id_normalization.py tests/test_mcp_unreachable.py tests/test_tool_error.py tests/test_health_mcp_up.py -v
```

---

## Risks

### R-1: MCP server unavailable

**Likelihood:** Medium  
**Impact:** High (agent cannot answer data questions)  
**Mitigation:**
- AC-3: clear error returned, no stdio fallback.
- AC-4: health endpoint surfaces MCP connectivity proactively.
- Test coverage: `test_mcp_unreachable.py`.
- Operational monitoring: alert on `mcp_reachable: false` in health endpoint.

### R-2: FastMCP HTTP transport mismatch

**Likelihood:** Low  
**Impact:** High (integration breaks)  
**Mitigation:**
- Assume JSON-RPC 2.0 format per PER-18 MCP server contract.
- Test coverage: `test_tool_call.py` validates payload structure.
- Integration test (verification item 1) against real MCP server (or mock matching PER-18 spec).
- If PER-18 uses different format (e.g. REST instead of JSON-RPC), update `mcp_client.py` payload structure.

**Decision:** Use JSON-RPC 2.0 format (`{"jsonrpc": "2.0", "method": "tools/call", "params": {...}, "id": 1}`) as placeholder. Verify against PER-18 implementation during integration test.

### R-3: ID normalization regression

**Likelihood:** Low  
**Impact:** Medium (wrong records returned)  
**Mitigation:**
- Exception criterion: preserve existing normalization (`emp-001` → `EMP-001`).
- Test coverage: `test_id_normalization.py`.
- Regex pattern `^([a-z]+)-(\d+)$` handles case-insensitive prefix; adjust if existing logic differs.

---

## Notes

- **FastMCP payload structure:** JSON-RPC 2.0 assumed per typical MCP/FastMCP conventions. Verify against PER-18 implementation. If PER-18 uses REST (`POST /tools/list_sales_order`), update `mcp_client.py` to match.
- **ID normalization:** `_normalize_id` uses regex to uppercase prefix. Handles `emp-001`, `EMP-001`, `so-123`, `SO-123`. Preserves existing logic (exception criterion).
- **Error handling:** Distinguishes connection errors (unreachable), timeouts, HTTP errors (5xx), and tool errors (MCP returns `{"error": "..."}`). All return `{"error": "..."}` to orchestrator; LLM synthesizes error message.
- **Timeout:** 10 seconds for tool calls (fail fast per AC-3). Health probe timeout remains 2 seconds (from Story 1).
- **No stdio fallback:** No subprocess logic in `mcp_client.py` or anywhere else. AC-3 explicitly prohibits stdio fallback.
- **Health probe:** Story 1 implementation uses HEAD request with 2s timeout. `mcp_client.py` uses POST with 10s timeout. Both acceptable; health probe is lightweight and fast-fail.
- **Flask and stdio removal:** Remove `agent/flask_app.py` and `agent/stdio_mcp.py` (if present) at this stage. Verify absence in Story 4 README.
- **`mcp_stub.py` removal:** Delete after updating orchestrator import. No longer needed.
- **Environment:** `MCP_URL` must be set (e.g. `http://127.0.0.1:8000/sse`). If unset, `mcp_client.call_tool` returns `{"error": "MCP_URL not configured"}`.

---

## Open Questions

_None. FastMCP HTTP client implementation, ID normalization, error handling, orchestrator integration, health endpoint update, and test coverage are fully specified._
