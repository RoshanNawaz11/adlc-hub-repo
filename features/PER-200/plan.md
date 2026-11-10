# Plan — PER-200 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: quick
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

A readiness endpoint for the agents service that checks dependencies before traffic arrives; fits quick_fix because it adds one endpoint in one repo without changing any existing interface or requiring a file plan.

## Steps

1. [ ] **analyst (inline_story)** — Turn the ticket into Given/When/Then requirements with concrete acceptance criteria.
   - focus: Keep the inline story focused on the new /ready endpoint; the ticket already states exactly what to check and how to handle timeouts and errors.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Only one document (inline story) so nothing to cross-check.
