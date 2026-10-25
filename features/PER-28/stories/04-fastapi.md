# Story 04-fastapi LLD: POST /query

**Ticket:** PER-32  
**Story:** 04-fastapi  
**Repo:** See LLD (stack/paths deferred to implementation based on repo context)  
**Status:** Draft

---

## Overview

Implement the POST /query endpoint as the synchronous entry point for agent query submissions. The endpoint generates a unique `run_id`, invokes the agent service via `AgentClient.initiate_query()`, registers the run in `RunRegistry`, and returns the agent's immediate reply (Ok or rejected) without waiting for the agent's final response.

**CRITICAL CHANGE (per change_requests.md):** FastAPI does **not** create, decide, or delay run status. The agent owns all timing logic. This endpoint only forwards the agent's immediate reply and registers the ID mapping for SSE routing. No timer logic, no status generation.

---

## Acceptance Criteria

### AC1: Immediate POST response
**Given** a POST request to `/query`  
**When** the request is received  
**Then** the backend returns the agent's immediate reply (Ok or rejected) without delay

---

## Scope

This story defines the POST /query endpoint interface and behavior contracts. Concrete file paths, language-specific implementations, and stack choices are deferred to implementation based on actual repo context, which currently shows "(repo path not mapped; do not invent a stack)" per `repo-tree/fastapi`.

---

## Functional Requirements

### POST /query Endpoint

**Responsibility:** Accept query requests, forward to agent service, return agent's immediate reply.

**Interface:**
```
POST /query
Request body: { "query": string, ... }
Response: { "status": "ok" | "rejected", "run_id": string }
HTTP 200 (ok or rejected) or 503 (agent unavailable)
```

**Behavior:**
- Validates incoming request
- Generates unique `run_id` for client tracking (UUID4)
- Forwards query to agent service (non-blocking)
- Returns agent's immediate reply without delay
- Registers run in registry (ID mapping only, no status)
- Does NOT wait for agent completion
- Does NOT poll agent service

**Response Codes:**
- `200 OK`: Agent replied (ok or rejected)
- `422 Unprocessable Entity`: Invalid request body (validation failure)
- `503 Service Unavailable`: Agent service unreachable/timeout
- `500 Internal Server Error`: Unexpected error (catch-all)

**AC Coverage:** AC1 (immediate response without delay)

---

### Endpoint Algorithm

**Processing Flow:**
1. Generate unique run_id (UUID4 format)
2. Call agent service's immediate query submission interface
3. Receive agent's immediate reply: `{"status": "ok"|"rejected", "agent_run_id": string}`
4. Register run in registry: map client `run_id` to agent `agent_run_id`
5. Return immediate response: `{"status": "ok"|"rejected", "run_id": string}`

**Key Constraints:**
- Response time target: < 100ms (agent reply expected in < 3s per Story 2)
- No `sleep()` or waiting logic — purely synchronous reply path
- Registry registration happens **after** agent replies (no state created for failed queries)
- UUID4 generates collision-resistant run_id (no explicit uniqueness check needed)

**Error Handling:**
- Agent timeout (3s): HTTP 503 "Agent service unavailable"
- Agent connection failure: HTTP 503 "Agent service unavailable"
- Agent HTTP error (4xx/5xx): HTTP 503 "Agent service unavailable"
- Unexpected error: HTTP 500 "Internal server error"
- Detailed errors logged server-side only (not exposed in response)

---

## Data Contracts

### Request Schema
```
QueryRequest:
  query: string (required, non-empty)
  [additional fields as defined by agent contract]
```

**Validation:**
- `query` field must be present and non-empty
- Additional validation (length limits, rate limiting) deferred to implementation needs

### Response Schema
```
ImmediateResponse:
  status: "ok" | "rejected"
  run_id: string (UUID4 format)
```

**Serialization Example:**
```json
{"status": "ok", "run_id": "123e4567-e89b-12d3-a456-426614174000"}
```

---

