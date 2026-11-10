---
key: PER-200
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Readiness check for the agents service
---

`GET /health` in `src/agents/runtime/server.py` always returns `{"status": "ok"}`, even when the service can't do any work. That happens when the run checkpointer can't be opened, Redis is down so queued runs never start, the model gateway rejects calls, or the MCP tools are unreachable. Deploys and the dashboard can't tell "up" from "able to process an order".

**Acceptance criteria**

1. **Checks:** one check each, in `src/agents/runtime/`:
   * checkpointer (`runtime/checkpointer.py`): a run database can be opened and closed
   * Redis (`queue/subtasks.py` `redis_settings`): a ping succeeds
   * model gateway (`model/gateway.py`): the gateway URL and key are set, and a request with no model call is accepted
   * MCP (`tools/mcp_client.py`): the configured server answers its tool list

   Each returns `{name, ok, ms, error}`.
2. **Timeouts:** each check gives up after `READY_CHECK_TIMEOUT` seconds (env, default 3). A check that times out is `ok: false` with `error: "timeout"`. The checks run in parallel, so the endpoint never takes longer than one timeout.
3. **API:** `GET /ready` returns 200 with `{ready: true, checks: [...]}` when every check passes, and 503 with `ready: false` when any fails. `GET /health` stays exactly as it is.
4. **No secrets:** check errors never include the gateway key, Redis password or full URLs with credentials.
5. **Startup log:** the server logs one line per check at startup (`ready: checkpointer ok 12ms`…), using the same checks.
6. `POST /runs`, `POST /runs/{thread_id}/resume`, `GET /runs/{thread_id}` and the queue are unchanged.
7. Tests cover: all ok; each check failing; a timeout; the 200 and 503 responses; no secret in an error; checks running in parallel.

**Out of scope:** auto-restart, metrics export, and checking the backend API.
