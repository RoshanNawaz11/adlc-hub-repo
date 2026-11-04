# Plan — PER-114 (v2)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Single-file bug fix in base/math.jl to return (missing, missing) from sincospi(missing); the ticket clearly states the change and has explicit acceptance criteria.

## Steps

1. [x] **analyst (inline_story)** — Convert the ticket into an implementable story with Given/When/Then and technical notes.
   - focus: The ticket already has clear scope and acceptance criteria. Focus on the file location (base/math.jl around line 1325), the specific method to change (sincospi(::Missing)), and the test file (test/math.jl). Emphasize that other Missing methods stay unchanged.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are explicit in the ticket; no product ambiguity.
- **designer** — The fix is a single method change with clear implementation detail in the ticket; no file plan needed.
- **test_architect** — Acceptance criteria map directly to test cases; no additional test design required.
- **critic** — Single inline story with no other artifacts to cross-check.
