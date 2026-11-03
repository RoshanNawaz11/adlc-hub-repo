# Low-Level Design — Story 02-agent

**Type:** Story  
**Ticket ID:** PER-112 (Story 02-agent)  
**Title:** Integrate idempotency store into runner and worker  
**Repo:** `trinet-adlc-agents`  
**Status:** Draft

---

## Acceptance Criteria

From **requirements.md** and **breakdown.md** (no additions, unchanged):

### AC1: In-memory replay skips (validated via runner integration)
**Given** a worker with the in-memory store (default)  
**When** a run finishes and the same key is replayed  
**Then** the second call returns `{"skipped": True, "reason": "idempotent replay"}`

### AC2: Concurrent Redis claims (validated via runner integration)
**Given** a worker with the Redis store  
**When** two workers claim the same key concurrently  
**Then** one returns `"claimed"` and starts the graph, the other returns `"running"` and skips with reason `"already running"`

### AC3: Completed run returns completed (validated via runner integration)
**Given** a Redis store with a completed run  
**When** the same key is claimed again  
**Then** it returns `"completed"` and skips with reason `"idempotent replay"`

### AC6: Redis unavailable raises error (validated via runner integration)
**Given** Redis is unreachable during `claim`  
**When** `run_query` is called  
**Then** it raises `IdempotencyUnavailable` and does not start the graph

### AC7: Redis failure logs but preserves result (validated via runner integration)
**Given** Redis is unreachable during `complete` or `release`  
**When** the operation fails  
**Then** an error is logged, the run result is preserved, and the claim TTL cleans up eventually

### AC8: Worker skips status write for "already running"
**Given** a worker receives a result with reason `"already running"`  
**When** `_run` in `orders.py` processes the result  
**Then** it does not write any status to `order-run:{thread_id}` (neither `running`, `done`, nor `failed`)

---

## File Plan

### Modified Files

| Path | Change Summary | Lines (est.) |
|------|----------------|--------------|
| `src/agents/runtime/runner.py` | Modify `run_query`: add store instantiation, claim logic after paused-thread guard, wrap `graph.ainvoke` in `try/except asyncio.CancelledError`, add complete/release calls based on run outcome. Handle `IdempotencyUnavailable` propagation. | ~80 (additions/modifications) |
| `src/agents/queue/orders.py` | Modify `_run`: add status write guard checking `result.get("reason") == "already running"` before all status writes. Early return when guard triggers. | ~20 (additions/modifications) |
| `tests/test_runner.py` | Add 8 integration tests for `run_query` with mocked `get_idempotency_store()`. Cover claim outcomes, pause/exception/cancellation release logic, resume flow. | ~280 |
| `tests/test_orders.py` | Add 1 test for worker status write guard (mock `run_query` to return `"already running"`, assert no Redis status write). | ~40 |

**Total estimated LOC:** ~420 (100 implementation + 320 tests).

---

## Interfaces

### Idempotency Store Interface (from Story 1)

```python
# src/agents/runtime/idempotency.py (already implemented in Story 1)

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

### Factory Function (from Story 1)

```python
# src/agents/runtime/idempotency.py (already implemented in Story 1)

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

### Exception (from Story 1)

```python
# src/agents/runtime/idempotency.py (already implemented in Story 1)

class IdempotencyUnavailable(Exception):
    """Raised when the idempotency backend is unreachable during claim."""
    pass
```

### `run_query` Return Schema Extension

Existing return shape (from `src/agents/runtime/runner.py`):

```python
{
    "thread_id": str,
    "trace_id": str,
    "skipped": bool,           # True when run was skipped
    "reason": str | None,      # NEW: clarifies why skipped
    "values": dict             # final state values or {}
}
```

**New `reason` values added in this story:**
- `"already running"`: Another worker is executing this run. **Worker must skip all status writes.**
- `"idempotent replay"`: Run already completed. (Existing behavior, now explicit in return value.)

---

## Implementation Details

### `src/agents/runtime/runner.py` — Modify `run_query`

**Current flow (simplified from repo context):**

