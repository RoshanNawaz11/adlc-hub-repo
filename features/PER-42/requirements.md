# requirements.md

## Acceptance Criteria

### AC-1: Immediate Accept
**Given** the agent is up  
**When** `POST /agent/query` receives `{"query": "<non-empty string>"}`  
**Then** the response is HTTP 200 `{"status": "ok", "agent_run_id": "<non-empty string>"}` before the LLM or MCP call finishes

### AC-2: Reject Without a Run
**Given** `query` is missing or blank  
**When** `POST /agent/query` is called  
**Then** the response is HTTP 200 `{"status": "rejected", "agent_run_id": "<non-empty string>", "reason": "invalid query"}`. The LLM and MCP are not called. `GET /agent/runs/{agent_run_id}/events` sends `failed` with that reason and does not send `completed`

### AC-3: One Event Stream
**Given** an accepted run  
**When** the caller sends `GET /agent/runs/{agent_run_id}/events` with `Accept: text/event-stream`  
**Then** the first event is `in_progress` with `data: {"run_id": "<agent_run_id>"}`. `run_id` in every event is the same id returned by the POST. An unknown id is HTTP 404

### AC-4: Five-Second Hold on Completed Only
**Given** the background job completes with an answer  
**When** that result is ready less than 5 seconds after `in_progress`  
**Then** the stream stays on `in_progress` until 5 seconds after that event, then sends `completed` with `data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}` and ends

**Given** the background job completes with an answer ready at 5 seconds or later  
**When** the answer is ready  
**Then** `completed` is sent as soon as the answer is ready and the stream ends. The hold does not delay the POST reply

### AC-5: Failure is Immediate
**Given** `process_question` returns `{"error": "..."}` or the LLM or MCP call fails  
**When** the failure is known  
**Then** the stream sends `failed` with `data: {"run_id": "<agent_run_id>", "error": "<that message>"}`. It does not wait out the 5 seconds and does not send `completed`

### AC-6: Existing POST /query Path Unchanged
**Given** `POST /query` with `{"question": "..."}`  
**When** the request is received  
**Then** the blocking response is `{"answer": "..."}` or `{"error": "..."}`. `GET /health` is unchanged. MCP is still called only from the agent process
