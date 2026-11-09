# Requirements — PER-119

**Ticket:** Skip the orchestrator's model call on hops with only one possible route, and record token usage per run

**Epic:** _(no epic)_

**Covers:** _(ticket is the source)_

---

## Summary

The orchestrator currently calls the model on every hop, even when only one route passes validation. Many hops have only one valid route (e.g., before the planner runs, after a review, once execution finishes). The model is paid to give an answer the code already knows.

This work adds forced routing to skip model calls when only one route is valid, unless the model still has work to do (writing a reviewer summary, merging duplicate flags). It also adds usage tracking per node so savings can be measured.

This is a performance optimization and observability improvement. The orchestrator, route rules, state schema, usage collection (across `run_skill`, MCP tool loop, `ModelGateway`), and runner all change. The state changes need backward compatibility. The routing logic must delegate to existing rule validation. Usage must aggregate correctly across retries and tool rounds.

---

## Requirements

### REQ-1: Forced routing (`route_rules.py`)

**Intent:** Skip model calls when only one route passes validation.

**Acceptance Criteria:**

**AC-1.1:** Given the orchestrator is choosing a route, when exactly one candidate route passes `broken_rules`, then `forced_route(state)` returns that route as `list[str]`

**AC-1.2:** Given the orchestrator is choosing a route, when two or more candidate routes pass `broken_rules`, then `forced_route(state)` returns `None`

**AC-1.3:** Given `forced_route` is selecting a route, when evaluating candidates, then candidates are: each node in `NODES` on its own, plus every non-empty set of stage agents

**AC-1.4:** Given `forced_route` is validating candidates, when checking if a route is valid, then `broken_rules` is the only source of routing rules (no duplication of rule logic)

**Success signal:**  
New tests (5 total) show `forced_route` returns `["planner"]` before the planner has run, `["__end__"]` after a review, `["output-validator"]` once execution has finished, `None` when two candidates are valid, and delegates all validation to `broken_rules`.

---

### REQ-2: Usage tracking per node (`State`, `run_skill`, `bindings.py`, `ModelGateway`, `runner.py`)

**Intent:** Track token usage per node across all model calls so savings can be measured.

**Acceptance Criteria:**

**AC-2.1:** Given a model call completes, when the call is from `run_skill`, the MCP tool loop in `bindings.py`, or `ModelGateway.complete`, then the call records `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens`

**AC-2.2:** Given `State` is defined, when tracking usage, then `State` has a `usage` field: `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`

**AC-2.3:** Given a node makes model calls, when the node updates state, then the update carries the usage of the calls it made, and the reducer adds the node's new counts to its existing ones

**AC-2.4:** Given a run was saved before this change, when the run is loaded, then the run loads and resumes successfully (backward compatibility: missing `usage` field does not break loading)

**AC-2.5:** Given a run ends or pauses, when `run_query` in `src/agents/runtime/runner.py` completes, then the run's total usage (summed across all nodes) is logged

**AC-2.6:** Given a client calls `GET /runs/{thread_id}`, when the run has usage data, then the response includes usage totals in `values`

**Success signal:**  
New tests (12 total across stories 1, 4, 5) show `usage` accumulates per node across calls (including JSON retries in `run_skill` and every MCP tool round in `bindings.py`), a run saved without `usage` still resumes, totals are logged when runs end or pause, and the API returns totals. PR description shows model calls and total tokens for one service order before and after the change.

---

### REQ-3: Orchestrator skip logic (`node.py`, `State`)

**Intent:** Skip `_decide` when `forced_route` returns a route, unless the model still has work to do.

**Acceptance Criteria:**

**AC-3.1:** Given the orchestrator is choosing a route, when `forced_route` returns a route and the model has no work to do, then the orchestrator skips `_decide` and uses that route

**AC-3.2:** Given the orchestrator is choosing a route, when `forced_route` returns a route and the route pauses for a person (`flag-review` or `review`), then the orchestrator calls `_decide` (the model writes the reviewer's `summary` before a pause)

**AC-3.3:** Given the orchestrator is choosing a route, when `forced_route` returns a route and an unresolved flag exists that the orchestrator's model has not seen yet, then the orchestrator calls `_decide` (merging duplicate flags needs the model)

**AC-3.4:** Given the orchestrator is tracking which flags it has seen, when the orchestrator processes a hop (model hop or forced hop), then the orchestrator updates a new state field with the flag IDs from `state.flags` (both paths track seen flags identically)

**AC-3.5:** Given a forced hop completes, when writing the step, then the step has title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`

**AC-3.6:** Given a forced hop completes, when updating state, then the hop updates `status`, `flagged`, `priority`, and `confidence` exactly as a model hop does, and leaves `summary` unchanged

**Success signal:**  
New tests (6 total) show the orchestrator makes no model call on a forced hop when the model has no work, calls the model when the forced route is `flag-review` or `review`, calls the model when a flag it hasn't seen is open, and a forced hop's step/status/priority/confidence match what a model hop would write for the same route. Tests verify seen flags are tracked on both model hop and forced hop paths.

---

## Constraints

- **Existing routing rules unchanged:** `broken_rules` behaves as today; routing logic itself does not change.
- **Backward compatibility:** A run saved before this change (with no `usage` or `orchestrator_seen_flags` fields) must load and resume.
- **Single source of truth:** `forced_route` must not duplicate rules from `broken_rules`; it delegates to `broken_rules` for validation.
- **No model or prompt changes:** Models, `policy.yaml` limits, skill content, and prompt text unchanged.
- **No orchestrator brief changes:** `_brief` size, skill container, and code-execution tool unchanged.
- **Existing tests pass:** `uv run pytest -q` passes; existing tests pass unchanged.

---

## Non-Goals

- Changing models, `policy.yaml` limits, skill or prompt content.
- Shrinking `_brief`, or removing the skill container and code-execution tool from orchestrator calls.
- The MCP server, the GraphQL service, or fastapi-backend.
- Graph routing rules themselves: `broken_rules` behaves as today.

---

## Success Signal

- New tests cover: `forced_route` behavior (5 tests in story 2), orchestrator skip logic (6 tests in story 3), usage schema and reducers (4 tests in story 1), usage collection at call sites (5 tests in story 4), usage logging and API (3 tests in story 5). Total: 23 tests.
- Existing tests pass unchanged; `uv run pytest -q` passes.
- PR description reports model calls and total tokens for one service order before and after the change.

---

## Open Questions

_(none at this time)_

---

## Notes

- **Design review required:** This work changes state schema, orchestrator logic, and usage collection across multiple call sites. Design review (HLD) must ensure state changes, orchestrator logic, and usage aggregation agree before coding.
- **Scope is minimal:** Three focused requirements (forced routing, usage tracking, orchestrator skip conditions); no framework/language/ORM/cloud stack choices (product outcomes only).
- **Test coverage per AC:** Each AC maps to specific test cases (5 tests for REQ-1, 12 tests for REQ-2 across stories 1/4/5, 6 tests for REQ-3). Total: 23 tests matching breakdown and story Tests sections exactly.
- **AC numbering added:** Each AC is now numbered (AC-1.1 through AC-3.6) for precise traceability to tests and stories.
- **Seen flags tracking clarified:** AC-3.4 now specifies that seen flags are tracked on both model hop and forced hop paths (matching HLD AD-5 clarification and Story 3 implementation). The wording "when the orchestrator's model receives a brief" in the original was too narrow; it has been replaced with "when the orchestrator processes a hop (model hop or forced hop)" to cover both paths.
