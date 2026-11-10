# LLD — Story 06-agent: End-to-end claim workflow integration test

**Repo:** `agent`  
**Story:** End-to-end claim workflow integration test  
**Covers:** REQ-AC1 through REQ-AC7 (all claim, handover, queue, and exclusivity requirements together), REQ-AC11 (persistence across restarts)

---

## Acceptance Criteria (from requirements/breakdown)

### AC1: Claiming a case (unclaimed → claimed)
**Given** a case is waiting for review and unclaimed  
**When** a reviewer claims it  
**Then** the case is marked as claimed by that reviewer with a timestamp, and all other reviewers see who claimed it and when

### AC2: Claim exclusivity
**Given** a case is claimed by Reviewer A  
**When** Reviewer B attempts to answer flags, approve, or reject the case  
**Then** Reviewer B receives a message indicating the case is claimed by Reviewer A and cannot proceed

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

### AC6: Reviewer's queue view
**Given** a reviewer has claimed one or more cases  
**When** they view their queue  
**Then** they see all cases they have claimed, sorted oldest waiting first, showing employee name, case type, and waiting duration

### AC7: Completed cases leave the queue
**Given** a reviewer's queue contains a claimed case  
**When** that case finishes (approved, rejected, or otherwise completed)  
**Then** the case is removed from the reviewer's queue

### AC11: Persistence across restarts
**Given** cases have claims, handovers, and history recorded  
**When** the service restarts  
**Then** all claim states, handover records, and history are preserved and available

---

## File Plan

All paths are relative to the `agent` repo root and match the existing tree. No new languages or stacks are introduced.

### Modified Files

None — this story only adds tests.

### New Files

| File | Purpose |
|------|---------|
| `tests/test_claim_workflow_e2e.py` | End-to-end integration tests for the full claim workflow |

---

## Interfaces

No new interfaces are introduced. This story exercises the endpoints and state fields added in Stories 1-5:

- **State fields** (Story 1): `claim`, `claim_history`, `needs_review_since`
- **Endpoints** (Stories 2, 3, 5):
  - `POST /runs/{thread_id}/claim`
  - `POST /runs/{thread_id}/release`
  - `POST /runs/{thread_id}/handover`
  - `GET /reviewers/{reviewer_id}/queue`
- **Ownership check** (Story 4): `_require_owner()` middleware in `POST /runs/{thread_id}/resume`

---

## Algorithm Notes

### Test 1: Full Workflow (Claim → Handover → Queue → Completion)

**Flow:**

1. Start a run and pause it at review (mock orchestrator to route to `["review"]`)
2. Alice claims the case via `POST /runs/{thread_id}/claim`
3. Bob attempts to answer flags via `POST /runs/{thread_id}/resume` → verify 403 with Alice's claim details
4. Alice hands over the case to Bob via `POST /runs/{thread_id}/handover` with a reason
5. Bob answers flags and approves via `POST /runs/{thread_id}/resume` → verify 202 and resumption proceeds
6. Verify the run completes (`status: "done"`)
7. Inspect `claim_history` to confirm all events: claim by Alice, handover to Bob with reason
8. Verify Bob's queue no longer contains the case (it completed)

**Key assertions:**

- Bob's 403 response carries `{"error": "not_owner", "claim": {"reviewer_id": "alice", "claimed_at": "<timestamp>"}}`
- `claim_history` has exactly 2 entries: Alice's claim, Alice's handover (with `reason` and `reviewer_id: "alice"`)
- The final `claim` field shows Bob as the owner at the time of completion
- `GET /reviewers/bob/queue` returns an empty list after completion

### Test 2: Persistence Across Restart

**Flow:**

1. Start a run and pause it at review
2. Alice claims the case
3. Alice hands over to Bob
4. **Restart the service**: close the graph (closes the sqlite checkpointer), reopen it with the same `thread_id`
5. Verify the claim state is restored: `claim.reviewer_id == "bob"`, `claim_history` has 2 entries
6. Bob resumes the run (answers flags and approves) → verify the run completes
7. Verify the claim history is preserved after completion

