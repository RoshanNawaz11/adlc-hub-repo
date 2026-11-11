# Plan — PER-234 (v3)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Adding reviewer-set priority with reason to existing cases; changes the case data model (new fields), case retrieval (show reviewer priority), and adds a new action (set priority API/command), so it needs HLD and designer for file-level planning.

## Steps

1. [ ] **analyst (requirements)** — Extract concrete requirements from the ticket's ACs.
   - focus: Focus on the data model changes (reviewer-set priority, reason, timestamp), the new action (set priority), validation rules, and state persistence.
2. [ ] **architect (hld)** — Design the schema changes (case model additions), the set-priority interface, and how reviewer priority overrides planner priority.
   - focus: Cover the case model extension (reviewer_priority, reviewer_priority_reason, reviewer_priority_set_at, reviewer_priority_set_by), the set priority operation's interface and validation, priority resolution logic, history recording, and activity feed integration.
3. [ ] **scrum_master (breakdown)** — Break down into stories: schema migration, set-priority logic, display changes, history, activity feed.
   - focus: Separate schema/model changes, business logic (set priority with validation), display updates, history tracking, and activity feed notification into implementable slices.
4. [ ] **designer (lld)** — Provide file-level implementation plan for each story slice.
   - focus: Identify which files handle case state, case operations, validation, history, activity feed, and display formatting.
5. [ ] **test_architect (tests)** — Map each AC to concrete test cases covering validation, state changes, and edge cases.
   - focus: Cover validation (empty reason, invalid priority), state transitions (running/done/rejected cases), persistence, priority override behavior, and history recording.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Ensure requirements, HLD, breakdown, stories, and tests agree on the data model, validation rules, and behavior.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket clearly states scope (reviewer sets priority with reason, specific ACs, explicit out-of-scope items); no ambiguity needing product sign-off.
