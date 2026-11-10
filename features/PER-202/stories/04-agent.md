# LLD — Story 04-agent: Add ownership check middleware and modify resume endpoint

**Repo:** `agent`  
**Story:** Add ownership check middleware and modify resume endpoint  
**Covers:** REQ-AC2 (claim exclusivity: only owner can act)

---

## Acceptance Criteria (from requirements/breakdown)

### AC2: Claim exclusivity
**Given** a case is claimed by Reviewer A  
**When** Reviewer B attempts to answer flags, approve, or reject the case  
**Then** Reviewer B receives a message indicating the case is claimed by Reviewer A and cannot proceed

---

## File Plan

All paths are relative to the `agent` repo root and match the existing tree. No new languages or stacks are introduced.

### Modified Files

| File | Change |
|------|--------|
| `src/agents/runtime/server.py` | Add `_require_owner()` middleware function; extend `FlagAnswers` and `ReviewDecision` models to include `reviewer_id`; modify `resume_run` endpoint to check ownership before processing |

### New Files

| File | Purpose |
|------|---------|
| `tests/test_ownership_check.py` | Unit and integration tests for ownership check middleware and resume endpoint authorization |

---

## Interfaces

### 1. Ownership Check Middleware

**File:** `src/agents/runtime/server.py`

Add the `_require_owner` function after the existing `_publish` function (before `_execute`):

```python
def _require_owner(state: dict, reviewer_id: str) -> tuple[bool, dict | None]:
    """Check if the reviewer owns the claim on a case. Returns (authorized, error).
    
    Authorization rules:
    - If the case is unclaimed (claim is None or absent), anyone can act → (True, None)
    - If the case is claimed and reviewer_id matches claim.reviewer_id → (True, None)
    - If the case is claimed by someone else → (False, error dict with claim details)
    """
    claim = state.get("claim")
    if not isinstance(claim, dict):
        # unclaimed case: no ownership restriction
        return True, None
    
    claimed_by = claim.get("reviewer_id")
    if claimed_by == reviewer_id:
        # reviewer owns the claim
        return True, None
    
    # reviewer does not own the claim
    return False, {
        "error": "not_owner",
        "message": f"Case is claimed by {claimed_by}",
        "claim": claim,
    }
```

**Logic:**

- `state` is the run's current state dict (from `snapshot.values`)
- `reviewer_id` is the reviewer attempting to act (from request body)
- Returns `(True, None)` when authorized (unclaimed case, or reviewer owns the claim)
- Returns `(False, error_dict)` when unauthorized (another reviewer owns the claim)
- The error dict carries `claim` so the client can show who has claimed it and when

**Design choice:** Unclaimed cases are not restricted (AC12: backward compatibility). Only claimed cases enforce ownership.

### 2. Request Model Extensions

**File:** `src/agents/runtime/server.py`

Extend the existing `FlagAnswers` and `ReviewDecision` models to include `reviewer_id`:

**Before (current code):**

```python
class ReviewDecision(BaseModel):
    """The reviewer's approval or rejection of the whole run (graph.states.Review)."""
    action: Literal["approve", "reject"]
    reason: str | None = None

    @model_validator(mode="after")
    def _reason_on_reject(self) -> "ReviewDecision":
        if self.action == "reject" and not self.reason:
            raise ValueError("a reject needs a reason")
        return self


class FlagAnswers(BaseModel):
    """The analyst's answers to every flag the run is paused on (graph.interrupts.flag_review_node)."""
    answers: list[FlagAnswer]
```

**After (modified code):**

```python
class ReviewDecision(BaseModel):
    """The reviewer's approval or rejection of the whole run (graph.states.Review)."""
    action: Literal["approve", "reject"]
    reason: str | None = None
    reviewer_id: str  # NEW: the reviewer performing the action

    @model_validator(mode="after")
    def _reason_on_reject(self) -> "ReviewDecision":
        if self.action == "reject" and not self.reason:
            raise ValueError("a reject needs a reason")
        return self


class FlagAnswers(BaseModel):
    """The analyst's answers to every flag the run is paused on (graph.interrupts.flag_review_node)."""
    answers: list[FlagAnswer]
    reviewer_id: str  # NEW: the reviewer performing the action
```

**Rationale:**

- `reviewer_id` is added as a required field to both models
- No default value → clients must explicitly provide it on every resume request
- The field is not optional because authorization depends on it (enforcing AC2 requires knowing who is acting)
- This is MVP-level auth (no cryptographic verification) — the client sends the ID, and we trust it. Real auth (JWT, session tokens) is deferred to future.

