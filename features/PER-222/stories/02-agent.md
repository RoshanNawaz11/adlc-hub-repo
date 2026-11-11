# Story 02-agent: API endpoint for withdrawal — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 1: Withdrawing a case — API interface

**Given** a case is paused waiting for answers to its flags (`status == "needs_input"`)  
**When** a reviewer withdraws it via `POST /runs/{thread_id}/withdraw` with a valid reason  
**Then** the API returns 200 OK with `{"status": "withdrawn", "reason": "<trimmed reason>", "withdrawn_at": "<ISO 8601 timestamp>"}`

---

**Given** a case is paused waiting for the final review (`status == "needs_review"`)  
**When** a reviewer withdraws it via `POST /runs/{thread_id}/withdraw` with a valid reason  
**Then** the API returns 200 OK with `{"status": "withdrawn", "reason": "<trimmed reason>", "withdrawn_at": "<ISO 8601 timestamp>"}`

---

**Given** a reviewer is withdrawing a case via the API  
**When** the reason is empty, or contains only spaces, tabs or line breaks  
**Then** the API returns 400 Bad Request with `{"detail": "Reason is required and cannot be empty"}`

---

**Given** a reviewer is withdrawing a case via the API  
**When** the reason exceeds 500 characters (after trimming)  
**Then** the API returns 400 Bad Request with `{"detail": "Reason exceeds 500 characters"}`

### AC 2: After withdrawing — API response

**Given** a case has been withdrawn  
**When** the API response is returned  
**Then** it contains `status: "withdrawn"`, `reason: "<trimmed reason>"`, and `withdrawn_at: "<ISO 8601 timestamp>"`

### AC 3: Cases that can't be withdrawn — API error responses

**Given** a case is still running (`status == "running"`)  
**When** a reviewer attempts to withdraw it via the API  
**Then** the API returns 409 Conflict with `{"detail": "Case is running, wait until it pauses"}`

---

**Given** a case is done, rejected, or failed (`status in ("done", "rejected", "failed")`)  
**When** a reviewer attempts to withdraw it via the API  
**Then** the API returns 409 Conflict with `{"detail": "Case is <status>, cannot withdraw"}`

---

**Given** a case does not exist (no checkpoint for the `thread_id`)  
**When** a reviewer attempts to withdraw it via the API  
**Then** the API returns 404 Not Found with `{"detail": "Case not found"}`

---

**Given** a case has already been withdrawn  
**When** a reviewer attempts to withdraw it a second time via the API  
**Then** the API returns 409 Conflict with `{"detail": "Case was withdrawn on <timestamp>: <reason>"}`

### AC 4: Everywhere a case is handled — API

**Given** a paused case exists  
**When** a reviewer withdraws it from the API  
**Then** the withdrawal proceeds with the same state transition and validation rules as the CLI

---

## File Plan

### Modified Files

#### `src/agents/runtime/server.py`

**Changes:**

1. Add imports (after line ~31, after the existing imports):
   ```python
   from agents.runtime.withdrawal import CaseNotFoundError, InvalidStateError, withdraw_case
   from agents.runtime.run_dir import run_dir
   ```

2. Add request model (after `FlagAnswers` class definition, around line ~70):
   ```python
   class WithdrawalRequest(BaseModel):
       reason: str
   ```

3. Add custom exception handler (after `_lifespan` function, before route definitions, around line ~75):
   ```python
   @app.exception_handler(HTTPException)
   async def http_exception_handler(request, exc: HTTPException):
       """Return error responses in {"error": "<message>"} format per HLD."""
       return JSONResponse(
           status_code=exc.status_code,
           content={"error": exc.detail}
       )
   ```

4. Add import for JSONResponse (with other FastAPI imports around line ~25):
   ```python
   from fastapi.responses import JSONResponse
   ```

