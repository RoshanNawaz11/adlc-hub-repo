# Plan — PER-189 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Build a CLI catalogue generator from rule YAML files — one repo, one story with a file plan for the parser and command structure.

## Steps

1. [ ] **analyst (inline_story)** — The ticket gives clear scope and acceptance criteria; the analyst writes one implementable story.
   - focus: Keep all seven acceptance criteria as-is; the implementation plan is for the designer.
2. [ ] **designer (lld)** — The work touches several files: the parser for rule YAML, the catalogue builder, the CLI command, the markdown writer, and tests; a file-by-file plan is needed.
   - focus: Show which modules parse the YAML, how flag extraction works across cases/checks/lines, how tables are built per agent, and where the --check diff logic goes.
3. [ ] **test_architect (tests)** — The acceptance criteria list specific test scenarios (flags in cases/checks/lines, flags list, agent/row order, broken files, --check modes) that map 1:1 to test cases.
   - focus: Map each scenario in AC 7 to a test case; ensure broken YAML parsing, table diffing, and deterministic output are all covered.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one spec document (the story); nothing to cross-check for consistency.
