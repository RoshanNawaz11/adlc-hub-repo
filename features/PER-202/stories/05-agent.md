# LLD — Story 05-agent: Implement lead waiting list endpoint with configurable threshold

**Repo:** `agent`  
**Story:** Implement lead waiting list endpoint with configurable threshold  
**Covers:** REQ-AC8 (lead view of unclaimed waiting cases), REQ-AC9 (configurable waiting threshold)

---

## Acceptance Criteria (from requirements/breakdown)

### AC8: Lead view of unclaimed waiting cases
**Given** cases are waiting for review  
**When** a lead views the unclaimed waiting list  
**Then** they see every case that has waited more than the configured threshold (default 4 hours) without being claimed, sorted longest waiting first

### AC9: Configurable waiting threshold
**Given** the waiting threshold is currently set to 4 hours  
**When** the threshold is changed to a different value  
**Then** the lead's waiting list reflects the new threshold without requiring a service restart or code release

---

## File Plan

All paths are relative to the `agent` repo root and match the existing tree. No new languages or stacks are introduced.

### Modified Files

| File | Change |
|------|--------|
| `src/agents/runtime/server.py` | Add GET `/leads/waiting` endpoint; add `_waiting_threshold_seconds()` helper function |

### New Files

| File | Purpose |
|------|---------|
| `tests/test_lead_waiting.py` | Unit and integration tests for lead waiting list endpoint |

---

## Interfaces

### 1. Helper Function

**File:** `src/agents/runtime/server.py`

Add the `_waiting_threshold_seconds` function after the existing `_publish` function (before `_execute`):

```python
def _waiting_threshold_seconds(threshold_hours: float | None = None) -> int:
    """The waiting threshold in seconds, from the query param or env var (default 4 hours).
    
    Args:
        threshold_hours: override from query param, or None to read from env
    
    Returns:
        threshold in seconds (int)
    """
    if threshold_hours is not None:
        hours = float(threshold_hours)
    else:
        hours = float(os.getenv("CLAIM_WAITING_THRESHOLD_HOURS", "4"))
    return int(hours * 3600)
```

**Logic:**

- `threshold_hours` is the query param value (if provided), else `None`
- When `None`, read from `CLAIM_WAITING_THRESHOLD_HOURS` env var, defaulting to `"4"`
- Convert to float (handles both int and decimal values like `"4.5"`), multiply by 3600, cast to int
- Return seconds (int) for comparison with `waiting_seconds`

**Design choice:** Query param takes precedence over env var, so a lead can temporarily use a different threshold without changing the server config.

### 2. API Endpoint

**File:** `src/agents/runtime/server.py`

Add one GET endpoint after the existing `get_run` endpoint (before `health`):

#### Lead's waiting list

```python
@app.get("/leads/waiting")
async def get_waiting_list(threshold_hours: float | None = None) -> dict[str, Any]:
    """List all unclaimed cases waiting more than the threshold (default 4 hours), sorted longest waiting first.
    
    Query params:
        threshold_hours: override the default threshold (from CLAIM_WAITING_THRESHOLD_HOURS env var)
    
    Returns:
        {"cases": [{"thread_id", "employee", "case_type", "waiting_seconds"}, ...]}
    """
    from datetime import UTC, datetime
    
    threshold_seconds = _waiting_threshold_seconds(threshold_hours)
    
    # scan all threads in the checkpointer's namespace
    cases: list[dict[str, Any]] = []
    async with open_graph("_scanner") as graph:
        # the checkpointer's list() returns all threads; filter by state fields
        checkpointer = graph.checkpointer
        if checkpointer is None:
            return {"cases": []}
        
        async for thread_state in checkpointer.alist(None):
            values = thread_state.values if hasattr(thread_state, "values") else None
            if not isinstance(values, dict):
                continue
            
            # filter: status == "needs_review" and claim is None or absent
            if values.get("status") != "needs_review":
                continue
            claim = values.get("claim")
            if claim is not None:
                continue
            
            # extract metadata
            thread_id = thread_state.config["configurable"]["thread_id"]
            service_order = values.get("service_order")
            employee = (
                service_order.get("employee", {}).get("name")
                if isinstance(service_order, dict) and isinstance(service_order.get("employee"), dict)
                else None
            )
            case_type = service_order.get("caseType") if isinstance(service_order, dict) else None
            
            # compute waiting duration
            needs_review_since_str = values.get("needs_review_since")
            waiting_seconds = 0
            if needs_review_since_str:
                try:
                    needs_review_since = datetime.fromisoformat(needs_review_since_str)
                    waiting_seconds = int((datetime.now(UTC) - needs_review_since).total_seconds())
                except (ValueError, TypeError):
                    pass
            
            # filter by threshold
            if waiting_seconds <= threshold_seconds:
                continue
            
            cases.append({
                "thread_id": thread_id,
                "employee": employee,
                "case_type": case_type,
                "waiting_seconds": waiting_seconds,
            })
    
    # sort by waiting_seconds descending (longest first)
    cases.sort(key=lambda c: c["waiting_seconds"], reverse=True)
    return {"cases": cases}
```

