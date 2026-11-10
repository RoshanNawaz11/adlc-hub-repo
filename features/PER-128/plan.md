# Plan — PER-128 (v3)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Multi-part state and routing refactor with new telemetry: needs architecture review for state schema changes, orchestrator logic changes, and cross-module consistency.

## Steps

1. [x] **analyst (requirements)** — Ticket already states requirements clearly; formalizing them as Given/When/Then for HLD input.
   - focus: Focus on the three separate concerns: usage telemetry schema and aggregation, forced routing logic, and orchestrator decision flow. Make the contract between State changes and orchestrator behavior explicit.
2. [x] **architect (hld)** — State schema changes, new routing component, and orchestrator decision logic must be designed together to ensure consistency.
   - focus: Design the usage accumulation pattern in State, the forced_route interface and its relationship to broken_rules, and how the orchestrator coordinates them. Address backward compatibility for runs without usage field.
3. [x] **scrum_master (breakdown)** — Work spans multiple modules (State, route_rules, node.py, runner.py, tools/bindings.py, model gateway) and must be sequenced for testability.
   - focus: Break down by dependency: State changes first, then route_rules, then orchestrator, then usage collection, then endpoint. Each story must be independently testable.
4. [x] **designer (lld)** — Implementation detail needed for State reducer logic, forced_route algorithm, and orchestrator decision tree.
   - focus: Specify exact State schema with migration strategy, forced_route's candidate enumeration, and orchestrator's decision flowchart with all guard conditions.
5. [x] **test_architect (tests)** — Complex logic with edge cases: multiple routes valid, flag deduplication, usage aggregation, backward compatibility.
   - focus: Map tests to ACs 1:1, especially forced_route's candidate logic, orchestrator's model-call guards, and usage accumulation across retries.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple documents must agree on State schema, routing contract, and orchestrator behavior.
   - focus: Verify State usage schema matches across HLD, stories and tests; forced_route contract is consistent between HLD and LLD; orchestrator decision logic in HLD matches implementation plan in stories.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket states scope and acceptance criteria clearly; no ambiguity or cross-team coordination needed.
