# Story 05-fastapi LLD: SSE Stream & Cleanup

**Ticket:** PER-33  
**Story:** 05-fastapi  
**Repo:** See LLD (stack/paths deferred to implementation based on repo context)  
**Status:** Draft

---

## Overview

Implement GET /query/{run_id}/stream endpoint as a pure SSE event forwarder. The endpoint subscribes to the agent's event stream via `AgentClient` and forwards events (`in_progress`, `completed`, `failed`) to the client in real-time. Closes the stream after a terminal event and cleans up the run registry mapping. Includes run_id validation, agent subscription timeout handling, and client disconnection detection.

**CRITICAL CHANGE (per change_requests.md):** FastAPI does **not** create, decide, or delay run status. The agent owns all timing logic, including the 5-second delay. This endpoint is a pure pass-through: it forwards the agent's events exactly as received, with no timer logic, no `in_progress`/`completed`/`failed` generation.

---

## Acceptance Criteria

### AC2: SSE stream forwarding
**Given** a query has been submitted  
**When** the SSE stream is opened for that run  
**Then** the stream forwards the status and response events the agent already sent

### AC3: No status creation
**Given** the integration is running  
**When** processing requests  
**Then** FastAPI does not create, decide, or delay run status (no timer logic, no in_progress/completed/failed generation)

### AC4: No polling behavior
**Given** the integration is running  
**When** waiting for agent responses  
**Then** FastAPI does not poll the agent for status or responses

---

## Scope

This story defines the SSE stream endpoint interface and behavior contracts. Concrete file paths, language-specific implementations, and stack choices are deferred to implementation based on actual repo context, which currently shows "(repo path not mapped; do not invent a stack)" per `repo-tree/fastapi`.

---

## Functional Requirements

### GET /query/{run_id}/stream Endpoint

**Responsibility:** Forward agent's SSE event stream to client.

**Interface:**
```
GET /query/{run_id}/stream
Accept: text/event-stream
Response: text/event-stream with events:
  - in_progress: {"run_id": "<uuid>"}
  - completed: {"run_id": "<uuid>", "response": {...}}
  - failed: {"run_id": "<uuid>", "error": "<message>"}
HTTP 200 (stream established) or 404 (run_id not found)
```

**Behavior:**
- Validates run_id exists in registry
- Looks up agent_run_id from registry
- Subscribes to agent's event stream
- Forwards events as received (pure pass-through)
- Closes stream after terminal event (completed or failed)
- Cleans up registry mapping in finally block
- NO timer logic, NO event creation, NO delay

**Response Codes:**
- `200 OK` with `Content-Type: text/event-stream`: SSE stream established
- `404 Not Found`: run_id not found in registry (query not submitted or already cleaned up)

**AC Coverage:** AC2 (forwards agent events), AC3 (no status creation), AC4 (single subscription)

---

## Endpoint Algorithm

**Processing Flow:**
1. Validate run_id exists in registry
2. Lookup agent_run_id from registry
3. Subscribe to agent's event stream (single connection)
4. For each agent event received:
   - Forward event exactly as received to SSE stream
   - If event is terminal (completed or failed), break
5. Close SSE stream
6. Clean up registry mapping in finally block

**Key Constraints:**
- Pure pass-through: forwards agent events exactly as received
- No event creation: does not generate in_progress/completed/failed
- No timer logic: does not delay or schedule events
- Single subscription: one connection to agent per SSE stream
- Cleanup always runs: registry mapping removed in finally block

**Error Handling:**
- Run_id not found in registry: HTTP 404 "Run not found"
- Agent subscription timeout (5 minutes): Send SSE failed event with "Agent timeout"
- Agent subscription error: Send SSE failed event with "Stream error"
- Client disconnection: Cleanup runs, agent subscription cancelled

---

## SSE Event Format

**Standard SSE Format:**
```
event: <type>
data: <json>

```

**Event Types:**
- `in_progress`: Run started, no response yet
- `completed`: Run finished successfully, includes response
- `failed`: Run failed, includes error message

**Example Events:**
```
event: in_progress
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000"}

event: completed
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000", "response": {"result": "success"}}

event: failed
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000", "error": "agent timeout"}
```

---

## Dependencies

**Component Dependencies:**
- Story 2: AgentClient (subscribe_to_run method)
- Story 3: RunRegistry (get_agent_run_id, remove_run methods)
- Story 4: POST /query endpoint (creates run registrations)

**Runtime Requirements:**
- SSE streaming library (framework-dependent)
- Async runtime support
- JSON serialization (typically stdlib)

**Test Requirements:**
- Test framework with async support
- SSE client test utilities
- Mocking library

---

## Implementation Guidance

When implementing this story in the actual codebase:

1. **Choose SSE library** based on repo stack (sse-starlette for FastAPI, EventSource wrapper for Node.js, etc.)
2. **Locate router/controller module** from Story 4
3. **Define endpoint handler** matching the interface contracts above
4. **Implement event generator** as async iterator/generator
5. **Use dependency injection** for AgentClient and RunRegistry (from Story 4)
6. **Handle errors** per the error translation strategy below