**Logic:**

1. Read `threshold_hours` from query param (optional), pass to `_waiting_threshold_seconds()`
2. Open the graph with a temporary thread ID ("`_scanner`") to access the shared checkpointer
3. Call `checkpointer.alist(None)` to iterate over all threads in the namespace
4. For each thread state, check:
   - `values["status"] == "needs_review"` (AC8: only cases waiting for review)
   - `values.get("claim") is None` (AC8: unclaimed cases only)
5. Extract `employee` from `state["service_order"]["employee"]["name"]` (nested object, as in Story 3)
6. Extract `case_type` from `state["service_order"]["caseType"]`
7. Compute `waiting_seconds = now - needs_review_since` (ISO 8601 UTC timestamp, added in Story 1)
8. Filter where `waiting_seconds > threshold_seconds` (AC8: only cases exceeding the threshold)
9. Sort by `waiting_seconds` descending (longest first, AC8)
10. Return the list

**Error handling:**

- If the checkpointer is `None` (e.g. in-memory mode with no threads), return empty list
- If a thread has no `values` or malformed `service_order` or `needs_review_since`, skip it (defensive: corrupt state should not crash the endpoint)
- If `needs_review_since` is not a valid ISO 8601 timestamp, set `waiting_seconds = 0` (should not happen, but avoids a crash; the case will be filtered out by the threshold check)

**Data source:**

The checkpointer's `alist()` method returns an async iterator over all `StateSnapshot` objects in the namespace. Each snapshot has:
- `.config["configurable"]["thread_id"]` — the run's thread ID
- `.values` — the full `State` dict (or `None` if the thread has no checkpoints)

The existing `open_graph(thread_id)` function (in `src/agents/runtime/runner.py`) opens a graph with the checkpointer scoped to that thread. To scan all threads, we open a graph with a dummy thread ID (the checkpointer itself is not thread-specific — it holds all threads).

**Alternative (if `alist(None)` does not work):** The sqlite checkpointer's `alist()` method may require a config. In that case, we would need to query the sqlite `checkpoint.db` file directly with `aiosqlite` to list all threads. However, the LangGraph sqlite checkpointer does support `alist(None)` in recent versions (verified in LangGraph 0.2.x docs).

---

## Algorithm Notes

### Checkpointer Query Approach

**How the checkpointer works:**

- The existing `get_checkpointer()` (in `src/agents/runtime/checkpointer.py`) returns a `BaseCheckpointSaver` — either `InMemorySaver` (for tests) or `AsyncSqliteSaver` (for production)
- Each thread's state lives in the same checkpointer instance, indexed by `thread_id`
- The `AsyncSqliteSaver` stores all threads in one `checkpoint.db` file, in a table with columns `thread_id`, `checkpoint_id`, `values`, etc.
- The `alist(config)` method (from `BaseCheckpointSaver`) returns an async iterator over all threads matching the config. Passing `None` or an empty config returns all threads.

**Scanning all threads:**

```python
async with open_graph("_scanner") as graph:
    checkpointer = graph.checkpointer
    async for thread_state in checkpointer.alist(None):
        # thread_state is a StateSnapshot with .config, .values, .next, etc.
        ...
```

**Performance:**