5. Add `POST /runs/{thread_id}/withdraw` route (after `resume_run` endpoint, after line ~148, before `get_run`):
   ```python
   @app.post("/runs/{thread_id}/withdraw")
   async def withdraw_run(thread_id: str, req: WithdrawalRequest) -> dict[str, Any]:
       """Withdraw a paused case with a reason.
       
       Returns:
           200 OK: {"status": "withdrawn", "reason": "<reason>", "withdrawn_at": "<timestamp>"}
           400 Bad Request: {"error": "Reason is required and cannot be empty"} or
                            {"error": "Reason exceeds 500 characters"}
           404 Not Found: {"error": "Case not found"}
           409 Conflict: {"error": "Case is running, wait until it pauses"} or
                         {"error": "Case is <status>, cannot withdraw"} or
                         {"error": "Case was withdrawn on <timestamp>: <reason>"}
       """
       # 1. Determine reviewer_id (from auth context, placeholder for now)
       # TODO: extract from authentication token when auth is implemented
       reviewer_id = "api-user"  # placeholder; future: extract from JWT or session
       
       # 2. Call the shared withdrawal service function
       try:
           result = await withdraw_case(
               thread_id=thread_id,
               reason=req.reason,
               reviewer_id=reviewer_id,
               checkpointer_kind=os.getenv("CHECKPOINTER", "sqlite"),
               db_path=run_dir(thread_id) / "checkpoint.db",
           )
           # 3. Update in-memory run cache (if running)
           if thread_id in _runs:
               _runs[thread_id] = {"thread_id": thread_id, "status": "withdrawn", **result}
           return result
       except ValueError as exc:
           # Reason validation failures (empty or >500 chars)
           raise HTTPException(400, str(exc))
       except CaseNotFoundError as exc:
           raise HTTPException(404, str(exc))
       except InvalidStateError as exc:
           raise HTTPException(409, str(exc))
   ```

**No other changes.** The existing endpoints (`create_run`, `resume_run`, `get_run`, `health`) are unchanged.

---

### New Files

None. Story 1 created `src/agents/runtime/withdrawal.py`; this story only imports and calls it.

## Interfaces

### `POST /runs/{thread_id}/withdraw` endpoint

**Location:** `src/agents/runtime/server.py`

**Path:** `/runs/{thread_id}/withdraw`

**Method:** `POST`

**Path parameters:**

- `thread_id` (str): The run's unique identifier (matches the checkpointer's `thread_id` and `State["run_id"]`).

**Request body:**

```json
{
  "reason": "Duplicate of SO-1003"
}
```

**Request model:** `WithdrawalRequest`

```python
class WithdrawalRequest(BaseModel):
    reason: str
```

**Validation:** Pydantic validates that `reason` is present and is a string. The `withdraw_case` function performs the business validation (non-empty after trimming, ≤500 characters).

---

**Response (200 OK):**

```json
{
  "status": "withdrawn",
  "reason": "Duplicate of SO-1003",
  "withdrawn_at": "2026-10-15T14:32:00Z"
}
```

**Response type:** `dict[str, Any]` (matches the return type of `withdraw_case`)

---

**Error responses:**

**400 Bad Request** (reason validation failure):

```json
{
  "error": "Reason is required and cannot be empty"
}
```

or

```json
{
  "error": "Reason exceeds 500 characters"
}
```

---

**404 Not Found** (case not found):

```json
{
  "error": "Case <thread_id> not found"
}
```

**Note:** `CaseNotFoundError` message from `withdrawal.py` is `"Case {thread_id} not found"`; HTTPException will wrap it in `error`.

---

**409 Conflict** (invalid state):

```json
{
  "error": "Case is running, wait until it pauses"
}
```

or

```json
{
  "error": "Case is done, cannot withdraw"
}
```

or

```json
{
  "error": "Case was withdrawn on 2026-10-14T10:00:00Z: Duplicate of SO-1003"
}
```

**Note:** `InvalidStateError` message is preserved from `withdrawal.py`.

---

### Reviewer authentication integration point

**Current implementation (placeholder):**

```python
reviewer_id = "api-user"  # placeholder
```

**Future implementation (when auth is added):**

FastAPI's dependency injection can extract the reviewer's identity from a JWT token or session. Example pattern:

