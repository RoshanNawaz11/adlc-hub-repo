# Low-Level Design — Story 01-agent

**Type:** Story  
**Ticket ID:** PER-111 (Story 01-agent)  
**Title:** Implement Redis-backed idempotency store with in-memory fallback  
**Repo:** `trinet-adlc-agents`  
**Status:** Draft

---

## Acceptance Criteria

From **requirements.md** and **breakdown.md** (no additions, unchanged):

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
**When** `claim` is called  
**Then** it raises `IdempotencyUnavailable` and does not start the graph

### AC7: Redis failure logs but preserves result
**Given** Redis is unreachable during `complete` or `release`  
**When** the operation fails  
**Then** an error is logged, the run result is preserved, and the claim TTL cleans up eventually

### AC9: Completed entry TTL
**Given** the Redis store writes a completed entry  
**When** the entry is created  
**Then** the key is `run-idempotency:{key}`, the value is JSON `{"state": "completed", "thread_id": "..."}`, and it has TTL `IDEMPOTENCY_COMPLETED_TTL_SECONDS` (default 604800)

### AC10: Claim atomicity and TTL
**Given** the Redis store writes a claim  
**When** the entry is created  
**Then** the key is `run-idempotency:{key}`, the value is JSON `{"state": "running", "thread_id": "..."}`, the operation uses SET with NX for atomicity, and it has TTL `IDEMPOTENCY_CLAIM_TTL_SECONDS` (default 3600)

---

## File Plan

### New Files

| Path | Purpose | Lines (est.) |
|------|---------|--------------|
| `tests/test_idempotency.py` | Unit tests for both stores (11 tests, no live Redis) | ~350 |

### Modified Files

| Path | Change Summary | Lines (est.) |
|------|----------------|--------------|
| `src/agents/runtime/idempotency.py` | **Major rewrite.** Remove `_seen`, `already_processed`, `mark_processed`. Add `IdempotencyStore` protocol, `MemoryIdempotencyStore`, `RedisIdempotencyStore`, `get_idempotency_store()`, `IdempotencyUnavailable`. Rewrite module docstring. | ~300 (net) |
| `pyproject.toml` | Add `redis>=5.0.0` to dependencies | +1 line |

**Total estimated LOC:** ~651 (300 implementation + 1 dependency + 350 tests).

---

## Interfaces

### `IdempotencyStore` Protocol

```python
# src/agents/runtime/idempotency.py

from typing import Protocol, Literal

class IdempotencyStore(Protocol):
    async def claim(self, key: str, thread_id: str) -> Literal["claimed", "running", "completed"]:
        """
        Atomically claim the run key.
        
        Returns:
            "claimed": This worker now owns the run. Proceed to execute.
            "running": Another worker owns the run. Skip execution.
            "completed": The run already finished. Skip execution.
        
        Raises:
            IdempotencyUnavailable: Backend is unreachable and claim cannot be guaranteed.
        """
        ...

    async def complete(self, key: str, thread_id: str) -> None:
        """
        Mark the run as completed. Idempotent (safe to call multiple times).
        Logs error if backend is unreachable, does not raise.
        """
        ...

    async def release(self, key: str, thread_id: str) -> None:
        """
        Release the claim (run paused, failed, or cancelled).
        Logs error if backend is unreachable, does not raise.
        """
        ...
```

### `IdempotencyUnavailable` Exception

```python
# src/agents/runtime/idempotency.py

class IdempotencyUnavailable(Exception):
    """Raised when the idempotency backend is unreachable during claim."""
    pass
```

### Factory Function

```python
# src/agents/runtime/idempotency.py

def get_idempotency_store() -> IdempotencyStore:
    """
    Factory function to instantiate the configured idempotency store.
    
    Reads IDEMPOTENCY_BACKEND env var (default: "memory").
    - "memory": Returns MemoryIdempotencyStore (in-process dict, no Redis).
    - "redis": Returns RedisIdempotencyStore (requires REDIS_URL).
    
    Raises:
        ValueError: Unknown IDEMPOTENCY_BACKEND value or missing REDIS_URL when backend=redis.
    """
    ...
```

