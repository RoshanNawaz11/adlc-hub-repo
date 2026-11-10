# LLD — Story 02-agent: Implement claim, release, and handover endpoints

**Repo:** `agent`  
**Story:** Implement claim, release, and handover endpoints  
**Covers:** REQ-AC1 (claiming a case), REQ-AC3 (releasing a claim), REQ-AC4 (handing over with reason), REQ-AC5 (claim history tracking), REQ-AC10 (claim state validation)

---

## Acceptance Criteria (from requirements/breakdown)

### AC1: Claiming a case (unclaimed → claimed)
**Given** a case is waiting for review and unclaimed  
**When** a reviewer claims it  
**Then** the case is marked as claimed by that reviewer with a timestamp, and all other reviewers see who claimed it and when

### AC3: Releasing a claim (claimed → unclaimed)
**Given** a reviewer has claimed a case  
**When** they release their claim  
**Then** the case becomes unclaimed again and available to any reviewer

### AC4: Handing over a claim (claimed by A → claimed by B)
**Given** Reviewer A has claimed a case  
**When** Reviewer A hands the case to Reviewer B with a reason  
**Then** the case is now claimed by Reviewer B, the reason is required and recorded, and Reviewer A no longer has claim

### AC5: Claim history tracking
**Given** a case has had claims, releases, and handovers  
**When** the history is viewed  
**Then** each claim, release, and handover is recorded showing who performed the action, when, and the reason (for handovers)

### AC10: Claim state validation
**Given** a case is not in "waiting for review" state (e.g., still running or already finished)  
**When** a reviewer attempts to claim it  
**Then** the claim is rejected with a message explaining why the case cannot be claimed

---

## File Plan

All paths are relative to the `agent` repo root and match the existing tree. No new languages or stacks are introduced.

### Modified Files

| File | Change |
|------|--------|
| `src/agents/runtime/server.py` | Add three POST endpoints: `/runs/{thread_id}/claim`, `/runs/{thread_id}/release`, `/runs/{thread_id}/handover`; add Pydantic request models `ClaimRequest`, `ReleaseRequest`, `HandoverRequest` |

### New Files

| File | Purpose |
|------|---------|
| `tests/test_claim_endpoints.py` | Unit and integration tests for claim, release, and handover endpoints |

---

## Interfaces

### 1. Request Models

**File:** `src/agents/runtime/server.py`

Add three Pydantic models after the existing `FlagAnswers` class (before the `_lifespan` function):

```python
class ClaimRequest(BaseModel):
    """A reviewer claims an unclaimed case."""
    reviewer_id: str


class ReleaseRequest(BaseModel):
    """A reviewer releases their claim on a case."""
    reviewer_id: str


class HandoverRequest(BaseModel):
    """A reviewer hands over their claim to another reviewer."""
    from_reviewer: str
    to_reviewer: str
    reason: str

    @model_validator(mode="after")
    def _reason_required(self) -> "HandoverRequest":
        if not (self.reason or "").strip():
            raise ValueError("handover requires a non-empty reason")
        return self
```

**Rationale:**

- `ClaimRequest` and `ReleaseRequest` carry only `reviewer_id` — the case is identified by the URL path parameter `{thread_id}`
- `HandoverRequest` carries both reviewers explicitly (not just `to_reviewer`) so the endpoint can validate `from_reviewer` owns the claim before transferring it
- The `@model_validator` on `HandoverRequest` enforces AC4 (reason is required) at the Pydantic layer, before the endpoint logic runs

### 2. API Endpoints

**File:** `src/agents/runtime/server.py`

Add three POST endpoints after the existing `resume_run` endpoint (before `get_run`):

#### 2.1. Claim a case

```python
@app.post("/runs/{thread_id}/claim")
async def claim_run(thread_id: str, req: ClaimRequest) -> dict[str, Any]:
    """Claim an unclaimed case waiting for review."""
    from datetime import UTC, datetime

    snapshot = await _snapshot(thread_id)
    if not snapshot.values:
        raise HTTPException(404, f"Unknown run {thread_id}")

    state = snapshot.values
    status = state.get("status")
    if status != "needs_review":
        raise HTTPException(400, json.dumps({
            "error": "not_claimable",
            "reason": f"case is {status}",
        }))

    claim = state.get("claim")
    if claim:
        raise HTTPException(409, json.dumps({
            "error": "already_claimed",
            "claim": claim,
        }))

    now = datetime.now(UTC).isoformat()
    new_claim = {"reviewer_id": req.reviewer_id, "claimed_at": now}
    history = list(state.get("claim_history") or [])
    history.append(new_claim)

    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        await graph.update_state(config, {"claim": new_claim, "claim_history": history})

    return {"status": "claimed", "claim": new_claim}
```

