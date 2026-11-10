# Plan — PER-189 (v2)

- made from: review feedback (plan_review)
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Flag catalogue command that reads rule YAML files and writes a markdown table; story_with_design fits because it needs a file-by-file plan for parser, builder, CLI, writer and tests, all in one repo.

## What changed

Added critic step to cross-check consistency between the story, LLD, and test plan as requested in feedback.

- Added: critic

## Feedback that informed this version

- **PR comment** by RoshanNawaz11 on hub#19 `[hub#19:issue-6767407859]`: add critic in it as well
  - addressed by: Added critic step after test_architect

## Steps

1. [ ] **analyst (inline_story)** — The ticket gives clear scope and acceptance criteria; the analyst writes one implementable story.
2. [ ] **designer (lld)** — The work touches several files: the parser for rule YAML, the catalogue builder, the CLI command, the markdown writer, and tests; a file-by-file plan is needed.
3. [ ] **test_architect (tests)** — The acceptance criteria list specific test scenarios (flags in cases/checks/lines, flags list, agent/row order, broken files, --check modes) that map 1:1 to test cases.
4. [ ] **critic (consistency_check)** — Cross-check consistency between the inline story, the LLD, and the test plan.
5. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Loop delivery.
7. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
8. [ ] **open_pr** _(auto)_ — Loop delivery.
9. [ ] **pr_review** _(auto)_ — Loop delivery.
10. [ ] **merge** _(auto)_ — Loop delivery.
