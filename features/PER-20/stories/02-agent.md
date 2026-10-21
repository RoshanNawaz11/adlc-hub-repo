# LLD: Story 02-agent — Integrate LLM client for direct answers and tool routing

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 02-agent  
**After:** Story 01-agent (FastAPI app must exist)

---

## Acceptance Criteria

### AC-1: Direct LLM answers (no tool call)

**Given** the agent is running  
**When** a caller sends a question that does not need data (a greeting or a help request)  
**Then** the LLM answers directly and no MCP tool is called

### Partial AC-2: LLM requests tool, synthesis (actual HTTP call in Story 3)

**Given** `MCP_URL` points at the FastAPI and FastMCP server  
**When** the LLM requests employee or sales-order data  
**Then** the agent routes to a stubbed MCP client and returns a placeholder or error  
**And** Story 3 implements the actual FastMCP HTTP client

### Partial AC-3: Error on LLM unavailable (MCP error in Story 3)

**Given** the LLM or the gateway is down  
**When** a question is sent  
**Then** the request fails with `{"error": "LLM unavailable"}` and the process stays up

---

## File Plan

### New Files

- `agent/orchestrator.py` — Routes questions to LLM client; routes tool requests to stubbed MCP client
- `agent/llm_client.py` — LLM client wrapper (Anthropic or gateway); decides direct answer vs. tool call; synthesizes tool results
- `agent/mcp_stub.py` — Stubbed MCP client; returns `{"error": "MCP not implemented"}` until Story 3
- `tests/test_query_direct.py` — AC-1: direct answer without tool call
- `tests/test_query_tool_stub.py` — Partial AC-2: tool request routed to stub
- `tests/test_llm_unavailable.py` — Partial AC-3: LLM unavailable error

### Modified Files

- `agent/main.py` — Replace stubbed `/query` endpoint with orchestrator call

### No changes outside `agent` repo

---

## Interfaces

### `/query` Endpoint (Updated)

**Method:** `POST`  
**Path:** `/query`  
**Request Body:**
```json
{
  "question": "string"
}
```

**Response (success, direct answer):**
```json
{
  "answer": "Hello! How can I help you?"
}
```

**Response (success, tool routed to stub):**
```json
{
  "error": "MCP not implemented"
}
```

**Response (error, LLM unavailable):**
```json
{
  "error": "LLM unavailable"
}
```

**Behavior:**
- AC-1: Greeting/help question → LLM answers directly, no MCP call logged.
- Partial AC-2: Data question → LLM requests tool → orchestrator routes to `mcp_stub` → placeholder or error.
- Partial AC-3: LLM client fails → returns `{"error": "LLM unavailable"}`, process stays up.

---

### Orchestrator Interface (`agent/orchestrator.py`)

**Function:** `async def process_question(question: str) -> dict`

**Input:** `question` (string)

**Output:**
```json
{
  "answer": "string"
}
```
or
```json
{
  "error": "string"
}
```

**Behavior:**
1. Route `question` to LLM client.
2. If LLM returns direct answer, return `{"answer": "..."}`.
3. If LLM requests tool, route to stubbed MCP client (`mcp_stub`).
4. Pass tool result (or error) to LLM for synthesis.
5. Return synthesized answer or error.
6. On LLM unavailable, return `{"error": "LLM unavailable"}`.

---

### LLM Client Interface (`agent/llm_client.py`)

**Function:** `async def ask_llm(question: str, tool_result: dict | None = None) -> dict`

**Input:**
- `question` (string): user question
- `tool_result` (optional dict): result from MCP tool (for synthesis)

**Output:**
```json
{
  "type": "direct_answer" | "tool_request",
  "answer": "string",  // if type == "direct_answer"
  "tool_name": "string",  // if type == "tool_request"
  "tool_args": {}  // if type == "tool_request"
}
```

**Behavior:**
- If `tool_result` is provided, synthesize it into natural language answer and return `{"type": "direct_answer", "answer": "..."}`.
- If `question` is greeting/help, return direct answer.
- If `question` needs data, return tool request (`{"type": "tool_request", "tool_name": "list_sales_order", "tool_args": {}}`).
- On LLM client failure (API down, env unset), raise exception caught by orchestrator.

