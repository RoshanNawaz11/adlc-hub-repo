# Plan — PER-222 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Adds a withdraw operation for paused cases with validation, persistence, and integration across CLI, API, queue and console; one repo with behaviour changes in several components (API endpoint, CLI command, queue handler, domain model, persistence) requires architecture and design.

## Steps

1. [ ] **analyst (requirements)** — Clear acceptance criteria are provided; requirements extraction confirms scope and edge cases.
   - focus: Focus on the validation rules (empty reason check), state transitions (what states allow withdrawal, what state it moves to), persistence requirements, and integration points (CLI, API, queue handler, console notification).
2. [ ] **architect (hld)** — Adds a new operation affecting domain model, API contract, CLI interface, queue handling, and external integration; needs component interaction design.
   - focus: Design the withdrawal operation's place in the domain model, the API endpoint contract, CLI command interface, queue message handling when a case is withdrawn, persistence schema changes if needed, and console notification contract.
3. [ ] **scrum_master (breakdown)** — Work spans multiple areas: domain model changes, API endpoint, CLI command, queue handler updates, persistence layer, and console integration; natural breakdown into stories.
   - focus: Split by technical area: domain model and business logic, API endpoint, CLI command, queue handler updates, persistence, console notification, and end-to-end validation.
4. [ ] **designer (lld)** — Each story needs file-level implementation detail for the domain logic, API routing, CLI parsing, queue handling, and database operations.
   - focus: Provide concrete file plans showing which modules handle state validation, where the withdrawal logic lives, how the API and CLI commands are structured, and where persistence changes go.
5. [ ] **test_architect (tests)** — Many edge cases stated in AC: validation rules, state transitions, persistence across restarts, queue behavior, console notification failure handling.
   - focus: Map each AC to concrete test cases, especially the state transition rules (AC 6, 8, 9), validation (AC 2), persistence (AC 7), and queue integration (AC 11).
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple documents (requirements, HLD, breakdown, stories with tests) must agree on state transitions, validation rules, API contract, and integration points.
   - focus: Verify consistency of withdrawal state transitions, reason validation rules, and integration contracts across all documents.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly defined; no ambiguity about user-facing behavior or product decisions.
