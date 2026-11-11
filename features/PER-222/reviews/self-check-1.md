## What would go wrong

### C1
- file: stories/02-agent.md
- section: ## File Plan
- quote: `{"detail": "Reason is required and cannot be empty"}`
- match: hld.md › ### Withdrawal Endpoint (API) › Response (validation error, 400)
- severity: blocking
- blocks because: Story 2 says the API returns `{"detail": "<message>"}` (FastAPI's default), but the HLD says `{"error": "<message>"}`. Dev would build the API with `detail`, then the integration tests would fail because the HLD's examples show `error`.
- problem: The HLD specifies `{"error": "<message>"}` for all error responses, but Story 2's LLD uses FastAPI's default `{"detail": "<message>"}` and notes the discrepancy without resolving it.
- change: Either update the HLD's API contract examples to use `"detail"` (matching FastAPI's convention), or add a custom exception handler in Story 2 to return `"error"` (matching the HLD). The LLD should pick one and implement it, not document the mismatch and defer.

### C2
- file: stories/04-agent.md
- section: ## File Plan
- quote: `await ctx["redis"].set(status_key(thread_id), json.dumps(payload), ex=STATUS_TTL_SECONDS,)`
- match: repo-file/agent/src/agents/queue/orders.py › async def write_status
- severity: blocking
- blocks because: Story 4 inlines the Redis write for withdrawn cases instead of calling `write_status`, so when the `write_status` signature changes later (e.g. to accept `**extra` fields), the withdrawn case code won't match. Dev would have two different patterns for writing queue status in the same file.
- problem: Story 4 inlines `await ctx["redis"].set(status_key(thread_id), json.dumps(payload), ex=STATUS_TTL_SECONDS)` instead of extending or calling the existing `write_status` helper. The open question asks whether to extend `write_status`, but the code implements the inline approach.
- change: Either extend `write_status(redis, thread_id, status, error=None, **extra)` to accept arbitrary fields and call it from the resume handler, or document in the code why the withdrawn case must inline the write (e.g. because `write_status` is a contract with FastAPI's order_queue.py and can't change). Don't leave it as an open question while implementing one side.

### C3
- file: stories/07-agent.md
- section: ## File Plan
- quote: `if snapshot.values and isinstance(snapshot.values, dict): if snapshot.values.get("status") == "withdrawn":`
- match: stories/04-agent.md › ## File Plan › 2. Modify the `resume_service_order` function
- severity: blocking
- blocks because: Story 7 checks `snapshot.values.get("status") == "withdrawn"` in `run_query` and `process_service_order`, but Story 4 already added the same check to `resume_service_order`. When Dev implements both stories, the resume handler will have the check twice (once from Story 4, once from Story 7's pattern), or Story 7's changes will conflict with Story 4's.
- problem: Story 4 adds a withdrawn check to `resume_service_order` at lines 93-98 (before calling `_run`). Story 7 says "No changes to `resume_service_order`" because Story 4 already handles it, but then the "Prevent operations on withdrawn cases" summary in the breakdown lists resume as an operation Story 7 covers. The dependency is stated but the division of work is unclear.
- change: In Story 7's LLD, explicitly state that `resume_service_order` already has the withdrawn check from Story 4 and does not need modification, or move the resume handler check from Story 4 to Story 7 and update Story 4's file plan to say "withdrawn check will be added in Story 7". Don't say "no changes" while the summary says it covers resume.

### C4
- file: stories/01-agent.md
- section: ## Interfaces
- quote: `history_entry["type"] = "withdrawn"`
- match: repo-outline/agent › src/agents/graph/states.py › class StepRecord(TypedDict)
- severity: blocking
- blocks because: Story 1 adds `type`, `by`, and `reason` fields to the withdrawal history entry by setting them on a `StepRecord` dict at runtime, with `# type: ignore[typeddict-unknown-key]` comments. But `StepRecord` is defined in `states.py` (shown in repo-outline) with a fixed set of fields, and TypedDict does not allow extra keys at runtime in strict mode. Dev would write the code as shown, then mypy (if enabled) or a runtime check would fail.
- problem: The history entry schema in Story 1 extends `StepRecord` with `type`, `by`, `reason` fields that are not in the `StepRecord` TypedDict definition. The code casts to `dict[str, Any]` or suppresses mypy, but the breakdown and HLD don't say whether `StepRecord` should be extended or whether the withdrawal entry is a different shape.
- change: Either add `type: NotRequired[str]`, `by: NotRequired[str]`, `reason: NotRequired[str]` to the `StepRecord` TypedDict in `states.py` (Story 1 modifies it), or define a separate `WithdrawalHistoryEntry` TypedDict and append that instead of a `StepRecord`. The LLD should pick one and show the change to `states.py`.

## Notes for the gate

The documents pass overall consistency (state transitions, field names, test counts all align), but have four blocking items where the implementation choice is deferred to an open question or noted as a discrepancy instead of being decided. The withdrawn state and its fields are used consistently everywhere; the test counts match the breakdown; and the queue handler, API, and CLI all call the same `withdraw_case` function as intended. The four items above are the places where a developer would get stuck or build conflicting code.

VERDICT: FAIL