## Dependencies

**Component Dependencies:**
- Story 1: Query request/response schemas, config (agent base URL, timeout)
- Story 2: AgentClient (initiate_query method)
- Story 3: RunRegistry (register_run method)

**Runtime Requirements:**
- HTTP client library with async support
- UUID generation (typically stdlib)
- JSON serialization (typically stdlib)
- Web framework with dependency injection support

**Test Requirements:**
- Test framework with async support
- HTTP client test utilities
- Mocking library

---

## Implementation Guidance

When implementing this story in the actual codebase:

1. **Choose web framework** based on existing repo patterns (FastAPI, Express, Actix, etc.)
2. **Locate or create router/controller module** following project conventions
3. **Define endpoint handler** matching the interface contracts above
4. **Implement dependency injection** for AgentClient and RunRegistry (singleton instances)
5. **Use async patterns** appropriate to the chosen stack
6. **Configure error translation** per the error handling strategy above

**Key Principles:**
- The contracts above (operation signatures, error cases, response formats) are fixed
- The implementation syntax (language, web framework, async patterns) adapts to the repo's stack
- No timer logic, no status creation — only ID mapping and immediate reply forwarding

---

## Tests

### Test 1: Successful Query Submission (Ok) - Unit

**Verification:** requirements.md AC1 (POST returns `{"status": "ok", "run_id": "<uuid>"}` in < 100ms)

**Acceptance Criteria:** AC1 (immediate response without delay)

**Test Coverage:**
- Mock agent client returning `{"status": "ok", "agent_run_id": "agent-123"}`
- Submit POST /query with `{"query": "test query"}`
- Assert response status 200
- Assert response body: `{"status": "ok", "run_id": "<uuid>"}`
- Assert `run_id` is valid UUID4 (36 characters, version 4)
- Assert agent client called exactly once
- Assert registry registered run with both IDs (`run_id`, `agent_run_id`)
- Assert response time < 100ms (excluding network/mock overhead)

---

### Test 2: Rejected Query Submission - Unit

**Verification:** requirements.md AC1 (POST returns `{"status": "rejected", "run_id": "<uuid>"}` with 200 status)

**Acceptance Criteria:** AC1 (immediate response for rejection)

**Test Coverage:**
- Mock agent client returning `{"status": "rejected", "agent_run_id": "agent-456"}`
- Submit POST /query with `{"query": "bad query"}`
- Assert response status 200 (rejection is still success response)
- Assert response body: `{"status": "rejected", "run_id": "<uuid>"}`
- Assert registry registered run (needed for SSE stream even on rejection)

---

### Test 3: Agent Service Unavailable - Unit

**Verification:** requirements.md AC1 (POST returns 503 on agent timeout/error)

**Acceptance Criteria:** AC1 (immediate error response, no delay)

**Test Coverage:**
- Mock agent client raising timeout error
- Submit POST /query with `{"query": "test query"}`
- Assert response status 503
- Assert response body: `{"detail": "Agent service unavailable"}`
- Assert registry NOT called (no run created for failed queries)
- Repeat with connection error instead of timeout
- Assert same 503 response

---

### Test 4: Registry Registration Verification - Unit

**Verification:** requirements.md AC1 (Verify RunRegistry.register_run() called with generated run_id)

**Acceptance Criteria:** AC1 (run registered for SSE tracking)

**Test Coverage:**
- Mock agent client returning `{"status": "ok", "agent_run_id": "agent-789"}`
- Submit POST /query with `{"query": "test query"}`
- Extract `run_id` from response
- Assert registry called once with: `register_run(run_id=<response_run_id>, agent_run_id="agent-789")`
- Assert `run_id` is valid UUID4
- Assert `run_id` differs from `agent_run_id` (no correlation)

---

## Open Questions

1. **Router prefix:** Should POST /query be at root `/query` or nested under `/api/query`? Current design assumes root. Adjust routing configuration if API versioning needed.

