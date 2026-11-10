# Plan — PER-202 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Case claim and handover workflow with reviewer queues and lead monitoring; introduces new state fields, persistence, and API endpoints in one repo, so architecture is needed despite being one story's scope.

## Steps

1. [ ] **analyst (requirements)** — Extract precise requirements from the ticket's acceptance criteria and examples.
   - focus: Focus on the state transitions (unclaimed → claimed → handover → release), the persistence requirement, and the 4-hour configurable threshold.
2. [ ] **architect (hld)** — New state fields, persistence layer, and API endpoints for claiming/handover require high-level design.
   - focus: Design the claim/handover state model, persistence approach for restarts, API contracts for claim/release/handover/query operations, and the configurable timeout mechanism.
3. [ ] **scrum_master (breakdown)** — The work splits into persistence layer, state management, API endpoints, and lead monitoring features.
   - focus: Break down by: persistence setup, claim state transitions, handover logic, reviewer queue view, lead monitoring view, and config for timeout threshold.
4. [ ] **designer (lld)** — Each story needs concrete file-by-file implementation plan for state, persistence, routes and business logic.
   - focus: Identify which modules handle claim state, where persistence hooks go, and how queries filter by reviewer or waiting time.
5. [ ] **test_architect (tests)** — Acceptance criteria include edge cases: claiming finished cases, concurrent claims, handover validation, and restart persistence.
   - focus: Map each AC to test cases, especially AC 2 (authorization), AC 5 (history), AC 10 (state validation), AC 11 (restart persistence), and AC 12 (backward compatibility).
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Requirements, HLD, breakdown, and stories must agree on state model, persistence approach, API contracts, and the 4-hour threshold.
   - focus: Verify claim state fields match across all documents, persistence design is consistent, and API contracts align with AC constraints.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly stated with concrete examples; no product ambiguity or cross-team coordination.