**Key Principles:**
- The contracts above (operation signatures, event formats, error cases) are fixed
- The implementation syntax (language, SSE library, async patterns) adapts to the repo's stack
- No timer logic, no status creation — only event forwarding and cleanup

---

## Algorithm Pseudocode

```
function stream_status(run_id, agent_client, registry):
    # Validate run_id exists
    try:
        agent_run_id = registry.get_agent_run_id(run_id)
    except KeyError:
        raise HTTPException(404, "Run not found")
    
    # Define SSE event generator
    async function event_generator():
        try:
            # Subscribe to agent event stream (single connection)
            async for agent_event in agent_client.subscribe_to_run(agent_run_id):
                # Forward event exactly as received (pure pass-through)
                event_type = agent_event["event"]
                event_data = agent_event["data"]
                
                # Serialize to SSE format
                sse_message = format("event: {event_type}\ndata: {json(event_data)}\n\n")
                yield sse_message
                
                # Close after terminal event
                if event_type in ("completed", "failed"):
                    break
        
        except TimeoutError:
            # Agent subscription timeout (5 minutes)
            yield format("event: failed\ndata: {{\"run_id\": \"{run_id}\", \"error\": \"Agent timeout\"}}\n\n")
        
        except Exception:
            # Unexpected error
            yield format("event: failed\ndata: {{\"run_id\": \"{run_id}\", \"error\": \"Stream error\"}}\n\n")
        
        finally:
            # Cleanup registry mapping (prevent memory leak)
            try:
                registry.remove_run(run_id)
            except KeyError:
                pass  # Already cleaned up
    
    # Return SSE stream response
    return SSEResponse(event_generator())
```

**Key Points:**
- Single subscription per SSE stream (no polling)
- Events forwarded exactly as received (no transformation)
- Cleanup in finally block (always runs)
- Terminal events close the stream

---

## Error Translation Strategy

| Error | SSE Event | HTTP Status | Notes |
|-------|-----------|-------------|-------|
| `KeyError` from `registry.get_agent_run_id()` | None | 404 | Run not found (query not submitted or already cleaned up) |
| Timeout from `agent_client.subscribe_to_run()` | `failed` with "Agent timeout" | 200 (stream established) | Agent took > 5 minutes |
| HTTP error from `agent_client.subscribe_to_run()` | `failed` with "Stream error" | 200 (stream established) | Agent subscription failed mid-stream |
| Client disconnects mid-stream | None | N/A (connection closed by client) | Cleanup still runs |

**Rationale:**
- 404 for missing run_id (client error: wrong ID or expired run)
- Once stream established (200), errors sent as SSE failed events
- Client disconnection detected automatically by SSE library

---

## Tests

### Test 1: SSE Stream Forwards Agent Events (Integration)

**Verification:** requirements.md AC2 (SSE stream forwards status and response events the agent already sent)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → completed
- Connect to SSE endpoint
- Assert 200 status, Content-Type: text/event-stream
- Assert 2 events received: in_progress, completed
- Assert event data matches agent events
- Assert registry cleanup called

---

### Test 2: SSE Stream Forwards Failure Event (Integration)

**Verification:** requirements.md AC2 (SSE stream forwards agent events), AC3 (FastAPI does not create status events)

**Acceptance Criteria:** AC2 (SSE stream forwarding), AC3 (no status creation)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → failed
- Connect to SSE endpoint
- Assert 2 events received: in_progress, failed
- Assert completed event never sent
- Assert registry cleanup called

---

### Test 3: SSE Stream Closes After Completed Event (Integration)

**Verification:** requirements.md AC2 (SSE stream closes after terminal event)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → completed → unexpected
- Connect to SSE endpoint
- Assert only 2 events received (unexpected not forwarded)
- Assert stream closed after completed

---

### Test 4: SSE Stream Closes After Failed Event (Integration)

**Verification:** requirements.md AC2 (SSE stream closes after terminal event)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → failed → unexpected
- Connect to SSE endpoint
- Assert only 2 events received (unexpected not forwarded)
- Assert stream closed after failed

---

### Test 5: SSE Stream Returns 404 for Missing Run ID (Unit)

**Verification:** requirements.md AC2 (validates run_id before forwarding)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock registry raises KeyError for run_id lookup
- Attempt to connect to SSE endpoint
- Assert 404 response with "Run not found" detail
- Assert agent client never called

---

### Test 6: SSE Stream Timeout Sends Failed Event (Integration)

**Verification:** requirements.md AC2 (handles agent timeout), AC3 (no artificial delay or status creation)

**Acceptance Criteria:** AC2 (SSE stream forwarding), AC3 (no status creation)