**Logic:**

1. Load the run's snapshot via `_snapshot(thread_id)` (existing helper)
2. Validate `status == "needs_review"` → else 400 with `not_claimable`
3. If `claim` exists → 409 with `already_claimed` and the current claim
4. Build the new claim dict with `reviewer_id` and ISO 8601 UTC timestamp
5. Append the claim to `claim_history`
6. Persist via `graph.update_state(config, {"claim": ..., "claim_history": ...})`
7. Return 200 with the new claim

**Error handling:**

- 404 if `thread_id` does not exist (snapshot has no values)
- 400 if the case is not in `needs_review` status (AC10)
- 409 if the case is already claimed (AC1: another reviewer sees who claimed it)

#### 2.2. Release a claim

```python
@app.post("/runs/{thread_id}/release")
async def release_run(thread_id: str, req: ReleaseRequest) -> dict[str, Any]:
    """Release the reviewer's claim on a case."""
    from datetime import UTC, datetime

    snapshot = await _snapshot(thread_id)
    if not snapshot.values:
        raise HTTPException(404, f"Unknown run {thread_id}")

    state = snapshot.values
    claim = state.get("claim")
    if not claim:
        raise HTTPException(400, json.dumps({"error": "not_claimed"}))

    if claim["reviewer_id"] != req.reviewer_id:
        raise HTTPException(403, json.dumps({
            "error": "not_owner",
            "claim": claim,
        }))

    now = datetime.now(UTC).isoformat()
    release_event = {"reviewer_id": req.reviewer_id, "claimed_at": now, "reason": "released"}
    history = list(state.get("claim_history") or [])
    history.append(release_event)

    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        await graph.update_state(config, {"claim": None, "claim_history": history})

    return {"status": "released"}
```

**Logic:**

1. Load snapshot
2. If no claim → 400 with `not_claimed`
3. If `claim["reviewer_id"] != req.reviewer_id` → 403 with `not_owner`
4. Build a release event: `{"reviewer_id": <id>, "claimed_at": <now>, "reason": "released"}`
5. Append to `claim_history`
6. Persist: `{"claim": None, "claim_history": ...}`
7. Return 200 with `{"status": "released"}`

**Error handling:**

- 404 if `thread_id` does not exist
- 400 if the case is unclaimed (AC3: only a claimed case can be released)
- 403 if the reviewer does not own the claim (AC3: only the owner can release)

#### 2.3. Hand over a claim

```python
@app.post("/runs/{thread_id}/handover")
async def handover_run(thread_id: str, req: HandoverRequest) -> dict[str, Any]:
    """Hand over a claim from one reviewer to another."""
    from datetime import UTC, datetime

    snapshot = await _snapshot(thread_id)
    if not snapshot.values:
        raise HTTPException(404, f"Unknown run {thread_id}")

    state = snapshot.values
    claim = state.get("claim")
    if not claim:
        raise HTTPException(400, json.dumps({"error": "not_claimed"}))

    if claim["reviewer_id"] != req.from_reviewer:
        raise HTTPException(403, json.dumps({
            "error": "not_owner",
            "claim": claim,
        }))

    now = datetime.now(UTC).isoformat()
    handover_event = {"reviewer_id": req.from_reviewer, "claimed_at": now, "reason": req.reason}
    new_claim = {"reviewer_id": req.to_reviewer, "claimed_at": now}
    history = list(state.get("claim_history") or [])
    history.append(handover_event)

    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        await graph.update_state(config, {"claim": new_claim, "claim_history": history})

    return {"status": "handed_over", "claim": new_claim}
```

**Logic:**

