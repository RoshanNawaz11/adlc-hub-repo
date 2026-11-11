# Plan — PER-223 (v2)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Input validation on case rejection reason across three entry points (console, CLI, queue); fits story_with_design because it needs a file-by-file plan for validation logic placement in existing rejection flows.

## Steps

1. [ ] **analyst (inline_story)** — Convert the detailed acceptance criteria into Given/When/Then requirements with technical notes.
   - focus: Focus on the three rejection entry points (console, CLI, queue) and the validation rules for the reason field. Keep the out-of-scope items clear.
2. [ ] **designer (lld)** — Multiple entry points for rejection need a file plan showing where validation is centralized and how each path calls it.
   - focus: Identify the console handler, CLI command, and queue processor for rejections. Show where the reason validation function lives and how each entry point uses it. Include the trim-before-save logic.
3. [ ] **test_architect (tests)** — Five acceptance criteria with specific edge cases (empty, whitespace-only, trimming) need explicit test coverage.
   - focus: Map tests 1:1 to the five ACs. Cover all three entry points, whitespace variations (spaces, tabs, line breaks), and the queue's paused-with-message behavior.
4. [ ] **critic (consistency_check)** — Requirements, LLD, and tests must agree on validation rules and the three entry points.
5. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
6. [ ] **dev** _(auto)_ — Loop delivery.
7. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
8. [ ] **open_pr** _(auto)_ — Loop delivery.
9. [ ] **pr_review** _(auto)_ — Loop delivery.
10. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are concrete and internal-facing; no product ambiguity or user-facing decisions.
