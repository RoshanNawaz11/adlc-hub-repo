## What would go wrong

### C1
- file: stories/02-agent.md
- section: ## Tests
- quote: Test 9: Integration test — Claim, release, claim again by another reviewer
- match: breakdown.md › Story 2 › Verification Approach
- problem: The breakdown lists 10 verification tests for Story 2, but the LLD only implements 8 tests (Tests 1-8). Test 9 starts but is cut off mid-implementation with `async def test_claim_release_claim_history(monkeypatch):` followed by incomplete code ending with `response_resume = await client.post(`. Test 10 is completely missing. A developer following this LLD would deliver 8 tests when the breakdown promised 10.
- change: Complete Test 9 with the full implementation (claim → release → claim by another reviewer, verify history has all three events), and add Test 10 (claim → handover with reason, verify history shows handover event with reason).

### C2
- file: stories/03-agent.md
- section: ## Interfaces
- quote: async for thread_state in checkpointer.alist(None):
- match: stories/05-agent.md › ## Interfaces › async for thread_state in checkpointer.alist(None):
- problem: Both Story 3 and Story 5 use the same checkpointer scanning pattern (`async for thread_state in checkpointer.alist(None)`), but neither verifies that this API actually exists in the repo's LangGraph version. The HLD notes "verified in LangGraph 0.2.x docs" but the repo context does not show the LangGraph version or confirm `alist(None)` works. If the checkpointer requires a config argument (not `None`), both stories would build the wrong query and the endpoints would return empty lists.
- change: Add a note in both Story 3 and Story 5 LLD Algorithm Notes sections: "Verify that `checkpointer.alist(None)` works in the repo's LangGraph version by checking `repo-file/agent/pyproject.toml` for the `langgraph` dependency version. If `alist(None)` is not supported, use the sqlite fallback documented in Risks section." This tells Dev to check before building, not after.

## Notes for the gate

The two stale items (C3 and C4) refer to text that no longer exists in the spec files, so they are resolved. The claim history format in Story 2 and the test assertions in Story 6 are consistent as written.

Story 4's LLD adds `reviewer_id` to the request models and the HLD's section 5 (Authorization Check) already shows the middleware checks `state["claim"]["reviewer_id"]` against `reviewer_id` from the request. The schema change is isolated to Story 4 and does not conflict with the HLD's ownership check design.

VERDICT: FAIL