**Environment:**
- LLM credentials: existing env vars for Anthropic or gateway (e.g. `ANTHROPIC_API_KEY` or `LLM_GATEWAY_URL`)

---

### Stubbed MCP Client Interface (`agent/mcp_stub.py`)

**Function:** `async def call_tool(tool_name: str, tool_args: dict) -> dict`

**Input:**
- `tool_name` (string): `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`
- `tool_args` (dict): tool arguments (e.g. `{"id": "EMP-001"}`)

**Output:**
```json
{
  "error": "MCP not implemented"
}
```

**Behavior:**
- Always returns `{"error": "MCP not implemented"}` until Story 3 replaces this with actual FastMCP HTTP client.
- No HTTP calls; no stdio subprocess.

---

## Algorithm Notes

### `agent/main.py` (Modified)

**Changes:**
- Replace stubbed `/query` endpoint logic with call to `orchestrator.process_question(question)`.

**Pseudocode:**
```python
from fastapi import FastAPI, Request
from agent.health import check_health
from agent.orchestrator import process_question

app = FastAPI()

@app.post("/query")
async def query(request: Request):
    body = await request.json()
    question = body.get("question", "")
    result = await process_question(question)
    return result

@app.get("/health")
async def health():
    return check_health()
```

---

### `agent/orchestrator.py` (New)

**Responsibilities:**
- Route question to LLM client.
- Route tool request to stubbed MCP client.
- Pass tool result to LLM for synthesis.
- Handle LLM unavailable error.

**Pseudocode:**
```python
from agent.llm_client import ask_llm
from agent.mcp_stub import call_tool

async def process_question(question: str) -> dict:
    try:
        # Step 1: Ask LLM (no tool result yet)
        llm_response = await ask_llm(question)
        
        if llm_response["type"] == "direct_answer":
            # AC-1: Direct answer, no MCP call
            return {"answer": llm_response["answer"]}
        
        elif llm_response["type"] == "tool_request":
            # Partial AC-2: Route to stubbed MCP client
            tool_name = llm_response["tool_name"]
            tool_args = llm_response["tool_args"]
            
            tool_result = await call_tool(tool_name, tool_args)
            
            # Synthesize tool result (or error) into answer
            synthesized = await ask_llm(question, tool_result=tool_result)
            return {"answer": synthesized["answer"]}
    
    except Exception as e:
        # Partial AC-3: LLM unavailable
        return {"error": "LLM unavailable"}
```

**Notes:**
- Exception handling is broad here; refine in production (distinguish LLM errors from other failures).
- Tool result is always `{"error": "MCP not implemented"}` until Story 3.

---

### `agent/llm_client.py` (New)

**Responsibilities:**
- Call LLM API (Anthropic or gateway).
- Decide direct answer vs. tool request.
- Synthesize tool results into natural language.
- Async-wrap if existing LLM client is sync.

**Pseudocode:**
```python
import os
import asyncio
from anthropic import Anthropic  # Example; replace with actual client

_client = None

def _get_client():
    global _client
    if _client is None:
        api_key = os.getenv("ANTHROPIC_API_KEY")
        if not api_key:
            raise ValueError("LLM credentials not set (ANTHROPIC_API_KEY)")
        _client = Anthropic(api_key=api_key)
    return _client

async def ask_llm(question: str, tool_result: dict | None = None) -> dict:
    client = _get_client()
    
    if tool_result:
        # Synthesize tool result into answer
        prompt = f"User asked: {question}\nTool returned: {tool_result}\nSynthesize a natural language answer."
        # Call LLM in thread pool if sync client
        response = await asyncio.to_thread(_call_llm_sync, client, prompt)
        return {"type": "direct_answer", "answer": response}
    
    # Decide: direct answer or tool request
    # Simple heuristic for demo; real implementation uses LLM function calling
    if _is_greeting_or_help(question):
        # AC-1: Direct answer
        response = await asyncio.to_thread(_call_llm_sync, client, question)
        return {"type": "direct_answer", "answer": response}
    else:
        # Partial AC-2: Tool request
        tool_name, tool_args = _infer_tool(question)
        return {"type": "tool_request", "tool_name": tool_name, "tool_args": tool_args}

def _call_llm_sync(client, prompt: str) -> str:
    # Example sync call; replace with actual LLM client method
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

def _is_greeting_or_help(question: str) -> bool:
    # Simple heuristic; real implementation may use LLM
    greetings = ["hello", "hi", "help", "what can you do"]
    return any(g in question.lower() for g in greetings)

def _infer_tool(question: str) -> tuple[str, dict]:
    # Simple heuristic; real implementation uses LLM function calling
    if "sales order" in question.lower():
        return "list_sales_order", {}
    elif "employee" in question.lower():
        return "list_employees", {}
    else:
        return "list_sales_order", {}  # Default fallback
```

