# Plan — PER-191 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Build a fact catalogue and consistency checker from rules metadata; entirely internal tooling in one repo with clear file structure and validation logic.

## Steps

1. [ ] **analyst (inline_story)** — The ticket clearly defines the feature but needs Given/When/Then format and tech notes for implementation.
   - focus: Focus on the catalogue generation logic, lookup command interface, validation checks, and error handling. Emphasize the requirement for idempotent output and byte-for-byte reproducibility.
2. [ ] **designer (lld)** — This needs a concrete file plan: modules for parsing rules and facts.yaml, building the catalogue structure, formatting markdown, implementing CLI commands, and validation logic.
   - focus: Design the module structure for parsing (rules, facts.yaml), catalogue building, markdown generation, CLI interface, and validation checks. Specify data structures for fact metadata and cross-references.
3. [ ] **test_architect (tests)** — The ticket explicitly lists seven test scenarios that need to be mapped to the acceptance criteria.
   - focus: Map the seven explicit test cases to ACs: facts read via inputs/lines.each (AC1), rule-produced facts (AC1), the three fix checks (AC3), section/row order (AC1), --fact known/unknown (AC2), broken file handling (AC5), --check pass/fail (AC4).
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one spec document (inline story with LLD and tests); nothing to cross-check.
