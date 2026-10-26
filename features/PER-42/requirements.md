# Requirements: Accept a FastAPI query immediately and publish the PER-20 answer on SSE

## Summary

When FastAPI submits a user question to `POST /agent/query`, the agent accepts it immediately with an HTTP 200 response and a run id, without waiting for the LLM or MCP call. The agent then executes the existing [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) work in the background and publishes status updates on a single Server-Sent Events stream. The 5-second hold applies only to `completed` events; failures are sent immediately.

## Acceptance criteria

### AC-1: Immediate accept

**Given** the agent is up and running  
**When** `POST /agent/query` receives `{"query": "<non-empty string>"}`  
**Then** the response is HTTP 200 `{"status": "ok", "agent_run_id": "<non-empty string>"}` before the LLM or MCP call finishes, and the handler does not wait for `process_question` to complete.

### AC-2: Reject without a run

**Given** `query` is missing or blank  
**When** `POST /agent/query` is called  
**Then** the response is HTTP 200 `{"status": "rejected", "agent_run_id": "<non-empty string>", "reason": "invalid query"}`, the LLM and MCP are not called, and `GET /agent/runs/{agent_run_id}/events` sends `failed` with that reason and does not send `completed`.

### AC-3: One event stream per run

**Given** an accepted run  
**When** the caller sends `GET /agent/runs/{agent_run_id}/events` with `Accept: text/event-stream`  
**Then** the first event is `in_progress` with `data: {"run_id": "<agent_run_id>"}`, every subsequent event includes the same `run_id`, and an unknown id returns HTTP 404.

### AC-4: Five-second hold on completed only

**Given** `process_question` returns an answer  
**When** the background job completes  
**Then** if the answer is ready less than 5 seconds after the `in_progress` event, the stream stays on `in_progress` until 5 seconds after that event, then sends `completed` with `data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}` and ends; if the answer is ready at 5 seconds or later, `completed` is sent as soon as the answer is ready and the stream ends. The hold does not delay the POST reply.

### AC-5: Failure is immediate

**Given** `process_question` returns `{"error": "..."}` or the LLM or MCP call fails  
**When** the error is detected  
**Then** the stream sends `failed` with `data: {"run_id": "<agent_run_id>", "error": "<that message>"}` immediately, without waiting for the 5-second hold and without sending `completed`.

### AC-6: Existing `/query` path unchanged

**Given** the existing `POST /query` endpoint  
**When** it receives `{"question": "..."}` and `GET /health` is called  
**Then** `POST /query` still returns the blocking `{"answer": "..."}` or `{"error": "..."}`, `GET /health` is unchanged, and MCP is still called only from the agent process.

## Notes

- The agent returns `agent_run_id` from the POST and uses that same value as `run_id` inside SSE event data.
- FastAPI forwards SSE events as sent and does not modify the `run_id` field.
- A second subscriber for the same `agent_run_id` may attach; the agent does not cancel the background job when a subscriber disconnects.
- Auth is out of scope, consistent with [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server).
- Tool names, id normalization, `MCP_URL`, and existing endpoints remain unchanged.
- Pydantic validation errors (HTTP 422 for missing or non-string `query`) are caught by a custom exception handler and converted to HTTP 200 `{"status": "rejected", "agent_run_id": "<id>", "reason": "invalid query"}` before the handler body executes.

## Verify

**Test Count:** 32 tests across 3 stories (8 + 15 + 9)

**Story 1 (PER-43: 01-agent) — AC-1, AC-2:** 8 tests verify immediate accept, rejection (including Pydantic validation and empty/whitespace query), no LLM/MCP calls on invalid input, and agent_run_id consistency in run store. Tests cover valid query response timing, background job spawn absence on rejection, empty/missing/whitespace input handling, and run store event enqueue. Tests: 1–2 (AC-1 immediate return), 3–8 (AC-2 rejection with no background job spawn).

**Story 2 (PER-44: 02-agent) — AC-3, AC-4, AC-5:** 15 tests verify SSE stream delivery with correct run_id, 5-second hold timing for fast/slow answers, immediate failure events, and stream closure. Tests cover HTTP 200 text/event-stream headers, first event in_progress, all events same run_id, 404 on unknown run_id, multi-subscriber event ordering, fast answer held to ≥5s, slow answer sent immediately with no extra hold, LLM/MCP errors sent immediately as failed, invalid query rejection as failed event, stream closure after completed or failed, subscriber disconnect (job continues), and late subscriber event replay. Tests: 1–5, 14–15 (AC-3 event stream), 6–8, 12 (AC-4 5-second hold), 9–11, 13 (AC-5 immediate failure).

**Story 3 (PER-45: 03-agent) — AC-6:** 9 tests verify existing POST /query and GET /health remain unchanged, no latency regression, MCP process isolation, tool names, and configuration stability. Tests cover POST /query blocking behavior with answer/error responses, GET /health ok response, latency regression within 10% tolerance, endpoint coexistence without interference, MCP calls originating only from main process, tool names and id normalization unchanged from baseline, and MCP_URL configuration unchanged.

All tests are listed in `stories/01-agent.md`, `stories/02-agent.md`, and `stories/03-agent.md`. Test counts align with breakdown and story verification sections: 8 + 15 + 9 = 32 total tests covering all 6 acceptance criteria.
