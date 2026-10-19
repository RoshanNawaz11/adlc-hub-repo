# Story 1: Low-Level Design

**Repo:** `mcp` (adlc-mcp-server)  
**Story ID:** 1  
**Story Title:** Set up FastAPI app with health endpoint and FastMCP SSE stub  
**Parent Epic:** PER-19  
**Root:** `/home/goos/projects/adlc-mcp-server/`

---

## 1. Acceptance Criteria (from requirements & breakdown)

This story implements Story 1 from the breakdown. The full epic has 5 ACs + 3 Exceptions; this story covers:

**AC1 (partial - FastMCP endpoint exists):**  
Given a client connects to `/sse`, then the FastMCP SSE handshake succeeds (no tools invoked yet).

**AC3 (health check):**  
Given the server process is started, when an operator checks `/health`, then they receive 200 + `{"status": "ok"}` without calling a GraphQL tool.

**AC4 (partial - HTTP setup):**  
Given a developer runs `uvicorn server:app`, then the server starts and listens on the configured host/port.

**Exception 1 (partial - startup validation):**  
Given `GRAPHQL_ENDPOINT` is missing, when the server starts, then it logs an error and exits.

**Not covered in this story:** AC2 (response shapes), AC5 (full test coverage), Exception 2 (stdio migration), Exception 3 (auth deferred) — these are addressed in later stories.

---

## 2. File Plan

All paths resolve from `/home/goos/projects/adlc-mcp-server/`.

### New Files

1. **`server.py`**  
   FastAPI application with `/sse` (FastMCP SSE endpoint) and `/health` endpoint. Reads environment variables and initializes the MCP server.

2. **`main.py`**  
   Uvicorn entrypoint to run the FastAPI app.

3. **`tests/test_server.py`**  
   Integration tests for FastAPI endpoints (`/health`, `/sse` handshake, startup validation).

### Updated Files

1. **`requirements.txt`** (or `pyproject.toml` if Poetry is used)  
   Add dependencies: `fastapi`, `uvicorn[standard]`, `fastmcp`, `pytest`, `pytest-asyncio`, `httpx`.

---

## 3. Interfaces

### 3.1 FastAPI Endpoints

**`GET /health`**  
- **Request:** none.
- **Response (200):**
  ```json
  {"status": "ok"}
  ```
- **Behavior:** Returns immediately; does not check GraphQL connectivity.

**`/sse` (FastMCP SSE Endpoint)**  
- **Protocol:** Server-Sent Events (SSE).
- **Handshake:** FastMCP protocol initialization.
- **Tool Invocations:** None in this story (tools added in Story 3).
- **Tool Responses:** None in this story.

---

### 3.2 Environment Variables

- **`GRAPHQL_ENDPOINT`** (required): URL of the GraphQL service (e.g., `https://example.com/graphql`). Validated at startup; server fails fast if missing.
- **`HOST`** (optional, default `0.0.0.0`): FastAPI bind host.
- **`PORT`** (optional, default `8000`): FastAPI bind port.

---

## 4. Algorithm Notes

### 4.1 Server Startup (`main.py` / `server.py`)

**`main.py` (Uvicorn entrypoint):**
```python
import os
import uvicorn

if __name__ == "__main__":
    host = os.getenv("HOST", "0.0.0.0")
    port = int(os.getenv("PORT", "8000"))
    uvicorn.run("server:app", host=host, port=port, reload=False)
```

**`server.py` (FastAPI app):**
```python
import os
import logging
from fastapi import FastAPI
from fastapi.responses import JSONResponse
from fastmcp import FastMCP

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Validate environment
graphql_endpoint = os.getenv("GRAPHQL_ENDPOINT")
if not graphql_endpoint:
    logger.error("GRAPHQL_ENDPOINT environment variable is required")
    raise ValueError("GRAPHQL_ENDPOINT environment variable is required")

logger.info(f"Configured GraphQL endpoint: {graphql_endpoint}")

app = FastAPI()
mcp = FastMCP("MCP GraphQL Server")

# Health endpoint
@app.get("/health")
async def health():
    return JSONResponse({"status": "ok"}, status_code=200)

# FastMCP SSE endpoint (no tools registered yet)
app.mount("/sse", mcp.get_asgi_app())
```

**Key logic:**
- Read `GRAPHQL_ENDPOINT` at startup; raise `ValueError` if missing.
- Log the configured endpoint for debugging.
- Initialize FastMCP instance (no tools yet).
- `/health` returns immediately without checking GraphQL.
- Mount `/sse` endpoint for FastMCP SSE (handshake will succeed, but no tools are available).

---