1. Load snapshot
2. If no claim → 400 with `not_claimed`
3. If `claim["reviewer_id"] != req.from_reviewer` → 403 with `not_owner`
4. Build handover event: `{"reviewer_id": <from>, "claimed_at": <now>, "reason": <reason>}`
5. Build new claim: `{"reviewer_id": <to>, "claimed_at": <now>}`
6. Append handover event to `claim_history`
7. Persist: `{"claim": <new_claim>, "claim_history": ...}`
8. Return 200 with `{"status": "handed_over", "claim": <new_claim>}`

**Error handling:**

- 404 if `thread_id` does not exist
- 400 if the case is unclaimed (AC4: only a claimed case can be handed over), or if `reason` is empty (enforced by Pydantic validator)
- 403 if `from_reviewer` does not own the claim (AC4: only the owner can hand over)

---

## Algorithm Notes

### Checkpoint Persistence

**How `graph.update_state` works:**

- The existing `open_graph(thread_id)` (in `src/agents/runtime/runner.py`) returns a LangGraph `CompiledGraph` with the run's checkpointer (sqlite or memory)
- `graph.update_state(config, values)` writes the given `values` dict into the state at the current checkpoint, creating a new checkpoint version
- The update is **not** a graph execution (no nodes run) — it is a direct state patch, persisted immediately
- The next time the graph runs (or `_snapshot` reads the state), it sees the updated `claim` and `claim_history` fields

**Concurrency:**

- The sqlite checkpointer (via `langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver`) uses optimistic locking: each checkpoint has a version number
- If two updates write simultaneously, the second one sees a version mismatch, retries the read, and then writes with the new version
- For claims: if two reviewers call `POST /claim` at the same time, one wins and the other gets a fresh snapshot on retry, sees `claim` is no longer `None`, and returns 409

**No new persistence code needed:** the checkpointer already handles the write and the concurrency.

### Claim History Format

**Structure:**

- Each entry in `claim_history` is a `Claim` TypedDict: `{"reviewer_id": str, "claimed_at": str, "reason": NotRequired[str]}`
- A claim action (new claim, release, handover) appends one entry
- The `reason` field is populated only for handovers (AC4) and releases (hardcoded to `"released"` for clarity in the history)

**Example history after claim → handover → release:**

```json
[
  {"reviewer_id": "alice", "claimed_at": "2025-01-15T10:42:00Z"},
  {"reviewer_id": "alice", "claimed_at": "2025-01-15T11:00:00Z", "reason": "Waiting on client for bonus details"},
  {"reviewer_id": "bob", "claimed_at": "2025-01-15T14:30:00Z", "reason": "released"}
]
```

**Reading the history:**

