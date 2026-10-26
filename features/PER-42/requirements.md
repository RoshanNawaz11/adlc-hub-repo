```markdown
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
```
