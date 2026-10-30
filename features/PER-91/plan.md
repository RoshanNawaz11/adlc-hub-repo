# Plan — PER-91 (v2)

- made from: review feedback (architect_review)
- source: LLM planner
- complexity: simple
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Small refactor in one repo removing LiteLLM/Cloudflare routing and calling Anthropic directly; the existing plan is correct and the inline comment does not require spec changes.

## What changed

No spec changes needed; the inline comment about setting ANTHROPIC_API_KEY in env is acknowledged but does not contradict the ticket requirements or existing specs.

- Kept (results reused): analyst, designer, test_architect

## Feedback that informed this version

- **PR inline comment** by RoshanNawaz11 on hub#4 `features/PER-91/stories/01-agent.md:20` `[hub#4:inline-4422809044]`: anthropic key can be set later in env it is not set in the env
  - addressed by: The comment notes that ANTHROPIC_API_KEY can be set later in env if not already set. This aligns with the existing spec: AC3 already specifies that MissingAPIKeyError is raised when the key is unset, and .env.example already documents ANTHROPIC_API_KEY= as required. No spec revision needed; the implementation will handle the env var lookup at runtime as designed.

## Steps

1. [x] **analyst (inline_story)** — The ticket already contains detailed requirements with clear AC; convert to Given/When/Then format with implementation notes.
2. [x] **designer (lld)** — The change touches multiple files (gateway.py, .env.example, tests, docs) with specific refactoring steps that need a concrete file plan.
3. [x] **test_architect (tests)** — The ticket specifies new test cases in test_gateway.py and modifications to test_tracing.py that need mapping to acceptance criteria.
4. [ ] **architect_gate (review)** _(auto)_ _(rerun)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no separate requirements, HLD or breakdown documents to cross-check for consistency.
