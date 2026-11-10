# Plan — PER-155 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Retry mechanism for failed runs from last checkpoint; single repo with clear API and queue contracts; needs concrete file plan before coding.

## Steps

1. [ ] **analyst (inline_story)** — Convert the ticket into a single implementable story with Given/When/Then acceptance criteria.
   - focus: Keep all six acceptance criteria; emphasize the retry limit persistence requirement and the existing checkpointer integration.
2. [ ] **designer (lld)** — Multi-component change (runner, API, queue) with state management and Redis contract needs concrete file plan.
   - focus: Detail the retry counter storage (Redis key structure), the runner's checkpoint recovery logic, and how the three components coordinate; map each AC to specific functions and data structures.
3. [ ] **test_architect (tests)** — Six acceptance criteria with edge cases (retry limits, status validation, checkpoint skip logic) worth explicit test mapping.
   - focus: Map test cases 1:1 to the six ACs; include edge cases for the retry limit, failed/paused/done status checks, and checkpoint continuation.
4. [ ] **critic (consistency_check)** — Requirements, LLD and tests must agree on retry semantics, status values and the Redis contract.
5. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Loop delivery.
7. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
8. [ ] **open_pr** _(auto)_ — Loop delivery.
9. [ ] **pr_review** _(auto)_ — Loop delivery.
10. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly stated in the ticket; no product ambiguity or cross-team coordination needed.