**Test Coverage:**
- Mock agent stream yielding: in_progress → timeout error
- Connect to SSE endpoint
- Assert 2 events received: in_progress, failed
- Assert failed event contains "Agent timeout" error

---

### Test 7: SSE Client Disconnection Triggers Cleanup (Integration)

**Verification:** requirements.md AC2 (cleanup on disconnection)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock long-running agent stream
- Connect to SSE endpoint
- Disconnect client after first event
- Assert registry cleanup called despite disconnection

---

### Test 8: Multiple Runs Cleanup After Terminal Events (Integration)

**Verification:** requirements.md AC2 (no memory leak)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Submit 10 queries
- Connect to 10 SSE streams
- Consume all streams fully
- Assert all 10 registry mappings cleaned up

---

## Open Questions

1. **SSE reconnection:** Should clients be able to reconnect mid-stream with `Last-Event-ID` header? Current design: no reconnection support (each stream is one-shot).

2. **Authentication on SSE endpoint:** Should SSE stream require authentication? Current design: no auth (if POST /query was authenticated, assume run_id is scoped per user).

3. **SSE event IDs:** Should we add `id: <sequence>` field to SSE events for client reconnection? Current design: no IDs (simpler implementation).

4. **Agent event schema validation:** Should we validate agent events before forwarding? Current design: forward as-is (trust agent contract).

5. **Multiple clients per run_id:** Can multiple clients connect to same SSE stream? Current design: yes (agent_run_id allows multiple subscriptions). Acceptable behavior?

6. **Stack confirmation:** Current repo-tree shows "(repo path not mapped; do not invent a stack)". Before implementing, confirm:
   - Language and web framework
   - Existing SSE library patterns
   - Async patterns and conventions
   - Router/controller organization

---

## Risks

### Risk 1: SSE client compatibility
**Impact:** Some clients (proxies, old browsers) may not support SSE or may buffer events  
**Likelihood:** Medium  
**Mitigation:** Document SSE requirements. Test with common clients. Consider WebSocket fallback if SSE proves unreliable.

### Risk 2: Agent event stream disconnection
**Impact:** SSE stream stalls, client never receives terminal event  
**Likelihood:** Medium  
**Mitigation:** 5-minute timeout on agent subscription. If agent disconnects, timeout sends failed event and closes stream.

### Risk 3: Memory leak if cleanup fails
**Impact:** Registry grows unbounded  
**Likelihood:** Low (simpler than old design)  
**Mitigation:** Cleanup in finally block (always runs). No periodic cleanup needed (no timestamps to leak).

### Risk 4: SSE library buffering
**Impact:** Events not streamed in real-time (buffered until threshold reached)  
**Likelihood:** Low (SSE libraries designed for streaming)  
**Mitigation:** Test with real agent to verify latency. If buffering occurs, adjust library settings.

### Risk 5: Client disconnection not detected
**Impact:** Registry entry not cleaned up, memory leak  
**Likelihood:** Low (SSE libraries handle disconnection)  
**Mitigation:** Test disconnection scenario (Test 7). If disconnection not detected, consider explicit ping/pong mechanism.

---

## Notes

- **No timer logic (critical):** Per change_requests.md, FastAPI does **not** create, decide, or delay run status. The agent owns all timing logic (including the 5-second delay mentioned in the original ticket). This endpoint forwards agent events exactly as received — pure pass-through.
- **No status creation:** This endpoint does not generate `in_progress`, `completed`, or `failed` events. It only forwards what the agent already sent.
- **Cleanup always runs:** Finally block ensures registry cleanup even if client disconnects or agent times out. No stale mappings left behind.
- **Terminal event handling:** Stream closes immediately after forwarding `completed` or `failed` (no further iteration on agent stream).
- **Agent subscription timeout:** 5 minutes per Story 2 (AgentClient). If agent hangs, SSE stream sends `failed` event after timeout.
- **SSE library responsibility:** SSE library handles headers (Content-Type, Cache-Control), chunked transfer encoding, and client disconnection detection automatically.
- **Error messages:** SSE `failed` events use generic messages ("Agent timeout", "Stream error"). Detailed errors logged server-side only.
- **Single subscription:** Agent client creates exactly one connection per SSE stream (no polling). Satisfies AC4.
- **Pure pass-through architecture:** This is the key invariant. FastAPI is a dumb pipe: agent → FastAPI → client. No transformation, no delay, no status authority. The agent owns the state machine; FastAPI only routes events.
- **Implementation deferred:** Concrete file paths, imports, and language-specific syntax removed per ADLC rule: "File paths and stack MUST match repo context / existing tree. Never invent another language or framework." Implementation awaits repo-tree mapping.
- **Stack-agnostic design:** Operations defined as behavioral contracts, not language-specific code. Implementation adapts to repo's actual stack.

---

**Document Status:** Draft  
**Story:** 05-fastapi (PER-33)  
**Epic:** FastAPI SSE Integration with Agent Service (requirements.md AC2, AC3, AC4)  
**Altitude:** Story-level LLD
