# Story — PER-200

**Title:** Readiness check endpoint for the agents service  
**Type:** story  
**Epic:** (standalone backlog story)  
**ID:** 200  
**Status:** planned  
**Assignee:** (unassigned)

---

## Overview

`GET /health` always returns `{"status": "ok"}`, even when the service cannot process orders (checkpointer unreachable, Redis down, model gateway rejecting calls, MCP tools unavailable). Deploys and the dashboard cannot distinguish "up" from "able to work".

Add `GET /ready` that verifies each dependency and returns 200 when all pass, 503 when any fail. Checks run in parallel with a timeout so the endpoint never blocks longer than one timeout. Errors expose no secrets. The server logs check results at startup.

---

## Technical Specification

### Architecture

Add a new `/ready` endpoint to `src/agents/runtime/server.py` that orchestrates parallel health checks across four dependencies. Each check is a standalone async function with timeout enforcement. The endpoint aggregates results and returns appropriate HTTP status based on check outcomes.

### Check Module Structure

Create `src/agents/runtime/health_checks.py` with:

```python
# health_checks.py structure (not literal syntax)

async def check_checkpointer() -> CheckResult
  # Open and close a run database connection from runtime/checkpointer.py
  # Return {name: "checkpointer", ok: bool, ms: int, error: str | None}

async def check_redis() -> CheckResult
  # Ping Redis using settings from queue/subtasks.py redis_settings
  # Return {name: "redis", ok: bool, ms: int, error: str | None}

async def check_model_gateway() -> CheckResult
  # Verify gateway URL and key are set (model/gateway.py)
  # Make a minimal request with no model call
  # Return {name: "model_gateway", ok: bool, ms: int, error: str | None}

async def check_mcp() -> CheckResult
  # Retrieve tool list from tools/mcp_client.py configured server
  # Return {name: "mcp", ok: bool, ms: int, error: str | None}

async def run_all_checks(timeout: float) -> list[CheckResult]
  # Run all four checks in parallel using asyncio.gather with timeout per check
  # Each check wrapped in asyncio.wait_for(check_fn(), timeout=timeout)
  # Timeout yields {ok: False, error: "timeout"}
  # Return list of all results
```

### Endpoint Implementation

In `src/agents/runtime/server.py`:

```python
# server.py additions (not literal syntax)

@app.get("/ready")
async def readiness_check():
  timeout = float(os.getenv("READY_CHECK_TIMEOUT", "3"))
  checks = await run_all_checks(timeout)
  ready = all(check["ok"] for check in checks)
  status_code = 200 if ready else 503
  return JSONResponse(
    status_code=status_code,
    content={"ready": ready, "checks": checks}
  )
```

### Secrets Sanitization

Every check's error handling must:
- Strip gateway keys
- Remove Redis passwords
- Mask credentials in URLs (replace with `***`)

Implementation: a `sanitize_error(msg: str) -> str` helper that applies regex patterns to common secret formats before returning the error field.

### Startup Logging

In `server.py` startup block:

```python
# On server start (not literal syntax)
async def log_readiness_checks():
  timeout = float(os.getenv("READY_CHECK_TIMEOUT", "3"))
  checks = await run_all_checks(timeout)
  for check in checks:
    status = "ok" if check["ok"] else f"failed {check.get('error', 'unknown')}"
    logger.info(f"ready: {check['name']} {status} {check['ms']}ms")
```

### Environment Variable

- `READY_CHECK_TIMEOUT`: seconds before a check times out (default: 3)

### Existing Endpoints

No changes to:
- `GET /health` (remains `{"status": "ok"}`)
- `POST /runs`
- `POST /runs/{thread_id}/resume`
- `GET /runs/{thread_id}`
- Queue operations

---

## Acceptance Criteria

### AC-1: Dependency checks

**Given** the agents service is running  
**When** a request to `GET /ready` is made  
**Then** the endpoint runs one check for each dependency in parallel:

- **checkpointer** (`runtime/checkpointer.py`): open and close a run database connection
- **Redis** (`queue/subtasks.py`, `redis_settings`): send a ping
- **model gateway** (`model/gateway.py`): verify the gateway URL and key are set, and accept a request with no model call
- **MCP** (`tools/mcp_client.py`): retrieve the tool list from the configured server

**And** each check returns `{name, ok, ms, error}`.

