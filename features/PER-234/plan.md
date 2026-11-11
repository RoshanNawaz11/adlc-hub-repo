# Plan — PER-234 (v2)

- made from: review feedback (plan_review)
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Add reviewer-set case priority with reason and persistence; story_with_design loop fits because it adds new API endpoints and database fields while keeping the changes in one repo.

## What changed

Switched from story_with_design to feature_with_architecture and added critic role because this feature adds API endpoints and database schema changes, requires breakdown into multiple stories, and needs cross-artifact consistency checks.

- Added: architect, scrum_master, critic

## Feedback that informed this version

- **PR comment** by RoshanNawaz11 on hub#30 `[hub#30:issue-6795427959]`: add critic to the plan
  - addressed by: Added critic step to cross-check requirements, HLD, breakdown, stories and tests for consistency on API contracts, database schema, validation rules and state transitions.

## Steps

1. [ ] **analyst (requirements)** _(rerun)_ — Convert the ticket into clear Given/When/Then requirements covering all priority-setting scenarios.
   - focus: Focus on the validation rules (allowed values, non-empty reason), state transitions (which case states allow changes), and persistence requirements.
2. [ ] **architect (hld)** — This adds API endpoints for setting priority, a database field for reviewer-set priority and reason, and changes how priority is calculated—contract changes that need high-level design.
   - focus: Design the API contract (endpoint, request/response shapes, error codes), the database schema additions (priority_set_by_reviewer, reviewer_priority, priority_reason fields), and how priority calculation logic changes to respect reviewer overrides.
3. [ ] **scrum_master (breakdown)** — The work splits into stories: API endpoint, business logic, persistence, validation, history tracking, activity feed integration.
   - focus: Break down by technical slice: API layer, domain logic for priority override, database persistence, validation, audit history, activity feed notification.
4. [ ] **designer (lld)** _(rerun)_ — Each story needs a concrete file plan covering API handlers, domain models, repository layer, and validation logic.
   - focus: Specify which files implement the API endpoint, where priority override logic goes in the domain model, how the repository persists the new fields, and where validation happens.
5. [ ] **test_architect (tests)** _(rerun)_ — The acceptance criteria have clear edge cases: invalid priorities, empty reasons, different case states, history tracking, activity feed.
   - focus: Map each AC to test cases covering happy paths, validation failures (invalid priority values, empty reasons), state restrictions (running/done/rejected cases), and persistence across restarts.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple spec documents (requirements, HLD, breakdown, stories, tests) must agree on API contracts, database schema, validation rules, and state transitions.
   - focus: Verify the API contract in HLD matches story implementation plans, database fields in HLD match persistence stories, validation rules are consistent across requirements and tests, and state restrictions are covered everywhere.
8. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket clearly states scope (reviewer priority setting only) and acceptance criteria; no product sign-off needed.
