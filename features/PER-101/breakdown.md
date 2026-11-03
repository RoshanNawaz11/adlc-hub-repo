# Breakdown — PER-101

**Type:** Story  
**Parent:** (none)  
**Title:** Make run idempotency durable across workers with a shared Redis store  
**Status:** Draft

---

## Overview

This breakdown splits PER-101 into three implementable stories sequenced to reach the epic's Done state: a durable Redis-backed idempotency store replacing the in-memory `_seen` set, integration into the runner and worker to guarantee at-most-once execution across processes, and documentation of the new configuration and behavior. The stories are ordered to establish the store contract first (story 1), integrate it into the runtime (story 2), and complete the documentation (story 3). Each story is a session-sized implementation slice with concrete file paths, prerequisite dependencies, and a verification approach specifying an exact test count that later LLD/tests must match.

---

## Requirements Traceability

All acceptance criteria below are copied from `requirements.md`. No new epic-level ACs are introduced. Each AC maps to exactly one story except where validation spans multiple stories (noted).

### From Requirements AC1-AC11

| AC | Maps to Stories |
|----|-----------------|
| AC1: In-memory replay skips | Story 1 (MemoryIdempotencyStore state transitions via tests 1-3), validated end-to-end in Story 2 (runner integration test 3) |
| AC2: Concurrent Redis claims (one claimed, other running) | Story 1 (RedisIdempotencyStore atomicity via SET NX, test 5), validated end-to-end in Story 2 (runner handles "running" outcome, test 2) |
| AC3: Completed run returns completed | Story 1 (store complete → claim returns "completed", tests 2 and 6), validated end-to-end in Story 2 (runner returns idempotent replay, test 1) |
| AC4: Released claim re-claimable | Story 1 (store release → claim returns "claimed", tests 3 and 8) |
| AC5: TTL expires stale claim | Story 1 (Redis claim TTL contract, test 4), Manual verification (PR) |
| AC6: Redis unavailable raises `IdempotencyUnavailable` | Story 1 (store error handling, test 9), validated end-to-end in Story 2 (runner propagates exception, test 8) |
| AC7: Redis failure logs but preserves result | Story 1 (store complete/release error logging, tests 10-11), validated end-to-end in Story 2 (runner integration preserves result after complete/release failure—implicit in all Story 2 tests with mocked store) |
| AC8: Worker skips status write for "already running" | Story 2 (worker status guard in orders.py, test 9) |
| AC9: Completed entry TTL | Story 1 (Redis `complete` TTL 604800, test 7) |
| AC10: Claim atomicity and TTL | Story 1 (Redis `SET NX EX` with TTL 3600, test 4) |
| AC11: Test suite passes | Story 1 (11 tests in test_idempotency.py) + Story 2 (9 tests in test_runner.py and test_orders.py) = **20 tests total** covering: in-memory/Redis store state transitions (tests 1-11), runner claim outcomes (tests 1-3), runner pause/exception/cancellation release (tests 4-6), runner resume flow (test 7), runner error propagation (test 8), worker status write guard (test 9) |

**AC11 Coverage Summary:** The 20 automated tests collectively satisfy AC11's requirement for test coverage across all load-bearing areas:

- **Store contracts (11 tests in Story 1):** Memory store state transitions (tests 1-3), Redis key/value/TTL contract (tests 4, 7, 9-10), Redis atomicity (test 4), Redis error handling (tests 9-11).
- **Runner integration (8 tests in Story 2):** Claim outcomes (tests 1-3), pause/exception/cancellation release logic (tests 4-6), resume flow (test 7), error propagation (test 8).
- **Worker status guard (1 test in Story 2):** Conditional status write for "already running" (test 9).
- **Manual verification (PR):** TTL expiry (AC5), two-worker end-to-end behavior (duplicate message handling, replay after restart).

---

## Dependencies

### External Dependencies

- **Redis:** Already a dependency (`REDIS_URL` in `.env.example`, `arq` in `pyproject.toml`). No new infrastructure required.
- **Python 3.11+:** Existing runtime (no change).
- **Pytest:** Existing test framework (no change).
- **redis Python package:** `redis>=5.0.0` provides `redis.asyncio.Redis` (Story 1 adds to `pyproject.toml`).

### Internal Dependencies

