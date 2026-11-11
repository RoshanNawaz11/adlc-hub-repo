# Plan — PER-234 (v4)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Add reviewer-settable priority to cases with validation, persistence, history tracking and activity feed integration; this is a feature with API and schema changes that requires architecture design.

## Steps

1. [ ] **analyst (requirements)** — Extract clear requirements from the ticket's acceptance criteria and examples.
   - focus: Focus on the state transitions, validation rules, and the distinction between reviewer-set vs system-set priority.
2. [ ] **architect (hld)** — Design the priority override mechanism, schema changes, API contract, and integration with history and activity feed.
   - focus: Address the persistence model for reviewer-set priority, the flag that distinguishes it from system priority, and the API endpoint contract.
3. [ ] **scrum_master (breakdown)** — Break down into stories covering API endpoint, schema migration, history tracking, and activity feed integration.
   - focus: Separate the data model changes from the API implementation and integration work.
4. [ ] **designer (lld)** — Provide file-by-file implementation plans for each story including validation logic and state handling.
   - focus: Detail the validation rules, the priority override storage, and the history entry structure.
5. [ ] **test_architect (tests)** — Map test cases to each acceptance criterion, especially edge cases around validation and state transitions.
   - focus: Cover all validation failures, state transition scenarios, and persistence across restarts.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Ensure consistency between requirements, HLD, breakdown and stories on priority logic and API contract.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly stated in the ticket with concrete examples; no ambiguity requiring product sign-off.

## Planner notes

- attempt 1: loop feature_with_architecture is ruled out: you answered that its not-when [N1] applies (one story's worth of work inside one component, and every interface stays as it is → quick_fix or story_with_design). Pick the loop that line points to, or the next loop down whose not-when lines do not apply
