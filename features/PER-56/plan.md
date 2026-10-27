# Plan — PER-56 (v2)

- made from: review feedback (architect_review)
- source: LLM planner
- complexity: moderate
- needs plan review: no
- track hint: standard

Bounded multi-step agent loop with step limit: architect and designer rerun to change endpoint from /query to /query_given.

## What changed

Changed HTTP endpoint from /query to /query_given per inline feedback; architect and designer regenerate specs with the new endpoint.

- Rerun: architect, scrum_master, designer, test_architect, critic
- Kept (results reused): analyst

## Feedback that informed this version

- **PR inline comment** by RoshanNawaz11 on hub#1 `features/PER-56/requirements.md:65` `[hub#1:inline-4387749082]`: change the endpoint to /query_given
  - addressed by: Architect and designer rerun with endpoint /query_given instead of /query

## Steps

1. [x] **analyst (requirements)** — Clarify loop semantics, step-limit behavior, and edge cases for the multi-step agent flow.
2. [x] **architect (hld)** — Design the bounded agent loop with message flow, state transitions, and error handling before implementation.
   - focus: Change the endpoint from /query to /query_given in all contracts and specifications.
3. [x] **scrum_master (breakdown)** — Split orchestrator changes, server config changes, and test expansion into ordered implementable stories.
4. [x] **designer (lld)** — Concrete file plan for orchestrator.py and run_server changes, plus test expansion, before coding.
   - focus: Update all endpoint references from /query to /query_given in file plans, route handlers, and test specifications.
5. [x] **test_architect (tests)** — Pin acceptance criteria as test cases, mapping loop scenarios and error conditions to test methods.
6. [x] **critic (consistency_check)** — Ensure requirements, HLD, breakdown, stories, and test cases agree on loop semantics and step-limit behavior.
7. [x] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
8. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Fixed execution step.
10. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
11. [ ] **open_pr** _(auto)_ — Fixed execution step.
12. [ ] **pr_review** _(auto)_ — Fixed execution step.
13. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Product scope is clear: bounded multi-step tool use with a step limit; no ambiguity requiring product sign-off.