- **Story 2** depends on **Story 1**: The runner integration requires `IdempotencyStore`, `get_idempotency_store()`, and `IdempotencyUnavailable` to exist.
- **Story 3** depends on **Story 1** and **Story 2**: Documentation must align with the implemented contract (Redis key format, TTL defaults, error semantics).

### Shared State

- **Redis namespace:** `run-idempotency:*` is owned by this subsystem. No other component writes to it.
- **Order status keys:** `order-run:{thread_id}` is written by the worker (`src/agents/queue/orders.py`). Story 2 adds a conditional write guard but does not change the key format or schema.

---

## Story Breakdown

### Story 1: Implement Redis-backed idempotency store with in-memory fallback (PER-111)

**Repo:** `trinet-adlc-agents`

**Description:**  
Replace the module-level `_seen: set[str]` in `src/agents/runtime/idempotency.py` with a durable idempotency store backed by Redis. Provide two implementations: `MemoryIdempotencyStore` (dict-backed, for local dev without Redis) and `RedisIdempotencyStore` (uses `redis.asyncio.Redis` from `redis>=5.0.0`, for production). The store protocol defines three operations: `claim(key, thread_id)` (atomic test-and-set returning `"claimed"`, `"running"`, or `"completed"`), `complete(key, thread_id)` (mark run finished with TTL), and `release(key, thread_id)` (drop claim on pause/fail/cancel). The Redis implementation uses the key format `run-idempotency:{key}` with JSON values `{"state": "running"|"completed", "thread_id": "...", ...}` and TTLs controlled by `IDEMPOTENCY_CLAIM_TTL_SECONDS` (default 3600) and `IDEMPOTENCY_COMPLETED_TTL_SECONDS` (default 604800). Redis failures during `claim` raise `IdempotencyUnavailable`; failures during `complete` or `release` log errors but do not raise. The factory function `get_idempotency_store()` reads `IDEMPOTENCY_BACKEND=memory|redis` and returns the appropriate instance.

**Covers (Requirements):**  
AC1 (in-memory replay skips—store contract), AC2 (concurrent Redis claims—atomicity contract), AC3 (completed run returns completed—store contract), AC4 (released claim re-claimable), AC5 (TTL expires stale claim), AC6 (Redis unavailable raises—store contract), AC7 (Redis failure logs—store contract), AC9 (completed entry TTL), AC10 (claim atomicity and TTL). Story 2 validates the end-to-end behavior for AC1, AC2, AC3, AC6, AC7 via runner integration tests.

**After:** (none—this is the first story)

**Allowed Paths:**  
- `src/agents/runtime/idempotency.py` (major rewrite: remove `_seen`, `already_processed`, `mark_processed`; add `IdempotencyStore` protocol, `MemoryIdempotencyStore`, `RedisIdempotencyStore`, `get_idempotency_store()`, `IdempotencyUnavailable`, rewrite module docstring per Story 3 contract)
- `pyproject.toml` (add `redis>=5.0.0` to dependencies)
- `tests/test_idempotency.py` (new file: unit tests for both stores)

**Verification Approach:**  
**11 unit tests** in `tests/test_idempotency.py` (no live Redis; use `unittest.mock.AsyncMock`):

1. `MemoryIdempotencyStore`: first claim → `"claimed"`, second claim → `"running"`.
2. `MemoryIdempotencyStore`: after `complete`, claim → `"completed"`.
3. `MemoryIdempotencyStore`: after `release`, claim → `"claimed"`.
4. `RedisIdempotencyStore` (mocked): `SET NX EX` called with correct key format `run-idempotency:{key}` and JSON value, TTL 3600.
5. `RedisIdempotencyStore` (mocked): `SET NX` returns 0 (exists), `GET` returns `{"state": "running"}` → claim returns `"running"`.
6. `RedisIdempotencyStore` (mocked): `SET NX` returns 0, `GET` returns `{"state": "completed"}` → claim returns `"completed"`.
7. `RedisIdempotencyStore` (mocked): `complete` calls `SET` (not `SET NX`) with TTL 604800.
8. `RedisIdempotencyStore` (mocked): `release` calls `DEL`.
9. `RedisIdempotencyStore` (mocked): `claim` raises `redis.exceptions.ConnectionError` → raises `IdempotencyUnavailable`.
10. `RedisIdempotencyStore` (mocked): `complete` raises connection error → no exception, error logged (use `caplog`).
11. `RedisIdempotencyStore` (mocked): `release` raises connection error → no exception, error logged.

**Test count: 11 tests.**

