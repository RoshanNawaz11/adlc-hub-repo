## What would go wrong

### C1
- file: stories/02-agent.md
- section: ## Tests
- quote: async def test_claim_release_claim_history(monkeypatch):
- match: breakdown.md › Story 2 › Verification Approach
- problem: The breakdown promises 10 tests for Story 2, but the LLD cuts off mid-sentence in Test 9 and omits Test 10 entirely. A developer following this LLD would deliver 8 complete tests when the breakdown specified 10.
- change: Complete Test 9 (claim → release → claim by another reviewer, verify history has all three events) and add Test 10 (claim → handover with reason, verify history shows handover event with reason) as full implementations matching the breakdown's verification approach.

VERDICT: FAIL
