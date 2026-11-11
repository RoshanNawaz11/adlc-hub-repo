# Plan — PER-223 (v3)

- made from: review feedback (plan_review)
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

This feature adds reason validation to case rejections across three entry points (console, CLI, queue); story_with_design fits because it affects several files in one component and needs a concrete file plan before coding.

## What changed

Added critic step as requested by the reviewer to ensure consistency across requirements, LLD, and test specs.


## Feedback that informed this version

- **PR comment** by RoshanNawaz11 on hub#28 `[hub#28:issue-6791757443]`: add critic to planner
  - addressed by: Added critic step to the plan with consistency_check activity.
- **PR comment** by RoshanNawaz11 on hub#28 `[hub#28:issue-6791891194]`: add critic to planner
  - addressed by: Added critic step to the plan with consistency_check activity.

## Steps

1. [x] **analyst (inline_story)** — Convert the detailed acceptance criteria into Given/When/Then requirements with technical notes.
   - focus: Focus on the validation rules (empty, whitespace-only, trimming) and the three rejection entry points.
2. [x] **designer (lld)** — Multiple entry points for rejection need a file plan showing where validation is centralized and how each path calls it.
   - focus: Show which files handle console, CLI, and queue rejections, where the shared validation logic lives, and how trimming is applied.
3. [x] **test_architect (tests)** — Five acceptance criteria with specific edge cases (empty, whitespace-only, trimming) need explicit test coverage.
   - focus: Map tests 1:1 to the ACs, covering each entry point and each invalid reason case.
4. [ ] **critic (consistency_check)** _(rerun)_ — Requirements, LLD, and tests must agree on validation rules and the three entry points.
   - focus: Verify that the validation logic (empty, whitespace, trimming) is described consistently across all spec documents and that all three entry points are covered.
5. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Loop delivery.
7. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
8. [ ] **open_pr** _(auto)_ — Loop delivery.
9. [ ] **pr_review** _(auto)_ — Loop delivery.
10. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are concrete and internal-facing; no product ambiguity or user-facing decisions.