- Entry 1: Alice claimed the case
- Entry 2: Alice handed the case to Bob (reason is in the entry with Alice's ID)
- Entry 3: Bob released the case (reason is `"released"`)

**Note:** The handover event carries the `from_reviewer`'s ID (Alice in the example), not the `to_reviewer`. The new claim (`state["claim"]`) carries Bob's ID. This matches the HLD's design: the history entry is the *action* (Alice handing over), not the *result* (Bob now owning it).

### Timestamp Precision

**ISO 8601 UTC:**

- All `claimed_at` timestamps are ISO 8601 strings in UTC (e.g. `"2025-01-15T10:42:00.123456+00:00"`)
- Generated via `datetime.now(UTC).isoformat()` (Python 3.11+)
- This matches the existing `State.step_record` timestamp format (in `src/agents/graph/states.py`)

---

## Tests

**File:** `tests/test_claim_endpoints.py` (new)

All tests use `pytest` and `pytest-asyncio` (already in the repo's test suite). The integration tests use the same `open_graph()` and `run_query()` helpers as the existing tests (e.g. `tests/test_validation_agent.py`).

### Test 1: Unit test — Claim a case in `needs_review` status

**What it checks:** POST `/runs/{thread_id}/claim` with valid `reviewer_id` on a case in `needs_review` status returns 200 with claim details and updates state.

**How:**

```python
import uuid
from datetime import UTC, datetime
import pytest
from httpx import AsyncClient
from agents.runtime.server import app
from agents.runtime.runner import open_graph, run_query

@pytest.mark.asyncio
async def test_claim_case_in_needs_review(monkeypatch):
    thread_id = str(uuid.uuid4())

    # mock fetch_service_order and report_progress to avoid external dependencies
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    # start a run and pause it at review (mock orchestrator to route to ["review"])
    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    # verify the case is paused at review
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        assert snapshot.values["status"] == "needs_review"
        assert snapshot.values.get("claim") is None

    # claim the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "claimed"
    assert body["claim"]["reviewer_id"] == "alice"
    assert "claimed_at" in body["claim"]

    # verify the state is updated
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        claim = snapshot.values["claim"]
        assert claim["reviewer_id"] == "alice"
        assert len(snapshot.values["claim_history"]) == 1
        assert snapshot.values["claim_history"][0] == claim
```

**Why:** Exercises the happy path for claiming a case (AC1).

---

### Test 2: Unit test — Claim a case not in `needs_review` status

**What it checks:** POST `/runs/{thread_id}/claim` on a case not in `needs_review` status returns 400 with error.

**How:**

```python
@pytest.mark.asyncio
async def test_claim_case_not_in_needs_review(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same mocks as Test 1
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    # start a run and complete it (mock orchestrator to route to [END])
    async def mock_decide(state, gateway, rejected):
        from langgraph.graph import END
        return {"route": [END], "summary": "Done"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    # verify the case is done
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        assert snapshot.values["status"] == "done"

    # attempt to claim the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

    assert response.status_code == 400
    body = response.json()
    assert body["error"] == "not_claimable"
    assert "done" in body["reason"]
```

**Why:** Verifies AC10 (claim state validation: case must be `needs_review`).

---

### Test 3: Unit test — Claim an already claimed case

**What it checks:** POST `/runs/{thread_id}/claim` on an already claimed case returns 409 with current claim.

**How:**

```python
@pytest.mark.asyncio
async def test_claim_already_claimed_case(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 1: pause at review
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    # alice claims the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response1 = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response1.status_code == 200

        # bob attempts to claim the same case
        response2 = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "bob"})

    assert response2.status_code == 409
    body = response2.json()
    assert body["error"] == "already_claimed"
    assert body["claim"]["reviewer_id"] == "alice"
```

**Why:** Verifies AC1 (other reviewers see who claimed it) and concurrent claim handling.

---

### Test 4: Unit test — Release a claim by the owner

**What it checks:** POST `/runs/{thread_id}/release` by the owning reviewer returns 200 and sets `claim` to `None`.

**How:**

```python
@pytest.mark.asyncio
async def test_release_claim_by_owner(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 1: pause at review
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    # alice claims the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

        # alice releases the case
        response = await client.post(f"/runs/{thread_id}/release", json={"reviewer_id": "alice"})

    assert response.status_code == 200
    assert response.json()["status"] == "released"

    # verify the state is updated
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        assert snapshot.values.get("claim") is None
        assert len(snapshot.values["claim_history"]) == 2  # claim + release
        assert snapshot.values["claim_history"][1]["reason"] == "released"
```

**Why:** Verifies AC3 (releasing a claim).

---

### Test 5: Unit test — Release a claim by a non-owner

**What it checks:** POST `/runs/{thread_id}/release` by a non-owner returns 403.

**How:**

```python
@pytest.mark.asyncio
async def test_release_claim_by_non_owner(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 4: alice claims
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    async with AsyncClient(app=app, base_url="http://test") as client:
        await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

        # bob attempts to release alice's claim
        response = await client.post(f"/runs/{thread_id}/release", json={"reviewer_id": "bob"})

    assert response.status_code == 403
    body = response.json()
    assert body["error"] == "not_owner"
    assert body["claim"]["reviewer_id"] == "alice"
```

**Why:** Verifies AC3 (only the owner can release).

---

### Test 6: Unit test — Handover with valid reason

**What it checks:** POST `/runs/{thread_id}/handover` with valid `from_reviewer`, `to_reviewer`, and `reason` returns 200 and updates claim.

**How:**

```python
@pytest.mark.asyncio
async def test_handover_with_reason(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 4: alice claims
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    async with AsyncClient(app=app, base_url="http://test") as client:
        await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

        # alice hands over to bob
        response = await client.post(f"/runs/{thread_id}/handover", json={
            "from_reviewer": "alice",
            "to_reviewer": "bob",
            "reason": "Waiting on client for bonus details"
        })

    assert response.status_code == 200
    body = response.json()
    assert body["status"] == "handed_over"
    assert body["claim"]["reviewer_id"] == "bob"

    # verify the state is updated
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        claim = snapshot.values["claim"]
        assert claim["reviewer_id"] == "bob"
        history = snapshot.values["claim_history"]
        assert len(history) == 2  # claim by alice + handover event
        assert history[1]["reviewer_id"] == "alice"
        assert history[1]["reason"] == "Waiting on client for bonus details"
```

**Why:** Verifies AC4 (handing over with reason) and AC5 (history tracking).

---

### Test 7: Unit test — Handover without reason

**What it checks:** POST `/runs/{thread_id}/handover` without `reason` returns 400.

**How:**

```python
@pytest.mark.asyncio
async def test_handover_without_reason(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 6: alice claims
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    async with AsyncClient(app=app, base_url="http://test") as client:
        await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

        # alice attempts to hand over without reason
        response = await client.post(f"/runs/{thread_id}/handover", json={
            "from_reviewer": "alice",
            "to_reviewer": "bob",
            "reason": ""
        })

    assert response.status_code == 422  # Pydantic validation error
    body = response.json()
    assert "reason" in str(body).lower()
```

**Why:** Verifies AC4 (reason is required for handover).

---

### Test 8: Unit test — Handover by non-owner

**What it checks:** POST `/runs/{thread_id}/handover` by non-owner returns 403.

**How:**

```python
@pytest.mark.asyncio
async def test_handover_by_non_owner(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 6: alice claims
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    async with AsyncClient(app=app, base_url="http://test") as client:
        await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})

        # bob attempts to hand over alice's claim
        response = await client.post(f"/runs/{thread_id}/handover", json={
            "from_reviewer": "bob",
            "to_reviewer": "charlie",
            "reason": "Not my case"
        })

    assert response.status_code == 403
    body = response.json()
    assert body["error"] == "not_owner"
    assert body["claim"]["reviewer_id"] == "alice"
```

**Why:** Verifies AC4 (only the owner can hand over).

---

### Test 9: Integration test — Claim, release, claim again by another reviewer

**What it checks:** Claim → release → claim again by another reviewer; verify history has all three events.

**How:**

```python
@pytest.mark.asyncio
async def test_claim_release_claim_history(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 4: pause at review
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    async with AsyncClient(app=app, base_url="http://test") as client:
        # alice claims the case
        response_claim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response_claim.status_code == 200

        # alice releases the case
        response_release = await client.post(f"/runs/{thread_id}/release", json={"reviewer_id": "alice"})
        assert response_release.status_code == 200

        # bob claims the case
        response_reclaim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "bob"})
        assert response_reclaim.status_code == 200

    # verify the history has all three events
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        history = snapshot.values["claim_history"]
        assert len(history) == 3
        assert history[0]["reviewer_id"] == "alice"
        assert "reason" not in history[0]
        assert history[1]["reviewer_id"] == "alice"
        assert history[1]["reason"] == "released"
        assert history[2]["reviewer_id"] == "bob"
        assert "reason" not in history[2]
```

**Why:** Verifies AC5 (claim history tracking: each claim, release shown).

---

### Test 10: Integration test — Claim, handover with reason, verify history

**What it checks:** Claim → handover with reason; verify history shows handover event with reason.

**How:**

```python
@pytest.mark.asyncio
async def test_claim_handover_history(monkeypatch):
    thread_id = str(uuid.uuid4())

    # same setup as Test 9: pause at review
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)

    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)

    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)

    await run_query(service_order_id="SO-TEST", thread_id=thread_id)

    async with AsyncClient(app=app, base_url="http://test") as client:
        # alice claims the case
        response_claim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response_claim.status_code == 200

        # alice hands over to bob
        response_handover = await client.post(f"/runs/{thread_id}/handover", json={
            "from_reviewer": "alice",
            "to_reviewer": "bob",
            "reason": "Waiting on client for bonus details"
        })
        assert response_handover.status_code == 200

    # verify the history has claim + handover event with reason
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        snapshot = await graph.aget_state(config)
        claim = snapshot.values["claim"]
        assert claim["reviewer_id"] == "bob"
        
        history = snapshot.values["claim_history"]
        assert len(history) == 2
        assert history[0]["reviewer_id"] == "alice"
        assert "reason" not in history[0]
        assert history[1]["reviewer_id"] == "alice"
        assert history[1]["reason"] == "Waiting on client for bonus details"
```

**Why:** Verifies AC5 (claim history tracking: handover event with reason shown).
