# Breakdown — PER-202

**Epic:** Reviewers can claim, hand over and track their cases  
**Source:** Requirements, HLD, and proposed tickets

---

## Overview

This epic introduces case claim and handover workflow with reviewer queues and lead monitoring. The work extends the existing LangGraph state machine to track which reviewer owns a paused case, preserves claim history across restarts, and exposes REST endpoints for claim operations and queue views. The implementation adds new state fields to the existing `State` TypedDict, leverages the current checkpointer for persistence, and builds API routes into the existing FastAPI server.

---

## Stories (Execution Order)

### Story 1: Add claim state fields and persistence

**Repo:** `agent`

**Description:**  
Add `Claim` TypedDict, `claim`, `claim_history`, and `needs_review_since` fields to the `State` schema in `src/agents/graph/states.py`. Set `needs_review_since` when the orchestrator or review flow transitions a case to `status: "needs_review"`. Verify that the existing checkpointer (Redis or SQLite) persists and restores these fields across service restarts without any new persistence code.

**Covers:**  
- REQ-AC11 (persistence across restarts)
- REQ-AC12 (backward compatibility: fields are `NotRequired`)

**After:** (none — first story)

**Verification Approach:**  
1. Unit test: `State` schema includes `claim`, `claim_history`, `needs_review_since` as `NotRequired` fields with correct types.
2. Unit test: `needs_review_since` is set when a case enters `status: "needs_review"` (mock the orchestrator or review node transition).
3. Integration test: start a run, pause it at review, restart the service, resume the run, and verify all claim fields are preserved in the restored state (read from checkpointer).
4. Integration test: run a case end-to-end without claiming it; verify no errors and `claim` is `None` or absent (backward compatibility).

**Verification Count:** 4 tests

**Dependencies:** None

**Risks/Unknowns:**  
- The exact node that sets `status: "needs_review"` must be identified (likely orchestrator or review node in `src/agents/graph/edges.py` or `src/agents/nodes/orchestrator/node.py`).
- Verify that the checkpointer serializes `TypedDict` fields correctly (currently used for `Flag`, `Need`, etc., so this should work).

**Allowed Paths:**  
- `src/agents/graph/states.py` (add `Claim` TypedDict, fields to `State`)
- `src/agents/graph/edges.py` or `src/agents/nodes/orchestrator/node.py` (set `needs_review_since`)
- `tests/test_claim_state.py` (new file: unit and integration tests)

---

### Story 2: Implement claim, release, and handover endpoints

**Repo:** `agent`

**Description:**  
Add three POST endpoints to `src/agents/runtime/server.py`: `/runs/{thread_id}/claim`, `/runs/{thread_id}/release`, and `/runs/{thread_id}/handover`. Each loads the run's snapshot via `_snapshot(thread_id)`, validates the request (case status, ownership, required fields), updates the `claim` and `claim_history` fields in the state, persists the checkpoint, and returns the appropriate JSON response. Include error handling for all validation failures (400, 403, 409).

**Covers:**  
- REQ-AC1 (claiming a case)
- REQ-AC2 (claim exclusivity — enforced in Story 4)
- REQ-AC3 (releasing a claim)
- REQ-AC4 (handing over with reason)
- REQ-AC5 (claim history tracking)
- REQ-AC10 (claim state validation: case must be `needs_review`)

**After:** Story 1 (depends on `claim`, `claim_history` fields)

**Verification Approach:**  
1. Unit test: POST `/runs/{thread_id}/claim` with valid `reviewer_id` on a case in `needs_review` status returns 200 with claim details and updates state.
2. Unit test: POST `/runs/{thread_id}/claim` on a case not in `needs_review` status returns 400 with error.
3. Unit test: POST `/runs/{thread_id}/claim` on an already claimed case returns 409 with current claim.
4. Unit test: POST `/runs/{thread_id}/release` by the owning reviewer returns 200 and sets `claim` to `None`.
5. Unit test: POST `/runs/{thread_id}/release` by a non-owner returns 403.
6. Unit test: POST `/runs/{thread_id}/handover` with valid `from_reviewer`, `to_reviewer`, and `reason` returns 200 and updates claim.
7. Unit test: POST `/runs/{thread_id}/handover` without `reason` returns 400.
8. Unit test: POST `/runs/{thread_id}/handover` by non-owner returns 403.
9. Integration test: claim → release → claim again by another reviewer; verify history has all three events.
10. Integration test: claim → handover with reason; verify history shows handover event with reason.

**Verification Count:** 10 tests

**Dependencies:**  
- Story 1 (state fields)

**Risks/Unknowns:**  
- Checkpoint persistence after state update: verify that calling `graph.update_state(config, {"claim": ...})` (or equivalent) persists the change without running the graph.
- Concurrent claims: two reviewers claim simultaneously. The checkpointer's optimistic locking should handle this (HLD notes this), but verify one gets 409.

**Allowed Paths:**  
- `src/agents/runtime/server.py` (add `/claim`, `/release`, `/handover` endpoints)
- `tests/test_claim_endpoints.py` (new file: unit and integration tests)

