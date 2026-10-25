# Story 02-fastapi LLD: Agent Client

**Ticket:** PER-30  
**Story:** 02-fastapi  
**Repo:** See LLD (stack/paths deferred to implementation based on repo context)  
**Status:** Draft

---

## Overview

Implement `AgentClient` service to interface with the external agent service. Provides synchronous immediate query submission via `initiate_query()` and asynchronous SSE event stream subscription via `subscribe_to_run()`. The client maintains a single subscription per run with no polling behavior and does not invoke MCP tools.

**CRITICAL CHANGE (per change_requests.md):** FastAPI does **not** create, decide, or delay run status. The agent owns all timing logic, including the 5-second delay. `AgentClient` is a pure pass-through: it forwards the agent's events exactly as received, with no timer logic, no `in_progress`/`completed`/`failed` generation.

---

## Acceptance Criteria

### AC1: FastAPI returns agent's immediate reply without delay (client provides reply)
**Given** a query is submitted to the agent service  
**When** `initiate_query()` is called  
**Then** the client returns the agent's immediate Ok or rejected reply synchronously

### AC4: FastAPI does not poll agent (client subscribes once)
**Given** a run is initiated  
**When** `subscribe_to_run()` is called  
**Then** the client creates exactly one SSE connection and does not poll for status

### AC5: FastAPI does not call MCP tools (code review: no MCP imports)
**Given** the agent client implementation  
**When** the code is reviewed  
**Then** no MCP tool imports or invocations are present

---

## Scope

This story defines the agent client interface and behavior contracts. Concrete file paths, language-specific implementations, and stack choices are deferred to implementation based on actual repo context, which currently shows "(repo path not mapped; do not invent a stack)" per `repo-tree/fastapi`.

---

## Functional Requirements

### Agent Client Interface

**Responsibilities:**
- Submit queries to agent service and return immediate replies
- Subscribe to agent SSE event streams for run status
- Maintain single subscription per run (no polling)
- Propagate agent errors to caller
- Remain MCP-free (agent service owns MCP interactions)

**Operations:**

#### Query Submission

**Inputs:**
- `query_data`: Query payload dictionary (minimally `{"query": string}`, additional fields TBD by agent contract)

**Outputs:**
- On success: `{"status": "ok" | "rejected", "agent_run_id": string}`
- On timeout: Timeout error (caller translates to 503)
- On connection failure: Connection error (caller translates to 503)
- On agent error (4xx/5xx): HTTP status error (caller translates to 503)

**Behavior:**
- POST to agent service at `{base_url}/agent/query`
- Enforce timeout (3 seconds default per Story 1 config)
- No retry logic (fail fast for immediate reply)
- Return raw JSON from agent (validation at caller level)

**Performance:**
- Response time target: < 100ms from agent (agent timeout: 3s)
- No delay or waiting beyond agent's immediate reply

---

#### SSE Stream Subscription

**Inputs:**
- `agent_run_id`: Agent's internal run identifier (from query submission response)

**Outputs:**
- Stream of events: `{"event": "in_progress" | "completed" | "failed", "data": object}`
  - `in_progress`: `{"run_id": string}`
  - `completed`: `{"run_id": string, "response": object}`
  - `failed`: `{"run_id": string, "error": string}`
- On timeout: Timeout error after 5 minutes
- On connection failure: Connection error

**Behavior:**
- GET to agent service at `{base_url}/agent/runs/{agent_run_id}/events`
- Accept: `text/event-stream` header
- Enforce timeout (300 seconds / 5 minutes)
- Single connection per call (no polling)
- Yield parsed events as they arrive (pure pass-through)
- Close connection after terminal event (`completed` or `failed`)
- **Does NOT create or delay events** — agent owns all timing logic

**Event Format (agent contract assumption):**
```
event: in_progress
data: {"run_id": "..."}

event: completed
data: {"run_id": "...", "response": {...}}

event: failed
data: {"run_id": "...", "error": "..."}
```

---

### Error Handling Strategy

**Timeout errors:**
- Query submission: 3s timeout → timeout error → caller returns 503
- SSE subscription: 5min timeout → timeout error → caller sends `failed` SSE event

