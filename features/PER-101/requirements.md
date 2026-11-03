# Requirements — PER-101

**Type:** Story  
**Parent:** (none)  
**Title:** Make run idempotency durable across workers with a shared Redis store  
**Status:** Draft

---

## Why

`src/agents/runtime/idempotency.py` currently uses a module-level `_seen: set[str]` to prevent replayed queue messages from running the graph twice. This approach has three critical failures:

1. A worker restart forgets every finished run.
2. Each worker process maintains its own set, so redelivery to a different worker re-runs the graph.
3. A run is marked processed only after completion, allowing concurrent execution of the same message across workers.

These failures matter because runs invoke non-idempotent live actions (`notify_on_call`, `create_ticket`). The module docstring already states the production implementation should use Redis SETNX with a TTL. This ticket implements that design.

---

## What

Replace the in-memory idempotency tracking with a durable, shared Redis store that guarantees:

- Across worker restarts and processes, one run key executes the graph at most once concurrently, and never again after completion.
- Local development runs (`run_so.py`) continue working without Redis via an in-memory fallback.

The store provides three operations:

- `claim(key)` → returns `"claimed"`, `"running"`, or `"completed"` atomically.
- `complete(key)` → marks the run finished with a TTL.
- `release(key)` → drops a claim (for paused, failed, or cancelled runs).

The runner integration changes:

- `run_query` in `src/agents/runtime/runner.py` claims before starting the graph (after the existing paused-thread guard).
- `"completed"` returns the existing skipped result shape with reason `"idempotent replay"`.
- `"running"` returns the same shape with reason `"already running"` without starting the graph.
- Resume flows do not claim; resumed runs call `complete` when they finish.
- Run completion without pause → `complete`. Run paused, failed, or cancelled → `release`.

The worker status integration:

- `_run` in `src/agents/queue/orders.py` must not write status to `order-run:{thread_id}` when the result reason is `"already running"`, because another worker owns that thread.

---

## Acceptance Criteria

### AC1: In-memory replay skips
**Given** a worker with the in-memory store (default)  
**When** a run finishes and the same key is replayed  
**Then** the second call returns `{"skipped": True, "reason": "idempotent replay"}`

### AC2: Concurrent Redis claims
**Given** a worker with the Redis store  
**When** two workers claim the same key concurrently  
**Then** one returns `"claimed"` and starts the graph, the other returns `"running"` and skips with reason `"already running"`

### AC3: Completed run returns completed
**Given** a Redis store with a completed run  
**When** the same key is claimed again  
**Then** it returns `"completed"` and skips with reason `"idempotent replay"`

### AC4: Released claim re-claimable
**Given** a claim that was released  
**When** the key is claimed again  
**Then** it returns `"claimed"` and allows the run

### AC5: TTL expires stale claim
**Given** a worker that crashes mid-run  
**When** `IDEMPOTENCY_CLAIM_TTL_SECONDS` elapses  
**Then** another worker can claim the key (TTL expires the stale claim)

### AC6: Redis unavailable raises error
**Given** Redis is unreachable during `claim`  
**When** `run_query` is called  
**Then** it raises `IdempotencyUnavailable` and does not start the graph

### AC7: Redis failure logs but preserves result
**Given** Redis is unreachable during `complete` or `release`  
**When** the operation fails  
**Then** an error is logged, the run result is preserved, and the claim TTL cleans up eventually

### AC8: Worker skips status write for already running
**Given** a worker receives a result with reason `"already running"`  
**When** `_run` in `orders.py` processes the result  
**Then** it does not write any status to `order-run:{thread_id}` (neither `running`, `done`, nor `failed`)

### AC9: Completed entry TTL
**Given** the Redis store writes a completed entry  
**When** the entry is created  
**Then** the key is `run-idempotency:{key}`, the value is JSON `{"state": "completed", "thread_id": "..."}`, and it has TTL `IDEMPOTENCY_COMPLETED_TTL_SECONDS` (default 604800)

### AC10: Claim atomicity and TTL
**Given** the Redis store writes a claim  
**When** the entry is created  
**Then** the key is `run-idempotency:{key}`, the value is JSON `{"state": "running", "thread_id": "..."}`, the operation uses SET with NX for atomicity, and it has TTL `IDEMPOTENCY_CLAIM_TTL_SECONDS` (default 3600)

### AC11: Test suite passes
**Given** a test suite in `tests/test_idempotency.py` and `tests/test_runner.py` and `tests/test_orders.py`  
**When** `uv run pytest -q` is executed  
**Then** all tests pass, covering: first claim → `"claimed"`, second claim → `"running"`, after `complete` → `"completed"`, after `release` → `"claimed"`, Redis key/value/TTL contract, Redis error → `IdempotencyUnavailable`, runner integration (claim outcomes, pause/exception/cancellation release), and worker status write guard for `"already running"`

---

## Out of Scope

- Changes to the `order-run:{thread_id}` status contract or fastapi-backend.
- The checkpointer and its Redis TODO.
- Altering how the idempotency key is built (`idempotency_key(task)` remains unchanged).
- Model gateway, tracing, node logic, prompts.

---

## References

- Ticket: PER-101
- Existing file: `src/agents/runtime/idempotency.py`
- Existing file: `src/agents/runtime/runner.py` (`run_query`)
- Existing file: `src/agents/queue/orders.py` (`_run`)
- Existing dependency: `arq` (Redis already in `.env.example`)

---

## Notes

- The design must be reviewed before coding begins (architect, test_architect, critic roles in plan).
- `IDEMPOTENCY_BACKEND=memory|redis` selects the store; `memory` is default.
- `IDEMPOTENCY_CLAIM_TTL_SECONDS` default 3600 matches `ORDER_JOB_TIMEOUT`.
- `IDEMPOTENCY_COMPLETED_TTL_SECONDS` default 604800 matches `STATUS_TTL_SECONDS`.
- `.env.example` must document all three new environment variables.
- The `idempotency.py` module docstring must be rewritten to describe the new behavior.
- Concurrency edge cases (concurrent claims, TTL expiry, Redis failures, resume without claim) are pinned as test requirements.
