---
key: PER-101
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Make run idempotency durable across workers with a shared Redis store
---

## Context

`src/agents/runtime/idempotency.py` stops a replayed queue message from running the graph twice. It keeps

finished runs in a module-level `_seen: set[str]`, through `idempotency_key(task)`, `already_processed(key)`

and `mark_processed(key)`. Its docstring says the real version should be Redis SETNX with a TTL.

Three problems follow:

* A worker restart forgets every finished run.
* Each worker process has its own set, so a redelivery on another worker runs the graph again.
* A run is only marked when it finishes, so two workers can run the same message at the same time.

This matters because a run calls non-idempotent live actions (`notify_on_call`, `create_ticket`).

`src/agents/runtime/runner.py` `run_query` is the only caller:

* It builds the key with `idempotency_key(f"{thread_id}:{service_order_id or query}")`.
* When `not resume and already_processed(key)`, it returns \`{"thread_id", "trace_id", "skipped": True,

  "reason": "idempotent replay", "values"}\`.
* It raises `RuntimeError` when a fresh start hits a thread paused mid-run (`snapshot.next`).
* It calls `mark_processed(key)` when the run ends without pausing.

`src/agents/queue/orders.py` `_run` calls `run_query`, then writes status `paused` or `done` to the Redis key

`order-run:{thread_id}`. Unexpected exceptions become status `failed`. `REDIS_URL` already exists in

`.env.example`, and `arq` is already a dependency.

This needs a new store component and a new Redis key contract shared by every worker, so it needs a design

review before coding. The requirements, store, runner and worker changes must agree with each other.

## Goal

Across restarts and worker processes, one run key executes the graph at most once at a time, and never again

after it has completed. Local runs keep working without Redis.

## Requirements

* `src/agents/runtime/idempotency.py`
  * Keep `idempotency_key(task)` unchanged.
  * Replace the module-level functions with an async store with these operations:
    * `claim(key)` returns `"claimed"`, `"running"` or `"completed"`.
    * `complete(key)` marks the run finished.
    * `release(key)` drops a claim.
  * Provide an in-memory store and a Redis store. `IDEMPOTENCY_BACKEND=memory|redis` selects one, with
    `memory` as the default so `run_so.py` works without Redis.
* Redis key contract
  * Key: `run-idempotency:{key}`.
  * Value: JSON `{"state": "running" | "completed", "thread_id": "..."}`.
  * `claim` must be atomic (SET with NX).
  * A claim expires after `IDEMPOTENCY_CLAIM_TTL_SECONDS` (default 3600, matching the default

    `ORDER_JOB_TIMEOUT`), so a crashed worker cannot block a run forever.
  * A completed entry expires after `IDEMPOTENCY_COMPLETED_TTL_SECONDS` (default 604800, matching

    `STATUS_TTL_SECONDS`).
* `run_query` in `src/agents/runtime/runner.py`
  * On a fresh start, claim after the existing paused-thread guard, so that guard's `RuntimeError` leaves no

    claim behind.
  * `"completed"`: return the existing skipped result with reason `"idempotent replay"`.
  * `"running"`: return the same shape with reason `"already running"`, without starting the graph.
  * Resume does not claim. A resumed run calls `complete` when it finishes.
  * Run ends without pausing: `complete`.
  * Run pauses, fails or is cancelled: `release`. Cancellation (`asyncio.CancelledError`) must be covered.
* Redis failures
  * If `claim` cannot reach Redis, raise `IdempotencyUnavailable` and do not start the graph.
  * If `complete` or `release` fails, log an error and keep the run's result. The claim TTL cleans up.
* `src/agents/queue/orders.py` `_run`
  * A result with reason `"already running"` must not write status to `order-run:{thread_id}`, because the

    other worker owns it. This includes the `running` status written before `run_query`.
  * The status JSON shape does not change.
* `.env.example`: document `IDEMPOTENCY_BACKEND`, `IDEMPOTENCY_CLAIM_TTL_SECONDS` and

  `IDEMPOTENCY_COMPLETED_TTL_SECONDS`.
* Rewrite the `idempotency.py` module docstring to describe the new behavior.

## Acceptance criteria

* New file `tests/test_idempotency.py`, with no live Redis (use a small fake in the test), covers:
  * The first `claim` returns `"claimed"`, and a second returns `"running"`.
  * After `complete`, `claim` returns `"completed"`.
  * After `release`, `claim` returns `"claimed"`.
  * The Redis store writes the agreed key, JSON value and both TTLs, and claims with NX.
  * A Redis error during `claim` raises `IdempotencyUnavailable`.
  * `_run` in `orders.py` writes no status for an `"already running"` result.
* `uv run pytest -q` passes.
* PR description: manual check with two workers on one Redis. The same message enqueued twice runs the graph

  once, and a replay after a worker restart is skipped.

## Out of scope

* The `order-run:{thread_id}` status contract and fastapi-backend.
* The checkpointer and its Redis TODO.
* Changing how the idempotency key is built.
* Model gateway, tracing, node logic and prompts.