```python
async def run_query(thread_id, query, service_order_id=None, resume=False, ...):
    # 1. Build idempotency key
    key = idempotency_key(f"{thread_id}:{service_order_id or query}")
    
    # 2. Load snapshot from checkpointer
    snapshot = await checkpointer.aget(...)
    
    # 3. Paused-thread guard
    if not resume and snapshot and snapshot.next:
        raise RuntimeError("Thread paused mid-run, cannot start fresh")
    
    # 4. Old idempotency check (module-level _seen set, to be removed)
    # (This is replaced by store.claim in the new flow)
    
    # 5. Run graph
    result = await graph.ainvoke(...)
    
    # 6. Mark processed if not paused (module-level _seen, to be removed)
    # (This is replaced by store.complete/release in the new flow)
    
    return result
```

**New flow (changes in bold):**

```python
async def run_query(thread_id, query, service_order_id=None, resume=False, ...):
    # 1. Build idempotency key (unchanged)
    key = idempotency_key(f"{thread_id}:{service_order_id or query}")
    
    # 2. Load snapshot (unchanged)
    snapshot = await checkpointer.aget(...)
    
    # 3. Paused-thread guard (unchanged)
    if not resume and snapshot and snapshot.next:
        raise RuntimeError("Thread paused mid-run, cannot start fresh")
    
    # **4. NEW: Instantiate store**
    from .idempotency import get_idempotency_store, IdempotencyUnavailable
    store = get_idempotency_store()
    
    # **5. NEW: Claim (only if not resume)**
    if not resume:
        try:
            claim_result = await store.claim(key, thread_id)
        except IdempotencyUnavailable:
            # Propagate to _run (which logs and marks job failed)
            raise
        
        if claim_result == "completed":
            return {
                "thread_id": thread_id,
                "trace_id": trace_id,
                "skipped": True,
                "reason": "idempotent replay",
                "values": {}
            }
        elif claim_result == "running":
            return {
                "thread_id": thread_id,
                "trace_id": trace_id,
                "skipped": True,
                "reason": "already running",
                "values": {}
            }
        # else: claim_result == "claimed", proceed to run
    
    # **6. Wrap graph.ainvoke to handle exceptions and cancellation**
    try:
        result = await graph.ainvoke(...)
    except asyncio.CancelledError:
        # Release claim on cancellation
        if not resume:
            await store.release(key, thread_id)
        raise
    except Exception as e:
        # Release claim on any exception
        if not resume:
            await store.release(key, thread_id)
        raise
    
    # **7. Complete or release based on pause state**
    # Refresh snapshot to check if run paused
    snapshot = await checkpointer.aget(...)
    if snapshot.next:
        # Run paused
        if not resume:
            await store.release(key, thread_id)
    else:
        # Run completed
        await store.complete(key, thread_id)
    
    return result
```

**Key decisions:**

- **Claim point:** After the paused-thread guard, so the guard's `RuntimeError` leaves no claim behind.
- **Resume flows:** Skip `claim` entirely. Resumed runs call `complete` when they finish (marks store entry `"completed"`, worker writes `done` status). Resumed runs that pause again call `release` (deletes store entry, worker writes `paused` status, allowing the run to be retried later).
- **Cancellation:** `asyncio.CancelledError` is caught, `release` is called, then the exception is re-raised.
- **Exception handling:** All exceptions (except `IdempotencyUnavailable`) trigger `release` before re-raising.
- **Snapshot refresh:** After `graph.ainvoke`, reload the snapshot to check `snapshot.next` (paused vs. completed state).

**Imports:**

```python
# Add to top of src/agents/runtime/runner.py
from .idempotency import get_idempotency_store, IdempotencyUnavailable
import asyncio
```

**Algorithm notes:**

- `trace_id` is assumed to be available in the function scope (existing code). If not, construct it per existing logic.
- `result` from `graph.ainvoke` is returned as-is when the run completes (existing behavior preserved).
- `store.complete` and `store.release` do not raise on Redis failures (per Story 1 contract); errors are logged internally.

---

### `src/agents/queue/orders.py` — Modify `_run`

**Current flow (simplified from repo context):**