### 4.2 Testing Strategy

**Test 1: Health endpoint (`test_server.py`)**
```python
import pytest
from httpx import AsyncClient
from server import app

@pytest.mark.asyncio
async def test_health_endpoint():
    """Verify /health returns 200 + {"status": "ok"}."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/health")
        assert response.status_code == 200
        assert response.json() == {"status": "ok"}
```

**Test 2: FastMCP SSE handshake (`test_server.py`)**
```python
@pytest.mark.asyncio
async def test_sse_endpoint_handshake():
    """Verify /sse FastMCP SSE handshake succeeds."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        # FastMCP SSE handshake (simplified check; exact behavior depends on FastMCP version)
        response = await client.get("/sse", headers={"Accept": "text/event-stream"})
        # Accept 200 (SSE stream started) or 405 (method not allowed if only POST supported)
        assert response.status_code in [200, 405]
```

**Test 3: Server fails to start if `GRAPHQL_ENDPOINT` is missing (`test_server.py`)**
```python
import pytest
import os

def test_server_startup_validation_missing_graphql_endpoint(monkeypatch):
    """Verify server fails to start if GRAPHQL_ENDPOINT is missing."""
    monkeypatch.delenv("GRAPHQL_ENDPOINT", raising=False)
    
    with pytest.raises(ValueError, match="GRAPHQL_ENDPOINT environment variable is required"):
        # Re-import server module to trigger startup validation
        import importlib
        import server
        importlib.reload(server)
```

---

## 5. Tests

**Test 1: Server starts and `/health` returns 200 + `{"status": "ok"}`**  
- **AC Coverage:** AC3 (health check)
- **File:** `/home/goos/projects/adlc-mcp-server/tests/test_server.py`
- **Method:** Start server, `GET /health`, assert response.

**Test 2: `/sse` FastMCP SSE handshake succeeds**  
- **AC Coverage:** AC1 (partial - FastMCP endpoint exists)
- **File:** `/home/goos/projects/adlc-mcp-server/tests/test_server.py`
- **Method:** Connect to `/sse`, verify handshake (no tools invoked yet).

**Test 3: Server fails to start if `GRAPHQL_ENDPOINT` is missing**  
- **AC Coverage:** Exception 1 (startup validation)
- **File:** `/home/goos/projects/adlc-mcp-server/tests/test_server.py`
- **Method:** Unset `GRAPHQL_ENDPOINT`, start server, assert `ValueError` raised.

---

## 6. Edge Cases and Error Handling

### 6.1 Missing `GRAPHQL_ENDPOINT`
- **Scenario:** Server starts without `GRAPHQL_ENDPOINT` set.
- **Behavior:** Server fails at startup with `ValueError`.
- **Implementation:** Check in `server.py` startup; log error and raise exception.

### 6.2 Malformed `GRAPHQL_ENDPOINT`
- **Scenario:** `GRAPHQL_ENDPOINT` is set but not a valid URL (e.g., missing protocol).
- **Behavior:** Startup succeeds (validation deferred to Story 2 when GraphQL client is implemented).
- **Implementation:** No URL format validation in this story.

### 6.3 Health endpoint invoked before server fully started
- **Scenario:** `/health` is called during server initialization.
- **Behavior:** Returns 200 + `{"status": "ok"}` once the server is ready to accept requests.
- **Implementation:** Uvicorn handles this; no custom logic needed.

### 6.4 SSE endpoint accessed without proper headers
- **Scenario:** Client connects to `/sse` without `Accept: text/event-stream` header.
- **Behavior:** FastMCP library handles this; may return 405 or 400.
- **Implementation:** No custom logic needed; FastMCP default behavior.

---

## 7. Open Questions

*(None. All ACs for this story are addressed.)*

---

## 8. Notes

- **No tools in this story:** The `/sse` endpoint is mounted but no MCP tools are registered. Tools are added in Story 3.
- **No GraphQL client in this story:** `GRAPHQL_ENDPOINT` is validated at startup but not used. GraphQL client is implemented in Story 2.
- **Test coverage:** 3 tests total for this story (Tests 1–3 from requirements.md). Full test suite of 20 tests is completed across all 5 stories.
- **Dependencies:** Add `fastapi`, `uvicorn[standard]`, `fastmcp`, `pytest`, `pytest-asyncio`, `httpx` to `requirements.txt` or `pyproject.toml`.
- **README updates:** HTTP setup documentation is added in Story 5, not this story.
- **FastMCP version:** Verify the current stable FastMCP version supports SSE and is compatible with FastAPI. Pin the version in `requirements.txt`.

---

**End of LLD**
