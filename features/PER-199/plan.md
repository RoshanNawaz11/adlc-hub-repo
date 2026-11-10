# Plan — PER-199 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Add persistent notes to paused cases: API endpoint, storage, and UI display; fits story_with_design because it adds a new endpoint and database field while staying in one repo.

## Steps

1. [x] **analyst (requirements)** — Extract precise requirements from the ticket's acceptance criteria.
   - focus: Focus on the state machine (when notes can be added), validation rules, and persistence requirements.
2. [x] **architect (hld)** — New API endpoint, database schema change, and case state checks require HLD.
   - focus: Design the notes storage schema, the POST endpoint contract, and how notes integrate with existing case state and retrieval.
3. [x] **scrum_master (breakdown)** — Work splits into storage, API, and UI display stories.
   - focus: Break down by: schema/migration, API endpoint with validation, UI integration.
4. [x] **designer (lld)** — Each story needs file-by-file implementation detail.
   - focus: Specify which files handle schema, validation, state checks, and rendering.
5. [x] **test_architect (tests)** — AC includes edge cases: empty notes, over-limit, wrong state, persistence across restarts.
   - focus: Map each AC to test cases, especially validation and state machine transitions.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [x] **critic (consistency_check)** — Requirements, HLD, breakdown, stories, and tests must agree on schema, endpoint contract, and validation rules.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clear and internal-facing; no product ambiguity.
