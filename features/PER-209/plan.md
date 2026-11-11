# Plan — PER-209 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: no
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Case list with filtering, sorting and counts needs persistent storage and a new API; HLD designs the schema and endpoints, breakdown splits UI from backend, critic ensures consistency across specs.

## Steps

1. [x] **analyst (requirements)** — Convert the ticket into structured requirements with clear Given/When/Then acceptance criteria.
2. [x] **architect (hld)** — Design the case storage schema, the list/count API endpoints, and how cases survive restarts.
   - focus: Focus on the database schema for case persistence, the REST API contract for list/filter/count operations, and how case state is preserved across restarts.
3. [x] **scrum_master (breakdown)** — Split into separate stories for backend (persistence, API) and any frontend integration.
   - focus: Consider separating storage/API implementation from any UI changes; each should be independently testable.
4. [x] **designer (lld)** — Provide file-by-file implementation plans for the storage layer, API endpoints, and query building.
   - focus: Detail the database migration, repository pattern for case queries, API route handlers, and filter/sort query construction.
5. [x] **test_architect (tests)** — Map test cases to each AC, especially pagination edge cases, filter combinations, and restart persistence.
   - focus: Cover pagination boundaries, filter combinations, sort ordering, count accuracy, and post-restart state verification.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Cross-check that the schema in HLD matches what breakdown and LLDs describe, and that all ACs trace through.
   - focus: Verify the database schema is consistent across HLD and stories, API contracts match in all documents, and every AC has corresponding tests.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clear and fully technical; no product ambiguity or user-facing decisions need product sign-off.
