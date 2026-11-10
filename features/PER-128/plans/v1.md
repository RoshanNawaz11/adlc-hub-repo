# Plan — PER-128 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Single-repo optimization with state changes, routing logic, and usage tracking; needs design review before coding.

## Steps

1. [x] **analyst (requirements)** — Requirements are already detailed in the ticket; formalize them in Given/When/Then format.
   - focus: Focus on the three requirement groups: token usage tracking with state changes, forced routing logic, and orchestrator integration. Keep the state migration requirement explicit.
2. [x] **architect (hld)** — State schema changes (usage field, seen flags), routing component contract, and cross-component agreement need HLD.
   - focus: Design the State.usage schema and reducer, forced_route contract with broken_rules, and how orchestrator tracks seen flags. Address state migration for runs without usage.
3. [x] **scrum_master (breakdown)** — Three distinct work areas need breakdown: usage tracking, routing logic, orchestrator changes.
   - focus: Split into stories by component: route_rules changes, State and reducer, orchestrator logic, usage logging. Order by dependency.
4. [x] **designer (lld)** — Implementation details for state reducer, forced_route algorithm, and orchestrator decision flow need concrete file plans.
   - focus: Detail the usage reducer merge logic, forced_route candidate generation without duplicating broken_rules, and orchestrator skip conditions.
5. [x] **test_architect (tests)** — The ticket lists specific test scenarios for forced routing, usage accumulation, and state migration.
   - focus: Map the ticket's test requirements 1:1 to test cases, covering forced_route behavior, orchestrator model call decisions, usage tracking, and state migration.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — State schema, routing contract, and orchestrator behavior must agree across HLD, breakdown, and stories.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Requirements and scope are clearly stated in the ticket; no product ambiguity or user-facing decisions.
