# PER-128 — Requirements

**Type:** story  
**Epic:** (none — standalone feature)  
**Covers:** (direct from ticket)

---

## Overview

Add usage telemetry aggregated per run and per node, and skip orchestrator model calls when the graph has only one valid route, to reduce token spend on deterministic decisions.

---

## Requirements

### R1: Token usage accumulation per run

**Given** a run is in progress  
**When** any node makes a model call (via `run_skill`, MCP tool loop in `bindings.py`, or `ModelGateway.complete`)  
**Then** the call's `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens` are recorded to `State.usage[node_name]`, accumulating counts across all calls that node makes.

**Given** a run saved before `State.usage` existed  
**When** the run is loaded and resumed  
**Then** the run continues without error, and `State.usage` initializes as empty for that run.

**Given** a run has ended or paused  
**When** `run_query` completes  
**Then** the total token counts (summed across all nodes) are logged, and `GET /runs/{thread_id}` returns them in the response `values`.

---

### R2: Forced route selection

**Given** the orchestrator is deciding the next route  
**When** `forced_route(state)` is called  
**Then** it returns a single-element route list if exactly one candidate route passes `broken_rules`, otherwise `None`.

**Given** `forced_route` needs to determine valid candidates  
**When** evaluating possible routes  
**Then** candidates include each node individually (from `NODES` list in `src/agents/nodes/__init__.py`, or derived from `src/agents/nodes/` directory structure: `planner`, `compliance`, `payroll_calculation`, `validation`, `execution`, `output_validator`, `orchestrator`), plus every non-empty set of stage agents (`compliance`, `payroll_calculation`, `validation`), and `broken_rules` (in `src/agents/nodes/orchestrator/route_rules.py`) remains the sole source of routing validity logic (no duplication in `forced_route`).

---

### R3: Orchestrator behavior on forced routes

**Given** `forced_route(state)` returns a route and the orchestrator has no model work to do  
**When** the orchestrator decides the next step  
**Then** it skips the model call and uses the forced route directly, recording a step with title `Next: <route>`, result `"forced: only one valid route"`, and `attempts: 0` in evidence.

**Given** `forced_route(state)` returns a route but the model still has work  
**When** pause conditions are detected (via `interrupt_before` in `src/agents/graph/interrupts.py`—no separate pause route nodes exist per `src/agents/nodes/` directory tree), OR an unresolved flag exists whose ID is not in `orchestrator_seen_flag_ids`  
**Then** the model call proceeds as usual, and the route is not forced.

**Given** the orchestrator receives control with flags present in state  
**When** it processes that hop  
**Then** the flag IDs from the state's flag field are recorded in `State.orchestrator_seen_flag_ids` (set of strings), to prevent duplicate model work on the same flags.

**Given** a forced hop completes  
**When** the orchestrator updates state  
**Then** `status`, `flagged`, `priority`, and `confidence` are updated exactly as a model hop would, and `summary` remains unchanged.

---

## Acceptance Criteria

### AC1: Forced route logic (6 tests)

**Given** the planner has not yet run  
**When** `forced_route(state)` is called  
**Then** it returns `["planner"]`.

**Given** execution has finished  
**When** `forced_route(state)` is called  
**Then** it returns `["output_validator"]` (node name format verified from `src/agents/nodes/output_validator/`).

