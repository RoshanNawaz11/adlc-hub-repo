# LLD — Story 01-agent: Add claim state fields and persistence

**Repo:** `agent`  
**Story:** Add claim state fields and persistence  
**Covers:** REQ-AC11 (persistence across restarts), REQ-AC12 (backward compatibility)

---

## Acceptance Criteria (from requirements/breakdown)

### AC11: Persistence across restarts
**Given** cases have claims, handovers, and history recorded  
**When** the service restarts  
**Then** all claim states, handover records, and history are preserved and available

### AC12: Backward compatibility
**Given** a team or environment that does not use the claim workflow  
**When** cases are processed  
**Then** case execution and review behavior remains unchanged for teams that never claim cases

---

## File Plan

All paths are relative to the `agent` repo root and match the existing tree. No new languages or stacks are introduced.

### Modified Files

| File | Change |
|------|--------|
| `src/agents/graph/states.py` | Add `Claim` TypedDict; add `claim`, `claim_history`, `needs_review_since` fields to `State` as `NotRequired` |
| `src/agents/nodes/orchestrator/node.py` | Set `needs_review_since` when `status` transitions to `"needs_review"` |

### New Files

| File | Purpose |
|------|---------|
| `tests/test_claim_state.py` | Unit and integration tests for claim state persistence and backward compatibility |

---

## Interfaces

### 1. State Schema Extensions

**File:** `src/agents/graph/states.py`

Add the `Claim` TypedDict before the `State` definition:

```python
class Claim(TypedDict):
    reviewer_id: str
    claimed_at: str  # ISO 8601 UTC timestamp
    reason: NotRequired[str]  # populated on handover only
```

Add three new fields to the `State` TypedDict (after the existing `review` field, before `priority`):

```python
class State(TypedDict):
    # ... existing fields ...
    review: NotRequired[Review]
    claim: NotRequired[Claim | None]
    claim_history: NotRequired[list[Claim]]
    needs_review_since: NotRequired[str]  # ISO 8601 UTC timestamp
    priority: NotRequired[Literal["high", "medium", "low"]]
    # ... rest of fields ...
```

**Rationale:**

- `claim`: current claim (reviewer ID, timestamp, optional handover reason), or `None` when unclaimed
- `claim_history`: append-only log of every claim, release, and handover (each entry is a `Claim` with `reason` populated where applicable)
- `needs_review_since`: timestamp when the case entered `status: "needs_review"`; used to compute waiting duration for lead monitoring (Story 5)
- All three are `NotRequired` → existing cases without them remain valid (AC12)

### 2. Setting `needs_review_since`

**File:** `src/agents/nodes/orchestrator/node.py`

The orchestrator node already computes `status` based on the route and sets it in the `update` dict returned at the end of `orchestrator_node()`. The current code (lines ~130-145 in the provided file) is:

```python
update["status"] = (
    "rejected" if review == "reject"
    else "needs_input" if route == [FLAG_REVIEW]
    else "needs_review" if route == ["review"]
    else "done" if route == [END]
    else "running"
)
```

**Change:** Add logic immediately after this block to set `needs_review_since` when the status transitions to `"needs_review"` for the first time:

```python
# Set needs_review_since when status transitions to needs_review for the first time
if update["status"] == "needs_review" and "needs_review_since" not in state:
    from datetime import UTC, datetime
    update["needs_review_since"] = datetime.now(UTC).isoformat()
```

**Rationale:**

- The orchestrator is the only node that sets `status` to `"needs_review"` (when routing to `["review"]`)
- Setting the timestamp only when the field is absent ensures it captures the *first* transition to review, not subsequent re-routes (e.g., after the output validator replans)
- The import of `datetime` is already present at the top of `states.py`; importing it locally here avoids adding a new module-level dependency to the orchestrator

**Alternative location (if needed):** If the orchestrator does not set `status` directly but the review node does, the same logic can be placed in `src/agents/graph/interrupts.py:review_node()`. However, examining the orchestrator code shows it already sets `status`, so the orchestrator is the correct location.

---

## Algorithm Notes

### Persistence Verification

**How the checkpointer works:**