### Unchanged Exports

```python
# src/agents/runtime/idempotency.py

def idempotency_key(task: str) -> str:
    """
    Generate idempotency key from task string. Unchanged from original.
    
    Args:
        task: Task identifier (e.g. "{thread_id}:{query}").
    
    Returns:
        Idempotency key (currently returns task unchanged; could hash in future).
    """
    ...
```

---

## Implementation Details

### `MemoryIdempotencyStore`

**Purpose:** In-memory dict-backed store for local development (no Redis required).

**State structure:**
```python
self._state: dict[str, dict] = {}
# Key: idempotency key (str)
# Value: {"state": "running" | "completed", "thread_id": str, "claimed_at": str, "completed_at": str (optional)}
```

**Concurrency:** Uses `asyncio.Lock` to serialize access (dict is not thread-safe in async context).

**Algorithm:**

- **`claim(key, thread_id)`:**
  1. Acquire lock.
  2. If `key` not in `_state`:
     - Insert `{"state": "running", "thread_id": thread_id, "claimed_at": <iso8601>}`.
     - Return `"claimed"`.
  3. Else return `_state[key]["state"]` (either `"running"` or `"completed"`).

- **`complete(key, thread_id)`:**
  1. Acquire lock.
  2. Update `_state[key]` to `{"state": "completed", "thread_id": thread_id, "claimed_at": <existing or now>, "completed_at": <iso8601>}`.
  3. (No TTL in memory store; acceptable for local dev.)

- **`release(key, thread_id)`:**
  1. Acquire lock.
  2. Delete `_state[key]` if present (`.pop(key, None)`).

**Edge cases:**
- Calling `complete` before `claim`: Creates the entry (idempotent).
- Calling `release` on non-existent key: No-op.

---

### `RedisIdempotencyStore`

**Purpose:** Redis-backed store for production (durable, shared across workers).

**Dependencies:**
- `redis.asyncio.Redis` (from `redis>=5.0.0` package).
- Environment variables: `REDIS_URL`, `IDEMPOTENCY_CLAIM_TTL_SECONDS` (default 3600), `IDEMPOTENCY_COMPLETED_TTL_SECONDS` (default 604800).

**Constructor:**
```python
def __init__(self, redis_url: str, claim_ttl: int, completed_ttl: int):
    self._redis = Redis.from_url(redis_url, decode_responses=True)
    self._claim_ttl = claim_ttl
    self._completed_ttl = completed_ttl
```

**Key format helper:**
```python
def _redis_key(self, key: str) -> str:
    return f"run-idempotency:{key}"
```

**Value schema (JSON string):**
```json
{
  "state": "running" | "completed",
  "thread_id": "<uuid>",
  "claimed_at": "<iso8601>",
  "completed_at": "<iso8601>"  // only when state="completed"
}
```

**Algorithm:**

- **`claim(key, thread_id)`:**
  1. Build Redis key: `redis_key = self._redis_key(key)`.
  2. Build JSON value: `{"state": "running", "thread_id": thread_id, "claimed_at": <now>}`.
  3. Try atomic claim: `result = await self._redis.set(redis_key, value, nx=True, ex=self._claim_ttl)`.
     - `nx=True`: Only set if key does not exist (atomic test-and-set).
     - `ex=claim_ttl`: Set TTL on the key (expires stale claims).
  4. If `result` is `True` (claim succeeded), return `"claimed"`.
  5. Else (key already exists):
     - Read existing value: `existing = await self._redis.get(redis_key)`.
     - If `existing` is `None` (race: key expired between SET NX and GET), return `"claimed"` (treat as fresh claim).
     - Parse JSON: `data = json.loads(existing)`.
     - Return `data["state"]` (either `"running"` or `"completed"`).
  6. On `redis.exceptions.ConnectionError`: raise `IdempotencyUnavailable(f"Cannot reach Redis: {e}")`.