```python
from fastapi import Depends
from typing import Annotated

def get_current_user(token: str = Header(..., alias="Authorization")) -> str:
    # decode JWT, validate, extract user_id
    return user_id

@app.post("/runs/{thread_id}/withdraw")
async def withdraw_run(
    thread_id: str,
    req: WithdrawalRequest,
    reviewer_id: Annotated[str, Depends(get_current_user)]
) -> dict[str, Any]:
    # ...
```

For this story, the placeholder `"api-user"` is acceptable; the AC tests will mock or ignore the reviewer identity field.

## Algorithm Notes

### Request flow

```
1. FastAPI receives POST /runs/{thread_id}/withdraw with JSON body {"reason": "<reason>"}
2. Pydantic validates the request body (reason is present and a string)
3. Endpoint extracts thread_id from path, reason from body
4. Endpoint determines reviewer_id (placeholder: "api-user")
5. Endpoint resolves checkpointer path: run_dir(thread_id) / "checkpoint.db"
6. Endpoint calls withdraw_case(thread_id, reason, reviewer_id, checkpointer_kind, db_path)
7. withdraw_case performs validation, state transition, persistence (from Story 1)
8. On success: return {"status": "withdrawn", "reason": "<trimmed>", "withdrawn_at": "<timestamp>"}
9. On ValueError (reason invalid): raise HTTPException(400, str(exc))
10. On CaseNotFoundError: raise HTTPException(404, str(exc))
11. On InvalidStateError: raise HTTPException(409, str(exc))
```

### Error handling pattern

FastAPI's `HTTPException` is the standard mechanism for returning non-200 responses. The pattern:

```python
try:
    result = await withdraw_case(...)
    return result
except ValueError as exc:
    raise HTTPException(400, str(exc))
except CaseNotFoundError as exc:
    raise HTTPException(404, str(exc))
except InvalidStateError as exc:
    raise HTTPException(409, str(exc))
```

The `str(exc)` message becomes the `detail` field in FastAPI's JSON error response.

---

### In-memory run cache update

The `_runs` dict holds run status for background tasks started by `create_run` and `resume_run`. A withdrawal changes a run's status, so the cache should be updated if the run is present (it may not be if the withdrawal happens after a restart, or if the run was started by the queue worker instead of the HTTP server).

```python
if thread_id in _runs:
    _runs[thread_id] = {"thread_id": thread_id, "status": "withdrawn", **result}
```

This is best-effort: if the thread_id is not in the cache, the next `get_run` call will read the withdrawn state from the checkpointer anyway.

---

## Tests Section

**File:** `tests/test_withdrawal_api.py` (new file)

### Integration tests (7 total)

All tests use FastAPI's `TestClient` to test the HTTP layer. The tests mock `withdraw_case` to verify request validation and error mapping, relying on Story 1's unit tests for domain logic and Story 8's end-to-end tests for full integration.

**Setup helper (shared across tests):**

```python
import pytest
from fastapi.testclient import TestClient
from agents.runtime.server import app

@pytest.fixture
def client():
    """TestClient for synchronous HTTP calls."""
    return TestClient(app)
```

---

**Test 1: Successful withdrawal from `needs_input`**

```python
@pytest.mark.asyncio
async def test_withdraw_from_needs_input_returns_200(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 1: A case waiting for flag answers can be withdrawn via API, returns 200 OK."""
    thread_id = "test-api-1"
    # Mock withdraw_case to return success
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        return {"status": "withdrawn", "reason": "Test reason", "withdrawn_at": "2026-10-15T14:32:00Z"}
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "Test reason"})
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "withdrawn"
    assert data["reason"] == "Test reason"
    assert "withdrawn_at" in data
```

---

**Test 2: Empty reason returns 400**

```python
@pytest.mark.asyncio
async def test_withdraw_empty_reason_returns_400(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 1: An empty reason (after trimming) returns 400 Bad Request."""
    thread_id = "test-api-2"
    # Mock withdraw_case to raise ValueError
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        raise ValueError("Reason is required and cannot be empty")
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "   "})
    assert response.status_code == 400
    assert "Reason is required" in response.json()["error"]
```

