# Course correction PER-64 (auto from critic exhaustion)

Authoritative for the next run. Human: review/edit this file, then re-run:
`adlc start PER-64 --skip-dev` or `adlc review request-changes PER-64 --note "..."` then `adlc resume PER-64`.

## Critic mismatches to resolve

**Concrete mismatches detected:**

1. **Spec file structure violation (change_requests.md directive ignored):**
   - **Found:** Three spec files created: `specs/01.md`, `specs/02.md`, `specs/03.md`
   - **Required:** One spec file (`specs/01.md`) covering all three stories per change_requests.md: "One spec file (specs/01.md) covers all three stories; do not require specs/02.md or specs/03.md"
   - **Impact:** specs/02.md and specs/03.md should not exist; all spec content belongs in specs/01.md

2. **Missing file path in repo-tree (real file check required):**
   - **File:** `tests/test_tracing.py`
   - **Status:** Referenced throughout all artifacts (requirements, HLD, breakdown, all three story LLDs, spec) but not shown in repo-tree under `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/tests/`
   - **Action required:** Re-read the real file per change_requests.md directive: "re-read the real files (src/agents/model/gateway.py, skills/client.py, runtime/runner.py, tests/test_tracing.py) and fix any details that don't match"
   - **Current risk:** Test rewrites in Story 3 may not align with actual file structure/assertions

3. **Proposed tickets AC coverage incomplete:**
   - **Required:** change_requests.md states "In proposed_tickets.md list every AC each ticket covers"
   - **Found:** proposed_tickets.md lists ACs per ticket but does not enumerate every Requirements AC each ticket covers in the format specified
   - **Example of required format:** `01-agent` should list "Requirements AC-1 (Gateway client refactored...), Requirements AC-6 (partial: existing tests pass)" explicitly per ticket rather than summary references

4. **File path inconsistency in specs:**
   - **specs/02.md (`src/agents/model/policy.yaml`):** Shows path as `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/src/agents/model/policy.yaml` (correct per repo-tree)
   - **specs/02.md (comment references):** References file as `model/policy.yaml` in Algorithm Notes and Documentation Contracts sections (missing `src/agents/` prefix)
   - **Impact:** Inconsistent path references may confuse implementation

5. **Test count sum alignment issue (minor, but worth noting):**
   - **Requirements AC-6:** States "9 new tests are added (6 in `test_gateway.py` + 3 in `test_tracing.py`)"
   - **Breakdown Test Strategy:** States "Total test count: 9 automated tests across all stories (Story 1: 0 new, Story 2: 0 new, Story 3: 9 new = 6 gateway + 3 tracing; sum: 0+0+9=9; aligns with Requirements AC-4, AC-5, AC-6)"
   - **All story LLDs:** Correctly state test counts (Story 1: 0, Story 2: 0, Story 3: 9)
   - **Status:** Alignment is correct, but `tests/test_tracing.py` existence must be verified per mismatch #2

**Recommendation:** Before proceeding to architect_gate review, address:
- Delete `specs/02.md` and `specs/03.md`; consolidate all spec content into `specs/01.md` per change_requests.md
- Re-read `tests/test_tracing.py` from repo and update Story 3 LLD/spec to match actual file structure
- Update `proposed_tickets.md` to enumerate every Requirements AC each ticket covers in full detail
- Fix path references in specs/02.md to use consistent `src/agents/model/policy.yaml` throughout

## Repair rules

- Keep ticket/repo labels (`repo:*`) and out-of-scope constraints.
- Prefer one complete story LLD per involved repo when possible; do not leave truncated mid-sentence files.
- Align AC counts, test counts, and file paths across requirements ↔ HLD ↔ breakdown ↔ stories.
- Rewrite incomplete artifacts fully (no cutoffs).
