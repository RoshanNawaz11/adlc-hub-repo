# Plan — PER-155 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Retry mechanism for failed runs from last checkpoint: moderate complexity with queue job, API endpoint, and runner changes, but all in one repo with well-defined contracts.

## Steps

1. [ ] **analyst (inline_story)** — Ticket has clear acceptance criteria and scope; needs formalization into Given/When/Then format.
   - focus: Focus on the state transitions (failed → running → done/failed/paused), the retry limit enforcement, and the contract with the order-run key.
2. [ ] **designer (lld)** — Multi-component change spanning runner, API, and queue with shared state contracts requires concrete file plan.
   - focus: Map out the retry flow across run_query, the new API endpoint, and retry_service_order job. Show how retry count is persisted and checked, and how the checkpointer continuation works.
3. [ ] **test_architect (tests)** — Acceptance criteria describe specific edge cases: mid-node retry, wrong states, limit enforcement, status writes.
   - focus: Ensure 1:1 mapping between the six ACs and test cases, covering the retry flow, state guards, limit enforcement, and status key contract.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no breakdown or separate HLD; nothing to cross-check for consistency.