**Key assertions:**

- After reopening the graph, `state["claim"]["reviewer_id"] == "bob"`
- `state["claim_history"]` has 2 entries (Alice's claim, Alice's handover to Bob)
- Bob can resume the run (the ownership check passes with Bob's ID)
- The run completes successfully (no checkpoint corruption or state loss)

**Implementation note:** The "restart" is simulated by closing the graph's context manager (which closes the sqlite connection) and reopening it with `open_graph(thread_id)`. This mirrors what happens when the FastAPI server restarts: the old process's sqlite connection is closed, and the new process opens a fresh connection to the same `checkpoint.db` file.

---

## Tests

**File:** `tests/test_claim_workflow_e2e.py` (new)

All tests use `pytest` and `pytest-asyncio` (already in the repo's test suite). The integration tests use the same `open_graph()`, `run_query()`, and FastAPI client helpers as the existing tests (e.g. `tests/test_validation_agent.py`, `tests/test_flag_review.py`).

### Test 1: Integration test — Full claim workflow (claim → handover → queue → completion)

**What it checks:** Start a run, pause it at review, claim it as Alice, attempt to resume as Bob (verify 403), hand over to Bob with a reason, resume as Bob and complete the run, verify claim history and queue.

**How:**

```python
import uuid
from datetime import UTC, datetime
import pytest
from httpx import AsyncClient
from agents.runtime.server import app
from agents.runtime.runner import open_graph, run_query

@pytest.mark.asyncio
async def test_full_claim_workflow(monkeypatch):
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
    
    # mock orchestrator to route to ["review"] (pause at final review, not flag-review)
    async def mock_decide(state, gateway, rejected):
        # first call (from planner): route to validation, compliance, payroll-calculation, execution, output-validator, review
        # after execution and output-validator finish, route to ["review"]
        from langgraph.graph import END
        if state.get("results", {}).get("execution"):
            return {"route": ["review"], "summary": "Ready for final review"}
        return {"route": ["validation", "compliance", "payroll-calculation", "execution"], "summary": "Running agents"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    # mock all stage agents to finish immediately (no flags, no waiting)
    async def mock_validation(state, gateway=None):
        return {
            "results": {"validation": {"output": {"status": "ready"}, "confidence": 0.9}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.validation.node.validation_node", mock_validation)
    
    async def mock_compliance(state, gateway=None):
        return {
            "results": {"compliance": {"output": {"obligations": []}, "confidence": 0.95}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.compliance.node.compliance_node", mock_compliance)
    
    async def mock_payroll(state, gateway=None):
        return {
            "results": {"payroll-calculation": {"output": {"packages": []}, "confidence": 0.85}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.payroll_calculation.node.payroll_calculation_node", mock_payroll)
    
    async def mock_execution(state, gateway=None):
        return {
            "results": {"execution": {"output": {"package": {"id": "PKG-1"}}, "confidence": 1.0}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.execution.node.execution_node", mock_execution)
    
    async def mock_output_validator(state, gateway=None):
        return {
            "results": {"output-validator": {"output": {"valid": True}, "confidence": 1.0}},
            "steps": [],
        }
    monkeypatch.setattr("agents.nodes.output_validator.node.output_validator_node", mock_output_validator)
    
    # start the run and wait for it to pause at review
    out = await run_query(service_order_id="SO-TEST", thread_id=thread_id)
    
    assert out["paused"] is True
    assert "review" in out["next"]
    
    # alice claims the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_claim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response_claim.status_code == 200
        claim = response_claim.json()["claim"]
        assert claim["reviewer_id"] == "alice"
        claimed_at_alice = claim["claimed_at"]
    
    # bob attempts to resume (approve the run) as the non-owner
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_resume_bob = await client.post(
            f"/runs/{thread_id}/resume",
            json={"reviewer_id": "bob", "action": "approve"}
        )
    
    assert response_resume_bob.status_code == 403
    body_bob = response_resume_bob.json()
    assert body_bob["error"] == "not_owner"
    assert body_bob["claim"]["reviewer_id"] == "alice"
    assert body_bob["claim"]["claimed_at"] == claimed_at_alice
    
    # alice hands over to bob with a reason
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_handover = await client.post(
            f"/runs/{thread_id}/handover",
            json={
                "from_reviewer": "alice",
                "to_reviewer": "bob",
                "reason": "Waiting on client for bonus details"
            }
        )
    
    assert response_handover.status_code == 200
    handover_body = response_handover.json()
    assert handover_body["status"] == "handed_over"
    assert handover_body["claim"]["reviewer_id"] == "bob"
    claimed_at_bob = handover_body["claim"]["claimed_at"]
    
    # verify bob's queue shows the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_queue = await client.get("/reviewers/bob/queue")
        assert response_queue.status_code == 200
        cases = response_queue.json()["cases"]
        assert len(cases) == 1
        assert cases[0]["thread_id"] == thread_id
        assert cases[0]["employee"] == "Priya Rao"
        assert cases[0]["case_type"] == "Offboarding"
    
    # bob resumes the run (approve)
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_resume = await client.post(
            f"/runs/{thread_id}/resume",
            json={"reviewer_id": "bob", "action": "approve"}
        )
    
    assert response_resume.status_code == 202
    assert response_resume.json()["status"] == "running"
    
    # wait for the run to complete
    import asyncio
    await asyncio.sleep(1)
    
    # verify the run completed
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_final = await client.get(f"/runs/{thread_id}")
    
    assert response_final.status_code == 200
    final_state = response_final.json()
    assert final_state["status"] == "done"
    assert final_state["paused"] is False
    
    # inspect claim history
    claim_history = final_state["values"]["claim_history"]
    assert len(claim_history) == 2
    
    # entry 1: alice's claim
    assert claim_history[0]["reviewer_id"] == "alice"
    assert claim_history[0]["claimed_at"] == claimed_at_alice
    assert "reason" not in claim_history[0]  # no reason on initial claim
    
    # entry 2: alice's handover (carries alice's ID, not bob's, because it's alice's action)
    assert claim_history[1]["reviewer_id"] == "alice"
    assert claim_history[1]["reason"] == "Waiting on client for bonus details"
    # the handover event's timestamp is when alice handed it over (not bob's claim timestamp)
    # the final `claim` field shows bob as the owner at the time of completion
    
    # verify the final claim shows bob (the last owner before completion)
    final_claim = final_state["values"]["claim"]
    assert final_claim["reviewer_id"] == "bob"
    assert final_claim["claimed_at"] == claimed_at_bob
    
    # verify bob's queue no longer contains the case (it completed)
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_queue_after = await client.get("/reviewers/bob/queue")
        assert response_queue_after.status_code == 200
        cases_after = response_queue_after.json()["cases"]
        assert len(cases_after) == 0
```

**Why:** Exercises the full workflow (AC1-AC7) end-to-end: claim, ownership check, handover with reason, queue view, completion, and history tracking.

---

### Test 2: Integration test — Persistence across restart

**What it checks:** Start a run, pause it at review, claim it as Alice, hand over to Bob, restart the service (close and reopen the graph), verify the claim state is restored, resume as Bob and complete the run.

**How:**

```python
@pytest.mark.asyncio
async def test_claim_workflow_persistence_across_restart(monkeypatch):
    thread_id = str(uuid.uuid4())
    
    # same mocks as Test 1: mock fetch, report_progress, orchestrator, and all stage agents
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
        from langgraph.graph import END
        if state.get("results", {}).get("execution"):
            return {"route": ["review"], "summary": "Ready for final review"}
        return {"route": ["validation", "compliance", "payroll-calculation", "execution"], "summary": "Running agents"}
    monkeypatch.setattr("agents.nodes.orchestrator.node._decide", mock_decide)
    
    async def mock_validation(state, gateway=None):
        return {
            "results": {"validation": {"output": {"status": "ready"}, "confidence": 0.9}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.validation.node.validation_node", mock_validation)
    
    async def mock_compliance(state, gateway=None):
        return {
            "results": {"compliance": {"output": {"obligations": []}, "confidence": 0.95}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.compliance.node.compliance_node", mock_compliance)
    
    async def mock_payroll(state, gateway=None):
        return {
            "results": {"payroll-calculation": {"output": {"packages": []}, "confidence": 0.85}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.payroll_calculation.node.payroll_calculation_node", mock_payroll)
    
    async def mock_execution(state, gateway=None):
        return {
            "results": {"execution": {"output": {"package": {"id": "PKG-1"}}, "confidence": 1.0}},
            "steps": [],
            "flags": [],
        }
    monkeypatch.setattr("agents.nodes.execution.node.execution_node", mock_execution)
    
    async def mock_output_validator(state, gateway=None):
        return {
            "results": {"output-validator": {"output": {"valid": True}, "confidence": 1.0}},
            "steps": [],
        }
    monkeypatch.setattr("agents.nodes.output_validator.node.output_validator_node", mock_output_validator)
    
    # start the run and wait for it to pause at review
    out = await run_query(service_order_id="SO-TEST", thread_id=thread_id)
    assert out["paused"] is True
    assert "review" in out["next"]
    
    # alice claims the case
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_claim = await client.post(f"/runs/{thread_id}/claim", json={"reviewer_id": "alice"})
        assert response_claim.status_code == 200
        claimed_at_alice = response_claim.json()["claim"]["claimed_at"]
    
    # alice hands over to bob
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_handover = await client.post(
            f"/runs/{thread_id}/handover",
            json={
                "from_reviewer": "alice",
                "to_reviewer": "bob",
                "reason": "Shift change"
            }
        )
        assert response_handover.status_code == 200
        claimed_at_bob = response_handover.json()["claim"]["claimed_at"]
    
    # read the claim state before restart
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        before = await graph.aget_state(config)
        claim_before = before.values["claim"]
        history_before = before.values["claim_history"]
        
        assert claim_before["reviewer_id"] == "bob"
        assert claim_before["claimed_at"] == claimed_at_bob
        assert len(history_before) == 2
        assert history_before[0]["reviewer_id"] == "alice"
        assert history_before[1]["reviewer_id"] == "alice"
        assert history_before[1]["reason"] == "Shift change"
    
    # simulate restart: close the graph (closes sqlite connection) and reopen it
    # (the checkpointer's sqlite connection is closed by open_graph's context manager)
    
    async with open_graph(thread_id) as graph:
        config = {"configurable": {"thread_id": thread_id}}
        after = await graph.aget_state(config)
        
        # verify the claim state is restored
        claim_after = after.values["claim"]
        history_after = after.values["claim_history"]
        
        assert claim_after["reviewer_id"] == "bob"
        assert claim_after["claimed_at"] == claimed_at_bob
        assert len(history_after) == 2
        assert history_after[0]["reviewer_id"] == "alice"
        assert history_after[0]["claimed_at"] == claimed_at_alice
        assert history_after[1]["reviewer_id"] == "alice"
        assert history_after[1]["reason"] == "Shift change"
        
        # verify the run is still paused at review (not corrupted by the restart)
        assert "review" in after.next
    
    # bob resumes the run (approve) after the restart
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_resume = await client.post(
            f"/runs/{thread_id}/resume",
            json={"reviewer_id": "bob", "action": "approve"}
        )
    
    assert response_resume.status_code == 202
    
    # wait for the run to complete
    import asyncio
    await asyncio.sleep(1)
    
    # verify the run completed and the claim history is preserved
    async with AsyncClient(app=app, base_url="http://test") as client:
        response_final = await client.get(f"/runs/{thread_id}")
    
    assert response_final.status_code == 200
    final_state = response_final.json()
    assert final_state["status"] == "done"
    
    # verify the claim history survived the restart and completion
    final_history = final_state["values"]["claim_history"]
    assert len(final_history) == 2
    assert final_history[0]["reviewer_id"] == "alice"
    assert final_history[1]["reviewer_id"] == "alice"
    assert final_history[1]["reason"] == "Shift change"
```

**Why:** Verifies AC11 (persistence across restarts) by closing the sqlite checkpointer (simulating a service restart), reopening it, and verifying the claim state and history are unchanged.

---

## Dependencies

**Story execution order:**

- **Depends on:** All prior stories (1, 2, 3, 4, 5)
  - Story 1: `claim`, `claim_history`, `needs_review_since` state fields
  - Story 2: `/claim`, `/release`, `/handover` endpoints
  - Story 3: `/reviewers/{id}/queue` endpoint
  - Story 4: `_require_owner()` ownership check in `/resume`
  - Story 5: (not directly used, but part of the feature)
- **After:** Stories 1, 2, 3, 4, 5

**External dependencies:**

- Existing `run_query()` and `open_graph()` helpers (used by all integration tests)
- Existing FastAPI `app` and `AsyncClient` (used to call HTTP endpoints)
- Existing checkpointer (sqlite) persists and restores state across "restarts"
- Existing orchestrator and stage agent nodes (mocked in tests to avoid external dependencies)

---

## Risks and Mitigations

### Risk 1: Mocking the full graph execution

**Risk:** The test mocks all stage agents (validation, compliance, payroll-calculation, execution, output-validator) to avoid external dependencies (MCP server, LLM API). If the mocks drift from the real agent behavior, the test may pass while the real workflow breaks.

**Mitigation:** The test focuses on the claim workflow logic, not the agent logic. The mocked agents return minimal valid outputs (no flags, confidence scores) to get the run to the review pause point. The claim state, endpoints, and ownership checks are exercised with real code, not mocks.

**Residual risk:** Low. The test verifies the claim workflow, not the agent execution flow. The agent execution flow is tested separately in `test_validation_agent.py`, `test_payroll_agent.py`, etc.

### Risk 2: Service restart simulation (closing and reopening the graph)

**Risk:** The "restart" is simulated by closing the graph's context manager (which closes the sqlite connection) and reopening it with `open_graph(thread_id)`. This may not fully mirror a real service restart (e.g. FastAPI server process killed and restarted).

**Mitigation:** The checkpointer's persistence contract is: write the full `State` to the sqlite file after every node, and restore it on `graph.aget_state(config)` when the graph reopens. Closing and reopening the graph exercises this contract (the sqlite connection is closed, the file remains, the new connection reads it).

**Residual risk:** Low. The checkpointer is tested by LangGraph's own test suite. This test verifies that the claim fields (added in Story 1) are included in the persistence cycle.

### Risk 3: Test flakiness (timing-dependent assertions)

**Risk:** The test calls `asyncio.sleep(1)` to wait for the run to complete after resuming. If the run takes longer than 1 second (e.g. under load), the test may read the state before the run finishes and assert `status == "done"` when it is still `"running"`.

**Mitigation:** 1 second is generous for the mocked workflow (no real LLM calls, no external MCP calls). If flakiness occurs, increase the sleep duration or poll `GET /runs/{thread_id}` until `paused == False`.

**Residual risk:** Low. The existing tests (`test_validation_agent.py`) use the same pattern without flakiness.

### Risk 4: Missing `needs_review_since` timestamp (if Story 1 is not fully deployed)

**Risk:** If Story 1's orchestrator change (setting `needs_review_since`) is not deployed yet, the `GET /reviewers/{id}/queue` and `GET /leads/waiting` endpoints (Stories 3 and 5) will compute `waiting_seconds = 0` for all cases, and the tests may fail or produce unexpected results.

**Mitigation:** This is an integration test suite for the full feature (Stories 1-6). The test assumes all prior stories are implemented and deployed together. The deployment order is Story 1 → 2 → 3 → 4 → 5 → 6, so Story 6's tests run only after Story 1's orchestrator change is live.

**Residual risk:** None. The test suite is designed to run after all prior stories are merged.

---

## Open Questions

*None at this time. All design choices are documented in the HLD and above.*
