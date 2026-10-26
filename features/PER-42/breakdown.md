```markdown
# Breakdown: Accept a FastAPI query immediately and publish the PER-20 answer on SSE

## Overview

This breakdown translates the epic's [requirements.md](./requirements.md) and [hld.md](./hld.md) into three ordered, implementable stories. Each story covers one or more acceptance criteria from the ticket, with concrete file paths, allowed implementation patterns, and verification steps that build toward the epic's definition of done.

**Epic Acceptance Criteria:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6  
**Story Count:** 3  
**Total Test Count:** 32 (8 + 15 + 9, verified against requirements.md Verify section)

---

## Execution Order & Dependencies

```
Story 1 (01-agent) — Accept & validate
  ↓ (must complete before Story 2 starts)
Story 2 (02-agent) — Stream events & background orchestration
  ↓ (must complete before Story 3 starts)
Story 3 (03-agent) — Backward compatibility verification
```

**Rationale:**
- Story 1 establishes the `POST /agent/query` handler, run store initialization, and synchronous event enqueue (`in_progress` and rejection `failed` events); Story 2 depends on existing `run_store.create()`, `run_store.mark_failed()`, and `run_store.enqueue_event()`.
- Story 2 implements the background orchestrator, SSE stream handler, and TTL-based cleanup; Story 3 verifies it does not interfere with existing endpoints.
- All three must be complete for the epic to be marked done.

---

## Story 1: Accept FastAPI query without blocking; return agent_run_id immediately

**Repo:** `agent`

**Acceptance Criteria:**
- AC-1: Immediate accept
- AC-2: Reject without a run

**Description:**

Implement the `POST /agent/query` endpoint in `agent/main.py` to accept a user query immediately and return an HTTP 200 response with `status` and `agent_run_id` before any LLM or MCP call. The handler validates the input synchronously: if `query` is missing, `None`, or empty/whitespace-only, reject with status `"rejected"` and reason `"invalid query"`, create a run, enqueue both `in_progress` and `failed` events synchronously, and do not spawn a background job. On acceptance, generate a unique `agent_run_id`, create a run, enqueue an `in_progress` event synchronously, and spawn the background orchestrator without waiting for it to complete.

**Allowed Paths:**
- `agent/__init__.py` — module marker
- `agent/models.py` — Pydantic request/response models (`AgentQueryRequest`, `AgentQueryAccepted`, `AgentQueryRejected`)
- `agent/run_store.py` — in-memory run registry with `create()`, `get()`, `enqueue_event()`, `mark_failed()` methods
- `agent/background_task.py` — background job spawn logic (placeholder in Story 1; full implementation in Story 2)
- `agent/main.py` — implement `POST /agent/query` handler, custom exception handler for HTTP 422 conversion, run store initialization, background job spawn via `asyncio.create_task()`
- `agent/tests/test_agent_query_accept.py` — unit tests for AC-1 and AC-2

**Verification Approach:**

| Test # | Scenario | AC | Verification |
|--------|----------|----|----|
| 1 | POST with valid query returns ok with agent_run_id immediately | AC-1 | Response HTTP 200 with `{"status": "ok", "agent_run_id": "<uuid>"}` before `process_question` called |
| 2 | POST accepts and returns before background execution | AC-1 | Response received < 100ms; run status "pending" immediately after return |
| 3 | POST with empty query returns rejected with reason | AC-2 | Response HTTP 200 with `{"status": "rejected", "agent_run_id": "<uuid>", "reason": "invalid query"}` |
| 4 | POST with missing query field returns rejected | AC-2 | Custom exception handler converts HTTP 422 Pydantic validation error to HTTP 200 rejected response |
| 5 | POST with whitespace-only query returns rejected | AC-2 | Response HTTP 200 rejected after input strip and empty check |
| 6 | Rejection path does not spawn background job | AC-2 | `asyncio.create_task()` not called on rejection; `process_question` never invoked |
| 7 | Both paths return non-empty, unique agent_run_id | AC-1, AC-2 | Separate calls return different UUID v4 hex strings |
| 8 | Rejection path enqueues in_progress and failed events synchronously | AC-2 | Run store contains created run; `run.events` queue has exactly 2 events (in_progress, failed) before POST returns |

**Total Test Count: 8**

---

## Story 2: Stream PER-20 answer via SSE with in_progress, completed, or failed events

**Repo:** `agent`

**Acceptance Criteria:**
- AC-3: One event stream
- AC-4: Five-second hold on completed only
- AC-5: Failure is immediate

**Description:**

Implement the `GET /agent/runs/{agent_run_id}/events` endpoint in `agent/main.py` to serve Server-Sent Events for a given run. The handler validates the run exists (404 if unknown), then streams events in order: `in_progress` was enqueued by Story 1, followed by exactly one of `completed` or `failed`. Implement `agent/sse_handler.py` to read from the run's event queue and yield SSE-formatted events. Implement the full `agent/background_task.py` to call the existing `process_question(query)` function and enqueue the result. If the result is an answer, apply a 5-second hold only to the `completed` event: if the answer is ready < 5 seconds after `in_progress` was enqueued, wait until 5 seconds have elapsed; if it is ready >= 5 seconds after, send immediately. If the result is an error or an exception occurs, enqueue `failed` immediately (no hold) and close the stream. Multiple SSE subscribers are allowed for the same `run_id`; they all receive the same events in the same order. Add a cleanup task to delete expired runs (TTL-based, default 1 hour).

**Allowed Paths:**
- `agent/run_store.py` — extend with `enqueue_event()`, `cleanup_expired()` methods
- `agent/sse_handler.py` — SSE stream generator; yields lines in format `event: <type>\ndata: <json>\n\n`
- `agent/background_task.py` — implement `run_background_job(agent_run_id, query)` that calls `process_question()`, enforces 5-second hold, enqueues result events, handles exceptions
- `agent/main.py` — add `GET /agent/runs/{agent_run_id}/events` handler returning `StreamingResponse` with appropriate headers; add cleanup task startup on app startup
- `agent/tests/test_agent_sse_stream.py` — integration tests for AC-3, AC-4, AC-5

**Verification Approach:**

| Test # | Scenario | AC | Verification |
|--------|----------|----|----|
| 1 | GET with known run_id returns HTTP 200 text/event-stream | AC-3 | Response status 200, `Content-Type: text/event-stream`, cache/connection headers correct |
| 2 | First SSE event is in_progress with correct run_id | AC-3 | First event: `event: in_progress\ndata: {"run_id": "<agent_run_id>"}` |
| 3 | All subsequent events include same run_id | AC-3 | Every event data contains `"run_id": "<agent_run_id>"` unchanged |
| 4 | GET with unknown run_id returns HTTP 404 | AC-3 | Response status 404 with `{"detail": "Run not found"}` |
| 5 | Multiple concurrent subscribers receive identical events in order | AC-3 | Two subscribers for same run_id receive events in identical order, no desync |
| 6 | Fast answer (ready < 5s): completed held until 5s elapsed | AC-4 | Elapsed time from in_progress to completed ≥ 4.9s (allow 100ms jitter) |
| 7 | Slow answer (ready ≥ 5s): completed sent immediately when ready | AC-4 | Elapsed time from in_progress to completed ≥ 5.9s and ≤ 6.1s (tight range, no extra hold) |
| 8 | Edge case at exactly 5s: completed sent immediately | AC-4 | Answer ready at t=5.0s receives completed at t ≈ 5.0s (no extra hold) |
| 9 | LLM error: in_progress then failed immediately, no completed | AC-5 | Elapsed time < 1s; failed event sent before 5s; no completed event |
| 10 | MCP error: in_progress then failed immediately, no completed | AC-5 | Exception from MCP wrapped in failed event; elapsed < 1s; no completed |
| 11 | Invalid query rejection: in_progress then failed (from Story 1 path) | AC-5 | GET stream receives both events; error: "invalid query"; no completed |
| 12 | Stream closes after completed (exactly 2 events) | AC-4 | Last event is completed; no further data after stream close |
| 13 | Stream closes after failed (exactly 2 events) | AC-5 | Last event is failed; no further data after stream close |
| 14 | Subscriber disconnect does not cancel background job | AC-3 | Job continues after subscriber closes; new subscriber retrieves all events |
| 15 | Late subscriber (job already completed) retrieves all events | AC-3 | Events queued before subscriber connects; replay without loss |

**Total Test Count: 15**

---

## Story 3: Preserve existing POST /query and GET /health endpoints

**Repo:** `agent`

**Acceptance Criteria:**
- AC-6: Existing /query path unchanged

**Description:**

Audit and verify that the existing `POST /query` and `GET /health` endpoints in `agent/main.py` remain unchanged after Stories 1 and 2 are implemented. The `POST /query` handler must continue to block and call `process_question(question)` directly, returning `{"answer": "..."}` or `{"error": "..."}` synchronously. `GET /health` must return `{"status": "ok"}` unchanged. Verify that MCP is still called only from the main agent process (no child processes or threads), and that tool names, id normalization, and `MCP_URL` are unchanged. Confirm coexistence: both `POST /agent/query` (new, non-blocking, from Stories 1–2) and `POST /query` (existing, blocking) work correctly in the same session without interference.

**Allowed Paths:**
- `agent/main.py` — existing `POST /query` and `GET /health` handlers; audit only, no code changes
- `agent/tests/test_backward_compat.py` — verification tests for AC-6

**Verification Approach:**

| Test # | Scenario | AC | Verification |
|--------|----------|----|----|
| 1 | POST /query with valid question returns answer blocking | AC-6 | Response HTTP 200 `{"answer": "<text>"}` after process_question completes (blocking) |
| 2 | POST /query with error condition returns error blocking | AC-6 | Response HTTP 200 `{"error": "<message>"}` with blocking latency |
| 3 | POST /query latency unchanged (no regression) | AC-6 | Mean latency post-Stories ≤ baseline mean × 1.1 (10% tolerance) |
| 4 | GET /health returns ok with no side effects | AC-6 | Response HTTP 200 `{"status": "ok"}` with no exceptions |
| 5 | Both POST /agent/query and POST /query coexist without interference | AC-6 | Agent POST returns immediately; Query POST blocks; SSE stream delivers events; no global state corruption |
| 6 | MCP calls originate only from main process (no child processes) | AC-6 | All MCP HTTP calls from main FastAPI process PID; no child processes spawned |
| 7 | Tool names unchanged (exact match to baseline) | AC-6 | Current tool registry matches baseline exactly (no additions, removals, renames) |
| 8 | ID normalization logic unchanged | AC-6 | Any ID normalization in process_question or handlers produces same output as baseline |
| 9 | MCP_URL configuration unchanged | AC-6 | Current MCP_URL value matches baseline exactly |

**Total Test Count: 9**

---

## Coverage & Verification

### Acceptance Criteria Mapping

| AC | Stories | Test Count | Details |
|----|---------|-----------|---------|
| AC-1: Immediate accept | Story 1 | 2 | Tests 1–2: valid query returns ok immediately, no wait for process_question |
| AC-2: Reject without a run | Story 1 | 6 | Tests 3–8: empty/missing/whitespace query, no background job, both events enqueued synchronously |
| AC-3: One event stream | Story 2 | 7 | Tests 1–5, 14–15: event stream delivery, 404 on unknown, multi-subscriber, event replay |
| AC-4: Five-second hold | Story 2 | 4 | Tests 6–8, 12: hold timing for fast/slow/edge-case answers, stream closure |
| AC-5: Failure is immediate | Story 2 | 4 | Tests 9–11, 13: LLM/MCP errors, invalid query rejection, no hold, stream closes |
| AC-6: Existing paths unchanged | Story 3 | 9 | Tests 1–9: blocking behavior, latency, health, coexistence, process isolation, tool names |
| **Total** | **3 stories** | **32 tests** | **All 6 ACs covered, all requirements from requirements.md met** |

**Alignment with requirements.md Verify section:**
- ✓ 8 tests for AC-1, AC-2 (Story 1 immediate accept, rejection)
- ✓ 15 tests for AC-3, AC-4, AC-5 (Story 2 SSE stream, timing, failure)
- ✓ 9 tests for AC-6 (Story 3 backward compatibility)
- ✓ All test categories mentioned: immediate return, rejection, fast/slow completed, tool/LLM failure, unknown run id, existing POST /query

---

## Epic Definition of Done Alignment

From the ticket's definition of done:

> `POST /agent/query` returns `ok` or `rejected` without waiting for the model. Accepted runs execute the [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) orchestrator in the background. The SSE stream emits `in_progress`, then exactly one of `completed` or `failed`, with the 5-second rule applied only to `completed`. Tests cover immediate return, rejection, the fast and slow completed cases, a tool or LLM failure, an unknown run id, and the existing `POST /query` answer.

**Coverage:**
- ✓ Immediate return & rejection: Story 1, Tests 1–8
- ✓ Background orchestration: Story 2, Tests 6–11 (background_task.py full implementation)
- ✓ SSE stream with in_progress, completed, failed: Story 2, Tests 1–13
- ✓ 5-second hold rule: Story 2, Tests 6–8, 12
- ✓ Fast and slow completed cases: Story 2, Tests 6–7
- ✓ Tool/LLM failure: Story 2, Tests 9–10
- ✓ Unknown run id: Story 2, Test 4
- ✓ Existing POST /query: Story 3, Tests 1–3, 5, 9

---

## Implementation Locations

### Story 1 — New Files (agent/)

```
models.py              Pydantic models (AgentQueryRequest, AgentQueryAccepted, AgentQueryRejected)
run_store.py           RunStore class with create(), get(), enqueue_event(), mark_failed()
main.py (add)          POST /agent/query handler, custom exception handler, run store init
background_task.py     Placeholder run_background_job() function (full impl in Story 2)
tests/
  test_agent_query_accept.py     8 tests for AC-1, AC-2
