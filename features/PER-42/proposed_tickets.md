```markdown
# Proposed tickets

## 1
- title: Accept FastAPI query without blocking; return agent_run_id immediately
- repo: agent
- ac: AC-1, AC-2
- story_id: 01-agent
- type: backend-feature

## 2
- title: Stream PER-20 answer via SSE with in_progress, completed, or failed events
- repo: agent
- ac: AC-3, AC-4, AC-5
- story_id: 02-agent
- type: backend-feature

## 3
- title: Preserve existing POST /query and GET /health endpoints
- repo: agent
- ac: AC-6
- story_id: 03-agent
- type: backend-verification

## Relationships

### Parent Epic
- **Title:** Accept a FastAPI query immediately and publish the PER-20 answer on SSE
- **Ticket ID:** (assign by tracker; no ADLC ID generated in this spec)
- **Repository:** agent
- **Acceptance Criteria:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6
- **Total Test Count:** 32 tests (8 + 15 + 9, across stories 01-agent, 02-agent, 03-agent)
- **Stories:** 1 → 2 → 3 (strict order; each depends on prior)

### Child Story Dependencies

**Story 01-agent (PER-43):**
- Deliverables: POST /agent/query endpoint, run store creation, input validation, background job spawn via asyncio.create_task(), custom exception handler for HTTP 422 → 200 conversion
- Files: agent/models.py, agent/run_store.py, agent/main.py (add route), agent/background_task.py (placeholder), agent/tests/test_agent_query_accept.py (8 tests)
- Blocks: Story 02-agent (requires run_store.create(), run_store.get(), run_store.enqueue_event(), run_store.mark_failed() implemented)

**Story 02-agent (PER-44):**
- Depends on: Story 01-agent complete
- Deliverables: GET /agent/runs/{agent_run_id}/events endpoint, SSE stream handler, full background orchestrator with 5-second hold, cleanup task, TTL-based run expiration
- Files: agent/sse_handler.py, agent/background_task.py (full impl), agent/run_store.py (extend), agent/main.py (add route + cleanup), agent/tests/test_agent_sse_stream.py (15 tests)
- Blocks: Story 03-agent (requires both endpoints and background execution complete)

**Story 03-agent (PER-45):**
- Depends on: Stories 01-agent and 02-agent complete
- Deliverables: Audit and regression tests; no code changes (verification only)
- Files: agent/tests/test_backward_compat.py (9 tests)
- No blockers (final validation step)

## Acceptance Criteria Coverage

| AC | Stories | Test Count | Requirements Alignment |
|---|---------|-----------|---|
| AC-1: Immediate accept | 01-agent | 2 | requirements.md AC-1 |
| AC-2: Reject without a run | 01-agent | 6 | requirements.md AC-2 |
| AC-3: One event stream | 02-agent | 7 | requirements.md AC-3 |
| AC-4: Five-second hold | 02-agent | 4 | requirements.md AC-4 |
| AC-5: Failure is immediate | 02-agent | 4 | requirements.md AC-5 |
| AC-6: Existing paths unchanged | 03-agent | 9 | requirements.md AC-6 |
| **Total** | **3 stories** | **32 tests** | **All 6 ACs from ticket and requirements.md** |

## Implementation Constraints

### Routes Location (per change_requests.md)
All routes (POST /agent/query, GET /agent/runs/{agent_run_id}/events, existing POST /query, GET /health) registered in **`agent/main.py`** only. No separate `agent/src/`, `agent/routes/`, or `agent/routers/` tree.

### File Locations (all paths resolve from agent/ repository root)
```
agent/
├── __init__.py                                  (module marker)
├── main.py                                      (all routes; startup cleanup task)
├── models.py                                    (Pydantic request/response and event data models)
├── run_store.py                                 (RunStore singleton; Run dataclass)
├── sse_handler.py                               (SSE stream generator)
├── background_task.py                           (run_background_job async function)
├── orchestrator.py                              (existing PER-20; unchanged)
├── health.py                                    (existing PER-20; unchanged)
├── mcp_client.py                                (existing PER-20; unchanged)
├── requirements.txt                             (verify FastAPI, uvicorn, pydantic, httpx)
└── tests/
    ├── test_agent_query_accept.py               (Story 01-agent: 8 tests for AC-1, AC-2)
    ├── test_agent_sse_stream.py                 (Story 02-agent: 15 tests for AC-3, AC-4, AC-5)
    └── test_backward_compat.py                  (Story 03-agent: 9 tests for AC-6)