```python
async def _run(ctx, thread_id, query, service_order_id):
    # 1. Write running status to Redis
    await redis.set(f"order-run:{thread_id}", json.dumps({"status": "running", ...}))
    
    # 2. Call run_query
    try:
        result = await run_query(thread_id, query, service_order_id)
    except Exception as e:
        # Write failed status
        await redis.set(f"order-run:{thread_id}", json.dumps({"status": "failed", ...}))
        raise
    
    # 3. Write done/paused status based on result
    if result.get("skipped"):
        status = "done"  # idempotent replay case
    elif snapshot.next:
        status = "paused"
    else:
        status = "done"
    
    await redis.set(f"order-run:{thread_id}", json.dumps({"status": status, ...}))
```

**New flow (changes in bold):**

```python
async def _run(ctx, thread_id, query, service_order_id):
    # **1. Call run_query first (before writing any status)**
    try:
        result = await run_query(thread_id, query, service_order_id)
    except Exception as e:
        # Write failed status (unchanged, because run_query raised before returning)
        await redis.set(f"order-run:{thread_id}", json.dumps({"status": "failed", ...}))
        raise
    
    # **2. NEW: Guard against "already running"**
    if result.get("reason") == "already running":
        logger.info(f"Run {thread_id} already owned by another worker, skipping status write")
        return  # Early exit, no status write
    
    # **3. Write running status (only if we own the run)**
    await redis.set(f"order-run:{thread_id}", json.dumps({"status": "running", ...}))
    
    # 4. Write done/paused status (unchanged logic)
    if result.get("skipped"):
        status = "done"  # "idempotent replay" case
    elif snapshot.next:
        status = "paused"
    else:
        status = "done"
    
    await redis.set(f"order-run:{thread_id}", json.dumps({"status": status, ...}))
```

**Key decisions:**

- **Guard placement:** Immediately after `run_query` returns, before any status write.
- **Early return:** When `reason == "already running"`, skip all status writes (including the initial `running` write) and return early.
- **Idempotent replay:** When `reason == "idempotent replay"`, proceed with status writes (write `done`). This is acceptable because the run is already completed—no concurrent owner exists.
- **Exception handling:** Unchanged. If `run_query` raises (including `IdempotencyUnavailable`), the exception handler writes `failed` status and re-raises.

**Imports:**

```python
# Add to top of src/agents/queue/orders.py (if not already present)
import logging
logger = logging.getLogger(__name__)
```

**Alternative considered (rejected):** Move the `running` status write inside `run_query` after a successful claim. This would keep the status write and claim atomic but couples the runner to the worker's Redis contract. The guard approach is less coupled and aligns with the existing separation of concerns.

---

## Algorithm Notes

### Claim Point Placement

**Why after the paused-thread guard?**

The paused-thread guard (`if not resume and snapshot.next: raise RuntimeError`) prevents starting a fresh run when a thread is paused mid-execution. If we claim **before** this guard and the guard raises, the claim is left behind (stale `running` entry in the store). The claim TTL will eventually clean it up, but it blocks retries for up to 3600s.

**Sequence:**

1. Load snapshot.
2. Check paused-thread guard → raise if paused.
3. Claim → only if guard passes.

**Edge case:** If the checkpointer is slow or unavailable, the snapshot load may fail before the claim. This is acceptable—the failure prevents the run from starting, so no claim is needed.

---

### Resume Flow and `complete`

**Why resume flows skip `claim` but call `complete`?**

A resumed run is the continuation of a run that paused earlier. The original run already claimed the key when it started. When the run is resumed:

- Skipping `claim` avoids a spurious `"running"` return (the key is already `running` from the original claim).
- Calling `complete` when the resumed run finishes marks the run done (transitions the store entry from `running` to `completed`).

**Observable outcomes for resumed runs:**
- **Resumed run completes:** `complete` marks the store entry as `"completed"`, and the worker writes `done` status to `order-run:{thread_id}`.
- **Resumed run pauses again:** `release` deletes the store entry (allowing the run to be retried later), and the worker writes `paused` status to `order-run:{thread_id}`.

**Sequence for a paused run:**

1. Worker A: fresh start → `claim` → `"claimed"` → run graph → pauses → `release`.
2. Worker B: resume → skip `claim` → run graph → completes → `complete`.

**Edge case:** If the resumed run pauses again, it calls `release` (same as a fresh run). This is idempotent—multiple `release` calls on the same key are safe.

---

### Cancellation Handling

**Why wrap `graph.ainvoke` in `try/except asyncio.CancelledError`?**