```

### Story 2 — New/Modified Files (agent/)

```
sse_handler.py         stream_events() async generator for SSE delivery
background_task.py     Complete run_background_job() with orchestration, 5-sec hold, error handling
run_store.py (extend)  Add enqueue_event(), cleanup_expired() methods
main.py (add)          GET /agent/runs/{agent_run_id}/events handler, cleanup task startup
tests/
  test_agent_sse_stream.py       15 tests for AC-3, AC-4, AC-5
```

### Story 3 — Testing Only (agent/)

```
tests/
  test_backward_compat.py        9 tests for AC-6 (audit + regression tests, no code changes)
```

---

## Key Decisions

**From HLD & change_requests.md:**
1. Routes location: All routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) in `agent/main.py` only. No separate routes module.
2. Event queue: In-memory `asyncio.Queue` per run (PoC; upgrade to Redis for multi-process).
3. 5-second hold: Enforced via `asyncio.sleep()` in background task; non-blocking per-run.
4. Run cleanup: TTL-based (default 1 hour); configurable via `RUN_CLEANUP_TTL_SECONDS`.
5. Input validation: Synchronous O(1) check; custom exception handler converts HTTP 422 to HTTP 200 rejected response.
6. Run ID format: UUID v4 hex (32 chars, collision-free).

---

## Notes

- **No external dependencies added:** Stack (FastAPI, asyncio, uuid, json) already present per repo-context.
- **Backward compatibility:** Story 3 verifies Stories 1–2 do not break existing endpoints, tool names, MCP config, or process isolation.
- **Async-only:** No threads or child processes; all concurrency via asyncio (Stories 1–2 use `asyncio.create_task()`, `asyncio.Queue`, `asyncio.sleep()`).
- **Per-process (PoC):** In-memory run store supports single FastAPI process; multi-process deployment requires Redis upgrade (deferred).
- **Pydantic validation:** Custom exception handler ensures AC-1/AC-2 HTTP 200 contract even for FastAPI Pydantic validation errors (missing/non-string query).
```
