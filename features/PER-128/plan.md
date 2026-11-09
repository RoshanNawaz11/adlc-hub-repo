# Plan — PER-128 (v2)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Standard multi-concern refactoring that needs concrete file plan (State schema, routing logic, orchestrator flow, usage tracking) with cross-artifact consistency checks.

## Steps

1. [ ] **analyst (inline_story)** — Ticket already has clear Given/When/Then requirements and concrete acceptance criteria.
   - focus: Extract the requirements and acceptance criteria as stated; this is implementation detail for an existing system, not a user-facing feature.
2. [ ] **designer (lld)** — Multiple files must be modified in coordinated way: State schema, route_rules.py logic, node.py orchestrator flow, usage tracking across runner.py, client.py, bindings.py.
   - focus: Focus on State schema changes (usage field with reducer, seen_flags tracking), forced_route implementation without duplicating broken_rules logic, orchestrator decision tree (when to skip model vs when model work remains), and usage accumulation at each call site.
3. [ ] **test_architect (tests)** — Acceptance criteria specify several test scenarios with precise assertions about forced routes, model call skipping, usage tracking, and backwards compatibility.
   - focus: Map each AC bullet to concrete test cases; ensure coverage of forced route scenarios, orchestrator model skip logic, usage accumulation across retries and tool rounds, and state migration.
4. [ ] **critic (consistency_check)** — Requirements, design and tests must agree on State schema fields, forced_route behavior, orchestrator skip conditions, and usage tracking locations.
5. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Loop delivery.
7. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
8. [ ] **open_pr** _(auto)_ — Loop delivery.
9. [ ] **pr_review** _(auto)_ — Loop delivery.
10. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket clearly states scope (routing optimization and usage tracking in existing orchestrator), acceptance criteria are concrete implementation checks, no product sign-off needed for internal performance optimization.