`asyncio.CancelledError` is raised when a task is cancelled (e.g. worker shutdown, timeout). If we don't catch it, the claim is never released and the run is blocked until the claim TTL expires (3600s).

**Sequence:**

1. Worker A: claim → `"claimed"` → start `graph.ainvoke`.
2. Worker shutdown signal → `asyncio.CancelledError` raised.
3. Catch exception → `release(key, thread_id)` → re-raise.
4. Claim is deleted. Worker B can retry immediately.

**Edge case:** If `release` itself raises (Redis down), the exception is logged (per Story 1 contract), and the `CancelledError` is re-raised. The claim TTL provides eventual cleanup.

---

### Worker Status Write Guard Logic

**Why check `reason` before writing `running` status?**

Worker A claims and writes `running` status. Worker B calls `run_query`, gets `"already running"`, and returns. If Worker B writes `running` status after receiving this result, it overwrites Worker A's status entry with Worker B's `thread_id`. This corrupts the status (the status key should only be written by the worker that owns the run).

**Sequence:**

1. Worker A: `claim` → `"claimed"` → write `running` status.
2. Worker B: `claim` → `"running"` → check reason → early return (no status write).

**Alternative considered (rejected):** Write the `running` status unconditionally before calling `run_query`, then delete it if `reason == "already running"`. This is racy—Worker A could read the status between Worker B's write and delete, seeing an incorrect `thread_id`.

---

## Tests

### Test File: `tests/test_runner.py`

**Structure:**
- **8 new test functions** (integration tests for `run_query` with mocked store).
- Use `pytest.mark.asyncio` for async tests.
- Mock `get_idempotency_store()` to return a controllable test double (either `AsyncMock` or the real `MemoryIdempotencyStore` from Story 1).

**Fixtures:**

```python
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from src.agents.runtime.runner import run_query
from src.agents.runtime.idempotency import IdempotencyUnavailable

@pytest.fixture
def mock_store():
    """Controllable mock store for testing claim outcomes."""
    return AsyncMock()

@pytest.fixture
def mock_graph():
    """Mock LangGraph graph.ainvoke."""
    return AsyncMock()

@pytest.fixture
def mock_checkpointer():
    """Mock checkpointer.aget."""
    return AsyncMock()
```

**Test cases (8 total, validating AC1, AC2, AC3, AC6, AC7 from requirements.md via runner integration):**

1. **`test_run_query_claim_completed_skips`** (validates AC1 and AC3)
   - Mock `store.claim()` to return `"completed"`.
   - Call `run_query(thread_id="t1", query="q1")`.
   - Assert return value: `{"skipped": True, "reason": "idempotent replay", ...}`.
   - Assert `graph.ainvoke` is **not** called.

2. **`test_run_query_claim_running_skips`** (validates AC2)
   - Mock `store.claim()` to return `"running"`.
   - Call `run_query(thread_id="t1", query="q1")`.
   - Assert return value: `{"skipped": True, "reason": "already running", ...}`.
   - Assert `graph.ainvoke` is **not** called.

3. **`test_run_query_claim_succeeds_runs_graph`** (validates AC1 and AC2 happy path)
   - Mock `store.claim()` to return `"claimed"`.
   - Mock `graph.ainvoke()` to return a result.
   - Mock `checkpointer.aget()` to return a snapshot with `next=None` (run completed).
   - Call `run_query(thread_id="t1", query="q1")`.
   - Assert `store.complete(key, "t1")` is called.

4. **`test_run_query_paused_run_releases`**
   - Mock `store.claim()` to return `"claimed"`.
   - Mock `graph.ainvoke()` to return a result.
   - Mock `checkpointer.aget()` to return a snapshot with `next` exists (run paused).
   - Call `run_query(thread_id="t1", query="q1")`.
   - Assert `store.release(key, "t1")` is called, **not** `store.complete`.

5. **`test_run_query_exception_releases`**
   - Mock `store.claim()` to return `"claimed"`.
   - Mock `graph.ainvoke()` to raise `ValueError("test error")`.
   - Call `run_query(thread_id="t1", query="q1")` → expect `ValueError` raised.
   - Assert `store.release(key, "t1")` is called before the exception propagates.

