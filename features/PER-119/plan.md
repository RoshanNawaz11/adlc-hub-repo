# Plan — PER-119 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Performance optimization and observability improvement in a single repo with clear requirements and multiple design concerns (state schema, routing logic, usage tracking); feature_with_architecture fits to ensure the state changes, orchestrator logic and usage aggregation all agree before coding.

## Steps

1. [x] **analyst (requirements)** — Extract the three distinct requirements (forced routing, usage tracking, state changes) from the detailed ticket.
   - focus: Focus on the three main requirements: forced_route logic, usage aggregation per node, and orchestrator skip conditions. Keep the acceptance criteria concrete.
2. [x] **architect (hld)** — State schema changes, routing component design, and usage tracking across multiple call sites need architectural agreement before implementation.
   - focus: Design the state.usage schema with its reducer, the forced_route algorithm that delegates to broken_rules, and how usage flows from run_skill/bindings/ModelGateway through node updates to state. Address backward compatibility for runs saved before this change.
3. [x] **scrum_master (breakdown)** — Work spans route_rules.py, node.py, state schema, bindings.py, ModelGateway, and runner.py; breakdown into implementable slices needed.
   - focus: Break down by component: (1) state.usage schema and reducer, (2) forced_route in route_rules.py, (3) orchestrator skip logic in node.py, (4) usage collection in run_skill/bindings/ModelGateway, (5) logging and API changes in runner.py.
4. [x] **designer (lld)** — Each story needs concrete file plans for state changes, the forced_route algorithm, usage propagation, and test setup with fake gateway.
   - focus: Provide detailed file plans showing exactly where usage gets collected (run_skill, bindings tool loop, ModelGateway), how the reducer merges counts, and how forced_route delegates to broken_rules without duplicating rules.
5. [x] **test_architect (tests)** — Acceptance criteria include specific test cases for forced_route behavior, orchestrator skip conditions, usage aggregation, and backward compatibility.
   - focus: Map each AC to tests: forced_route cases, orchestrator model call skip, forced hop step format, usage accumulation across retries and tool rounds, backward compatibility for runs without usage.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Multiple artifacts (requirements, HLD, breakdown, stories, tests) must agree on state schema, forced_route contract, and usage flow.
   - focus: Verify consistency across all specs: state.usage schema matches in HLD and stories, forced_route algorithm agrees with broken_rules delegation, usage collection points match across HLD and implementation stories.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Technical performance optimization with clear requirements in ticket; no product scope ambiguity or user-facing changes needing product sign-off.
