# Plan — PER-129 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Building a complete bookstore API from scratch in an empty repo requires architecture design for the schema, components, and API contracts, making feature_with_architecture the right loop.

## Steps

1. [ ] **analyst (requirements)** — Extract Given/When/Then requirements and propose tickets for breakdown from the detailed acceptance criteria.
   - focus: The ticket already has very detailed ACs; keep the requirements minimal and focus on the functional boundaries (catalog, cart, checkout, shipping) rather than restating every AC.
2. [ ] **architect (hld)** — New application from empty repo needs HLD for the data schema, API contract, component structure, and tech stack decisions.
   - focus: Design the SQLite schema (product, cart, cart_item, order, order_item), the GraphQL or REST API surface, role-based auth approach, and the Node.js project structure. Address persistence, stock management, and cart-to-order lifecycle.
3. [ ] **scrum_master (breakdown)** — Work naturally splits into stories: scaffold, schema, catalog, cart, checkout, shipping, and tests.
   - focus: Break down into implementable slices: (1) project scaffold and health endpoint, (2) schema and migrations, (3) catalog CRUD with role checks, (4) cart operations, (5) checkout with stock validation, (6) shipping endpoint, (7) test coverage. Each story should be independently testable.
4. [ ] **designer (lld)** — Each story needs a concrete file plan and implementation detail before coding a greenfield API.
   - focus: Specify file structure, middleware for role checks, SQLite initialization, query patterns, and error handling for each story. Make the cart-id header handling and stock synchronization explicit.
5. [ ] **test_architect (tests)** — The ticket has many edge cases (stock limits, role checks, cart merge, persistence) that should be pinned as tests.
   - focus: Map each AC to test cases covering happy path, auth failures, stock validation, cart state transitions, and persistence across restarts.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple artifacts (requirements, HLD, breakdown, stories, tests) must agree on the schema, API contract, and role model.
   - focus: Verify the schema in HLD matches the stories, the role checks are consistent, the cart lifecycle is coherent, and stock reduction is covered in both checkout design and tests.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are explicitly stated in the ticket with clear in-scope and out-of-scope boundaries; no product ambiguity.