```

### Preserved Modules (No Changes)
- `agent/orchestrator.py` (existing PER-20 process_question)
- `agent/health.py` (existing PER-20 GET /health)
- `agent/mcp_client.py` (existing PER-20 MCP integration)

### Environment Variables (Configurable, Default Values)
- `RUN_CLEANUP_TTL_SECONDS` = 3600 (1 hour)
- `RUN_CLEANUP_INTERVAL_SECONDS` = 60 (seconds)
- `AGENT_COMPLETE_HOLD_SECONDS` = 5 (seconds, enforced in background task)
- `MCP_URL` = (existing from PER-20; unchanged)

## Test Execution Order

Execute stories in sequence (cumulative):
1. **Story 01-agent:** 8 tests (AC-1, AC-2 acceptance paths)
2. **Story 02-agent:** 15 tests (AC-3, AC-4, AC-5 SSE delivery and timing)
3. **Story 03-agent:** 9 tests (AC-6 backward compatibility audit and regression)

**Total: 32 tests** covering all 6 acceptance criteria from ticket and requirements.md.

## Key Design Decisions

- **Event Queue:** In-memory `asyncio.Queue` per run (PoC; upgrade to Redis if multi-process deployment required)
- **5-Second Hold:** Enforced via `asyncio.sleep()` in background task; non-blocking per-run, does not delay POST /agent/query response
- **Run ID Format:** UUID v4 hex (32-char string; globally unique, collision-free)
- **Input Validation:** Synchronous O(1) string check (strip + empty test) before task spawn; custom exception handler converts HTTP 422 (Pydantic validation error) to HTTP 200 rejected response per AC-1/AC-2 contract
- **Process Isolation:** Background tasks use `asyncio.create_task()` only (no threads, no child processes); MCP calls remain in main agent process
- **Cleanup Strategy:** TTL-based run expiration (default 1 hour); cleanup task runs every 60 seconds in background
- **No Auth/Rate Limiting:** Out of scope per AC-6 and requirements; assume upstream FastAPI middleware

## Deferred Items

- **Run history API:** No `GET /agent/runs` or `GET /agent/runs/{id}` metadata endpoint in this epic; query-only via SSE stream
- **Persistent event store:** In-memory queue is PoC; Redis upgrade when multi-process deployment required
- **Metrics/logging:** No instrumentation of latency, error rates, or run duration; basic startup logs only
- **Per-user subscription limits:** No cap on concurrent SSE subscribers; add if resource contention surfaces
- **.env.example or config schema:** Deferred; document all env vars in deployment guide or README
- **Custom validation error messages:** HTTP 200 response always returns `"reason": "invalid query"` (single error message); no field-level validation details

## Dependencies & Assumptions

### External (PER-20)
- **Stable Contract:** `process_question(query: str) → {"answer": str} | {"error": str}` (signature and return type frozen per AC-6)
- **No Changes Required:** PER-20 orchestrator, tool names, MCP URL, ID normalization logic all unchanged
- **Process Isolation:** MCP calls originate from main agent process only; PER-20 does not spawn child processes

### External (PER-28)
- **Consumer:** Consumes SSE events published by this epic's GET /agent/runs/{agent_run_id}/events endpoint
- **No Changes:** PER-28 reads events as sent; no modifications to SSE format or event ordering

### Internal (Same Repo)
- **FastAPI Version:** 0.104+ (already in requirements.txt)
- **Python Version:** 3.11+
- **Stack:** asyncio, uuid, json, httpx (all present in existing stack; no new external dependencies)

## Known Risks & Mitigations

| Risk | Severity | Mitigation |
|------|----------|-----------|
| Event loss if process crashes before SSE subscribes | Medium | Events queued in Run.events before subscriber connects; replay on stream open; TTL cleanup prevents stale runs |
| Unbounded run accumulation in memory | High | TTL-based cleanup; delete runs 1 hour after creation; configurable via RUN_CLEANUP_TTL_SECONDS |
| 5-second hold blocks other work | Low | Hold enforced per-run via asyncio.sleep() in background task; non-blocking per-run; does not block event queue or other jobs |
| Multiple subscribers desync | Low | All subscribers read from same queue in order; identical event timing and content guaranteed |
| Unknown run_id returns 404 immediately | Low | Keep runs ≥ 1 hour; clarify API docs: 404 means run never created or TTL expired |
| PER-20 contract changes | High | Design depends on current process_question return type; any change must preserve {"answer": "…"} \| {"error": "…"}; lock contract in AC-6 tests |
| HTTP 422 bypasses custom exception handler | Medium | Handler must be registered BEFORE routes; verify handler registration order in app startup |
| TTL boundary condition ambiguity | Low | Cleanup uses strict inequality: delete if (now - created_at) > ttl_seconds; run at exactly ttl_seconds is not deleted (safe boundary) |

## Verification Checklist (For Build)

Before marking epic done:

- [ ] All 32 tests pass (8 + 15 + 9 across stories 01-agent, 02-agent, 03-agent)
- [ ] POST /agent/query returns HTTP 200 immediately (< 100ms) before process_question called
- [ ] POST /agent/query accepts valid query and returns agent_run_id
- [ ] POST /agent/query rejects empty/missing/whitespace query with HTTP 200 rejected response
- [ ] Custom exception handler converts HTTP 422 to HTTP 200 rejected response
- [ ] GET /agent/runs/{agent_run_id}/events returns HTTP 200 text/event-stream or 404 on unknown id
- [ ] SSE stream delivers in_progress event first, then exactly one of (completed | failed)
- [ ] Fast answer (< 5s) held until 5s elapsed before completed sent
- [ ] Slow answer (≥ 5s) completed sent immediately with no extra hold
- [ ] Failed event sent immediately on error; no hold; no completed event sent
- [ ] Multiple subscribers receive identical events in same order
- [ ] Subscriber disconnect does not cancel background job; new subscriber retrieves all queued events
- [ ] POST /query still blocks and returns {"answer": "…"} or {"error": "…"}
- [ ] GET /health still returns {"status": "ok"}
- [ ] POST /query latency unchanged (no regression; ≤ baseline + 10% tolerance)
- [ ] MCP calls originate from main process only (no child processes)
- [ ] Tool names unchanged from baseline
- [ ] ID normalization logic unchanged from baseline
- [ ] MCP_URL configuration unchanged from baseline
- [ ] All routes registered in agent/main.py only (no agent/src tree)
- [ ] agent/orchestrator.py, agent/health.py, agent/mcp_client.py unchanged
- [ ] Environment variables documented (RUN_CLEANUP_TTL_SECONDS, RUN_CLEANUP_INTERVAL_SECONDS, AGENT_COMPLETE_HOLD_SECONDS)
- [ ] No new external dependencies added (FastAPI, asyncio, uuid, json, httpx already present)
```
