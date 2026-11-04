# Plan — PER-116 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Single-repo behavioral fix to add --threads=1 to lld on Linux CI; ticket states the file, change location, and acceptance criteria clearly.

## Steps

1. [x] **analyst (inline_story)** — Convert the ticket into one implementable story with Given/When/Then AC and tech notes.
   - focus: Focus on the conditional logic (Linux + CI=true → add --threads=1) and the test cases in test/misc.jl. Emphasize that CI must be read at call time, not build time.
2. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [x] **dev** _(auto)_ — Loop delivery.
4. [x] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [x] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no other artifacts to cross-check.