### 3. Resume Endpoint Modification

**File:** `src/agents/runtime/server.py`

Modify the existing `resume_run` endpoint to check ownership before processing:

**Before (current code, lines ~150-165):**

```python
@app.post("/runs/{thread_id}/resume", status_code=202)
async def resume_run(thread_id: str, decision: FlagAnswers | ReviewDecision) -> dict[str, Any]:
    if _runs.get(thread_id, {}).get("status") == "running":
        raise HTTPException(409, f"Run {thread_id} is running")
    # the checkpointer, not _runs, says whether it is paused — it outlives a restart
    snapshot = await _snapshot(thread_id)
    if not snapshot.next:
        raise HTTPException(409, f"Run {thread_id} is not paused")
    on_flags = FLAG_REVIEW in snapshot.next
    if on_flags != isinstance(decision, FlagAnswers):
        expected = "the answers to its flags" if on_flags else "approve or reject"
        raise HTTPException(422, f"Run {thread_id} is paused at {list(snapshot.next)} — send {expected}")
    body = decision.model_dump(exclude_none=True)
    if on_flags and (missing := missing_answers(snapshot.values or {}, body)):
        raise HTTPException(422, f"Answer every flag before the case continues — no answer yet for {missing}")
    return _start(thread_id, None, resume=True, decision=body)
```

**After (modified code):**

```python
@app.post("/runs/{thread_id}/resume", status_code=202)
async def resume_run(thread_id: str, decision: FlagAnswers | ReviewDecision) -> dict[str, Any]:
    if _runs.get(thread_id, {}).get("status") == "running":
        raise HTTPException(409, f"Run {thread_id} is running")
    # the checkpointer, not _runs, says whether it is paused — it outlives a restart
    snapshot = await _snapshot(thread_id)
    if not snapshot.next:
        raise HTTPException(409, f"Run {thread_id} is not paused")
    on_flags = FLAG_REVIEW in snapshot.next
    if on_flags != isinstance(decision, FlagAnswers):
        expected = "the answers to its flags" if on_flags else "approve or reject"
        raise HTTPException(422, f"Run {thread_id} is paused at {list(snapshot.next)} — send {expected}")
    
    # NEW: check ownership before allowing the action
    authorized, error = _require_owner(snapshot.values or {}, decision.reviewer_id)
    if not authorized:
        raise HTTPException(403, json.dumps(error))
    
    body = decision.model_dump(exclude_none=True)
    if on_flags and (missing := missing_answers(snapshot.values or {}, body)):
        raise HTTPException(422, f"Answer every flag before the case continues — no answer yet for {missing}")
    return _start(thread_id, None, resume=True, decision=body)
```

**Changes:**

1. After loading the snapshot and before processing the decision, call `_require_owner(snapshot.values, decision.reviewer_id)`
2. If unauthorized (`authorized == False`), raise `HTTPException(403, json.dumps(error))`
3. The `error` dict is serialized to JSON so the client can parse it and show the claim details (who has claimed it, when)
4. The rest of the endpoint logic is unchanged — if authorized, the resume proceeds as before

**Error response format:**

```json
{
  "error": "not_owner",
  "message": "Case is claimed by alice",
  "claim": {
    "reviewer_id": "alice",
    "claimed_at": "2025-01-15T10:42:00.123456+00:00"
  }
}
```

**HTTP status codes:**

- 403 Forbidden: reviewer does not own the claim (AC2)
- 409 Conflict: run is not paused, or already running (existing validation)
- 422 Unprocessable Entity: wrong decision type for the pause point, or missing flag answers (existing validation)

---

## Algorithm Notes

### Ownership Check Logic

**When is a case "claimed"?**

- The `claim` field in `State` (added in Story 1) is `NotRequired[Claim | None]`
- A case is **unclaimed** when `claim` is `None` or absent (not in the state dict)
- A case is **claimed** when `claim` is a dict with `reviewer_id` and `claimed_at`

**Authorization rules:**

1. **Unclaimed case:** Anyone can answer flags, approve, or reject (no ownership restriction)
   - Rationale: AC12 (backward compatibility) — teams that never claim cases should see no change
   - The claim workflow is opt-in — if no one ever calls `POST /runs/{thread_id}/claim`, the case remains unclaimed throughout