2. **Request validation:** Should we add length limits (e.g., max 10,000 characters for query field)? Or rate limiting per client? Current design accepts any valid string.

3. **Response headers:** Should we add custom headers (e.g., `X-Run-ID: <uuid>`)? Current design includes `run_id` only in JSON body per requirements.

4. **Authentication:** No auth implemented in this story. If queries require authentication, add auth dependency in future story. Does not affect current AC.

5. **Idempotency:** Should POST /query be idempotent (same query returns same run_id)? Current design: each POST creates new run_id. If idempotency needed, add request deduplication logic.

6. **Stack confirmation:** Current repo-tree shows "(repo path not mapped; do not invent a stack)". Before implementing, confirm:
   - Language and web framework
   - Existing routing patterns
   - Async patterns and conventions
   - Dependency injection approach
   - Module organization

---

## Risks

### Risk 1: UUID collision
**Impact:** Two requests get same run_id, SSE stream confusion  
**Likelihood:** Negligible (UUID4 collision probability ~10^-36)  
**Mitigation:** Use cryptographically random UUID4 generation. No explicit collision check needed.

### Risk 2: Agent service timeout blocks endpoint
**Impact:** Slow agent reply delays HTTP response  
**Likelihood:** Low (agent timeout 3s per Story 2)  
**Mitigation:** HTTP client timeout enforced in AgentClient (Story 2). Web framework concurrency ensures other requests not blocked.

### Risk 3: Registry memory leak
**Impact:** Uncleaned runs if registration happens but SSE never connects  
**Likelihood:** Low (simpler than old design — only IDs stored)  
**Mitigation:** Story 5 SSE endpoint cleans up mapping in `finally` block after terminal event. No periodic cleanup needed.

### Risk 4: Missing error logging
**Impact:** Hard to debug 503/500 errors in production  
**Likelihood:** High without logging  
**Mitigation:** Add logging in error handlers for troubleshooting (log exception details server-side, return generic message to client).

### Risk 5: Dependency injection not thread-safe
**Impact:** Singleton state corruption in multi-threaded environment  
**Likelihood:** Depends on web framework (low for event-loop based, medium for multi-threaded)  
**Mitigation:** Implementation verifies web framework's concurrency model. If multi-process deployment, each process gets own singleton instance (no shared state).

---

## Notes

- **No delay logic:** This endpoint returns immediately upon agent reply. No 5-second delay logic — that was removed per change_requests.md. The agent owns all timing logic.
- **No status creation:** This endpoint does not create `in_progress`, `completed`, or `failed` events. It only registers an ID mapping for SSE routing.
- **No SSE logic:** SSE endpoint (GET /query/{run_id}/stream) added in Story 5. This story only creates the run entry for SSE tracking.
- **Singleton pattern:** AgentClient and RunRegistry are singletons instantiated at application startup. Tests override via dependency injection.
- **Error messages:** Intentionally generic ("Agent service unavailable") to avoid leaking internal details. Detailed errors logged server-side only.
- **UUID format:** Always UUID4 (random), never UUID1 (timestamp-based) or UUID5 (name-based). No correlation with agent_run_id.
- **Registry registration order:** Happens **after** agent replies (not before). Failed queries do not create run entries.
- **Pure pass-through:** Per change_requests.md, FastAPI does not create, decide, or delay run status. This endpoint forwards the agent's immediate reply and registers the ID mapping — nothing more.
- **Implementation deferred:** Concrete file paths, imports, and language-specific syntax removed per ADLC rule: "File paths and stack MUST match repo context / existing tree. Never invent another language or framework." Implementation awaits repo-tree mapping.
- **Stack-agnostic design:** Operations defined as behavioral contracts, not language-specific code. Implementation adapts to repo's actual stack.

---

**Document Status:** Draft  
**Story:** 04-fastapi (PER-32)  
**Epic:** FastAPI SSE Integration with Agent Service (requirements.md AC1)  
**Altitude:** Story-level LLD