- **`complete(key, thread_id)`:**
  1. Build Redis key: `redis_key = self._redis_key(key)`.
  2. Build JSON value: `{"state": "completed", "thread_id": thread_id, "claimed_at": <now>, "completed_at": <now>}`.
     - (Note: `claimed_at` is re-set to current time; the original claim time is not preserved. This is acceptable per HLD.)
  3. Unconditional SET (not NX): `await self._redis.set(redis_key, value, ex=self._completed_ttl)`.
     - Overwrites any existing entry (idempotent).
     - TTL is `completed_ttl` (default 604800, 7 days).
  4. On `redis.exceptions.ConnectionError`: log `logger.error(f"Failed to mark run {key} complete: {e}")`. Do not raise.

- **`release(key, thread_id)`:**
  1. Build Redis key: `redis_key = self._redis_key(key)`.
  2. Delete: `await self._redis.delete(redis_key)`.
  3. On `redis.exceptions.ConnectionError`: log `logger.error(f"Failed to release claim {key}: {e}")`. Do not raise.

**Edge cases:**
- Concurrent claims: Redis `SET NX` is atomic; only one worker gets `result=True`.
- Claim expires mid-run: Another worker can claim after TTL. The first worker's `complete` call overwrites (last writer wins). Both workers' `thread_id` values are recorded in separate `order-run:{thread_id}` status keys (handled in story 2).
- `complete` called twice: Idempotent (same JSON, TTL refreshed).
- `release` on non-existent key: Redis `DEL` returns 0 (no-op).

---

### `get_idempotency_store()` Factory

**Algorithm:**
1. Read `IDEMPOTENCY_BACKEND` env var (default `"memory"`).
2. If `backend == "memory"`: return `MemoryIdempotencyStore()`.
3. Elif `backend == "redis"`:
   - Read `REDIS_URL` env var. If missing, raise `ValueError("REDIS_URL must be set when IDEMPOTENCY_BACKEND=redis")`.
   - Read `IDEMPOTENCY_CLAIM_TTL_SECONDS` env var, parse as int (default 3600).
   - Read `IDEMPOTENCY_COMPLETED_TTL_SECONDS` env var, parse as int (default 604800).
   - Return `RedisIdempotencyStore(redis_url, claim_ttl, completed_ttl)`.
4. Else: raise `ValueError(f"Unknown IDEMPOTENCY_BACKEND: {backend}")`.