6. **`test_run_query_cancellation_releases`**
   - Mock `store.claim()` to return `"claimed"`.
   - Mock `graph.ainvoke()` to raise `asyncio.CancelledError`.
   - Call `run_query(thread_id="t1", query="q1")` → expect `asyncio.CancelledError` raised.
   - Assert `store.release(key, "t1")` is called before the exception propagates.

7. **`test_run_query_resume_does_not_claim`**
   - Mock `checkpointer.aget()` to return a snapshot with `next=None` (run completes).
   - Call `run_query(thread_id="t1", query="q1", resume=True)`.
   - Assert `store.claim()` is **not** called.
   - Assert `store.complete(key, "t1")` is called when the run finishes.

8. **`test_run_query_claim_unavailable_raises`** (validates AC6)
   - Mock `store.claim()` to raise `IdempotencyUnavailable("Redis down")`.
   - Call `run_query(thread_id="t1", query="q1")` → expect `IdempotencyUnavailable` raised.
   - Assert the exception propagates (not caught).

**Test setup notes:**

- Patch `get_idempotency_store()` at the module level (`src.agents.runtime.runner.get_idempotency_store`) to return the mock store.
- Patch `graph.ainvoke()` at the module level (`src.agents.runtime.runner.graph.ainvoke`) or inject the mock graph via a test-only parameter (if the existing code supports it).
- Patch `checkpointer.aget()` similarly.
- Use `assert_called_with()` or `assert_called_once_with()` to verify store method calls.
- Use `assert_not_called()` to verify `graph.ainvoke` is skipped when claim returns `"completed"` or `"running"`.

**Example test (pseudo-code):**

```python
@pytest.mark.asyncio
async def test_run_query_claim_completed_skips(mock_store, mock_graph, mock_checkpointer):
    mock_store.claim.return_value = "completed"
    
    with patch("src.agents.runtime.runner.get_idempotency_store", return_value=mock_store), \
         patch("src.agents.runtime.runner.graph.ainvoke", mock_graph), \
         patch("src.agents.runtime.runner.checkpointer.aget", mock_checkpointer):
        
        mock_checkpointer.return_value = MagicMock(next=None)  # Not paused
        
        result = await run_query(thread_id="t1", query="q1")
        
        assert result["skipped"] is True
        assert result["reason"] == "idempotent replay"
        mock_graph.assert_not_called()
```

---

### Test File: `tests/test_orders.py`

**Structure:**
- **1 new test function** (worker status write guard).
- Mock `run_query` to return a result with `reason == "already running"`.
- Mock `redis.set()` (the status write operation).

**Fixture:**

```python
import pytest
from unittest.mock import AsyncMock, patch
from src.agents.queue.orders import _run

@pytest.fixture
def mock_redis():
    """Mock Redis client."""
    return AsyncMock()
```

**Test case (1 total, validating AC8 from requirements.md):**

1. **`test_run_already_running_skips_status_write`** (validates AC8)
   - Mock `run_query()` to return `{"skipped": True, "reason": "already running", "thread_id": "t1", "trace_id": "tr1", "values": {}}`.
   - Mock `redis.set()`.
   - Call `_run(ctx, thread_id="t1", query="q1", service_order_id=None)`.
   - Assert `redis.set()` is **not** called for key `order-run:t1`.

**Test setup notes:**

- Patch `run_query` at the module level (`src.agents.queue.orders.run_query`) to return the mock result.
- Patch `redis.set()` at the module level (`src.agents.queue.orders.redis.set`) or inject the mock Redis client via a test-only parameter.
- Use `assert_not_called()` to verify no status write occurs.

**Example test (pseudo-code):**

```python
@pytest.mark.asyncio
async def test_run_already_running_skips_status_write(mock_redis):
    mock_result = {
        "skipped": True,
        "reason": "already running",
        "thread_id": "t1",
        "trace_id": "tr1",
        "values": {}
    }
    
    with patch("src.agents.queue.orders.run_query", return_value=mock_result), \
         patch("src.agents.queue.orders.redis.set", mock_redis.set):
        
        await _run(ctx=None, thread_id="t1", query="q1", service_order_id=None)
        
        mock_redis.set.assert_not_called()
```

---

**Test count: 9 tests (8 in `test_runner.py`, 1 in `test_orders.py`).**

