# Plan — PER-154 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Single-repo validation fix in the flag resume logic; needs a concrete file plan before coding since it touches two distinct resume paths with shared validation logic.

## Steps

1. [ ] **analyst (inline_story)** — The ticket already provides clear Given/When/Then acceptance criteria for both resume paths.
   - focus: Keep the existing ACs exactly as stated; focus on the two resume paths and their shared validation requirements.
2. [ ] **designer (lld)** — Two different code paths need validation logic; a file plan will identify where shared validation goes and how both paths call it.
   - focus: Design a shared validation function for flag answers, show how both resume paths (HTTP endpoint and queue service) call it, and specify the error responses for each path.
3. [ ] **test_architect (tests)** — Five distinct test scenarios are stated in AC 5 (unknown ids, duplicate ids, valid reply, both paths), worth pinning as explicit test cases.
   - focus: Map the five test scenarios from AC 5 to test cases covering both resume paths.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope is clear and fully technical; no user-facing behavior or product decision needed.
- **critic** — Single inline story with no separate requirements or HLD; nothing to cross-check.