**HTTP errors:**
- Agent returns 4xx/5xx → HTTP status error → caller returns 503

**Connection errors:**
- Agent service unreachable → connection error → caller returns 503

**No MCP handling:**
- Agent client has zero MCP-related imports or logic
- Agent service owns all MCP tool interactions

---

## Implementation Guidance

When implementing this story in the actual codebase:

1. **Choose HTTP client** based on existing repo patterns (e.g., `httpx` for Python, `fetch`/`axios` for Node.js, `reqwest` for Rust)
2. **Locate or create services module** following project conventions
3. **Define client class/module** matching the interface contracts above
4. **Implement query submission** with timeout, error handling, and no retries
5. **Implement SSE subscription** with line-by-line parsing, single connection, and terminal event detection
6. **Use async patterns** appropriate to the chosen stack (async/await, promises, futures)
7. **Configure timeouts** from Story 1 config (3s for query, 5min for SSE)

**Key Principles:**
- The contracts above (operation signatures, event formats, error cases) are fixed
- The implementation syntax (language, HTTP client library, async patterns) adapts to the repo's stack
- No MCP imports or calls (AC5 verification: code review ensures zero MCP dependencies)

---

## Tests

### Test 1: Successful Query Submission (Unit)

**Verification:** requirements.md AC1 (client returns agent's immediate reply)

**Acceptance Criteria:** AC1 (immediate reply without delay)

**Test Coverage:**
- Mock agent POST endpoint returning `{"status": "ok", "agent_run_id": "run-123"}`
- Call query submission operation
- Assert result matches expected reply: `{"status": "ok", "agent_run_id": "run-123"}`
- Assert exactly one HTTP POST call made
- Assert response time < 100ms (excluding network/mock overhead)

---

### Test 2: Rejected Query Submission (Unit)

**Verification:** requirements.md AC1 (client handles rejection replies)

**Acceptance Criteria:** AC1 (immediate reply for rejection)

**Test Coverage:**
- Mock agent POST endpoint returning `{"status": "rejected", "reason": "invalid query"}`
- Call query submission operation
- Assert result contains `{"status": "rejected"}` and optional `reason` field
- Assert exactly one HTTP POST call made

---

### Test 3: Query Submission Timeout (Unit)

**Verification:** requirements.md AC1 (client propagates timeout errors)

**Acceptance Criteria:** AC1 (timeout error handling)

**Test Coverage:**
- Mock agent POST endpoint to timeout after > 3s
- Call query submission operation
- Assert timeout error raised/thrown
- Assert caller can catch error and return 503

---

### Test 4: SSE Stream Success Path (Unit)

**Verification:** requirements.md AC4 (single subscription yields agent events)

**Acceptance Criteria:** AC4 (single subscription, no polling)

**Test Coverage:**
- Mock agent GET endpoint returning SSE stream:
  ```
  event: in_progress
  data: {"run_id": "run-123"}

  event: completed
  data: {"run_id": "run-123", "response": {"result": "success"}}
  ```
- Call SSE subscription operation
- Collect all yielded/emitted events
- Assert 2 events received: `in_progress` followed by `completed`
- Assert `completed` event data matches expected format
- Assert exactly one HTTP GET call made (no polling)

---

### Test 5: SSE Stream Failure Event (Unit)

**Verification:** requirements.md AC4 (handles failure events correctly)

**Acceptance Criteria:** AC4 (single subscription yields failure)

**Test Coverage:**
- Mock agent GET endpoint returning SSE stream:
  ```
  event: in_progress
  data: {"run_id": "run-123"}

  event: failed
  data: {"run_id": "run-123", "error": "agent timeout"}
  ```
- Call SSE subscription operation
- Collect all yielded/emitted events
- Assert 2 events received: `in_progress` followed by `failed`
- Assert `failed` event data contains expected error message
- Assert exactly one HTTP GET call made

---

### Test 6: Single Connection Per Subscription (Unit)

**Verification:** requirements.md AC4 (no polling behavior)

**Acceptance Criteria:** AC4 (no polling)

**Test Coverage:**
- Mock agent GET endpoint returning SSE stream with 2 events
- Call SSE subscription operation
- Consume all events
- Assert exactly one HTTP GET call made (verify no additional polling requests)
- Assert connection closed after terminal event (`completed` or `failed`)

---

## Dependencies

**Runtime:**
- Async-capable HTTP client library (determined by repo stack)
- JSON parsing library (typically stdlib)
- SSE stream parsing capability (line-by-line iteration)
- Config loading mechanism (Story 1 dependency)

**Test:**
- Project's test framework
- Async test support (if needed)
- HTTP mocking library (for unit tests)

**Notes:**
- Dependencies listed abstractly since stack not yet mapped
- Implementation adds concrete dependencies based on chosen stack

---

## Open Questions

1. **Agent service authentication:** Current implementation assumes no auth. If agent requires API keys/tokens, add to client initialization and operation headers.

2. **SSE event parsing robustness:** Should the client handle malformed SSE streams (missing event type, invalid JSON)? Consider error handling around JSON parsing if agent SSE reliability is unknown.

3. **Agent response schema validation:** Should the client validate agent responses against expected schema, or defer to caller? Current design: defer to caller (Story 4).

4. **Reconnection strategy:** If SSE stream disconnects mid-run, should the client retry subscription? Current design: fail fast, let caller (Story 5) handle.

5. **Stack confirmation:** Current repo-tree shows "(repo path not mapped; do not invent a stack)". Before implementing, confirm:
   - Language and framework
   - Existing HTTP client patterns
   - Async patterns and conventions
   - File structure and module organization

---

## Risks

### Risk 1: HTTP client SSE stream parsing
**Impact:** Incomplete/incorrect event parsing  
**Likelihood:** Low (most HTTP clients support streaming)  
**Mitigation:** Test with real agent SSE stream during integration testing (Story 5). Unit tests verify parsing logic with mock streams.

### Risk 2: Agent service endpoint contract mismatch
**Impact:** 404s, malformed responses  
**Likelihood:** Medium (agent contract not finalized)  
**Mitigation:** Clearly document assumed endpoints (`/agent/query`, `/agent/runs/{id}/events`) in this LLD. Adjust when real contract known.

### Risk 3: Timeout configuration mismatch
**Impact:** Premature timeouts or hanging connections  
**Likelihood:** Low  
**Mitigation:** Use timeouts from Story 1 config (3s for query, 5min for SSE). Verify in production load testing.

### Risk 4: Stack-specific async patterns
**Impact:** Implementation differs from standard patterns for chosen stack  
**Likelihood:** Medium (stack unknown)  
**Mitigation:** Survey existing async patterns in repo before implementing. Follow established conventions.

---

## Notes

- **No MCP imports:** Code review checkpoint for AC5. Agent client must remain MCP-free — all MCP logic stays in agent service.
- **Error propagation strategy:** Agent client raises/throws errors; caller (Story 4, 5) catches and translates to HTTP responses. This keeps client logic simple and testable.
- **SSE format assumption:** Parser expects `event:` line before `data:` line. If agent uses different SSE format (e.g., data-only events), parser needs adjustment when contract finalized.
- **Pure pass-through (change_requests.md):** Agent client does **not** create `in_progress`, `completed`, or `failed` events. It only forwards what the agent already sent. No timer logic, no status generation. The agent owns all timing logic (including the 5-second delay described in the original ticket — that logic will live in a later agent-side ticket, not in FastAPI).
- **Implementation deferred:** Concrete file paths, imports, and language-specific syntax removed per ADLC rule: "File paths and stack MUST match repo context / existing tree. Never invent another language or framework." Implementation awaits repo-tree mapping.
- **Stack-agnostic design:** Operations defined as behavioral contracts, not language-specific code. Implementation adapts to repo's actual stack.

---

**Document Status:** Draft  
**Story:** 02-fastapi (PER-30)  
**Epic:** FastAPI SSE Integration with Agent Service (requirements.md AC1, AC4, AC5)  
**Altitude:** Story-level LLD