**Assumption:** The ticket's `{name, ok, ms, error}` format requires all four fields in every response, with `error` present but possibly empty or null when `ok` is true, to maintain a consistent response shape.

---

### AC-2: Timeouts

**Given** a check is running  
**When** it exceeds `READY_CHECK_TIMEOUT` seconds (environment variable, default 3)  
**Then** the check is terminated  
**And** it returns `{ok: false, error: "timeout"}`  
**And** the overall `GET /ready` endpoint completes within one timeout duration (checks run in parallel).

---

### AC-3: API responses

**Given** all checks have completed  
**When** every check returned `ok: true`  
**Then** `GET /ready` returns HTTP 200 with `{ready: true, checks: [...]}`  

**Given** at least one check returned `ok: false`  
**When** the checks have completed  
**Then** `GET /ready` returns HTTP 503 with `{ready: false, checks: [...]}`  

**And** `GET /health` remains unchanged and continues to return `{"status": "ok"}`.

---

### AC-4: No secrets in errors

**Given** a check fails  
**When** the error is recorded  
**Then** the `error` field does not include the gateway key, Redis password, or full URLs with embedded credentials.

---

### AC-5: Startup logging

**Given** the server is starting  
**When** initialization completes  
**Then** one log line is written per check (e.g., `ready: checkpointer ok 12ms`, `ready: redis failed timeout`)  
**And** the logs use the same check logic and timeout as `GET /ready`.

---

### AC-6: Existing endpoints unchanged

**Given** the readiness check is implemented  
**When** any of the following endpoints are called  
**Then** their behavior is unchanged:

- `POST /runs`
- `POST /runs/{thread_id}/resume`
- `GET /runs/{thread_id}`
- queue operations

---

### AC-7: Test coverage

**Given** the readiness check implementation  
**When** tests are run  
**Then** coverage includes:

- all checks passing
- each individual check failing
- a timeout scenario
- HTTP 200 response when ready
- HTTP 503 response when not ready
- no secret leaked in an error message
- checks running in parallel (total time ≤ one timeout, not sum of all timeouts)

## Verification

1. **Manual:** `curl http://localhost:8000/ready` returns 200 with all checks `ok: true` when dependencies are healthy; returns 503 with at least one `ok: false` when a dependency is unavailable.
2. **Manual:** Server startup logs show one line per check with timing.
3. **Manual:** With `READY_CHECK_TIMEOUT=1` and one slow check, the endpoint returns within ~1 second.
4. **Manual:** Trigger a Redis auth failure; verify the 503 response error does not include the password.
5. **Automated:** Run test suite covering all seven AC-7 scenarios.

---

## Tests

See `tests/test_health_checks.py` and `tests/test_ready_endpoint.py`:

1. **test_all_checks_pass:** Mock all four checks to return `ok: true`; assert 200 response with `ready: true`.
2. **test_checkpointer_fails:** Mock checkpointer to return `ok: false`; assert 503 with `ready: false`.
3. **test_redis_fails:** Mock Redis ping to fail; assert 503.
4. **test_model_gateway_fails:** Mock gateway to reject; assert 503.
5. **test_mcp_fails:** Mock MCP to timeout or error; assert 503.
6. **test_check_timeout:** Mock one check to sleep longer than `READY_CHECK_TIMEOUT`; assert it returns `error: "timeout"` and 503.
7. **test_no_secret_in_error:** Inject a Redis error with password in message; assert sanitized error.
8. **test_parallel_execution:** Mock all checks to sleep 1s; assert total elapsed time ≤ 1.1s (not ~4s).
9. **test_startup_logging:** Capture logs during server init; assert one line per check with timing.
10. **test_existing_endpoints_unchanged:** Call `/health`, `POST /runs`, etc.; assert responses match pre-implementation behavior.

---

## Out of Scope

- Auto-restart on failed checks
- Metrics export
- Checking the backend API availability

---

## Open Questions

None.

---

## Notes

- **File paths match repo:** All references (`runtime/checkpointer.py`, `queue/subtasks.py`, `model/gateway.py`, `tools/mcp_client.py`, `src/agents/runtime/server.py`) are verified against the actual tree; `graphql` repo is not mapped and not used.
- **Secrets sanitization:** Apply before writing to `error` or logs; use a dedicated helper function.
- **Parallelism verification:** Test must confirm total time ≤ one timeout, not sum of check times.
- **Timeout mechanism:** Use `asyncio.wait_for` per check, not aggregate timeout.