2. **Claimed case, matching reviewer:** The reviewer who claimed the case can answer flags, approve, or reject
   - Rationale: AC2 (only the owner can act)
3. **Claimed case, non-matching reviewer:** Return 403 with the claim details
   - Rationale: AC2 (everyone else sees who has claimed it and is blocked)

**Edge cases:**

- **Claim field malformed (not a dict):** Treat as unclaimed → allow the action (defensive: corrupt state should not lock out all reviewers)
- **Claim missing `reviewer_id` field:** Treat as unclaimed → allow the action (same rationale)
- **Case transitions from claimed to unclaimed (release) mid-resume:** The ownership check reads the snapshot at the moment the request arrives. If the claim is released between the check and the graph execution, the graph sees the release (no conflict). If the claim is released after the check but before the response, the resume still proceeds (the release is a separate operation, not a conflicting one).

### Placement in the Request Flow

**Why check ownership in `resume_run`, not in the graph?**

- The graph (LangGraph nodes) operates on the state machine — it does not know about HTTP requests or who sent them
- The `reviewer_id` is an HTTP-level concern (request body), not a graph-level concern (state)
- Checking ownership in the endpoint keeps the graph logic clean and the auth logic in the HTTP layer, where it belongs

**Sequence:**

1. Client sends `POST /runs/{thread_id}/resume` with `{"reviewer_id": "alice", ...}`
2. Server checks if the run is paused (existing logic)
3. Server loads the run's state from the checkpointer (`await _snapshot(thread_id)`)
4. **NEW:** Server calls `_require_owner(state, "alice")` before processing the decision
5. If authorized → server processes the decision (existing logic: calls `_start(thread_id, None, resume=True, decision=body)`)
6. If unauthorized → server returns 403 with the claim details (new logic)

**Why not check ownership in `_start` or `_execute`?**

- `_start` and `_execute` are generic run-execution functions used by both `create_run` (no reviewer) and `resume_run` (has reviewer)
- Checking ownership in `resume_run` keeps the logic close to where `reviewer_id` is available (the request body)
- If future endpoints need ownership checks (e.g. `POST /runs/{thread_id}/cancel`), they can call `_require_owner` directly

### Backward Compatibility

**Cases without claims (AC12):**

- Existing runs (created before Story 1) have no `claim` field in their state
- The ownership check treats these as unclaimed → any reviewer can act
- This matches the current behavior (no ownership restrictions before this feature)

**Cases that never claim:**

- A team that never calls `POST /runs/{thread_id}/claim` processes cases as today — no reviewer IDs, no ownership checks
- The `reviewer_id` field in `FlagAnswers` and `ReviewDecision` is required by the schema, but the ownership check allows any value when the case is unclaimed
- For teams that never claim, the reviewer ID could be a placeholder (e.g. `"system"` or the team name) with no functional impact

---

## Tests

**File:** `tests/test_ownership_check.py` (new)