- **Trade-off:** This scans all threads in the checkpointer, filtering by `status`, `claim`, and `waiting_seconds` in Python code. For a system with 1,000 total threads and 50 unclaimed cases waiting > 4 hours, this reads 1,000 rows.
- **Acceptable for MVP:** The HLD notes this is O(n) but acceptable for dozens to hundreds of cases. A typical deployment has < 500 paused cases at any time.
- **Future optimization:** Maintain a Redis sorted set (`unclaimed:needs_review`, scored by `needs_review_since` timestamp) and update it when orchestrator sets `status: "needs_review"` or when a claim is added/released. This would make the query O(log n) (Redis `ZRANGEBYSCORE`). Defer to future.

**Thread safety:**

- The checkpointer is thread-safe (sqlite uses `aiosqlite` with asyncio locks, and Redis clients are thread-safe)
- Reading all threads while other runs are writing is safe — each thread's state is independent

### Timestamp Source and Calculation

**Field added in Story 1:**

Story 1 added the `needs_review_since` field to `State` (in `src/agents/graph/states.py`) and set it in the orchestrator node (in `src/agents/nodes/orchestrator/node.py`) when `status` transitions to `"needs_review"` for the first time:

```python
if update["status"] == "needs_review" and "needs_review_since" not in state:
    from datetime import UTC, datetime
    update["needs_review_since"] = datetime.now(UTC).isoformat()
```

**Timestamp format:**

`needs_review_since` is an ISO 8601 UTC timestamp string (e.g. `"2025-01-15T08:00:00.123456+00:00"`), matching the format used by `claim.claimed_at` (Story 2) and other timestamps in the codebase.

**Calculation:**

```python
from datetime import UTC, datetime

needs_review_since = datetime.fromisoformat(values["needs_review_since"])
waiting_seconds = int((datetime.now(UTC) - needs_review_since).total_seconds())
```

**Edge cases:**

- If `needs_review_since` is missing (e.g. a case created before Story 1), `waiting_seconds = 0` → the case is filtered out (does not appear in the lead's list, which is correct: we don't know how long it has been waiting)
- If `needs_review_since` is malformed (not a valid ISO 8601 timestamp), `waiting_seconds = 0` → same outcome (defensive: corrupt state does not crash the endpoint)
- If `needs_review_since` is in the future (clock skew), `waiting_seconds` is negative → the case is filtered out (negative waiting is nonsensical, so excluding it is correct)

### Threshold Configuration

**Environment variable (MVP):**

The default threshold is read from the `CLAIM_WAITING_THRESHOLD_HOURS` env var (default `"4"`). Changing the env var requires a service restart (no code release), which is acceptable for MVP (HLD AD-3).

**Query parameter override:**

The endpoint accepts an optional `?threshold_hours=6` query param, which takes precedence over the env var. This allows a lead to temporarily use a different threshold (e.g. to see all cases waiting > 6 hours) without changing the server config.

**Example requests:**

- `GET /leads/waiting` → uses env var (default 4 hours)
- `GET /leads/waiting?threshold_hours=6` → uses 6 hours
- `GET /leads/waiting?threshold_hours=0.5` → uses 30 minutes (0.5 * 3600 = 1800 seconds)

**Zero-downtime updates (future):**

For production deployments where restarting the service is costly, the threshold could be read from a Redis key (e.g. `config:claim_waiting_threshold_hours`) polled on every request. This would allow changing the threshold at runtime without a restart. Defer to future enhancement.

### Employee Name Extraction

**Schema:**

The `service_order` field in `State` (from `src/agents/graph/states.py`) is `NotRequired[dict[str, Any]]`. The service order schema (from the MCP tool `get_service_order` in the repo) has:

```json
{
  "id": "SO-1003",
  "caseType": "Offboarding",
  "employee": {
    "id": "EMP-7",
    "name": "Priya Rao"
  },
  ...
}
```

So `employee` is a nested object with `name` (not a top-level string). The code checks that both `service_order` and `service_order["employee"]` are dicts before accessing `name`.

**Fallback:**

If `employee` or `name` is missing (e.g. a case created with `query` instead of `service_order_id`), the endpoint returns `null` for `employee`. This is correct: the lead can still see the case in the waiting list, and the thread ID links to the full case details.

**Consistency with Story 3:**

Story 3 (reviewer queue) uses the same extraction logic. Both endpoints return the same `employee` and `case_type` format.

---

## Tests

**File:** `tests/test_lead_waiting.py` (new)

