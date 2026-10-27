# Course correction PER-64 (auto from critic exhaustion)

Authoritative for the next run. Human: review/edit this file, then re-run:
`adlc start PER-64 --skip-dev` or `adlc review request-changes PER-64 --note "..."` then `adlc resume PER-64`.

## Critic mismatches to resolve

- **AC count mismatch — proposed tickets vs requirements:** Proposed tickets define 3 tickets with "see requirements.md AC-X" references, but requirements.md has 7 ACs (AC-1 through AC-7). Proposed ticket 01 references AC-1 only (1 AC), ticket 02 references AC-2 only (1 AC), ticket 03 references AC-3, AC-4, AC-5, AC-6, AC-7 (5 ACs). This maps to breakdown stories correctly (Story 1 has 2 ACs covering requirements AC-1, Story 2 has 2 ACs covering requirements AC-2, Story 3 has 5 ACs covering requirements AC-3/4/5/6/7), but proposed_tickets.md oversimplifies by saying "see requirements.md AC-X" without enumerating all covered ACs. Requirements AC count (7) matches breakdown total story AC count (2+2+5=9), but proposed_tickets.md AC references are incomplete.

- **Test count mismatch — breakdown vs stories:** Breakdown Story 3 verification approach says "9 automated tests (6 gateway + 3 tracing)" and breakdown "Test Strategy" section says "Total test count: 9 automated tests (6 gateway + 3 tracing; aligns with Requirements AC-4, AC-5, AC-6)". Story 03-agent LLD Tests section says "Total automated tests: 9 tests (6 gateway + 3 tracing)" and lists 6 tests in test_gateway.py + 3 tests in test_tracing.py. Story 01-agent LLD Tests section says "0 new automated tests in this story (Story 3 adds 6 gateway tests + 3 tracing tests = 9 total automated tests per breakdown)". Story 02-agent LLD Tests section says "0 new tests in this story; Story 3 adds 9 automated tests (6 gateway + 3 tracing)". Requirements AC-4 says "6 test cases" (gateway), AC-5 says "3 test cases" (tracing), AC-6 says "9 new tests". **Counts align** across all artifacts (requirements 6+3=9, breakdown 6+3=9, stories 6+3=9).

- **File paths — no stack/language mismatch:** Ticket context establishes Python repo at `/home/regularinsaan/projects/trinet-adlc-agents`. All file paths (`src/agents/model/gateway.py`, `runtime/runner.py`, `model/policy.yaml`, `tests/test_gateway.py`, `tests/test_tracing.py`, `.env.example`, `README.md`, `docs/claude-via-litellm-cloudflare.md`) are Python paths consistent with ticket. Repo-tree shows empty `existing files:` list but paths are specified in ticket context. Spec 01 declares "Python 3.11+", "anthropic SDK", "pytest", "uv" — matches ticket context. No TypeScript, JavaScript, or other language paths invented. **No mismatch.**

- **HLD outcomes vs LLD:** HLD "Contracts" section describes `require_client() -> anthropic.AsyncAnthropic` returning singleton, `MissingAPIKeyError` message "ANTHROPIC_API_KEY environment variable required", `traceparent()` W3C format, `session_id` ContextVar label "run_session_id". Story 01-agent LLD "Interfaces" section matches: `require_client() -> anthropic.AsyncAnthropic`, `MissingAPIKeyError` message text identical, `traceparent()` format identical, `session_id: ContextVar[str] = ContextVar("run_session_id")`. Story 03-agent LLD "Tests" section validates these contracts in test cases. **No mismatch.**

- **Missing specs/<id>.md for child tickets:** Proposed tickets define `01-agent`, `02-agent`, `03-agent`. `specs/01.md` exists (Story 01-agent spec). No `specs/02.md` or `specs/03.md` present in provided artifacts. Stories `02-agent` and `03-agent` have LLD files (`stories/02-agent.md`, `stories/03-agent.md`) but no corresponding `specs/02.md`, `specs/03.md`. Per ADLC consistency rule 2, breakdown → story LLD must align; per plan.md step 6 "designer (lld)" produces story LLDs, but spec files under `specs/` directory are missing for tickets 02 and 03. **Missing `specs/02.md` and `specs/03.md`.**

- **change_requests.md:** Not present in provided artifacts. No contradiction to check.

- **merged_plan.md:** Not present in provided artifacts (plan.md step 9 "merge_pack (pack)" not yet executed per `[ ]` status). No order gaps to check.

**Concrete mismatches:**
- Proposed tickets AC references incomplete (ticket 01 says "see AC-1" but should enumerate both story ACs; ticket 02 says "see AC-2" but should enumerate both story ACs; ticket 03 says "see AC-3, AC-4, AC-5, AC-6, AC-7" which is correct).
- Missing `specs/02.md` for proposed ticket `02-agent`.
- Missing `specs/03.md` for proposed ticket `03-agent`.

## Repair rules

- Keep ticket/repo labels (`repo:*`) and out-of-scope constraints.
- Prefer one complete story LLD per involved repo when possible; do not leave truncated mid-sentence files.
- Align AC counts, test counts, and file paths across requirements ↔ HLD ↔ breakdown ↔ stories.
- Rewrite incomplete artifacts fully (no cutoffs).
