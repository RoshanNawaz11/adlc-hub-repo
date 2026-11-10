## What would go wrong

### C4
- file: stories/01-graphql.md
- section: ## Acceptance Criteria
- quote: **And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string present only when `ok` is false.
- match: ticket › Acceptance criteria › 1. **Checks:**
- problem: The ticket says each check returns `{name, ok, ms, error}` without specifying when `error` appears, but the story adds "present only when `ok` is false". This means a successful check would return `{name, ok, ms}` without an `error` key, which is a new constraint the ticket does not state. The developer would omit the field on success when the ticket may have expected it always present (null or empty string).
- change: Match the ticket exactly: state `{name, ok, ms, error}` without adding the conditional presence rule, or confirm with the ticket author whether `error` should be omitted or set to null/empty on success.

### C5
- file: requirements.md
- section: AC-1: Dependency checks
- quote: **And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string present only when `ok` is false.
- match: ticket › Acceptance criteria › 1. **Checks:**
- problem: Same as C4. The requirements add "present only when `ok` is false" where the ticket says only `{name, ok, ms, error}`. The developer would build one behavior from the story and requirements, but the ticket may have intended another.
- change: Match the ticket exactly or clarify the conditional presence of `error` with the ticket author.

## Notes for the gate

- **C1, C2, C3:** The fixer marked these as `cannot_fix` because the story file has a structure problem (sections not appearing exactly once). The file name `01-graphql.md` is wrong for a Python REST endpoint story, the test count (10 vs. 7) still disagrees with AC-7, and the manual verification steps still duplicate automated coverage. A person at the gate must decide whether to rename the file, reconcile the test count, and clarify the verification steps.

VERDICT: FAIL
