# Plan — PER-101 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Durable idempotency with Redis requires architect (data contract and store design), scrum_master (split into store, runner, worker and test stories), designer (file plan for each), test_architect (edge cases for concurrency and TTL), and critic (ensure contract consistency across specs).

## Steps

1. [x] **analyst (requirements)** — Extract the idempotency contract, store API, runner changes and worker changes into structured requirements for architecture review.
2. [x] **architect (hld)** — Design the Redis key contract, claim/complete/release atomicity guarantees, error handling strategy and the two-store abstraction before implementation.
3. [x] **scrum_master (breakdown)** — Split into separate stories for the store component, runner integration, worker status logic and tests so each can be implemented and reviewed independently.
4. [x] **designer (lld)** — Each story needs a concrete file plan: the store module structure, where claim/complete/release calls go in runner.py, the conditional status write in orders.py, and test file organization.
5. [x] **test_architect (tests)** — Pin edge cases as tests: concurrent claims, TTL expiry, Redis failure modes, resume path without claim, and the worker status write guard.
6. [ ] **critic (consistency_check)** — Cross-check that the HLD contract, breakdown stories, LLD file plans and test cases all agree on key format, claim semantics, TTL values and error handling.
7. [ ] **merge_pack (pack)** _(auto)_ — Pack per-ticket specs into merged_plan.md for review.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope is clear from the ticket: durable idempotency with Redis, no product ambiguity or user-facing decision.

## Planner notes

- attempt 1: No loop: set `loop` to one loop id from the loop catalog
