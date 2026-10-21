# LLD: Story 01-agent — Migrate agent from Flask to FastAPI and add health endpoint

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 01-agent  
**After:** None (first story; opens epic)

---

## Acceptance Criteria

### AC-4 (health endpoint reports process up and MCP reachable)

**Given** the agent process is started  
**When** an operator checks health on the FastAPI app  
**Then** they can see the process is up and whether the MCP HTTP endpoint is reachable, without calling a tool

### AC-5 (FastAPI is primary path)

**Given** the `adlc-agents` README  
**When** a developer follows setup  
**Then** they run the agent with FastAPI, set `MCP_URL` to the FastMCP SSE URL, set the LLM env, and `POST` a question  
**And** HTTP MCP is the documented primary path

### Partial AC-3 (process stays up on error)

**Given** a tool returns `{"error": "..."}` or the MCP server is unreachable  
**When** the question needed that tool  
**Then** the agent returns a clear error and stays up

---

## File Plan

### New Files

- `agent/main.py` — FastAPI app entry point; defines `/query` and `/health` endpoints
- `agent/health.py` — Health check logic; probes `MCP_URL` for reachability
- `tests/test_health.py` — Tests for `/health` endpoint (AC-4)
- `tests/test_query_stub.py` — Tests for stubbed `/query` endpoint (process stays up, placeholder response)
- `requirements.txt` — Dependencies: `fastapi`, `uvicorn`, `httpx`

### Modified Files

_None in this story. README update deferred to Story 4._

### Removed Files

_Flask app and stdio MCP logic removal deferred to Story 3 (or implicit here if no existing code found)._

---

## Interfaces

### `/query` Endpoint (Stubbed)

**Method:** `POST`  
**Path:** `/query`  
**Request Body:**
```json
{
  "question": "string"
}
```

**Response (success, placeholder):**
```json
{
  "answer": "Placeholder response. LLM integration pending Story 2."
}
```

**Response (error, stub):**
```json
{
  "error": "LLM not yet integrated"
}
```

**Behavior:**
- Accepts any `question` string.
- Returns placeholder answer until LLM client integrated in Story 2.
- Process stays up after each request (AC-3 partial).

---

### `/health` Endpoint

**Method:** `GET`  
**Path:** `/health`  
**Response:**
```json
{
  "status": "up",
  "mcp_reachable": true
}
```

**Behavior:**
- `status` is always `"up"` if the process is responding.
- `mcp_reachable`:
  - `true` when `MCP_URL` is set and a lightweight probe (HEAD request or socket connect) succeeds.
  - `false` when `MCP_URL` is unset, empty, or probe fails.
- Probe does NOT call MCP tools; uses HEAD request to `MCP_URL` or attempts socket connection.
- Probe result may be cached (TTL: 5–10 seconds) to avoid latency on every health check.

**Environment:**
- `MCP_URL` (optional): FastMCP SSE URL (e.g. `http://127.0.0.1:8000/sse`). If unset, `mcp_reachable: false`.

---

## Algorithm Notes

### `agent/main.py`

**Responsibilities:**
- Initialize FastAPI app.
- Define `/query` endpoint (stubbed response).
- Define `/health` endpoint (delegates to `agent/health.py`).

**Pseudocode:**
```python
from fastapi import FastAPI, Request
from agent.health import check_health

app = FastAPI()

@app.post("/query")
async def query(request: Request):
    body = await request.json()
    question = body.get("question", "")
    # Stub: return placeholder until LLM integrated (Story 2)
    return {"answer": "Placeholder response. LLM integration pending Story 2."}

@app.get("/health")
async def health():
    return check_health()
```

**Notes:**
- No LLM or MCP client logic in this story.
- Error handling: FastAPI default exception handlers suffice; process stays up on request errors.

---

### `agent/health.py`

**Responsibilities:**
- Probe `MCP_URL` for reachability.
- Return `{"status": "up", "mcp_reachable": bool}`.

**Pseudocode:**
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
        # Async HEAD request with short timeout
        with httpx.Client(timeout=2.0) as client:
            response = client.head(url)
            return response.status_code < 500  # 2xx, 3xx, 4xx acceptable; 5xx not
    except Exception:
        return False
