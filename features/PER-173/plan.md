# Plan — PER-173 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Add a GET /runs endpoint that lists runs from the logs/ folder with status filtering and paging; this is a new API endpoint so it needs the feature_with_architecture loop.

## Steps

1. [ ] **analyst (requirements)** — Convert the ticket into structured requirements with Given/When/Then acceptance criteria.
   - focus: Focus on the API contract (endpoint, query params, response shape, error codes) and the status determination logic that must match the existing GET /runs/{thread_id} endpoint.
2. [ ] **architect (hld)** — This adds a new API endpoint, which is a contract change requiring HLD.
   - focus: Define the OpenAPI spec for GET /runs with query parameters and response schema; document how status is determined from checkpoint.db to ensure consistency with the existing endpoint; address error handling for unreadable folders.
3. [ ] **scrum_master (breakdown)** — The work may split into implementation stories (endpoint handler, folder scanning logic, status determination, paging, filtering, tests).
   - focus: Break down into implementable stories covering the endpoint, folder scanning, status logic, filters, paging, and comprehensive test coverage.
4. [ ] **designer (lld)** — Each story needs a file-by-file plan: which modules to create or modify, function signatures, and how components interact.
   - focus: Specify which files in src/agents/runtime/ need changes; detail the folder scanning logic, checkpoint reading, status mapping, and how to integrate with the existing FastAPI router.
5. [ ] **test_architect (tests)** — The acceptance criteria include specific test coverage requirements (newest-first, status agreement, filters, paging, broken folders).
   - focus: Map each AC to concrete test cases; ensure tests verify status consistency between list and detail endpoints, and cover edge cases like missing or corrupted checkpoint files.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple artifacts (requirements, HLD with OpenAPI spec, breakdown, stories with LLD, tests) must agree on the API contract and status logic.
   - focus: Verify the OpenAPI schema in the HLD matches the response structure in stories; confirm status determination is identical across all documents; check test cases cover all acceptance criteria.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket clearly states scope and acceptance criteria; no product sign-off needed for this internal support endpoint.