**AC coverage mapping (validates requirements.md AC1, AC2, AC3, AC6, AC7, AC8 via integration tests):**
- **AC1 (in-memory replay skips):** tests 1 and 3
- **AC2 (concurrent Redis claims):** tests 2 and 3
- **AC3 (completed run returns completed):** test 1
- **AC6 (Redis unavailable raises):** test 8
- **AC7 (Redis failure logs but preserves result):** implicit in all tests (Story 1 contract ensures complete/release failures log; runner tests verify complete/release are called)
- **AC8 (worker skips status write for "already running"):** test 9

---

## Dependencies

### External Dependencies

None (all dependencies already exist from Story 1: `redis`, `arq`, `pytest`).

### Internal Dependencies

**Blocked by Story 1 (PER-111):**

- `IdempotencyStore` protocol (`src/agents/runtime/idempotency.py`).
- `get_idempotency_store()` factory function.
- `IdempotencyUnavailable` exception.

**This story cannot start until Story 1 is merged.**

---

## Risks & Mitigations

### Risk 1: `run_query` Cancellation Handling Incomplete

**Impact:** If `graph.ainvoke` is cancelled but the `try/except asyncio.CancelledError` block is not added, the claim is never released. The run is blocked until the claim TTL expires (3600s).

**Mitigation:** The test suite includes a cancellation test (test case 6). The test will fail if the `release` call is missing.

**Residual risk:** Low. The test catches this before merge.

---

### Risk 2: Worker Status Write Race

**Impact:** If Worker A claims and Worker B calls `run_query` concurrently, Worker B gets `"already running"` and exits. If Worker A then crashes before writing the `running` status, the job appears stuck (no status entry exists). The claim TTL eventually expires, allowing retry.

**Mitigation:** The existing `ORDER_JOB_TIMEOUT` already bounds this. The claim TTL provides eventual recovery. This is an acceptable trade-off (prioritize correctness of the claim over status consistency).

**Residual risk:** Low. This is an existing failure mode (worker crashes are always possible). The idempotency layer does not worsen it.

---

### Risk 3: Resume Flow Calls `complete` Twice

**Impact:** A resumed run calls `complete` when it finishes. If the original run also called `complete` (before pausing), the key is written twice. This is safe because `complete` is idempotent (simple SET, not conditional), but it's wasteful.

**Mitigation:** The HLD already specifies that `complete` is idempotent. No code change needed. Document this in the `complete` method's docstring (Story 1 responsibility).

**Residual risk:** None. This is the intended behavior.

---

### Risk 4: `_run` Status Write Logic Becomes Complex

**Impact:** The status write guard introduces conditional logic (check reason, early exit). If future changes add more status write points, the guard must be applied consistently.

**Mitigation:** Keep the guard as a single early-exit check at the top of `_run`, immediately after `run_query` returns. All status writes happen after this check. Add a comment linking to the HLD AD-4 and this LLD.

**Residual risk:** Low. The structure is simple (one check, one exit).

---

## Out of Scope

- Implementing the idempotency store (Story 1 / PER-111).
- Redis key contract implementation (Story 1 / PER-111).
- `.env.example` documentation (Story 3 / PER-113).
- Module docstring rewrite (Story 1 / PER-111).
- Live Redis testing (manual verification only, per HLD).
- Changing the `order-run:{thread_id}` status contract (out of scope per breakdown).
- Altering the checkpointer or its Redis TODO (out of scope per breakdown).

---

## Definition of Done

- [ ] `src/agents/runtime/runner.py` modified: `run_query` includes store instantiation, claim logic, cancellation handling, complete/release calls.
- [ ] `src/agents/queue/orders.py` modified: `_run` includes status write guard for `"already running"`.
- [ ] `tests/test_runner.py` extended: 8 integration tests added and passing.
- [ ] `tests/test_orders.py` extended: 1 worker status test added and passing.
- [ ] `uv run pytest -q tests/test_runner.py tests/test_orders.py` passes (all 9 tests green).
- [ ] No regressions in existing tests (`uv run pytest -q` full suite passes).
- [ ] Code review confirms: claim point is after paused-thread guard, resume flows skip claim and call complete when finishing (observable outcome: `done` status written), cancellation releases, status write guard is correct.

---

**End of LLD — Story 02-agent**
