## What would go wrong

### C9
- file: stories/04-agent.md
- section: ## File Plan
- quote: `async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]:`
- match: repo-file/agent/src/agents/queue/orders.py › resume_service_order
- severity: blocking
- blocks because: Dev would add withdrawal checking code to `resume_service_order` when Story 4's purpose is to add that check, but the breakdown says Story 7 handles "prevent operations on withdrawn cases" and its LLD modifies `resume_service_order` again. Two stories would modify the same function for overlapping purposes, so Dev would either apply both changes (duplicating the check) or skip one, leaving gaps.
- problem: Story 4's LLD adds a withdrawn status check to `resume_service_order` (lines 93-98 in the existing code become a much larger block with checkpointer reads and status checking). Story 7's breakdown says it "modifies `resume_service_order`" but Story 7's LLD says "**No changes to `resume_service_order`:** Story 4 already added the withdrawn check" in the "Modified Files" section. The division is unclear: if Story 4 owns the resume withdrawn check, Story 7 should not list `resume_service_order` in its breakdown; if Story 7 owns preventing all operations, Story 4 should only handle the queue status write when already withdrawn, not the operation prevention.
- change: Clarify in Story 4's LLD whether it adds the full withdrawn check to `resume_service_order` (as the code shows) or only the queue status write (deferring the check to Story 7). Update Story 7's breakdown and LLD to match: either remove `resume_service_order` from Story 7's file list (if Story 4 owns it) or move the check from Story 4 to Story 7 (if Story 7 owns operation prevention).

---

### C10
- file: breakdown.md
- section: ## Summary
- quote: **Total test count:** 31 tests (10 + 7 + 5 + 2 + 2 + 1 + 4 + 5 = 36 tests across all stories)
- match: breakdown.md › ## Summary
- severity: blocking
- blocks because: The breakdown says 31 tests in the summary sentence but the addition gives 36, so Dev would write 31 tests (trusting the summary) and miss 5, or write 36 and wonder which 5 to drop. Either way the test count would not match what the stories deliver.
- problem: The summary line says "31 tests" but the addition "(10 + 7 + 5 + 2 + 2 + 1 + 4 + 5 = 36 tests)" gives 36. One is wrong.
- change: Recalculate the test count from the stories' test sections and update the summary to match. If the stories deliver 36 tests, say "36 tests (10 + 7 + 5 + 2 + 2 + 1 + 4 + 5)". If they deliver 31, correct the addition.

---

### C11
- file: stories/06-agent.md
- section: ## Tests Section
- quote: `async def test_withdrawn_case_persists_across_restart() -> None:`
- match: stories/06-agent.md › ## File Plan
- severity: blocking
- blocks because: The story's purpose is "Persistence across restarts" and it creates a new test file `tests/test_withdrawal_persistence.py`, but the test implementation in the "Tests Section" shows the full test body while the "File Plan" section says the same test file contains the same test. Dev would write the test file once (in the File Plan section) and then see the test body again in the Tests Section and either duplicate it or skip the Tests Section, not knowing which is the source of truth.
- problem: The LLD repeats the full test implementation twice: once in the "File Plan" section (under `tests/test_withdrawal_persistence.py`) and again in the "Tests Section". The text is identical. A developer following the LLD would see the test body in the File Plan, write it, then reach the Tests Section and see the same test quoted in full with `# (full implementation above in File Plan)` in the docstring but the body still present below it.
- change: In the "Tests Section", replace the full test implementation with a reference: `(See File Plan for full implementation)` and remove the test body. The File Plan already shows the complete test; the Tests Section should only describe coverage and assertions, not repeat the code.

---

### C12
- file: stories/08-agent.md
- section: ## File Plan
- quote: `async def test_example_1_duplicate_case_withdrawn_flags_not_answered() -> None:`
- match: stories/08-agent.md › ## Tests Section
- severity: blocking
- blocks because: Same issue as C11: the LLD shows each test's full implementation in the File Plan section, then repeats the full implementation in the Tests Section with a comment `# (full implementation above in File Plan)`. Dev would write the tests once in the File Plan, then see the same code in the Tests Section and either duplicate them or skip the section, not knowing the Tests Section is only documentation.
- problem: The File Plan section of Story 8 contains the full implementations of all 5 tests inside the test file `tests/test_withdrawal_e2e.py`. The Tests Section then repeats the same 5 tests in full (with `# (full implementation above in File Plan)` in each docstring but the entire test body still shown). This is the same pattern as C11.
- change: In the "Tests Section", replace each full test implementation with a reference: `(See File Plan for full implementation)` and remove the test bodies. Only the coverage and assertions descriptions belong in the Tests Section; the code is already in the File Plan.

---

## Notes for the gate

### N1
- file: stories/02-agent.md
- section: ## File Plan
- note: The story adds a custom exception handler to `server.py` to transform FastAPI's default `{"detail": "<message>"}` error responses into the HLD's specified `{"error": "<message>"}` format. This is a global change affecting all endpoints. The fixer decided this approach (per `critic-items.md` C4) instead of wrapping each `HTTPException` manually. The handler is added after `_lifespan` (around line ~75) and before route definitions. This works, but it means all endpoints in `server.py` (including ones added by other features) will return `{"error": ...}` instead of FastAPI's default `{"detail": ...}`. If the rest of the codebase expects FastAPI's default shape, this could break other callers. The person at the gate should check whether the console or other services rely on `{"detail": ...}` in error responses from this server.

---

### N2
- file: breakdown.md
- section: ## Summary
- note: The "Dependencies visualized" diagram shows Story 8 depending on "all prior stories" (1-7). The prose says "Story 8 (end-to-end validation) depends on all prior stories". However, Story 8's "After:" field in the breakdown header says "Story 1, Story 2, Story 3, Story 4, Story 5, Story 6, Story 7", which is correct, but the diagram's arrow from "Story 7" to "Story 8" is the only one drawn. The other dependencies (1, 2, 3, 4, 5, 6 → 8) are implied by the prose but not shown in the diagram. This is not wrong (the prose is clear), but the diagram could be more explicit by drawing all 7 arrows or using a note "Story 8 depends on 1-7" at the Story 8 node.

---

### N3
- file: stories/05-agent.md
- section: ## Open Questions (carried forward from Story 1 / HLD)
- note: Open question 2 asks "Event dedupe_key collision: if multiple withdrawals of the same case are allowed (e.g. a case is resumed and then withdrawn again), this key will not deduplicate them correctly." The requirements AC 6 says "A withdrawn case can't be resumed… or be withdrawn a second time", so multiple withdrawals of the same case are not allowed by design. The question is moot under the current requirements. If the requirement changes in the future (e.g. allowing a case to be un-withdrawn and withdrawn again), the dedupe_key would need a unique component (timestamp or counter). For now, this is not a problem, just a note for future design.

---

### N4
- file: stories/07-agent.md
- section: ## Open Questions (carried forward from prior stories)
- note: Open question 1 asks "Should `create_run` also load the snapshot and check for withdrawn status explicitly (for a clearer 409 response instead of a 500 from the `RuntimeError`)?" The current implementation delegates to `run_query`, which raises `RuntimeError` when `resume=False` and the case is withdrawn. This surfaces as a 500 error to the API caller (FastAPI does not catch `RuntimeError` by default). A 409 Conflict is more semantically correct for "case is withdrawn, cannot start it again". The person at the gate should decide whether `create_run` should add an explicit withdrawn check (before calling `_start`) to return 409, or accept the current 500 response as good enough for this edge case.

---

VERDICT: FAIL
