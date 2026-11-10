# Plan — PER-190 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Add timeline tracking to runs by reading LangGraph checkpoint history; this is a contained enhancement in one repo that needs a file-by-file implementation plan.

## Steps

1. [ ] **analyst (inline_story)** — Turn the ticket into implementable Given/When/Then requirements with acceptance criteria.
   - focus: Focus on the checkpoint history reading logic, parallel step detection, and the two API endpoints; keep the scope tight to what's in the ACs.
2. [ ] **designer (lld)** — This needs a file-by-file plan: new timeline builder function, API endpoint additions, and checkpoint history reading logic across multiple files.
   - focus: Detail which files implement the timeline builder, where the API endpoints go, how to extract timing from checkpoints, and how parallel steps are detected; address the existing runner.py and server.py structure.
3. [ ] **test_architect (tests)** — Seven explicit test scenarios in AC 7 need concrete test cases mapped to acceptance criteria.
   - focus: Map each of the seven test scenarios to specific test cases with setup and assertions.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no separate HLD or breakdown to cross-check.
