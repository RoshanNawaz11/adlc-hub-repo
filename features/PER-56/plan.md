# Plan — PER-56 (v1)

- made from: ticket
- source: LLM planner
- complexity: moderate
- needs plan review: no
- track hint: standard

Multi-step tool orchestration with bounded loop; requires detailed requirements, high-level loop design, breakdown into implementable stories, concrete file plan, and test cases.

## Steps

1. [x] **analyst (requirements)** — Clarify loop semantics, step-limit behavior, and edge cases for the multi-step agent flow.
   - focus: Focus on the loop termination conditions (text answer vs step limit), what happens when max_steps is reached, and the exact message-history structure after each tool round.
2. [x] **architect (hld)** — Design the bounded agent loop with message flow, state transitions, and error handling before implementation.
   - focus: Detail the loop control flow, message-list evolution across steps, the final tools=None fallback, and how existing ValueError handling integrates with the new loop.
3. [x] **scrum_master (breakdown)** — Split orchestrator changes, server config changes, and test expansion into ordered implementable stories.
   - focus: Separate the core loop logic, max_steps parameter and env var parsing, test updates, and README update into distinct stories that can be implemented and tested incrementally.
4. [x] **designer (lld)** — Concrete file plan for orchestrator.py and run_server changes, plus test expansion, before coding.
   - focus: Specify the exact loop structure in handle_query, where max_steps checking happens, AGENT_MAX_STEPS parsing in run_server, and which test cases cover the new paths.
5. [x] **test_architect (tests)** — Pin acceptance criteria as test cases, mapping loop scenarios and error conditions to test methods.
   - focus: Map each AC scenario (direct text, two-step chain, step limit with answer, step limit without answer, unknown tool on step 2, AGENT_MAX_STEPS parsing) to a test case with assertions on call counts and message structure.
6. [x] **critic (consistency_check)** — Ensure requirements, HLD, breakdown, stories, and test cases agree on loop semantics and step-limit behavior.
7. [x] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Fixed execution step.
10. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
11. [ ] **open_pr** _(auto)_ — Fixed execution step.
12. [ ] **pr_review** _(auto)_ — Fixed execution step.
13. [ ] **merge** _(auto)_ — Fixed execution step.

## Skipped roles

- **tpm_gate** — Product scope is clear: bounded multi-step tool use with a step limit; no ambiguity requiring product sign-off.

## Planner notes

- attempt 1: designer needs ['stories'] but no earlier step produces it (plan must produce stories: analyst `inline_story` or scrum_master)
