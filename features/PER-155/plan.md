# Plan — PER-155 (v2)

- made from: review feedback (plan_review)
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Retry mechanism for failed runs requires new API endpoint, queue job, and shared retry-count contract across runner, API, and queue components; feature_with_architecture with full breakdown and critic.

## What changed

Switched from story_with_design to feature_with_architecture per reviewer feedback; added architect for contract design, scrum_master for breakdown, and critic for cross-artifact consistency.

- Added: architect, scrum_master, critic

## Feedback that informed this version

- **PR comment** by RoshanNawaz11 on hub#17 `[hub#17:issue-6758978900]`: Use feature_with_architecture: this adds a new API endpoint, a new queue job and a retry count that must survive restarts, shared by the runner, API and queue. Split it into stories and keep the critic."
  - addressed by: Changed loop to feature_with_architecture, added architect and scrum_master steps, kept critic.

## Steps

1. [ ] **analyst (requirements)** _(rerun)_ — Formalize acceptance criteria and state the shared retry-count contract that must survive restarts.
   - focus: Emphasize the retry-count persistence requirement and the status-write contract with fastapi-backend.
2. [ ] **architect (hld)** — New API endpoint, queue job, and shared state (retry count) across runner, API, and queue require HLD to define contracts and persistence strategy.
   - focus: Define where retry count is stored, how it survives restarts, and the contract shape for order-run status writes.
3. [ ] **scrum_master (breakdown)** — Multi-component change (runner, API, queue) needs breakdown into separate implementable stories per component.
   - focus: Split into stories for runner retry logic, API endpoint, queue job, retry-count persistence, and tests.
4. [ ] **designer (lld)** _(rerun)_ — Each story needs concrete file plan for runner changes, new API route, and queue job implementation.
   - focus: Provide file-level implementation detail for checkpoint continuation, API route, queue job, and shared state access.
5. [ ] **test_architect (tests)** _(rerun)_ — Acceptance criteria describe specific edge cases: mid-node retry, wrong states, limit enforcement, status writes.
   - focus: Map each AC to test cases covering retry from middle node, state validation, limit enforcement, and status-write contract.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple artifacts (requirements, HLD, breakdown, stories) must agree on retry-count persistence and status-write contract.
   - focus: Verify consistency of retry-count storage, status contract, and limit enforcement across all specs.
8. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly stated in the ticket; no product ambiguity.