All tests use `pytest` and `pytest-asyncio` (already in the repo's test suite). The integration tests use the same `open_graph()` and `run_query()` helpers as the existing tests (e.g. `tests/test_validation_agent.py`).

### Test 1: Unit test — `_require_owner` returns `(True, None)` when reviewer owns the claim

**What it checks:** The middleware function authorizes the action when the reviewer owns the claim.

**How:**

```python
import pytest
from agents.runtime.server import _require_owner

def test_require_owner_authorizes_claim_owner():
    state = {
        "claim": {
            "reviewer_id": "alice",
            "claimed_at": "2025-01-15T10:42:00.123456+00:00",
        }
    }
    
    authorized, error = _require_owner(state, "alice")
    
    assert authorized is True
    assert error is None
```

**Why:** Verifies the happy path (AC2: the owner can act).

---

### Test 2: Unit test — `_require_owner` returns `(False, error)` when reviewer does not own the claim

**What it checks:** The middleware function blocks the action when the reviewer does not own the claim, and returns the claim details in the error.

**How:**

```python
def test_require_owner_blocks_non_owner():
    state = {
        "claim": {
            "reviewer_id": "alice",
            "claimed_at": "2025-01-15T10:42:00.123456+00:00",
        }
    }
    
    authorized, error = _require_owner(state, "bob")
    
    assert authorized is False
    assert error is not None
    assert error["error"] == "not_owner"
    assert "alice" in error["message"]
    assert error["claim"]["reviewer_id"] == "alice"
```

**Why:** Verifies AC2 (non-owners are blocked and see who has claimed it).

---

### Test 3: Unit test — `_require_owner` returns `(True, None)` when there is no claim

**What it checks:** The middleware function allows the action when the case is unclaimed (backward compatibility).

**How:**

```python
def test_require_owner_allows_unclaimed_case():
    # case 1: claim is None
    state_none = {"claim": None}
    authorized, error = _require_owner(state_none, "bob")
    assert authorized is True
    assert error is None
    
    # case 2: claim is absent
    state_absent = {}
    authorized, error = _require_owner(state_absent, "bob")
    assert authorized is True
    assert error is None
    
    # case 3: claim is not a dict (malformed)
    state_malformed = {"claim": "invalid"}
    authorized, error = _require_owner(state_malformed, "bob")
    assert authorized is True
    assert error is None
```

**Why:** Verifies AC12 (backward compatibility: unclaimed cases have no ownership restriction).

---

### Test 4: Integration test — Claim as Reviewer A, attempt to resume (answer flags) as Reviewer B

**What it checks:** When a case is claimed by Reviewer A, Reviewer B cannot answer its flags and receives a 403 response with the claim details.

**How:**

```python
import uuid
from datetime import UTC, datetime
import pytest
from httpx import AsyncClient
from agents.runtime.server import app
from agents.runtime.runner import open_graph, run_query

@pytest.mark.asyncio
async def test_non_owner_cannot_answer_flags(monkeypatch):
    thread_id = str(uuid.uuid4())
    
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
    # and raise a flag so FLAG_REVIEW runs first
    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Ready for review"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    # mock validation agent to raise a flag
    async def mock_validation(state, gateway=None):
        return {
            "flags": [{
                "flag_id": "validation-1",
                "type": "missingData",
                "stage": "validation",
                "title": "Missing employee ID",
                "detail": {},
                "resolved": False,
                "decisions": [],
            }],
            "results": {"validation": {"output": {"status": "flagged"}}},
            "steps": [],
        }
    monkeypatch.setattr("agents.nodes.validation.node.validation_node", mock_validation)
    
    # start a run and pause it at flag review
    await run_query(service_order_id="SO-TEST", thread_id=thread_id)
    
    # alice claims the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_claim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response_claim.status_code == 200
    
    # bob attempts to answer the flag
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_resume = await client.post(
            f"/runs/{thread_id}/resume",
            json={
                "reviewer_id": "bob",
                "answers": [{"flag_id": "validation-1", "action": "accept"}]
            }
        )
    
    assert response_resume.status_code == 403
    body = response_resume.json()
    assert body["error"] == "not_owner"
    assert "alice" in body["message"]
    assert body["claim"]["reviewer_id"] == "alice"
```

**Why:** Exercises the full flow (claim → attempt to resume by non-owner) and verifies AC2 (403 with claim details).

---

### Test 5: Integration test — Claim as Reviewer A, resume (answer flags) as Reviewer A

**What it checks:** When a case is claimed by Reviewer A, Reviewer A can answer its flags and the resume proceeds (202 response).

**How:**

```python
@pytest.mark.asyncio
async def test_owner_can_answer_flags(monkeypatch):
    thread_id = str(uuid.uuid4())
    
    # same setup as Test 4: mock fetch, report_progress, orchestrator, validation agent
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
        # first call: route to flag-review (pause on flag)
        # second call (after flag answer): route to review (pause at final review)
        if state.get("results", {}).get("validation", {}).get("rerun"):
            return {"route": ["review"], "summary": "Ready for review"}
        return {"route": ["flag-review"], "summary": "Flagged"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    async def mock_validation(state, gateway=None):
        # first call: raise a flag
        # second call (after flag answer): clear the flag
        if state.get("results", {}).get("validation", {}).get("answered"):
            return {
                "flags": [],
                "results": {"validation": {"output": {"status": "ready"}}},
                "steps": [],
            }
        return {
            "flags": [{
                "flag_id": "validation-1",
                "type": "missingData",
                "stage": "validation",
                "title": "Missing employee ID",
                "detail": {},
                "resolved": False,
                "decisions": [],
            }],
            "results": {"validation": {"output": {"status": "flagged"}}},
            "steps": [],
        }
    monkeypatch.setattr("agents.nodes.validation.node.validation_node", mock_validation)
    
    # start a run and pause it at flag review
    await run_query(service_order_id="SO-TEST", thread_id=thread_id)
    
    # alice claims the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_claim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response_claim.status_code == 200
    
    # alice answers the flag
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_resume = await client.post(
            f"/runs/{thread_id}/resume",
            json={
                "reviewer_id": "alice",
                "answers": [{"flag_id": "validation-1", "action": "accept"}]
            }
        )
    
    assert response_resume.status_code == 202
    assert response_resume.json()["status"] == "running"
    
    # wait for the run to pause again (at final review)
    import asyncio
    await asyncio.sleep(1)
    
    # verify the run continued (reached final review)
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_status = await client.get(f"/runs/{thread_id}")
    
    assert response_status.status_code == 200
    state = response_status.json()
    assert "review" in state["next"]  # paused at final review
    assert state["values"]["results"]["validation"]["output"]["status"] == "ready"
```

**Why:** Verifies AC2 (the owner can act) and that the resume proceeds normally (no authorization errors).

---

## Dependencies

**Story execution order:**

- **Depends on:** Story 2 (claim endpoints) — the ownership check reads the `claim` field added by Story 2
- **After:** Story 2

**External dependencies:**

- Existing `_snapshot` helper (reads the run's state from the checkpointer) — used by `resume_run`
- Existing `FlagAnswers` and `ReviewDecision` models (extended to include `reviewer_id`)
- Existing `missing_answers` helper (validates flag answers) — called after the ownership check
- Existing `_start` function (enqueues the resume task) — called after the ownership check

---

## Risks and Mitigations

### Risk 1: Reviewer ID is not authenticated (MVP-level security)

**Risk:** The client sends `reviewer_id` in the request body, and the server trusts it without cryptographic verification. A malicious client could impersonate another reviewer by sending their ID.

**Mitigation:** Accept this for MVP (the ticket says "defer real auth to future"). The reviewer ID is a trust-based identifier, like a username in a low-security environment. Real auth (JWT tokens, session cookies, API keys with signature verification) is a separate feature.

**Residual risk:** In a real deployment, this must be hardened (extract `reviewer_id` from a signed JWT or session token, not the request body). For MVP, document this as a known limitation.

**Documentation note:** Add to the HLD or a separate security doc: "The reviewer_id field is currently trust-based (sent by the client). Before production deployment, replace this with authenticated claims from a session token or JWT."

### Risk 2: Race condition (claim released between check and execution)

**Risk:** Reviewer A claims a case, starts resuming it (passes the ownership check), but another reviewer releases the claim (or A releases it themselves) before the graph executes. The resume proceeds with stale authorization.

**Mitigation:** Accept this for MVP (the ownership check is a point-in-time check, not a lock). The claim state is eventually consistent: if the claim is released during the resume, the next action will see it as unclaimed.

**Residual risk:** Low. The window is small (milliseconds between the ownership check and the graph starting), and the impact is minor (one resume proceeds with stale authorization, but the case is no longer claimed afterward, so future actions are correctly unrestricted).

**Alternative (future):** Use optimistic locking (check the claim version in the checkpointer, fail the resume if it changed). Defer to future optimization.

### Risk 3: `FlagAnswers` and `ReviewDecision` schema breaking change

**Risk:** Adding `reviewer_id` as a required field breaks existing clients that call `POST /runs/{thread_id}/resume` without it.

**Mitigation:** This is acceptable for MVP (the feature introduces claim ownership, so clients must opt in by adding `reviewer_id`). For a smoother rollout:

1. Make `reviewer_id` optional (`str | None = None`) in the MVP
2. If absent, treat the request as coming from an "anonymous" reviewer (any ID works for unclaimed cases, but fails for claimed cases with a 403)
3. Clients update their code to send `reviewer_id` explicitly
4. After all clients are updated, make `reviewer_id` required (remove the default)

**Recommendation:** Start with required `reviewer_id` (cleaner API, forces clients to think about ownership up front). If rollout friction is too high, switch to the optional approach.

### Risk 4: Existing tests break (schema change)

**Risk:** Tests that call `resume_run` (e.g. `tests/test_flag_review.py`) do not send `reviewer_id` and fail with Pydantic validation errors.

**Mitigation:** Update those tests to send `reviewer_id` (a placeholder like `"test-reviewer"` or `"system"`). The ownership check allows it (unclaimed cases have no restriction).

**Affected tests:** Any test that calls `POST /runs/{thread_id}/resume` or constructs a `FlagAnswers` or `ReviewDecision` object directly. Grep for `resume_run`, `FlagAnswers(`, `ReviewDecision(`.

---

## Open Questions

*None at this time. All design choices are documented in the HLD and above.*