```

**Notes:**
- Cache prevents health check latency on every call.
- HEAD request chosen over socket connect for HTTP-level verification.
- Timeout: 2 seconds (fast-fail if MCP server down).
- `httpx` used for consistency with Story 3 FastMCP HTTP client.

---

### `requirements.txt`

**New dependencies:**
```
fastapi
uvicorn[standard]
httpx
pytest
pytest-asyncio
```

**Rationale:**
- `fastapi`: web framework (AC-5).
- `uvicorn[standard]`: ASGI server with HTTP/2 and WebSocket support.
- `httpx`: async HTTP client for health probe and future MCP calls (Story 3).
- `pytest`, `pytest-asyncio`: test framework.

---

## Tests

### `tests/test_health.py`

**Coverage:** AC-4 (health endpoint reports process up and MCP reachable)

**Test 1: Health endpoint when `MCP_URL` unset**

**Given:** `MCP_URL` environment variable is unset or empty  
**When:** `GET /health` is called  
**Then:** Response is `{"status": "up", "mcp_reachable": false}`

**Pseudocode:**
```python
import os
from fastapi.testclient import TestClient
from agent.main import app

def test_health_mcp_url_unset():
    os.environ.pop("MCP_URL", None)
    client = TestClient(app)
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "up", "mcp_reachable": False}
```

---

**Test 2: Health endpoint when `MCP_URL` unreachable**

**Given:** `MCP_URL` is set to an unreachable endpoint (e.g. `http://127.0.0.1:9999/sse`)  
**When:** `GET /health` is called  
**Then:** Response is `{"status": "up", "mcp_reachable": false}`

**Pseudocode:**
```python
import os
from fastapi.testclient import TestClient
from agent.main import app

def test_health_mcp_unreachable(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:9999/sse")
    client = TestClient(app)
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json() == {"status": "up", "mcp_reachable": False}
```

**Notes:**
- No actual MCP server needed; health probe fails fast (2s timeout).
- Cache cleared between tests (or use monkeypatch to reset `_probe_cache`).

---

### `tests/test_query_stub.py`

**Coverage:** Partial AC-3 (process stays up on error); stubbed `/query` endpoint

**Test 1: Query endpoint returns placeholder response**

**Given:** The agent is running (FastAPI app)  
**When:** `POST /query` is called with `{"question": "hello"}`  
**Then:** Response is `{"answer": "Placeholder response. LLM integration pending Story 2."}` and process stays up

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app

def test_query_stub_response():
    client = TestClient(app)
    response = client.post("/query", json={"question": "hello"})
    assert response.status_code == 200
    assert "answer" in response.json()
    assert "Placeholder" in response.json()["answer"]
```

---

**Test 2: Query endpoint stays up after multiple requests**

**Given:** The agent is running  
**When:** `POST /query` is called 5 times with different questions  
**Then:** Each call returns 200 OK and process does not crash

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app

def test_query_stays_up():
    client = TestClient(app)
    for i in range(5):
        response = client.post("/query", json={"question": f"test {i}"})
        assert response.status_code == 200
```

**Notes:**
- Validates FastAPI app resilience (AC-3 partial: process stays up).
- No LLM or MCP logic yet; Stories 2 and 3 add those.

---

## Verification Summary

**Total test count:** 2 tests (per breakdown Verification Approach)

| Test | File | Coverage |
|---|---|---|
| Health endpoint when `MCP_URL` unset or unreachable | `tests/test_health.py` | AC-4 (MCP unreachable case) |
| Query endpoint returns placeholder and stays up | `tests/test_query_stub.py` | Partial AC-3 (process resilience), AC-5 (FastAPI primary path) |

**Additional validation tests (4 total in code above):**
- `test_health_mcp_url_unset` (AC-4, unset case)
- `test_health_mcp_unreachable` (AC-4, unreachable case)
- `test_query_stub_response` (placeholder response)
- `test_query_stays_up` (process resilience)

**Run tests:**
```bash
pytest tests/test_health.py tests/test_query_stub.py -v
```

---

## Risks

### R-5: Async/sync HTTP client choice

**Likelihood:** Low  
**Impact:** Medium (performance or compatibility issues)  
**Mitigation:**
- FastAPI is async-native; `httpx` async client used for health probe.
- Story 3 will use same `httpx` client for FastMCP HTTP calls, ensuring consistency.

**Decision:** Use `httpx` (async) for all HTTP operations. Sync fallback (`requests`) not needed.

---

## Notes

- **Flask removal:** No Flask code exists in repo tree (`README.md` only); nothing to remove. If Flask artifacts discovered later, remove in Story 3.
- **stdio MCP logic:** None detected in repo tree; deferred check to Story 3.
- **LLM client:** Not integrated in this story. Stubbed `/query` endpoint placeholder suffices.
- **Health probe cache:** 10-second TTL prevents excessive probes; adjust if operational needs differ.
- **Environment:** `MCP_URL` optional in this story; required in Story 3 for actual MCP calls.
- **FastAPI server:** Run with `uvicorn agent.main:app --host 0.0.0.0 --port 8001` (documented in Story 4 README update).

---

## Open Questions

_None. Health probe contract, stubbed query behavior, and test coverage are fully specified._