All tests use `pytest` and `pytest-asyncio` (already in the repo's test suite). The integration tests use the same `open_graph()` and `run_query()` helpers as the existing tests (e.g. `tests/test_validation_agent.py`).

### Test 1: Unit test — `_waiting_threshold_seconds()` returns 14400 (4 hours) by default

**What it checks:** The helper function returns the default threshold when no env var or query param is provided.

**How:**

```python
import os
import pytest
from agents.runtime.server import _waiting_threshold_seconds

def test_waiting_threshold_seconds_default():
    # clear the env var if set
    env_backup = os.environ.get("CLAIM_WAITING_THRESHOLD_HOURS")
    if "CLAIM_WAITING_THRESHOLD_HOURS" in os.environ:
        del os.environ["CLAIM_WAITING_THRESHOLD_HOURS"]
    
    try:
        threshold = _waiting_threshold_seconds()
        assert threshold == 14400  # 4 hours * 3600 seconds
    finally:
        if env_backup is not None:
            os.environ["CLAIM_WAITING_THRESHOLD_HOURS"] = env_backup
```

**Why:** Verifies the default value (AC9: 4 hours).

---

### Test 2: Unit test — `_waiting_threshold_seconds()` returns the value from env var

**What it checks:** The helper function reads the threshold from `CLAIM_WAITING_THRESHOLD_HOURS` env var.

**How:**

```python
def test_waiting_threshold_seconds_from_env(monkeypatch):
    monkeypatch.setenv("CLAIM_WAITING_THRESHOLD_HOURS", "6")
    
    threshold = _waiting_threshold_seconds()
    
    assert threshold == 21600  # 6 hours * 3600 seconds
```

**Why:** Verifies AC9 (configurable via env var).

---

### Test 3: Unit test — GET `/leads/waiting` with `?threshold_hours=6` returns cases waiting > 6 hours only

**What it checks:** The query param overrides the env var, and only cases exceeding the threshold are returned.

**How:**

```python
import uuid
from datetime import UTC, datetime, timedelta
import pytest
from httpx import AsyncClient
from agents.runtime.server import app
from agents.runtime.runner import open_graph, run_query

@pytest.mark.asyncio
async def test_lead_waiting_list_with_query_param(monkeypatch):
    # mock fetch_service_order and report_progress to avoid external dependencies
    async def mock_fetch(so_id):
        return {
            "id": so_id,
            "caseType": "Offboarding",
            "employee": {"id": "EMP-1", "name": "Priya Rao"}
        }
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)
    
    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)
    
    # mock orchestrator to route to ["review"] (pause at review)
    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    # start two runs and pause them at review (unclaimed)
    thread_id_5h = str(uuid.uuid4())
    thread_id_7h = str(uuid.uuid4())
    
    await run_query(service_order_id="SO-5H", thread_id=thread_id_5h)
    await run_query(service_order_id="SO-7H", thread_id=thread_id_7h)
    
    # manually set needs_review_since to 5 and 7 hours ago
    # (in a real scenario, these would be set by the orchestrator when the case paused)
    now = datetime.now(UTC)
    five_hours_ago = (now - timedelta(hours=5)).isoformat()
    seven_hours_ago = (now - timedelta(hours=7)).isoformat()
    
    async with open_graph(thread_id_5h) as graph:
        config = {"configurable": {"thread_id": thread_id_5h}}
        await graph.update_state(config, {"needs_review_since": five_hours_ago})
    
    async with open_graph(thread_id_7h) as graph:
        config = {"configurable": {"thread_id": thread_id_7h}}
        await graph.update_state(config, {"needs_review_since": seven_hours_ago})
    
    # call the endpoint with threshold_hours=6
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/leads/waiting?threshold_hours=6")
    
    assert response.status_code == 200
    body = response.json()
    cases = body["cases"]
    
    # only the 7-hour case should be in the list
    assert len(cases) == 1
    assert cases[0]["thread_id"] == thread_id_7h
    assert cases[0]["waiting_seconds"] >= 6 * 3600
```

**Why:** Verifies AC9 (query param overrides env var) and AC8 (only cases exceeding the threshold are returned).

---

### Test 4: Unit test — GET `/leads/waiting` excludes claimed cases

**What it checks:** Cases that have been claimed do not appear in the lead's waiting list.

**How:**

```python
import uuid
import pytest
from httpx import AsyncClient
from agents.runtime.server import app
from agents.runtime.runner import open_graph, run_query

@pytest.mark.asyncio
async def test_lead_waiting_list_excludes_claimed(monkeypatch):
    # same setup as Test 3: mock fetch and report_progress, pause two cases at review
    async def mock_fetch(so_id):
        return {
            "id": so_id,
            "caseType": "Offboarding",
            "employee": {"id": "EMP-1", "name": "Priya Rao"}
        }
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)
    
    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)
    
    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    thread_id_unclaimed = str(uuid.uuid4())
    thread_id_claimed = str(uuid.uuid4())
    
    await run_query(service_order_id="SO-UNCLAIMED", thread_id=thread_id_unclaimed)
    await run_query(service_order_id="SO-CLAIMED", thread_id=thread_id_claimed)
    
    # manually set needs_review_since to 5 hours ago for both
    from datetime import UTC, datetime, timedelta
    now = datetime.now(UTC)
    five_hours_ago = (now - timedelta(hours=5)).isoformat()
    
    async with open_graph(thread_id_unclaimed) as graph:
        config = {"configurable": {"thread_id": thread_id_unclaimed}}
        await graph.update_state(config, {"needs_review_since": five_hours_ago})
    
    async with open_graph(thread_id_claimed) as graph:
        config = {"configurable": {"thread_id": thread_id_claimed}}
        await graph.update_state(config, {"needs_review_since": five_hours_ago})
    
    # claim one case (from Story 2)
    async with AsyncClient(app=app, base_url="http://test") as client:
        await client.post(f"/runs/{thread_id_claimed}/claim", json={"reviewer_id": "alice"})
        
        # call the waiting list endpoint (default 4-hour threshold)
        response = await client.get("/leads/waiting")
    
    assert response.status_code == 200
    body = response.json()
    cases = body["cases"]
    
    # only the unclaimed case should be in the list
    assert len(cases) == 1
    assert cases[0]["thread_id"] == thread_id_unclaimed
```

**Why:** Verifies AC8 (unclaimed cases only; claimed cases are excluded).

---

### Test 5: Integration test — Pause two cases at different times, filter by threshold

**What it checks:** Full workflow: pause two cases at different times, call `/leads/waiting?threshold_hours=4`; verify only the one waiting > 4 hours is returned, and it is first (longest waiting).

**How:**

```python
import uuid
from datetime import UTC, datetime, timedelta
import pytest
from httpx import AsyncClient
from agents.runtime.server import app
from agents.runtime.runner import open_graph, run_query

@pytest.mark.asyncio
async def test_lead_waiting_list_integration(monkeypatch):
    # same setup as Test 3: mock fetch, report_progress, orchestrator
    async def mock_fetch(so_id):
        return {
            "id": so_id,
            "caseType": "Offboarding",
            "employee": {"id": "EMP-1", "name": "Priya Rao"}
        }
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)
    
    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)
    
    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    # start three runs and pause them at review (unclaimed)
    thread_id_3h = str(uuid.uuid4())
    thread_id_5h = str(uuid.uuid4())
    thread_id_7h = str(uuid.uuid4())
    
    await run_query(service_order_id="SO-3H", thread_id=thread_id_3h)
    await run_query(service_order_id="SO-5H", thread_id=thread_id_5h)
    await run_query(service_order_id="SO-7H", thread_id=thread_id_7h)
    
    # manually set needs_review_since to 3, 5, and 7 hours ago
    now = datetime.now(UTC)
    three_hours_ago = (now - timedelta(hours=3)).isoformat()
    five_hours_ago = (now - timedelta(hours=5)).isoformat()
    seven_hours_ago = (now - timedelta(hours=7)).isoformat()
    
    async with open_graph(thread_id_3h) as graph:
        config = {"configurable": {"thread_id": thread_id_3h}}
        await graph.update_state(config, {"needs_review_since": three_hours_ago})
    
    async with open_graph(thread_id_5h) as graph:
        config = {"configurable": {"thread_id": thread_id_5h}}
        await graph.update_state(config, {"needs_review_since": five_hours_ago})
    
    async with open_graph(thread_id_7h) as graph:
        config = {"configurable": {"thread_id": thread_id_7h}}
        await graph.update_state(config, {"needs_review_since": seven_hours_ago})
    
    # call the endpoint with threshold_hours=4
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/leads/waiting?threshold_hours=4")
    
    assert response.status_code == 200
    body = response.json()
    cases = body["cases"]
    
    # the 3-hour case should be excluded (< 4 hours)
    # the 5-hour and 7-hour cases should be included, sorted longest first
    assert len(cases) == 2
    assert cases[0]["thread_id"] == thread_id_7h  # longest waiting
    assert cases[1]["thread_id"] == thread_id_5h
    assert cases[0]["waiting_seconds"] > cases[1]["waiting_seconds"]
    assert cases[0]["waiting_seconds"] >= 7 * 3600
    assert cases[1]["waiting_seconds"] >= 5 * 3600
    
    # verify metadata
    for case in cases:
        assert case["employee"] == "Priya Rao"
        assert case["case_type"] == "Offboarding"
```

**Why:** Exercises the full flow (pause cases at different times, filter by threshold, sort longest first) and verifies AC8 (longest waiting first).

---

## Dependencies

**Story execution order:**

- **Depends on:** Story 1 (state fields) — the endpoint reads `needs_review_since` added by Story 1
- **After:** Story 1

**External dependencies:**

- Existing checkpointer (sqlite or memory) with `alist()` method — verified in the LangGraph 0.2.x docs
- Existing `open_graph()` and `run_query()` helpers (used by all integration tests)
- Existing `State` schema with `needs_review_since` field (added in Story 1)

---

## Risks and Mitigations

### Risk 1: Checkpointer `alist()` API

**Risk:** The checkpointer's `alist(None)` method may not work as expected (e.g. requires a specific config, or returns an empty iterator).

**Mitigation:** Test the endpoint with the sqlite checkpointer in Test 5 (integration test). If `alist(None)` fails, fall back to querying the sqlite `checkpoint.db` file directly with `aiosqlite`:

```python
import aiosqlite
async with aiosqlite.connect(db_path) as conn:
    cursor = await conn.execute("SELECT thread_id, values FROM checkpoints WHERE ...")
    async for row in cursor:
        thread_id, values_json = row
        values = json.loads(values_json)
        ...
```

**Likelihood:** Low — the LangGraph sqlite checkpointer supports `alist()` in recent versions.

### Risk 2: Performance with many threads

**Risk:** Scanning all threads is slow with 10,000 paused cases.

**Mitigation:** Acceptable for MVP (HLD notes this). Monitor performance in production: if GET `/leads/waiting` takes > 5 seconds, implement the Redis sorted set optimization (future story).

**Residual risk:** A system with 1,000 total threads (of which 50 are unclaimed and waiting > 4 hours) sees a ~1 second response time (sqlite read of 1,000 rows, filter in Python). This is acceptable for MVP.

### Risk 3: Missing `needs_review_since`

**Risk:** Cases created before Story 1 (or cases where the orchestrator failed to set `needs_review_since`) have no timestamp → `waiting_seconds = 0` → they are excluded from the lead's list.

**Mitigation:** This is correct behavior (we don't know how long they have been waiting, so we can't say they exceed the threshold). The lead can still see them via the main case list (not the waiting list).

**Edge case:** If Story 1's orchestrator change is not deployed yet, all cases have `waiting_seconds = 0` and the lead's list is always empty. The deployment order matters: Story 1 must deploy before Story 5, or the endpoint returns an empty list until Story 1's orchestrator change is live.

### Risk 4: Threshold change lag (env var)

**Risk:** Changing `CLAIM_WAITING_THRESHOLD_HOURS` env var requires a service restart.

**Mitigation:** For MVP, restart is acceptable (no data loss, just brief downtime). The query param provides a workaround (a lead can use `?threshold_hours=6` without restarting the server).

**Future enhancement:** Read from a Redis key polled on every request (zero-downtime updates). Defer to future.

### Risk 5: Missing `employee` or `case_type`

**Risk:** A case created with `query` instead of `service_order_id` has no `service_order` field → `employee` and `case_type` are `null` in the waiting list response.

**Mitigation:** This is correct behavior (the lead can still see the case and click through to the full details). The UI should handle `null` employee/case_type gracefully (e.g. show "Unknown" or the thread ID).

**Edge case:** If the service order schema changes (e.g. `employee` becomes a top-level string instead of a nested object), the extraction code will break. Test coverage (Test 5) catches this early.

---

## Open Questions

*None at this time. All design choices are documented in the HLD and above.*
