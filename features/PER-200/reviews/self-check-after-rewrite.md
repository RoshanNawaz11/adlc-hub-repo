## What would go wrong

### C1
- file: stories/01-graphql.md
- section: (filename)
- quote: # Story — PER-200
- problem: The file is named `01-graphql.md` but the repository context states this is a Python repo (`app`), not a GraphQL service. The ticket references `src/agents/runtime/server.py` and Python modules throughout. A developer looking for this story would be confused by the graphql naming.
- change: Rename the file to match the story content (e.g., `01-readiness-check.md` or `01-ready-endpoint.md`) or verify the filename convention with the team if `graphql` is a placeholder pattern used regardless of stack.

### C2
- file: stories/01-graphql.md
- section: ## Acceptance Criteria
- quote: **And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string present but possibly empty or null when `ok` is true, to maintain a consistent response shape.
- match: requirements.md › AC-1: Dependency checks
- problem: The story's AC-1 now says `error` is "present but possibly empty or null when `ok` is true", but the requirements say `error` is "a string present only when `ok` is false". These contradict each other. The developer following the story would include the field always, while the requirements say to omit it on success.
- change: Match the decision in C4/C5: state that all four fields are always present, with `error` null or empty when `ok` is true, in both documents.

### C3
- file: requirements.md
- section: ## Given/When/Then Acceptance Criteria
- quote: **And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string present only when `ok` is false.
- match: stories/01-graphql.md › AC-1
- problem: Same as C2. The requirements still say `error` is "present only when `ok` is false" despite the decision in C5 that all four fields are always present. The developer would see conflicting instructions.
- change: Apply the C5 decision: replace "present only when `ok` is false" with "present in every response, null or empty when `ok` is true" to match the ticket and the story assumption.

## Notes for the gate

- The `READY_CHECK_TIMEOUT` environment variable, timeout behavior, parallel execution, secrets sanitization, and startup logging are all consistently specified across ticket, requirements, and story.
- The seven test scenarios in AC-7 are listed identically in ticket, requirements, and story, and the story's Tests section plans all seven.
- File paths (`runtime/checkpointer.py`, `queue/subtasks.py`, `model/gateway.py`, `tools/mcp_client.py`, `src/agents/runtime/server.py`) match the ticket throughout and are stated to be verified against the repo tree in the story notes.
- The decision in C4/C5 to always include all four fields is sound and should be applied consistently once the contradictions above are resolved.

VERDICT: FAIL