**Notes:**
- **Async wrapper:** `asyncio.to_thread` wraps sync LLM client calls.
- **Heuristics:** `_is_greeting_or_help` and `_infer_tool` are placeholders. Real implementation uses LLM function calling (e.g. Anthropic's tool use API) to decide tool vs. direct answer and parse tool arguments. This story preserves existing LLM choice logic; heuristics stand in for that.
- **Error handling:** `_get_client()` raises `ValueError` if env unset; caught by orchestrator as LLM unavailable.
- **Synthesis:** When `tool_result` is provided, LLM synthesizes it into natural language. Example: `tool_result = {"error": "MCP not implemented"}` → synthesized answer: "I'm sorry, the MCP service is not yet available."

---

### `agent/mcp_stub.py` (New)

**Responsibilities:**
- Stub MCP client; always returns `{"error": "MCP not implemented"}`.
- Replaced by actual FastMCP HTTP client in Story 3.

**Pseudocode:**
```python
async def call_tool(tool_name: str, tool_args: dict) -> dict:
    # Stub: no actual MCP call
    return {"error": "MCP not implemented"}
```

**Notes:**
- No HTTP calls, no stdio subprocess.
- Tool name and args are ignored; Story 3 implements actual tool logic.

---

## Tests

### `tests/test_query_direct.py`

**Coverage:** AC-1 (direct LLM answers when no data needed)

**Test 1: Direct answer for greeting**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "hello"}`  
**Then:** LLM answers directly, no MCP call logged, returns `{"answer": "Hello! How can I help you?"}`

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_direct_answer():
    client = TestClient(app)
    
    # Mock LLM client to return direct answer
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm:
        mock_llm.return_value = {
            "type": "direct_answer",
            "answer": "Hello! How can I help you?"
        }
        
        response = client.post("/query", json={"question": "hello"})
        
        assert response.status_code == 200
        assert response.json() == {"answer": "Hello! How can I help you?"}
        
        # Verify LLM called once, no tool result
        mock_llm.assert_called_once()
        args = mock_llm.call_args
        assert args[0][0] == "hello"  # question
        assert args[1].get("tool_result") is None  # no tool result
```

**Notes:**
- Mock `ask_llm` to verify orchestrator calls it correctly.
- No actual LLM API call in test; mock response.

---

### `tests/test_query_tool_stub.py`

**Coverage:** Partial AC-2 (tool request routed to stub)

**Test 1: Tool request routed to stubbed MCP client**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "list sales orders"}`  
**Then:** LLM requests tool, orchestrator routes to stubbed MCP client, returns synthesized error

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_tool_stub():
    client = TestClient(app)
    
    # Mock LLM to return tool request, then synthesize stub error
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_stub.call_tool", new_callable=AsyncMock) as mock_mcp:
        
        # First call: LLM requests tool
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "list_sales_order", "tool_args": {}},
            {"type": "direct_answer", "answer": "I'm sorry, the MCP service is not yet available."}
        ]
        
        # Stub MCP client returns error
        mock_mcp.return_value = {"error": "MCP not implemented"}
        
        response = client.post("/query", json={"question": "list sales orders"})
        
        assert response.status_code == 200
        assert "MCP" in response.json()["answer"]  # Synthesized error message
        
        # Verify LLM called twice (tool request, then synthesis)
        assert mock_llm.call_count == 2
        
        # Verify MCP stub called once
        mock_mcp.assert_called_once_with("list_sales_order", {})
```