**Given** the graph has reached terminal state  
**When** `forced_route(state)` is called  
**Then** it returns `["__end__"]` (LangGraph's terminal sentinel, verified from `src/agents/graph/build.py` edge logic).

**Given** two stage agents are unfinished and no flags or pauses are waiting  
**When** `forced_route(state)` is called  
**Then** it returns `None` (multiple valid candidates).

**Given** only a parallel set of stage agents is valid (e.g., compliance and validation together, both individually rejected by `broken_rules`)  
**When** `forced_route(state)` is called  
**Then** it returns the parallel set (e.g., `["compliance", "validation"]`) if `broken_rules` enforces parallel execution, OR returns `None` if `broken_rules` never requires parallel execution.

**Given** all candidates are rejected by `broken_rules` (invalid state)  
**When** `forced_route(state)` is called  
**Then** it returns `None`.

---

### AC2: Orchestrator model call skipping (6 tests)

**Given** `forced_route(state)` returns a route and no model work is required (not a pause condition, and no unseen flags)  
**When** the orchestrator processes the hop  
**Then** no model call is made, and a step is recorded with `result: "forced: only one valid route"` and `attempts: 0`.

**Given** `forced_route(state)` returns a route but pause conditions are detected (via `interrupt_before` in `src/agents/graph/interrupts.py`)  
**When** the orchestrator processes the hop  
**Then** the model is called to generate the reviewer's `summary` before pausing.

**Given** `forced_route(state)` returns a route but an unresolved flag exists whose ID is not in `State.orchestrator_seen_flag_ids`  
**When** the orchestrator decides the next route  
**Then** the model is called to handle flag merging.

**Given** the orchestrator receives control with flags in state  
**When** the hop is processed  
**Then** the flag IDs are extracted and added to `State.orchestrator_seen_flag_ids` before any decision logic runs.

**Given** a forced hop completes  
**When** the step is written  
**Then** the step's title is `Next: <route>`, and `status`, `priority`, `confidence`, and `flagged` match what a model hop would produce for the same route, and `summary` is unchanged.

**Given** a forced hop completes  
**When** the step is recorded  
**Then** the step's evidence includes `attempts: 0` and any other evidence fields that a normal model hop would include.

---

### AC3: Usage tracking and aggregation (7 tests)

**Given** `State.usage` reducer receives updates for the same node across multiple calls  
**When** the reducer processes them  
**Then** counts for `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens` sum into existing counts for that node, and missing keys initialize to zero.

**Given** `State.orchestrator_seen_flag_ids` reducer receives sets of flag IDs  
**When** the reducer processes them  
**Then** incoming sets union into the existing set.

**Given** a checkpoint saved before `State.usage` and `State.orchestrator_seen_flag_ids` existed  
**When** the checkpoint is loaded  
**Then** the run resumes without error, and both fields initialize as empty (empty dict and empty set respectively).

**Given** a checkpoint with existing `State.usage` and `State.orchestrator_seen_flag_ids`  
**When** the checkpoint is loaded  
**Then** existing counts and seen flag IDs are preserved.

**Given** a node makes multiple model calls including JSON retries (in `run_skill` from `src/agents/skills/client.py`) and MCP tool rounds (in `src/agents/tools/bindings.py`)  
**When** usage is aggregated  
**Then** `State.usage[node_name]` contains the sum of all calls' counts.

**Given** a run ends (next step is `["__end__"]`) or pauses (detected by `next` being empty and `status != "complete"`, or by `interrupted` flag, per `src/agents/runtime/runner.py` logic)  
**When** `run_query` completes  
**Then** it logs total token counts summed across all nodes.

**Given** a client requests run state via `GET /runs/{thread_id}` (endpoint in `src/agents/runtime/backend.py` or `src/agents/runtime/server.py`)  
**When** the response is returned  
**Then** `values.usage` includes per-node counts matching `State.usage`.

---

### AC4: Test coverage

**Given** the new logic is implemented  
**When** tests are run  
**Then** all new tests covering AC1 (6 tests), AC2 (6 tests), and AC3 (7 tests) pass with a fake gateway (no live services), existing tests in `tests/test_backend.py` and `tests/test_evaluate.py` pass unchanged, and `uv run pytest -q` completes successfully.

---

### AC5: PR evidence

**Given** the PR is opened  
**When** the description is reviewed  
**Then** it includes before-and-after counts of model calls and total tokens for one service order (e.g., `run_so.py` with SO-1003 or equivalent), showing measurable savings.

---

## Out of Scope

- Changing models, `policy.yaml` limits, skill or prompt content
- Shrinking `_brief`, or removing the skill container and code-execution tool from orchestrator calls
- The MCP server, GraphQL service, or fastapi-backend
- Graph routing rules themselves (`broken_rules` behavior in `src/agents/nodes/orchestrator/route_rules.py` remains unchanged)

---

## Verification

Run the new test suite (19 new tests total: 6 for AC1, 6 for AC2, 7 for AC3) with a fake model gateway and no live services. Verify all tests pass, existing tests in `tests/test_backend.py` and `tests/test_evaluate.py` remain unchanged, and `uv run pytest -q` passes. Confirm the PR description includes measured token savings from a representative service order.

---

## Notes

- `State.usage` schema: `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}` with a reducer that sums new counts into existing ones (Story 1).
- `State.orchestrator_seen_flag_ids` schema: `set[str]` with a union reducer (Story 1).
- `forced_route` must not duplicate routing logic; it delegates all validity checks to `broken_rules` (in `src/agents/nodes/orchestrator/route_rules.py`). Story 2 verifies the exact signature and return type of `broken_rules` before implementing `forced_route`.
- Track which flags the orchestrator has seen in `State.orchestrator_seen_flag_ids` to avoid redundant model calls for flag merging (Story 3).
- Pause detection: No separate pause route nodes exist per `src/agents/nodes/` directory. Pause logic is handled via `interrupt_before` in `src/agents/graph/interrupts.py`. Story 3 verifies the actual pause detection mechanism before implementing Guard 1 in the orchestrator.
- Terminal node: `["__end__"]` is LangGraph's terminal sentinel (not a real node directory). Story 2 includes it in the enumerated candidates for `forced_route` if `broken_rules` accepts sentinel values.
- **All calling nodes of `run_skill` must be identified and updated** when `run_skill`'s return type changes to `tuple[dict, dict]` in Story 4. This includes at minimum: `planner`, `orchestrator`, and any stage agents that call skills. The list is verified by searching `src/agents/nodes/` for `run_skill` calls before implementing Story 4.
- **Story verification responsibilities (no cross-story verification assumptions):**
  - **Story 1** verifies and documents actual flag field name and dict structure in `src/agents/graph/states.py` (may be `flags`, `open_flags`, or other; flag dicts may have `id`/`flag_id` and `resolved`/`is_resolved` or other keys). Produces `verification-findings.md` with concrete schema.
  - **Story 2** verifies exact signature and return type of `broken_rules` in `src/agents/nodes/orchestrator/route_rules.py` (what it accepts for `next_step`, what it returns). Verifies actual node names from `src/agents/nodes/` directory (planner, compliance, payroll_calculation, validation, execution, output_validator, orchestrator). Verifies whether `NODES` list exists in `src/agents/nodes/__init__.py`, and derives it from directory names if not. Verifies correct node name format (underscores vs hyphens: `output_validator` not `output-validator`). Verifies terminal sentinel handling from `src/agents/graph/build.py`. Verifies pause mechanism from `src/agents/graph/interrupts.py`. Produces `verification-findings.md` with concrete findings.
  - **Story 3** reads Story 1's `verification-findings.md` for flag schema, reads Story 2's `verification-findings.md` for pause mechanism. Verifies orchestrator function structure in `src/agents/nodes/orchestrator/node.py` (main function name, model decision function signature, step structure, state field update pattern). Produces `verification-findings.md` with concrete findings.
  - **Story 4** verifies current signature and return type of `run_skill` in `src/agents/skills/client.py` (sync vs async, current return type). Verifies all nodes that call `run_skill` (searches `src/agents/nodes/` for imports/calls). Verifies response object structure from `ModelGateway.complete` in `src/agents/model/gateway.py` (how to access `usage` metadata: `response.usage.input_tokens` or other). Produces `verification-findings.md` listing all calling nodes.
  - **Story 5** verifies Story 4's implementations do NOT pass `node_name` to `ModelGateway.complete`. Verifies current signature of `ModelGateway.complete` method. Produces `verification-findings.md` confirming non-interference.
  - **Story 6** verifies run end/pause detection in `src/agents/runtime/runner.py` (how `run_query` identifies completion vs pause: `next == ["__end__"]`, `interrupted` flag, or other). Verifies API endpoint location (`src/agents/runtime/backend.py` or `server.py`) and response structure (whether `values` includes all state fields or is filtered). Produces `verification-findings.md` with concrete findings.

---

**Dependencies:** None  
**Blocks:** None
