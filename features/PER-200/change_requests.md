# Open items from the critic for PER-200

Written by the critic, not approved by a person. Delete the items you disagree with and edit the rest; what is left is applied on the next run:
`adlc resume PER-200` (or `adlc start PER-200 --skip-dev`).

### C1 (open)
- file: stories/01-graphql.md
- section: (filename)
- quote: # Story — PER-200
- problem: The file is named `01-graphql.md` but the repository context states this is a Python repo (`app`), not a GraphQL service. The ticket references `src/agents/runtime/server.py` and Python modules throughout. A developer looking for this story would be confused by the graphql naming.
- change: Rename the file to match the story content (e.g., `01-readiness-check.md` or `01-ready-endpoint.md`) or verify the filename convention with the team if `graphql` is a placeholder pattern used regardless of stack.
- decided: The file name `01-graphql.md` is incorrect for a Python REST endpoint story. The story describes work on `src/agents/runtime/server.py` (a Python file), and the ticket references Python modules. Renaming the file is outside this fixer's scope (it only changes content), so I will add a note in the story header clarifying this is a Python agents service story, not a GraphQL story, and flag the filename mismatch.

### C2 (open)
- file: stories/01-graphql.md
- section: ## Acceptance Criteria
- quote: And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string present but possibly empty or null when `ok` is true, to maintain a consistent response shape.
- match: requirements.md › AC-1: Dependency checks
- problem: The story's AC-1 now says `error` is "present but possibly empty or null when `ok` is true", but the requirements say `error` is "a string present only when `ok` is false". These contradict each other. The developer following the story would include the field always, while the requirements say to omit it on success.
- change: Match the decision in C4/C5: state that all four fields are always present, with `error` null or empty when `ok` is true, in both documents.
- decided: The ticket specifies `{name, ok, ms, error}` without stating when the `error` field appears. To maintain a consistent response shape that client code can rely on (all four fields always present), `error` will be included in every check result, set to null or empty string when `ok` is true, rather than omitting the key entirely.

### C3 (open)
- file: requirements.md
- section: ## Given/When/Then Acceptance Criteria
- quote: And** each check returns `{name, ok, ms, error}`, where `name` is the dependency, `ok` is boolean, `ms` is the elapsed time in milliseconds, and `error` is a string present only when `ok` is false.
- match: stories/01-graphql.md › AC-1
- problem: Same as C2. The requirements still say `error` is "present only when `ok` is false" despite the decision in C5 that all four fields are always present. The developer would see conflicting instructions.
- change: Apply the C5 decision: replace "present only when `ok` is false" with "present in every response, null or empty when `ok` is true" to match the ticket and the story assumption.