---

### Story 3: Implement reviewer queue endpoint

**Repo:** `agent`

**Description:**  
Add GET `/reviewers/{reviewer_id}/queue` endpoint to `src/agents/runtime/server.py`. Query all threads with `status: "needs_review"` and `claim.reviewer_id == <id>` from the checkpointer, compute `waiting_seconds = now - claim.claimed_at` for each, sort by `waiting_seconds` descending (oldest first), and return a list with `thread_id`, `employee`, `case_type`, and `waiting_seconds`. Extract `employee` from `state["service_order"]["employee"]` and `case_type` from `state["service_order"]["caseType"]`.

**Covers:**  
- REQ-AC6 (reviewer's queue view)
- REQ-AC7 (completed cases leave the queue — cases not in `needs_review` are excluded by the query)

**After:** Story 2 (depends on claim endpoints to populate claims)

**Verification Approach:**  
1. Unit test: GET `/reviewers/{id}/queue` with no claimed cases returns empty list.
2. Unit test: GET `/reviewers/{id}/queue` with two claimed cases returns both, sorted oldest first by `waiting_seconds`.
3. Unit test: GET `/reviewers/{id}/queue` excludes cases claimed by other reviewers.
4. Integration test: claim a case, complete it (set `status: "done"`), call the queue endpoint; verify the completed case is not in the queue.

**Verification Count:** 4 tests

**Dependencies:**  
- Story 2 (claim endpoints)

**Risks/Unknowns:**  
- Querying all threads: the checkpointer may not have a direct "query by field" API. May need to scan all threads in the namespace and filter in code. Acceptable for MVP (HLD notes this).
- Performance with many threads: defer optimization (HLD: sorted sets) to future.

**Allowed Paths:**  
- `src/agents/runtime/server.py` (add `/reviewers/{reviewer_id}/queue` endpoint)
- `tests/test_reviewer_queue.py` (new file: unit and integration tests)

---

### Story 4: Add ownership check middleware and modify resume endpoint

**Repo:** `agent`

**Description:**  
Add `_require_owner(state: dict, reviewer_id: str) -> tuple[bool, dict | None]` function to `src/agents/runtime/server.py`. Modify POST `/runs/{thread_id}/resume` to accept `reviewer_id` in the request body (for flag answers and review decisions), call `_require_owner`, and return 403 if unauthorized. This enforces AC2: only the owning reviewer can answer flags, approve, or reject the case.

**Covers:**  
- REQ-AC2 (claim exclusivity: only owner can act)

**After:** Story 2 (depends on claim endpoints and state fields)

**Verification Approach:**  
1. Unit test: `_require_owner` returns `(True, None)` when the reviewer owns the claim.
2. Unit test: `_require_owner` returns `(False, error)` when the reviewer does not own the claim.
3. Unit test: `_require_owner` returns `(True, None)` when there is no claim (unclaimed case).
4. Integration test: claim a case as Reviewer A, attempt to resume (answer flags) as Reviewer B; verify 403 response.
5. Integration test: claim a case as Reviewer A, resume (answer flags) as Reviewer A; verify 202 response and resumption proceeds.

**Verification Count:** 5 tests

**Dependencies:**  
- Story 2 (claim state and endpoints)

**Risks/Unknowns:**  
- The existing `resume_run` endpoint accepts `FlagAnswers | ReviewDecision`. Need to extend both models to include `reviewer_id` (or extract it from headers/auth if available).
- How `reviewer_id` is authenticated: assume it is passed in the request for MVP; defer real auth to future.

**Allowed Paths:**  
- `src/agents/runtime/server.py` (add `_require_owner`, modify `resume_run`)
- `tests/test_ownership_check.py` (new file: unit and integration tests)

---

### Story 5: Implement lead waiting list endpoint with configurable threshold

**Repo:** `agent`

**Description:**  
Add GET `/leads/waiting` endpoint to `src/agents/runtime/server.py` with optional `?threshold_hours=4` query parameter. Read the threshold from the query param or from `CLAIM_WAITING_THRESHOLD_HOURS` env var (default 4). Query all threads with `status: "needs_review"` and `claim == None`, compute `waiting_seconds = now - needs_review_since`, filter where `waiting_seconds > threshold * 3600`, sort by `waiting_seconds` descending (longest first), and return a list with `thread_id`, `employee`, `case_type`, and `waiting_seconds`. Add `_waiting_threshold_seconds()` helper function to read and parse the threshold.

**Covers:**  
- REQ-AC8 (lead view of unclaimed waiting cases)
- REQ-AC9 (configurable waiting threshold)

**After:** Story 1 (depends on `needs_review_since` field)

**Verification Approach:**  
1. Unit test: `_waiting_threshold_seconds()` returns 14400 (4 hours) by default.
2. Unit test: `_waiting_threshold_seconds()` returns the value from `CLAIM_WAITING_THRESHOLD_HOURS` env var.
3. Unit test: GET `/leads/waiting` with `?threshold_hours=6` returns cases waiting > 6 hours only.
4. Unit test: GET `/leads/waiting` excludes claimed cases.
5. Integration test: pause two cases at different times, call `/leads/waiting?threshold_hours=4`; verify only the one waiting > 4 hours is returned, and it is first (longest waiting).

**Verification Count:** 5 tests

**Dependencies:**  
- Story 1 (`needs_review_since` field)

**Risks/Unknowns:**  
- Same query performance concern as Story 3 (acceptable for MVP).
- Changing the env var requires service restart (HLD notes this is acceptable for MVP).

**Allowed Paths:**  
- `src/agents/runtime/server.py` (add `/leads/waiting` endpoint, `_waiting_threshold_seconds` helper)
- `tests/test_lead_waiting.py` (new file: unit and integration tests)

---

### Story 6: End-to-end claim workflow integration test

**Repo:** `agent`

**Description:**  
Write an end-to-end integration test that exercises the full claim workflow: start a run and pause it for review, claim the case as Reviewer A, attempt to answer flags as Reviewer B (verify 403), hand over the case from A to B with a reason, answer flags and approve as B, verify the run completes, and inspect the claim history to confirm all events (claim by A, handover to B with reason, completion). Also verify that the case leaves B's queue when completed.

**Covers:**  
- REQ-AC1 through REQ-AC7 (all claim, handover, queue, and exclusivity requirements together)
- REQ-AC11 (persistence: restart the service mid-workflow and verify state is preserved)

**After:** Stories 1, 2, 3, 4 (all prior work)

**Verification Approach:**  
1. Integration test: full workflow as described above (claim, handover, queue, ownership check, completion).
2. Integration test: same workflow, but restart the service after handover and before resumption; verify claim state is restored and resumption proceeds.

**Verification Count:** 2 tests

**Dependencies:**  
- All prior stories (1, 2, 3, 4, 5)

**Risks/Unknowns:**  
- Service restart test: requires stopping and restarting the FastAPI server (or graph runner) in the test. May mock this by closing and reopening the graph with the same `thread_id`.

**Allowed Paths:**  
- `tests/test_claim_workflow_e2e.py` (new file: end-to-end integration tests)

---

## Dependencies

**Story execution order:**

```
1 (state fields) → 2 (claim/release/handover endpoints) → 4 (ownership check)
                ↘ 3 (reviewer queue)
                ↘ 5 (lead waiting list)

4, 3, 5 → 6 (end-to-end workflow)
```

**External dependencies:**  
- Existing checkpointer (Redis or SQLite) persists `State` fields (verified in Story 1).
- Existing FastAPI server and `_snapshot` helper (used by all endpoint stories).
- Existing LangGraph state machine and orchestrator (sets `status: "needs_review"` and `needs_review_since`).

---

## Allowed Paths

All stories modify or create files under the `agent` repo:

**Modified:**
- `src/agents/graph/states.py` (Story 1: add `Claim`, fields)
- `src/agents/graph/edges.py` or `src/agents/nodes/orchestrator/node.py` (Story 1: set `needs_review_since`)
- `src/agents/runtime/server.py` (Stories 2, 3, 4, 5: endpoints, ownership check, threshold helper)

**New:**
- `tests/test_claim_state.py` (Story 1)
- `tests/test_claim_endpoints.py` (Story 2)
- `tests/test_reviewer_queue.py` (Story 3)
- `tests/test_ownership_check.py` (Story 4)
- `tests/test_lead_waiting.py` (Story 5)
- `tests/test_claim_workflow_e2e.py` (Story 6)

---

## Risks and Mitigations

### 1. Concurrent claims (two reviewers claim the same case simultaneously)

**Mitigation:** The checkpointer uses optimistic locking (checkpoint version increment). The second claim write fails and retries, sees the case is already claimed, returns 409. No data corruption.  
**Residual risk:** One reviewer sees a stale "available" state briefly. Acceptable for MVP; UI should poll or use server-sent events.

### 2. Unbounded claim history

**Risk:** A case handed over 100 times balloons the checkpoint size.  
**Mitigation:** Accept for MVP (rare). Future: archive `claim_history` to a separate store after case completion.

### 3. Query performance (scanning all threads for queue and waiting list)

**Risk:** Slow with 10,000 paused cases.  
**Mitigation:** Acceptable for MVP (dozens to hundreds of cases). Future: maintain Redis sorted sets indexed by reviewer and wait time.

### 4. Threshold change lag

**Risk:** Changing `CLAIM_WAITING_THRESHOLD_HOURS` env var requires service restart.  
**Mitigation:** For MVP, restart is acceptable (no data loss, brief downtime). Future: read from Redis key polled every request.

### 5. Setting `needs_review_since`

**Risk:** Uncertainty about which node sets `status: "needs_review"`.  
**Mitigation:** Story 1 investigates the orchestrator and review nodes to identify the transition point and adds the timestamp setter there.

---

## Open Questions

*None at this time. All technical decisions are documented in the HLD and stories above.*
