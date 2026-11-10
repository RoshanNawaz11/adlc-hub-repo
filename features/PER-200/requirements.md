# Requirements — PER-200

**Story:** Readiness check endpoint for the agents service  
**Epic:** (none — standalone backlog story)  
**Type:** story

---

## Overview

`GET /health` always returns `{"status": "ok"}`, even when the service cannot process orders (checkpointer unreachable, Redis down, model gateway rejecting calls, MCP tools unavailable). Deploys and the dashboard cannot distinguish "up" from "able to work".

Add `GET /ready` that verifies each dependency and returns 200 when all pass, 503 when any fail. Checks run in parallel with a timeout so the endpoint never blocks longer than one timeout. Errors expose no secrets. The server logs check results at startup.

---

## Given/When/Then Acceptance Criteria

### AC-1: Dependency checks

**Given** the agents service is running  
**When** a request to `GET /ready` is made  
**Then** the endpoint runs one check for each dependency in parallel:

- **checkpointer** (`runtime/checkpointer.py`): open and close a run database connection
- **Redis** (`queue/subtasks.py`, `redis_settings`): send a ping
- **model gateway** (`model/gateway.py`): verify the gateway URL and key are set, and accept a request with no model call
- **MCP** (`tools/mcp_client.py`): retrieve the tool list from the configured server

**And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string.

**Assumption:** The ticket's `{name, ok, ms, error}` format requires all four fields in every response, with `error` present but possibly empty or null when `ok` is true, to maintain a consistent response shape.

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

- **Secrets sanitization:** Error messages must be scrubbed before being written to `error` or logged.
- **Parallelism verification:** Tests should confirm that the endpoint does not serialize checks (e.g., four 1-second checks should complete in ~1 second, not ~4 seconds).
- **Timeout mechanism:** Implementation must use a timeout context (e.g., `asyncio.wait_for` or equivalent) that applies per check, not to the aggregate.