**Notes:**  
- `idempotency_key(task)` function remains unchanged (still exported, used by runner).
- The module docstring rewrite (part of this story per LLD) must align with Story 3's `.env.example` entries (the contract is established together, but documented in Story 3).
- The Redis TTL defaults (3600, 604800) must match `ORDER_JOB_TIMEOUT` and `STATUS_TTL_SECONDS` (cross-check during review).

---

### Story 2: Integrate idempotency store into runner and worker (PER-112)

**Repo:** `trinet-adlc-agents`

**Description:**  
Integrate the idempotency store (from Story 1) into `run_query` in `src/agents/runtime/runner.py` and `_run` in `src/agents/queue/orders.py` to guarantee at-most-once execution across workers. The runner claims before starting the graph (after the existing paused-thread guard), handles claim outcomes (`"claimed"` → proceed, `"running"` → skip with reason `"already running"`, `"completed"` → skip with reason `"idempotent replay"`), wraps `graph.ainvoke` in `try/except asyncio.CancelledError` to release on cancellation, and calls `complete` when the run finishes or `release` when it pauses/fails/cancels. Resume flows do not claim; resumed runs call `complete` when they finish (marking the store entry as `completed` and triggering the worker's `done` status write). Resumed runs that pause again call `release` (same as fresh runs), allowing the run to be retried later. The worker adds a status write guard: when `run_query` returns `{"reason": "already running"}`, skip all status writes to `order-run:{thread_id}` (including the initial `running` write) because another worker owns that thread. Redis failures during `claim` propagate `IdempotencyUnavailable` to `_run` (which logs and marks the job `failed`); failures during `complete` or `release` are logged but do not raise.

**Covers (Requirements):**  
AC1 (in-memory replay skips—end-to-end runner validation), AC2 (concurrent claims—runner handles "running" outcome), AC3 (completed run skips—runner returns idempotent replay), AC6 (claim failure raises—runner propagates), AC7 (complete/release failure logged—runner integration preserves result), AC8 (worker skips status write for already running).

**After:** Story 1 (requires `IdempotencyStore`, `get_idempotency_store()`, `IdempotencyUnavailable`)

**Allowed Paths:**  
- `src/agents/runtime/runner.py` (modify `run_query`: add store instantiation, claim logic, cancellation handling, complete/release calls)
- `src/agents/queue/orders.py` (modify `_run`: add status write guard checking `result.get("reason") == "already running"`)
- `tests/test_runner.py` (new file: 8 integration tests for `run_query` with mocked store)
- `tests/test_orders.py` (new file: 1 test for worker status write guard)

**Verification Approach:**  
**9 integration/worker tests** (8 in `tests/test_runner.py`, 1 in `tests/test_orders.py`):

1. `tests/test_runner.py`: `claim` returns `"completed"` → `run_query` returns `{"skipped": True, "reason": "idempotent replay"}`, `graph.ainvoke` not called.
2. `tests/test_runner.py`: `claim` returns `"running"` → `run_query` returns `{"skipped": True, "reason": "already running"}`, `graph.ainvoke` not called.
3. `tests/test_runner.py`: `claim` returns `"claimed"` → graph runs, `complete` called.
4. `tests/test_runner.py`: graph pauses (`snapshot.next` exists) → `release` called, not `complete`.
5. `tests/test_runner.py`: graph raises exception → `release` called before re-raise.
6. `tests/test_runner.py`: `graph.ainvoke` raises `asyncio.CancelledError` → `release` called, exception re-raised.
7. `tests/test_runner.py`: `run_query(..., resume=True)` → `claim` not called, `complete` called when run finishes.
8. `tests/test_runner.py`: `claim` raises `IdempotencyUnavailable` → exception propagates.
9. `tests/test_orders.py`: mock `run_query` returns `{"reason": "already running"}` → `redis.set()` not called for `order-run:{thread_id}`.

**Test count: 9 tests.**

**Notes:**  
- The claim point in `run_query` must be after the paused-thread guard (`if not resume and snapshot.next: raise RuntimeError`) so the guard's exception leaves no claim behind.
- Resume flows (where `resume=True`) skip the claim but still call `complete` when the run finishes. This is load-bearing for resumed runs: `complete` marks the store entry as `completed` (transitioning from `running` to `completed`), and the worker writes `done` status to `order-run:{thread_id}`. Resumed runs that pause again call `release` (same as fresh runs), deleting the store entry and allowing the run to be retried.
- The status write guard in `_run` must check the reason **before** writing the `running` status; otherwise Worker B would overwrite Worker A's status. The simplest fix is to call `run_query` first, check the reason, and return early if `"already running"`.

---

### Story 3: Document idempotency configuration and update module docstring (PER-113)

**Repo:** `trinet-adlc-agents`

**Description:**  
Document the three new environment variables controlling the idempotency subsystem in `.env.example` with inline comments: `IDEMPOTENCY_BACKEND=memory` (default, selects store; allowed values `memory` or `redis`), `IDEMPOTENCY_CLAIM_TTL_SECONDS=3600` (claim TTL in seconds, must be >= `ORDER_JOB_TIMEOUT` to avoid duplicate execution), `IDEMPOTENCY_COMPLETED_TTL_SECONDS=604800` (completed run TTL in seconds, bounds memory usage, matches `STATUS_TTL_SECONDS`). The comments must reference the related config variables (`ORDER_JOB_TIMEOUT`, `STATUS_TTL_SECONDS`) to reinforce the alignment. The module docstring in `src/agents/runtime/idempotency.py` is rewritten (as part of Story 1's file changes but validated here) to describe: the purpose (prevent duplicate execution across workers), the store abstraction (`IdempotencyStore` protocol with `claim`/`complete`/`release`), the two implementations (memory and Redis), configuration (via `IDEMPOTENCY_BACKEND`), the Redis key contract (`run-idempotency:{key}`, JSON value schema, TTLs), claim semantics (`"claimed"`, `"running"`, `"completed"`), error handling (claim failures raise `IdempotencyUnavailable`, complete/release failures log), and TTL cleanup (claim TTL expires stale claims, completed TTL bounds memory).

**Covers (Requirements):**  
`.env.example` documents `IDEMPOTENCY_BACKEND`, `IDEMPOTENCY_CLAIM_TTL_SECONDS`, `IDEMPOTENCY_COMPLETED_TTL_SECONDS`; `idempotency.py` module docstring describes new behavior.

**After:** Story 1 (module docstring rewrite happens in Story 1's file changes), Story 2 (documentation must align with implemented integration)

**Allowed Paths:**  
- `.env.example` (add three new variables with inline comments)
- `src/agents/runtime/idempotency.py` (validate module docstring against this story's contract—no code changes, docstring already rewritten in Story 1)

**Verification Approach:**  
**Manual verification** (no automated tests; human review via PR):

1. Open `.env.example`. Verify all three variables are present with the specified defaults and inline comments. Check that the comments reference `ORDER_JOB_TIMEOUT` and `STATUS_TTL_SECONDS` where applicable.
2. Open `src/agents/runtime/idempotency.py`. Verify the module docstring (written in Story 1) includes all required sections: Purpose, Store Operations, Redis Key Contract, TTLs, Atomicity, Error Handling, Integration. Cross-check against HLD AD-1, AD-2, AD-5, AD-6 for alignment.
3. Verify the new `.env.example` entries follow the same format and placement as existing queue/worker config (e.g. near `REDIS_URL`, `ORDER_JOB_TIMEOUT`).

**Test count: 0 automated tests** (documentation-only verification).

**Notes:**  
- This story is documentation-only; no tests are required. The contracts documented here are verified by the tests in Stories 1 and 2.
- The module docstring serves as the canonical reference for the idempotency subsystem after this story merges. All future changes to the store, config, or integration must update the docstring.
- The `.env.example` comments are the operator-facing contract. Keep them concise (one-line) but load-bearing (must state the constraint and default).

---

## Risks & Mitigations

### Risk 1: Redis Single Point of Failure

**Impact:** If Redis is down, all workers fail fast on `claim` and cannot start runs.

**Mitigation:** Redis is already a required dependency (`arq` queue). The system's availability already depends on Redis. Fail-fast on `claim` prevents silent corruption. Operational monitoring should alert on Redis downtime.

**Residual risk:** Low. The failure mode is explicit (`IdempotencyUnavailable` logged and job marked failed).

---

### Risk 2: Claim TTL Too Short Causes Duplicate Execution

**Impact:** If `IDEMPOTENCY_CLAIM_TTL_SECONDS < typical_job_duration`, a long-running job's claim will expire mid-run, allowing a second worker to claim and start a duplicate execution.

**Mitigation:** Default TTL (3600s) matches `ORDER_JOB_TIMEOUT`. Document the relationship in `.env.example` (Story 3). Recommend adding a startup validation warning if `IDEMPOTENCY_CLAIM_TTL_SECONDS < ORDER_JOB_TIMEOUT` (future work, out of scope).

**Residual risk:** Medium. Operational misconfiguration could cause duplicates. Manual verification (PR description) will test the default case.

---

### Risk 3: `complete` Failure Leaves Stale `running` Entry

**Impact:** If Redis is unreachable at completion time, the entry remains `running` until the claim TTL expires. Retries within the TTL window will skip (return `"running"`), even though the run actually finished.

**Mitigation:** Log the failure. The claim TTL bounds the unavailability window (default 1 hour). The trade-off favors preserving the run result over blocking retries—non-idempotent actions (`create_ticket`, `notify_on_call`) have already executed.

**Residual risk:** Medium. Acceptable because re-running on `complete` failure would duplicate non-idempotent actions.

---

### Risk 4: Worker Crash Between `claim` and First Status Write

**Impact:** Worker A claims and crashes before writing `running` status to `order-run:{thread_id}`. The job appears stuck. Worker B cannot claim (entry is `running`). Claim TTL eventually expires, allowing retry.

**Mitigation:** Existing `ORDER_JOB_TIMEOUT` already bounds this. The status write and claim are not atomic, but the claim TTL provides eventual recovery.

**Residual risk:** Low. This is an existing failure mode; the idempotency layer does not worsen it.

---

### Risk 5: Concurrent Completion Race

**Impact:** Worker A's claim expires while the graph is still running. Worker B claims and starts a second execution. Worker A finishes and calls `complete` while Worker B is running. The last writer wins (Worker A or B), and both executions' `thread_id` values appear in separate `order-run:{thread_id}` status keys.

**Mitigation:** The default claim TTL (3600s) is tuned to typical job duration. If jobs routinely exceed 1 hour, operators must increase both `IDEMPOTENCY_CLAIM_TTL_SECONDS` and `ORDER_JOB_TIMEOUT`.

**Residual risk:** Medium. Rare in practice (requires a job to run longer than the TTL and Redis to be available). Observable via duplicate status keys.

---

## Story Ordering Rationale

1. **Story 1 first:** Establishes the store contract (protocol, implementations, factory, error handling). No dependencies. Self-contained and testable in isolation (mocked Redis client).
2. **Story 2 next:** Integrates the store into the runtime. Depends on Story 1 (requires `IdempotencyStore` interface and factory). Changes the runner and worker behavior; tests verify the integration (mocked store).
3. **Story 3 last:** Documents the configuration and behavior. Depends on Stories 1 and 2 (must align with implemented contract). No code changes; human review verifies alignment.

This ordering allows Story 1 to be implemented and merged independently (establishing the contract), Story 2 to follow immediately (integration), and Story 3 to complete the documentation after the code is stable. All three stories are session-sized (Story 1 ~300 LOC + 11 tests, Story 2 ~150 LOC + 9 tests, Story 3 ~72 LOC documentation).

---

## Out of Scope

- The `order-run:{thread_id}` status contract and fastapi-backend (no changes to status key format or schema).
- The checkpointer and its Redis TODO (deferred per ticket).
- Changing how the idempotency key is built (`idempotency_key(task)` remains unchanged).
- Model gateway, tracing, node logic, prompts.
- Startup validation warning for TTL misconfiguration (future work per HLD Risk 2).

---

## Definition of Done

All three stories completed, with:

- Story 1 (PER-111): 11 unit tests passing in `tests/test_idempotency.py`, `redis>=5.0.0` dependency added to `pyproject.toml`, module docstring rewritten in `src/agents/runtime/idempotency.py`.
- Story 2 (PER-112): 9 integration/worker tests passing (8 in `tests/test_runner.py`, 1 in `tests/test_orders.py`), runner and worker integration complete in `src/agents/runtime/runner.py` and `src/agents/queue/orders.py`, no regressions in existing tests.
- Story 3 (PER-113): `.env.example` updated with three new variables and inline comments, module docstring validated against HLD.
- PR description includes manual verification steps: two workers on one Redis, same message enqueued twice runs graph once, replay after worker restart is skipped.
- `uv run pytest -q` passes (all 20 tests green: 11 from Story 1 + 9 from Story 2).
- No contradictions or omissions relative to the HLD or requirements.
