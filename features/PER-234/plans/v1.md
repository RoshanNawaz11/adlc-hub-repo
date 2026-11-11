# Plan — PER-234 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

A reviewer sets a case's priority with a reason, overriding the agent's default; this is a multi-file change in one existing component that needs a file-by-file plan before coding.

## Steps

1. [ ] **analyst (inline_story)** — Convert the ticket into a single implementable story with Given/When/Then acceptance criteria.
2. [ ] **designer (lld)** — This touches several files (API, business logic, persistence, validation) and needs a concrete file plan before coding.
   - focus: Focus on where priority overrides are stored, how they persist across restarts, and how they interact with the existing priority logic that raises on flags.
3. [ ] **test_architect (tests)** — The acceptance criteria have clear edge cases (invalid priorities, empty reasons, different case states) that should map to specific tests.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one spec document (the story with LLD and tests), so there are no cross-artifact consistency checks needed.