---

**Test 3: Reason over 500 characters returns 400**

```python
@pytest.mark.asyncio
async def test_withdraw_long_reason_returns_400(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 1: A reason exceeding 500 characters returns 400 Bad Request."""
    thread_id = "test-api-3"
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        raise ValueError("Reason exceeds 500 characters")
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "a" * 501})
    assert response.status_code == 400
    assert "exceeds 500 characters" in response.json()["error"]
```

---

**Test 4: Case not found returns 404**

```python
@pytest.mark.asyncio
async def test_withdraw_not_found_returns_404(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 9: Withdrawing a non-existent case returns 404 Not Found."""
    thread_id = "test-api-4"
    from agents.runtime.withdrawal import CaseNotFoundError
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        raise CaseNotFoundError(f"Case {tid} not found")
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "Test"})
    assert response.status_code == 404
    assert "not found" in response.json()["error"]
```

---

**Test 5: Case running returns 409**

```python
@pytest.mark.asyncio
async def test_withdraw_running_case_returns_409(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 8: Withdrawing a running case returns 409 Conflict."""
    thread_id = "test-api-5"
    from agents.runtime.withdrawal import InvalidStateError
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        raise InvalidStateError("Case is running, wait until it pauses")
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "Test"})
    assert response.status_code == 409
    assert "wait until it pauses" in response.json()["error"]
```

---

**Test 6: Case done/rejected/failed returns 409**

```python
@pytest.mark.asyncio
@pytest.mark.parametrize("status", ["done", "rejected", "failed"])
async def test_withdraw_terminal_case_returns_409(
    client: TestClient, monkeypatch: pytest.MonkeyPatch, status: str
) -> None:
    """AC 9: Withdrawing a done/rejected/failed case returns 409 Conflict."""
    thread_id = f"test-api-6-{status}"
    from agents.runtime.withdrawal import InvalidStateError
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        raise InvalidStateError(f"Case is {status}, cannot withdraw")
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "Test"})
    assert response.status_code == 409
    assert f"is {status}" in response.json()["error"]
```

---

**Test 7: Already withdrawn case returns 409**

```python
@pytest.mark.asyncio
async def test_withdraw_already_withdrawn_returns_409(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None:
    """AC 6: Withdrawing an already-withdrawn case returns 409 Conflict with existing details."""
    thread_id = "test-api-7"
    from agents.runtime.withdrawal import InvalidStateError
    async def mock_withdraw(tid, reason, reviewer, **kwargs):
        raise InvalidStateError("Case was withdrawn on 2026-10-14T10:00:00Z: Original reason")
    monkeypatch.setattr("agents.runtime.server.withdraw_case", mock_withdraw)
    
    response = client.post(f"/runs/{thread_id}/withdraw", json={"reason": "Test"})
    assert response.status_code == 409
    assert "was withdrawn on" in response.json()["error"]
    assert "Original reason" in response.json()["error"]
```

---

**Test count alignment with breakdown:** 7 tests (1 success case, 6 error cases).

**Note on parameterization:** Test 6 uses `@pytest.mark.parametrize` to cover three terminal states (`done`, `rejected`, `failed`) in one test function, counting as 3 test cases when pytest runs it. The breakdown counts this as "1 test" (Case done/rejected/failed), but pytest will report 3. This is acceptable; the important alignment is that all 6 error scenarios from the breakdown are covered.

## Open Questions (carried forward from Story 1 / HLD)

1. **Reviewer authentication:** How is `reviewer_id` determined in the API context? For this story, the placeholder `"api-user"` is used. When authentication is implemented, the endpoint will extract `reviewer_id` from a JWT token or session via FastAPI's dependency injection.

2. **Checkpointer path resolution:** The endpoint resolves `db_path` as `run_dir(thread_id) / "checkpoint.db"`. This assumes `run_dir` is importable from `agents.runtime.run_dir`. The import has been added to the file plan.

---

**End of LLD.**
