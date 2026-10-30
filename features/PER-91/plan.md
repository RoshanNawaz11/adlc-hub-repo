# Plan — PER-91 (v1)

- made from: ticket
- source: LLM planner
- complexity: moderate
- needs plan review: yes
- track hint: standard
- loop: story_with_design
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Remove LiteLLM and Cloudflare AI Gateway proxies and call Anthropic directly; this is a moderate refactor of the model gateway layer with test updates, requiring design for the concrete file plan.

## Steps

1. [ ] **analyst (inline_story)** — The ticket already contains detailed requirements with clear AC; convert to Given/When/Then format with implementation notes.
   - focus: Focus on the model gateway refactor, tracing setup changes, test updates, and documentation cleanup. Keep the out-of-scope boundaries explicit.
2. [ ] **designer (lld)** — The change touches multiple files (gateway.py, .env.example, tests, docs) with specific refactoring steps that need a concrete file plan.
   - focus: Provide a precise file-by-file plan for removing LiteLLM/Cloudflare code, rewiring the Anthropic client, updating tracing comments, rewriting docs, and adding new gateway tests. Include the exact lines or sections to change in gateway.py.
3. [ ] **test_architect (tests)** — The ticket specifies new test cases in test_gateway.py and modifications to test_tracing.py that need mapping to acceptance criteria.
   - focus: Map the four test_gateway.py scenarios (missing key, default URL, custom URL, client identity) and the test_tracing.py changes (remove header tests, add traceparent stability test) to the AC.
4. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
5. [ ] **dev** _(auto)_ — Loop delivery.
6. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
7. [ ] **open_pr** _(auto)_ — Loop delivery.
8. [ ] **pr_review** _(auto)_ — Loop delivery.
9. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **critic** — Single inline story with no separate requirements, HLD or breakdown documents to cross-check for consistency.