- The existing `get_checkpointer()` (in `src/agents/runtime/checkpointer.py`) returns a `BaseCheckpointSaver` — either `InMemorySaver` (for tests) or `AsyncSqliteSaver` (for production)
- Every LangGraph node update writes the full `State` dict to the checkpointer (via `graph.update_state()` or the graph's internal checkpoint write after a node)
- On service restart, `open_graph(thread_id)` (in `src/agents/runtime/runner.py`) opens the same sqlite file and `graph.aget_state(config)` restores the last checkpoint, including all fields in `State`

**No new persistence code required:**

- The new fields are part of the `State` TypedDict → they are automatically included in every checkpoint write
- The sqlite checkpointer serializes `TypedDict` fields as JSON → `Claim` (with `str` and optional `str` fields) and `list[Claim]` are natively serializable
- Restoration reads the JSON back into the `State` dict → the `NotRequired` fields are present if they were written, absent if they were not (backward compatibility)

**Test approach:**

1. Start a run, pause it at review (triggers `status: "needs_review"` and sets `needs_review_since`)
2. Close the graph (closes the sqlite connection)
3. Restart the graph with the same `thread_id` (reopens the same checkpoint.db)
4. Resume the run (`run_query(..., resume=True)`)
5. Verify `state["needs_review_since"]` is the same value as before the restart (read from the restored state via `graph.aget_state(config)`)

### Setting `needs_review_since` Logic

**Conditions:**

- Set only when `status` transitions to `"needs_review"` (not on every re-route)
- Check `"needs_review_since" not in state` to avoid overwriting the original timestamp if the orchestrator re-routes to review multiple times (e.g., after a replan)

**Edge case:** If a run never reaches review (e.g., rejected at planning or failed mid-run), `needs_review_since` remains absent. This is correct: the field is only meaningful for cases in `status: "needs_review"`.

---

## Tests

**File:** `tests/test_claim_state.py` (new)

All tests use `pytest` and `pytest-asyncio` (already in the repo's test suite). The integration tests use the same `open_graph()` and `run_query()` helpers as the existing tests (e.g., `tests/test_validation_agent.py`).

### Test 1: Unit test — State schema includes new fields

**What it checks:** The `State` TypedDict has `claim`, `claim_history`, and `needs_review_since` as `NotRequired` fields with correct types.

**How:**

```python
from typing import get_type_hints
from agents.graph.states import State, Claim

def test_state_schema_has_claim_fields():
    hints = get_type_hints(State)
    # verify claim is NotRequired[Claim | None]
    assert "claim" in hints
    # verify claim_history is NotRequired[list[Claim]]
    assert "claim_history" in hints
    # verify needs_review_since is NotRequired[str]
    assert "needs_review_since" in hints
```

**Why:** Catches a typo or missing field at the schema level before runtime.

---

### Test 2: Unit test — `needs_review_since` is set on transition

**What it checks:** When the orchestrator sets `status: "needs_review"`, it also sets `needs_review_since` if not already present.

**How:**

```python
from datetime import UTC, datetime
from agents.nodes.orchestrator.node import orchestrator_node
from agents.graph.states import State

@pytest.mark.asyncio
async def test_needs_review_since_set_on_transition(monkeypatch):
    # mock the orchestrator's skill call to return a route to review
    async def mock_decide(state, gateway, rejected):
        return {"route": ["review"], "summary": "Test summary"}
    
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    state: State = {
        "schema_version": 1,
        "run_id": "test-run",
        "status": "running",
        "flagged": False,
        "results": {"planner": {"priority": "medium"}},
        "steps": [],
        "flags": [],
        "needs": [],
        "runs": {},
    }
    
    update = await orchestrator_node(state)
    
    assert update["status"] == "needs_review"
    assert "needs_review_since" in update
    # verify it's a valid ISO 8601 timestamp
    ts = datetime.fromisoformat(update["needs_review_since"])
    assert ts.tzinfo == UTC
```

**Why:** Verifies the orchestrator logic without running the full graph.

---

### Test 3: Integration test — claim fields persist across restart

**What it checks:** Start a run, pause it at review (sets `needs_review_since`), restart the service (close and reopen the graph), resume the run, and verify all claim fields are preserved.

**How:**

```python
import uuid
from pathlib import Path
from agents.runtime.runner import open_graph, run_query
from agents.runtime.run_dir import run_dir

@pytest.mark.asyncio
async def test_claim_fields_persist_across_restart(monkeypatch):
    thread_id = str(uuid.uuid4())
    
    # mock fetch_service_order to avoid external dependency
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)
    
    # mock report_progress to avoid external dependency
    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)
    
    # start a run and pause at review
    out = await run_query(service_order_id="SO-TEST", thread_id=thread_id)
    assert out["paused"]
    assert "review" in out["next"]
    
    # read the state before restart
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        before = await graph.aget_state(config)
        needs_review_since_before = before.values.get("needs_review_since")
        assert needs_review_since_before is not None  # set by orchestrator
    
    # simulate restart: close the graph and reopen it
    # (the checkpointer's sqlite connection is closed by open_graph's context manager)
    
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        after = await graph.aget_state(config)
        
        # verify needs_review_since is preserved
        assert after.values.get("needs_review_since") == needs_review_since_before
        
        # verify claim and claim_history are present (even if None/empty)
        assert "claim" in after.values or after.values.get("claim") is None
        assert "claim_history" in after.values or after.values.get("claim_history") == []
```

**Why:** Exercises the full persistence path (write → close → reopen → read) without new persistence code.

---

### Test 4: Integration test — backward compatibility (unclaimed case)

**What it checks:** Run a case end-to-end without claiming it; verify no errors and `claim` is `None` or absent.

**How:**

```python
@pytest.mark.asyncio
async def test_backward_compatibility_unclaimed_case(monkeypatch):
    thread_id = str(uuid.uuid4())
    
    # same mocks as Test 3
    async def mock_fetch(so_id):
        return {"id": so_id, "caseType": "Offboarding", "employee": {"id": "EMP-1"}}
    monkeypatch.setattr("agents.runtime.runner.fetch_service_order", mock_fetch)
    
    async def mock_report(*args, **kwargs):
        pass
    monkeypatch.setattr("agents.runtime.backend.report_progress", mock_report)
    
    # run a case end-to-end (mock the orchestrator to route straight to END)
    async def mock_decide(state, gateway, rejected):
        return {"route": ["END"], "summary": "Test case completed"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    out = await run_query(service_order_id="SO-TEST", thread_id=thread_id)
    
    # verify the run completed without errors
    assert not out["paused"]
    assert out["values"]["status"] == "done"
    
    # verify claim is None or absent (not set)
    claim = out["values"].get("claim")
    assert claim is None or "claim" not in out["values"]
    
    # verify claim_history is empty or absent
    history = out["values"].get("claim_history")
    assert history == [] or "claim_history" not in out["values"]
```

**Why:** Confirms that the new fields do not break existing workflows (AC12).

---

## Dependencies

**None.** This is the first story.

**External dependencies:**

- Existing checkpointer (sqlite or memory) persists `State` fields → verified in Test 3
- Existing orchestrator node sets `status` → verified in Test 2
- Existing `run_query()` and `open_graph()` helpers → used by Tests 3 and 4

---

## Risks and Mitigations

### Risk 1: Checkpointer does not serialize `TypedDict` correctly

**Likelihood:** Low — the checkpointer already serializes `Flag`, `Need`, and other `TypedDict` fields in the current `State`.

**Mitigation:** Test 3 exercises the full write → close → reopen → read path. If the checkpointer fails to serialize `Claim`, the test will catch it.

**Fallback:** If `TypedDict` serialization fails, define `Claim` as a `dict[str, Any]` instead (less type-safe, but functionally identical).

### Risk 2: The orchestrator is not the node that sets `status: "needs_review"`

**Likelihood:** Low — the provided `orchestrator_node()` code shows it explicitly sets `status` based on the route.

**Mitigation:** Test 2 verifies the logic in isolation. If the orchestrator does not set `status`, the test will fail and we will move the logic to the correct node (likely `src/agents/graph/interrupts.py:review_node()`).

**Investigation:** The provided `edges.py` and `orchestrator/node.py` files show the orchestrator is the only node that writes `status` (except `runner.py` at initialization). The `review_node()` in `interrupts.py` does not set `status` — it only validates the review decision.

### Risk 3: `needs_review_since` is set multiple times

**Likelihood:** Low — the check `"needs_review_since" not in state` prevents overwriting.

**Edge case:** If a run is resumed with a fresh `State` dict (e.g., a bug in the resume path), the timestamp would be reset. However, the current `run_query()` code (in `runner.py`) does not pass a fresh state on resume — it uses `Command(resume=decision)`, which updates the existing checkpoint, not the state schema.

**Mitigation:** Test 3 restarts the service and resumes the run, verifying the timestamp is unchanged.

---

## Open Questions

*None at this time. All design choices are documented in the HLD and above.*