**Notes:**
- `ask_llm` called twice: first to decide tool request, second to synthesize stub error.
- Synthesized answer contains "MCP" to indicate error context.

---

### `tests/test_llm_unavailable.py`

**Coverage:** Partial AC-3 (error on LLM unavailable)

**Test 1: LLM unavailable error**

**Given:** LLM client fails (API down or env unset)  
**When:** `POST /query` with any question  
**Then:** Returns `{"error": "LLM unavailable"}`, process stays up

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_llm_unavailable():
    client = TestClient(app)
    
    # Mock LLM to raise exception
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm:
        mock_llm.side_effect = Exception("LLM API down")
        
        response = client.post("/query", json={"question": "hello"})
        
        assert response.status_code == 200
        assert response.json() == {"error": "LLM unavailable"}
```

**Notes:**
- Exception in `ask_llm` caught by orchestrator.
- Process stays up (no crash).

---

## Verification Summary

**Total test count:** 3 tests (per breakdown Verification Approach: 3 tests for Story 2)

| Test | File | Coverage |
|---|---|---|
| Direct answer for greeting | `tests/test_query_direct.py` | AC-1 (direct LLM answer, no MCP call) |
| Tool request routed to stub | `tests/test_query_tool_stub.py` | Partial AC-2 (LLM requests tool, stub returns error) |
| LLM unavailable error | `tests/test_llm_unavailable.py` | Partial AC-3 (LLM failure, process stays up) |

**Run tests:**
```bash
pytest tests/test_query_direct.py tests/test_query_tool_stub.py tests/test_llm_unavailable.py -v
```

---

## Risks

### R-4: LLM decision divergence

**Likelihood:** Low  
**Impact:** Medium (unexpected tool calls or direct answers)  
**Mitigation:**
- Constraint: LLM choice logic unchanged (preserve existing behavior).
- Test coverage: `test_query_direct.py` (direct answer path), `test_query_tool_stub.py` (tool request path).
- Heuristics in `llm_client.py` are placeholders; real implementation uses LLM function calling (e.g. Anthropic tool use API). This story does not change that decision logic — it integrates the existing client.

**Decision:** Use simple heuristics (`_is_greeting_or_help`, `_infer_tool`) as placeholders. Replace with actual LLM function calling in production or if existing client already has it.

---

## Notes

- **LLM client async wrapper:** `asyncio.to_thread` wraps sync LLM client calls (e.g. Anthropic SDK). If existing client is already async, remove wrapper.
- **Heuristics vs. function calling:** `_is_greeting_or_help` and `_infer_tool` are demo placeholders. Real implementation uses LLM's tool use API (e.g. Anthropic's function calling) to decide direct answer vs. tool request and parse arguments. This story preserves that existing logic; if it doesn't exist yet, heuristics suffice for Story 2 and are replaced in Story 3 or post-epic.
- **Stubbed MCP client:** `mcp_stub.py` always returns `{"error": "MCP not implemented"}`. Story 3 replaces it with actual FastMCP HTTP client.
- **Environment:** `ANTHROPIC_API_KEY` (or equivalent LLM gateway env var) must be set. If unset, `_get_client()` raises `ValueError`, caught as `{"error": "LLM unavailable"}`.
- **No Flask or stdio logic:** None added in this story. Story 1 already migrated to FastAPI; Story 3 removes any remaining Flask/stdio artifacts.
- **Synthesis:** When `ask_llm` is called with `tool_result`, it synthesizes the result into natural language. Example: stub error `{"error": "MCP not implemented"}` → synthesized answer: "I'm sorry, the MCP service is not yet available."

---

## Open Questions

_None. LLM client integration, orchestrator routing, stub MCP client, and test coverage are fully specified._