**Error handling:**
- `ValueError` if backend is unknown or `REDIS_URL` is missing when `backend=redis`.
- `int()` parse failures on TTL env vars: propagate the exception (caller's responsibility to set valid integers).

---

### Module Docstring

**New docstring** (per story 3 contract, implemented here):

```python
"""
Durable idempotency for agent runs across worker restarts and processes.

This module prevents replayed queue messages from executing the graph multiple times
by tracking run keys in a shared store. The store abstraction (`IdempotencyStore`)
supports two backends:

- `MemoryIdempotencyStore`: In-process dict-backed store for local development.
  No Redis required. State is lost on restart (acceptable for dev).
- `RedisIdempotencyStore`: Durable shared state backed by Redis. Required for
  production deployments with multiple workers.

Configuration
-------------
Set `IDEMPOTENCY_BACKEND=memory|redis` to select the store (default: memory).
Factory function `get_idempotency_store()` reads the variable and validates it.

Store Operations
----------------
- `claim(key, thread_id)`: Atomically claim a run key. Returns:
  - `"claimed"`: This worker now owns the run. Proceed to execute the graph.
  - `"running"`: Another worker currently owns the run. Skip execution.
  - `"completed"`: The run already finished. Skip execution.
  
- `complete(key, thread_id)`: Mark the run as completed. Idempotent (safe to call
  multiple times). Sets a TTL to bound memory usage.
  
- `release(key, thread_id)`: Drop the claim (called when a run pauses, fails, or
  is cancelled). Allows the run to be retried later.

Redis Key Contract
------------------
Key format: `run-idempotency:{idempotency_key}`
Value: JSON `{"state": "running"|"completed", "thread_id": "<uuid>", "claimed_at": "<iso8601>", ...}`

TTLs:
- Claim TTL (`IDEMPOTENCY_CLAIM_TTL_SECONDS`, default 3600): Expires stale claims
  from crashed workers. Must be >= ORDER_JOB_TIMEOUT to prevent duplicate execution
  of long-running jobs.
- Completed TTL (`IDEMPOTENCY_COMPLETED_TTL_SECONDS`, default 604800): Expires old
  completed runs to bound Redis memory usage. Matches STATUS_TTL_SECONDS.

Atomicity: Claims use Redis `SET ... NX EX` (atomic test-and-set). Only one worker
can claim a given key at a time.

Error Handling
--------------
- Claim failures (Redis unreachable): Raise `IdempotencyUnavailable`. The caller
  (runner) must not start the graph when idempotency cannot be guaranteed.
- Complete/release failures (Redis unreachable): Log an error but do not raise.
  The run result is preserved. The claim TTL cleans up stale entries eventually.

Integration
-----------
The runner (`src/agents/runtime/runner.py`) calls `claim` before starting the graph,
`complete` when the run finishes, and `release` when the run pauses, fails, or is
cancelled. The worker (`src/agents/queue/orders.py`) skips status writes when
another worker owns the run (claim returned `"running"`).

See Also
--------
- PER-101 HLD: Full architecture and Redis key contract.
- `src/agents/runtime/runner.py`: Integration into `run_query`.
- `src/agents/queue/orders.py`: Worker status write guard.
"""
```

---

## Algorithm Notes

### Claim Race Condition Handling

**Scenario:** Worker A and Worker B call `claim(key)` concurrently.

**Redis store behavior:**
1. Both workers call `SET run-idempotency:{key} ... NX EX 3600`.
2. Redis processes one request first (serialized at the Redis server). Let's say Worker A's SET succeeds (returns `True`), Worker B's SET fails (returns `False`).
3. Worker A returns `"claimed"`.
4. Worker B calls `GET run-idempotency:{key}`, reads `{"state": "running", ...}`, returns `"running"`.

**Memory store behavior:**
1. Both workers await `asyncio.Lock`.
2. Worker A acquires lock first, inserts entry, releases lock, returns `"claimed"`.
3. Worker B acquires lock, finds entry exists, returns `"running"`.

**Outcome:** Exactly one worker gets `"claimed"`, all others get `"running"`. No double-execution.

---

### TTL Expiry During Run

**Scenario:** Worker A claims at T=0. Claim TTL is 3600s. Worker A's graph execution takes 4000s. At T=3600, the claim expires. Worker B claims at T=3700 and starts a second execution.

**Redis behavior:**
1. At T=3600, Redis expires the key `run-idempotency:{key}`.
2. At T=3700, Worker B calls `SET ... NX`, succeeds (key does not exist), returns `"claimed"`.
3. Worker B starts the graph.
4. Worker A finishes at T=4000, calls `complete`, writes `{"state": "completed", ...}` (overwrites Worker B's claim or coexists if Worker B hasn't completed yet).

**Risk:** Duplicate execution (both Worker A and Worker B run the graph). This is acceptable per HLD Risk 2 (claim TTL must be tuned to job duration; default 3600s matches `ORDER_JOB_TIMEOUT`).

---

### Complete Idempotence

**Scenario:** Worker A calls `complete(key, thread_id)` twice (e.g. retry logic or accidental double-call).

**Redis behavior:**
1. First call: `SET run-idempotency:{key} {"state": "completed", ...} EX 604800` (creates entry).
2. Second call: Same `SET` command (overwrites with identical value, TTL refreshed to 604800s from now).

**Outcome:** No error. The completed entry persists with a fresh TTL. This is the intended idempotent behavior.

---

### Release and Re-Claim

**Scenario:** Worker A claims, graph pauses, Worker A calls `release`. Worker B claims the same key.

**Redis behavior:**
1. Worker A: `SET ... NX` → `True`, returns `"claimed"`.
2. Graph pauses. Worker A calls `release`: `DEL run-idempotency:{key}`.
3. Worker B: `SET ... NX` → `True` (key does not exist), returns `"claimed"`.

**Outcome:** Worker B can claim and start the run (resume flow or fresh retry). This is the intended behavior (paused runs are released to allow retries).

---

## Tests

### Test File: `tests/test_idempotency.py`

**Structure:**
- **11 test functions** (per breakdown verification approach).
- No live Redis (use `unittest.mock.AsyncMock` for `redis.asyncio.Redis`).
- Pytest fixtures: `memory_store`, `mocked_redis_store`.

**Fixtures:**

```python
import pytest
from unittest.mock import AsyncMock, patch
from src.agents.runtime.idempotency import (
    MemoryIdempotencyStore,
    RedisIdempotencyStore,
    IdempotencyUnavailable,
)

@pytest.fixture
def memory_store():
    return MemoryIdempotencyStore()

@pytest.fixture
def mocked_redis_store():
    """Redis store with mocked Redis client."""
    with patch("src.agents.runtime.idempotency.Redis") as mock_redis_class:
        mock_redis = AsyncMock()
        mock_redis_class.from_url.return_value = mock_redis
        store = RedisIdempotencyStore("redis://localhost", 3600, 604800)
        store._redis = mock_redis  # Replace with mock
        yield store, mock_redis
```

**Test cases:**

1. **`test_memory_store_first_claim_returns_claimed`**
   - Call `claim("key1", "tid1")` → assert returns `"claimed"`.
   - Assert `store._state["key1"]` contains `{"state": "running", "thread_id": "tid1", "claimed_at": ...}`.

2. **`test_memory_store_second_claim_returns_running`**
   - Call `claim("key1", "tid1")` → `"claimed"`.
   - Call `claim("key1", "tid2")` → assert returns `"running"`.

3. **`test_memory_store_completed_claim_returns_completed`**
   - Call `claim("key1", "tid1")` → `"claimed"`.
   - Call `complete("key1", "tid1")`.
   - Call `claim("key1", "tid2")` → assert returns `"completed"`.

4. **`test_memory_store_release_allows_reclaim`**
   - Call `claim("key1", "tid1")` → `"claimed"`.
   - Call `release("key1", "tid1")`.
   - Call `claim("key1", "tid2")` → assert returns `"claimed"`.

5. **`test_redis_store_claim_set_nx_called_with_correct_key`**
   - Mock `redis.set()` to return `True`.
   - Call `claim("key1", "tid1")`.
   - Assert `redis.set` called with `("run-idempotency:key1", <json>, nx=True, ex=3600)`.
   - Parse JSON, assert `{"state": "running", "thread_id": "tid1", ...}`.

6. **`test_redis_store_claim_existing_running_returns_running`**
   - Mock `redis.set()` to return `False` (key exists).
   - Mock `redis.get()` to return `'{"state": "running", "thread_id": "tid1", ...}'`.
   - Call `claim("key1", "tid2")` → assert returns `"running"`.

7. **`test_redis_store_claim_existing_completed_returns_completed`**
   - Mock `redis.set()` to return `False`.
   - Mock `redis.get()` to return `'{"state": "completed", "thread_id": "tid1", ...}'`.
   - Call `claim("key1", "tid2")` → assert returns `"completed"`.

8. **`test_redis_store_complete_calls_set_with_ttl`**
   - Call `complete("key1", "tid1")`.
   - Assert `redis.set` called with `("run-idempotency:key1", <json>, ex=604800)`.
   - Parse JSON, assert `{"state": "completed", ...}`.

9. **`test_redis_store_release_calls_delete`**
   - Call `release("key1", "tid1")`.
   - Assert `redis.delete` called with `("run-idempotency:key1",)`.

10. **`test_redis_store_claim_connection_error_raises`**
    - Mock `redis.set()` to raise `redis.exceptions.ConnectionError("Redis down")`.
    - Call `claim("key1", "tid1")` → assert raises `IdempotencyUnavailable`.

11. **`test_redis_store_complete_connection_error_logs`** (uses `caplog` fixture)
    - Mock `redis.set()` to raise `redis.exceptions.ConnectionError("Redis down")`.
    - Call `complete("key1", "tid1")` → assert no exception raised.
    - Assert `caplog` contains `"Failed to mark run key1 complete: Redis down"`.

**Total: 11 tests** (matching breakdown verification approach count).

**Test setup notes:**
- Use `pytest.mark.asyncio` for async test functions.
- Mock `redis.asyncio.Redis` at import time (`patch("src.agents.runtime.idempotency.Redis")`).
- Use `AsyncMock()` for async Redis methods (`set`, `get`, `delete`).
- Use `caplog` fixture for log assertions (test 11).

---

## Dependencies

### External Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `redis` | `>=5.0.0` | Redis async client (`redis.asyncio.Redis`) |

**Added to `pyproject.toml`:**

```toml
dependencies = [
    # ... existing deps ...
    "redis>=5.0.0",
]
```

### Internal Dependencies

None (this story does not depend on other stories; it is the first in the sequence).

---

## Risks & Mitigations

### Risk 1: Redis Connection Leaks

**Impact:** If `RedisIdempotencyStore` does not close the Redis connection, long-running processes may exhaust connection pools.

**Mitigation:** `redis.asyncio.Redis.from_url()` returns a connection pool (not a single connection). The pool is shared across all `RedisIdempotencyStore` instances created by the factory. The pool is automatically closed on process exit. For explicit cleanup (e.g. in tests), add a `close()` method to the store (future work, out of scope for this story).

**Residual risk:** Low. The default connection pool behavior is acceptable for production.

---

### Risk 2: JSON Serialization Failures

**Impact:** If `json.dumps()` or `json.loads()` fails (e.g. corrupted data in Redis), the store raises an exception.

**Mitigation:** The JSON schema is simple (strings only). `json.dumps()` cannot fail on our value dicts. `json.loads()` can fail if Redis contains invalid JSON (e.g. manual edit). This is acceptable—raise the exception and let the caller handle it (the runner will fail the job).

**Residual risk:** Low. Invalid JSON in Redis is a data corruption issue, not a code bug.

---

### Risk 3: Timestamp Consistency

**Impact:** `claimed_at` and `completed_at` timestamps are generated on the worker. If workers have clock skew, timestamps may be inconsistent.

**Mitigation:** Timestamps are for observability (not load-bearing for correctness). The TTL is set by Redis (based on Redis server time, not worker time). Clock skew does not affect correctness.

**Residual risk:** None. Timestamps are informational only.

---

## Out of Scope

- Integration into `src/agents/runtime/runner.py` (story 2).
- Integration into `src/agents/queue/orders.py` (story 2).
- `.env.example` documentation (story 3).
- Module docstring validation (story 3 validates; this story implements the docstring).
- Live Redis testing (manual verification only, per HLD).
- Connection pooling configuration (use Redis client defaults).
- Startup validation warning for TTL misconfiguration (future work per HLD Risk 2).

---

## Definition of Done

- [ ] `src/agents/runtime/idempotency.py` rewritten: `IdempotencyStore` protocol, `MemoryIdempotencyStore`, `RedisIdempotencyStore`, `get_idempotency_store()`, `IdempotencyUnavailable`, module docstring.
- [ ] `pyproject.toml` updated: `redis>=5.0.0` dependency added.
- [ ] `tests/test_idempotency.py` created: 11 unit tests passing (no live Redis).
- [ ] `uv run pytest -q tests/test_idempotency.py` passes (all tests green).
- [ ] No regressions in existing tests (`uv run pytest -q` full suite passes).
- [ ] Code review confirms: Redis key format matches HLD AD-2, TTL defaults match `ORDER_JOB_TIMEOUT`/`STATUS_TTL_SECONDS`, error handling matches HLD AD-6.

---

**End of LLD — Story 01-agent**
