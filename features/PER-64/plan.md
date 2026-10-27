# Plan — PER-64 (v1)

- made from: ticket
- source: fallback (planner not used)
- complexity: moderate
- needs plan review: no
- track hint: standard

Track template (standard).

## Steps

1. [x] **analyst (requirements)** — standard track template
2. [x] **tpm_gate (review)** — standard track template
3. [x] **ticketize (normalize)** _(auto)_ — Normalize proposed tickets after TPM approval.
4. [x] **architect (hld)** — standard track template
5. [x] **scrum_master (breakdown)** — standard track template
6. [x] **designer (lld)** — standard track template
7. [x] **test_architect (tests)** — standard track template
8. [ ] **critic (consistency_check)** — standard track template
9. [ ] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
10. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
11. [ ] **dev** _(auto)_ — Fixed execution step.
12. [ ] **pre_pr_qa** _(auto)_ — Fixed execution step.
13. [ ] **open_pr** _(auto)_ — Fixed execution step.
14. [ ] **pr_review** _(auto)_ — Fixed execution step.
15. [ ] **merge** _(auto)_ — Fixed execution step.

## Planner notes

- attempt 1: designer needs ['stories'] but no earlier step produces it (plan must produce stories: analyst `inline_story` or scrum_master)
- attempt 2: Duplicate role: analyst; designer needs ['stories'] but no earlier step produces it (plan must produce stories: analyst `inline_story` or scrum_master)
- planner rejected; using track template
