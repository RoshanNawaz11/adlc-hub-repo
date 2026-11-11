## What would go wrong

### C1
- file: stories/01-agent.md
- section: ## Interfaces
- quote: `validate_rejection_reason(self.reason)`
- match: stories/01-agent.md › ## Algorithm Notes › ### CLI Entry Point (REQ-3)
- severity: blocking
- blocks because: The pseudo-code shows calling `validate_rejection_reason(self.reason)` and assigning its return to `self.reason`, but the interface description says "Let `ValueError` propagate", so Dev would write the validator without the assignment and the trimmed reason would never be saved.
- problem: The interface says "Let `ValueError` propagate" and "Store the trimmed return value in `self.reason`", but the pseudo-code in Algorithm Notes shows `self.reason = validate_rejection_reason(self.reason)` with no comment about propagation. The two readings disagree on whether the validator both assigns and propagates, or only assigns.
- change: In the pseudo-code, add a comment after the assignment: `# ValueError propagates; Pydantic converts to 422` so Dev knows the validator does both.

### C2
- file: stories/01-agent.md
- section: ## Algorithm Notes
- quote: `reason.strip()`
- match: repo-file/agent/src/agents/runtime/server.py › ReviewDecision._reason_on_reject
- severity: blocking
- blocks because: The current validator uses `not self.reason` to check for empty, which treats `None` and `""` the same, but the new validation function spec says to check `if reason is None` and `if reason == ""` as separate steps before calling `strip()`. Dev following the spec would write three separate checks when the existing code shows one check is enough.
- problem: The validation function spec lists three steps: check `None`, check `""`, check `strip() == ""`. The existing code does `if self.action == "reject" and not self.reason`, which covers `None` and `""` in one check. The spec would lead Dev to write more checks than needed.
- change: Simplify the validation function spec to match Python idiom: "1. If `reason is None or reason == ""` → invalid. 2. If `reason.strip() == ""` → invalid. 3. Else valid → return `reason.strip()`." The function signature already shows `reason: str | None`, so the two-step check is clear.

### C3
- file: stories/01-agent.md
- section: ## File Plan
- quote: `src/agents/graph/validation.py`
- match: repo-tree/agent
- severity: blocking
- blocks because: The file plan introduces a new module `src/agents/graph/validation.py` that does not exist in the repo tree, and no document explains why this validation belongs in the `graph` package rather than in `runtime` where `server.py` lives or in a shared utilities module. Dev would create the file where the plan says, but the choice of location is not justified and may be wrong.
- problem: The repo tree shows no `src/agents/graph/validation.py`, and the graph package (from repo-outline) contains state, edge routing and interrupt handling, not data validation. The runtime package holds `server.py` (one of the three entry points) and `runner.py` (which calls the graph). Placing validation in `graph` when two of the three users are in other packages is unexplained.
- change: Move `validation.py` to `src/agents/runtime/validation.py` in the file plan and interfaces section, because two of the three files that import it (`server.py`, `orders.py`) are in `runtime`, and `interrupts.py` can import across packages. Update the import statements in the pseudo-code to `from agents.runtime.validation import ...`.

VERDICT: FAIL
