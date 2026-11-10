# HLD — PER-128: Forced Routing and Usage Telemetry

**Feature:** Skip orchestrator model calls on deterministic hops and track token usage per run  
**Epic:** (standalone)  
**Version:** 1.2  
**Status:** Draft  
**Altitude:** Story

---

## Overview

The orchestrator currently calls the model at every hop, even when only one route is valid. Many hops are deterministic: before the planner runs, only `["planner"]` is valid; after review completes, the graph reaches its terminal state; after execution finishes, only the output validation node is valid. The model is paid to answer what the code already knows.

This feature adds **forced routing** — when exactly one candidate route passes `broken_rules`, skip the model call and use that route — and **per-run usage telemetry** so savings can be measured. The orchestrator still calls the model when it has real work: writing a reviewer summary before pause, or merging flags it hasn't seen yet.

Three separable concerns are coordinated:

1. **Usage accumulation** — `State.usage` aggregates token counts per node, across all calls (skill, MCP tool loop, gateway direct).
2. **Forced route logic** — `forced_route(state)` returns the single valid route when deterministic, delegates validity to `broken_rules`.
3. **Orchestrator decision flow** — skip `_decide` on forced hops unless model work remains (pause summary, unseen flags).

Backward compatibility: runs saved before `State.usage` existed must load and resume without error.

**Repo tree context confirms:**

- Node directories exist: `compliance/`, `execution/`, `orchestrator/`, `output_validator/`, `payroll_calculation/`, `planner/`, `validation/`
- No `review/` or `flag-review/` directories exist
- `interrupts.py` exists, suggesting pause logic is interrupt-based
- Stage agents: compliance, payroll_calculation, validation

---

## Architectural Decisions

### AD-1: Usage schema and accumulation pattern

**Binds:** `State.usage` structure, reducer behavior, backward compatibility  
**Prevents:** divergent usage tracking per node, migration breaks on old runs  
**Rule:**

- `State.usage` is a top-level field in the `State` dataclass: `{node_name: {calls: int, input_tokens: int, output_tokens: int, cache_creation_input_tokens: int, cache_read_input_tokens: int}}`.
- The reducer is additive: new counts for a node sum into existing counts; missing keys initialize to zero.
- A run loaded without `usage` in its checkpoint initializes `usage` as an empty dict and continues.
- Every model call site (`run_skill`, MCP tool loop in `bindings.py`, `ModelGateway.complete`) extracts usage from the response and returns it in the node's state update dict (LangGraph pattern: nodes return `{"usage": {node_name: {...}}}`, which the reducer merges into state).

**Rationale:** A flat per-node dict is sufficient — no timestamp or per-call detail needed, only aggregates. Additive reducer ensures resilience to retries and multiple rounds. Empty-dict default handles missing field gracefully. Returning usage in the node's state update dict matches LangGraph's state update pattern (no callback needed).

---

### AD-2: Forced route interface and contract

**Binds:** `forced_route(state)` signature, candidate enumeration, delegation to `broken_rules`  
**Prevents:** routing logic duplication, forced routes that violate graph rules  
**Rule:**

- `forced_route(state) -> list[str] | None` lives in `route_rules.py`.
- Enumerate candidates: each node in `NODES` (from `__init__.py` or derived from `src/agents/nodes/` directory) individually, plus all 7 non-empty subsets of stage agents (compliance, payroll_calculation, validation).
- **Terminal node handling:** Include `["__end__"]` as a candidate if graph edges in `build.py` show it's used as a terminal target. If `broken_rules` rejects sentinels, exclude it and rely on `broken_rules` to enforce terminal logic through other means.
- For each candidate, call `broken_rules(state, candidate)`. Count candidates that `broken_rules` accepts as valid (what constitutes "valid" is determined by `broken_rules`' return type: empty list, falsy bool, etc.).
- If exactly one candidate is valid, return it. Otherwise return `None`.
- `forced_route` contains no routing validity logic itself — all rules stay in `broken_rules`.

**Rationale:** Single source of truth for routing rules (`broken_rules`). Enumerating all nodes plus all stage agent subsets is exhaustive; `broken_rules` filters to what's actually valid. No risk of missing a valid route. Must delegate to existing `broken_rules` contract, not invent a new one.

---

### AD-3: Orchestrator decision guards and flag tracking

**Binds:** when the orchestrator calls the model vs. uses forced route, state fields tracking model visibility of flags  
**Prevents:** skipping model calls that write pause summaries or merge unseen flags  
**Rule:**

- Add `State.orchestrator_seen_flag_ids: set[str]` as a top-level field (default empty set, reducer unions new IDs into existing set).
- When the orchestrator receives control, extract flag IDs from the state's flag field and add them to `orchestrator_seen_flag_ids` before any decision (return `{"orchestrator_seen_flag_ids": {flag_id_1, flag_id_2, ...}}` in the state update).
- Call `forced_route(state)`. If it returns `None`, proceed with `_decide` as today.
- If it returns a route:
  - **Guard 1 (pause detection):** Check if pause is triggered. Pause detection mechanism (interrupt-based vs routing-based) determines the check:
    - If pause is triggered via `interrupt_before` in `interrupts.py`: check the state condition that triggers interrupts (e.g., `state.flagged`, `state.needs_review`, or other field).
    - If pause routes exist as separate nodes: check if `route` matches a pause route node name.
    - If pause is triggered, call `_decide` (model must write summary).
  - **Guard 2 (unseen flags):** Check for unresolved flags whose IDs are not in `orchestrator_seen_flag_ids`. If any exist, call `_decide` (model must merge/deduplicate).
  - **Otherwise (no model work):** skip `_decide`, use the forced route, write a step with `title: "Next: {route}"`, `result: "forced: only one valid route"`, `attempts: 0` in evidence. Update `status`, `flagged`, `priority`, `confidence` as a model hop would (reuse existing logic or compute deterministically); leave `summary` unchanged. Return the state update dict.

**Rationale:** The orchestrator's model does two things: decide the route, and write text (summaries, flag merges). Forced routing skips only the decision when deterministic; text generation still requires the model. Tracking seen flags prevents redundant merges. Updating `status`/`priority`/`confidence` on forced hops keeps state consistent with model hops (same downstream logic).

---

### AD-4: Usage collection integration points

**Binds:** where usage data is extracted and returned in state updates, logging and API return  
**Prevents:** missed calls, partial aggregation  
**Rule:**

- **`run_skill` in `src/agents/skills/client.py`:** After each `ModelGateway.complete` call (including JSON retry calls), extract usage from the response and return it in the modified return value `(result, usage_data)`. All calling nodes must be updated to unpack the tuple and include usage in their state updates.
- **MCP tool loop in `src/agents/tools/bindings.py`:** After each round of tool calls (including the final no-tools call), extract usage and include it in the node's returned state update dict.
- **`ModelGateway.complete` in `src/agents/model/gateway.py`:** When called directly by a node (not via `run_skill` or MCP loop), the calling node extracts usage from the response and includes it in its returned state update dict. If `node_name` is needed, the caller provides it. **`run_skill` and MCP tool loop must NOT pass `node_name`** — they extract usage themselves to avoid double-dispatch.
- **`run_query` in `src/agents/runtime/runner.py`:** When a run ends (next step is terminal state marker) or pauses (detected by graph state: `next` empty with `status != "complete"`, or `interrupted` flag), sum `State.usage` across all nodes and log totals.
- **`GET /runs/{thread_id}` in `src/agents/runtime/backend.py` or `server.py`:** Return `State.usage` in the response `values` dict, keyed by node name. If `values` is filtered, ensure `usage` is not excluded.

**Rationale:** Coverage at every model call site ensures complete aggregation. Returning usage in state update dicts matches LangGraph's pattern (no callback needed). Logging totals on pause/end makes savings visible. API return exposes usage to clients.

---

## System Components

### Modified Components

#### `src/agents/graph/states.py`

**Changes:**

- Add `usage: dict[str, dict[str, int]]` as a top-level field in the `State` dataclass (node → counts dict).
- Add `orchestrator_seen_flag_ids: set[str]` as a top-level field.
- Define reducers:
  - `usage`: for each node in the incoming update, sum `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` into existing counts (initialize missing keys to zero).
  - `orchestrator_seen_flag_ids`: union incoming set into existing set.
- Handle missing `usage` and `orchestrator_seen_flag_ids` fields on checkpoint load: if absent, initialize as empty dict and empty set respectively (migration logic may be in `states.py` or `src/agents/runtime/checkpointer.py`).

**Contracts:**

- `State.usage` schema: `{node_name: {calls: int, input_tokens: int, output_tokens: int, cache_creation_input_tokens: int, cache_read_input_tokens: int}}`.
- `State.orchestrator_seen_flag_ids` schema: `set[str]` (flag IDs).

---

#### `src/agents/nodes/orchestrator/route_rules.py`

**Changes:**

- Add `forced_route(state: State) -> list[str] | None`.
- Enumerate candidates: each node in `NODES` (from `__init__.py` or derived from `src/agents/nodes/` directory) individually, plus all 7 non-empty subsets of stage agents (compliance, payroll_calculation, validation).
- Include `["__end__"]` as a candidate if terminal edges in `build.py` use it; exclude if `broken_rules` only validates runnable nodes.
- For each candidate, call `broken_rules(state, candidate)`. Count valid candidates based on `broken_rules`' return type.
- If exactly one candidate is valid, return it. Otherwise return `None`.

**Contracts:**

- `forced_route(state)` returns `list[str]` (single valid route) or `None` (zero or multiple valid routes).
- No routing logic duplication: all validity checks delegate to `broken_rules`.

---

#### `src/agents/nodes/orchestrator/node.py`

**Changes:**

- Before calling `_decide`, extract flag IDs from the state's flag field and include them in the state update: `{"orchestrator_seen_flag_ids": {flag_id_1, flag_id_2, ...}}`.
- Call `forced_route(state)`. If `None`, proceed with `_decide` as today.
- If a route is returned, check model work guards:
  - **Guard 1 (pause):** Check if pause is triggered (via state condition for interrupt-based pause, or route match for routing-based pause). If yes, call `_decide`.
  - **Guard 2 (unseen flags):** Check if any unresolved flag's ID is not in `orchestrator_seen_flag_ids`. If yes, call `_decide`.
  - **Otherwise:** skip `_decide`, write forced hop step, update state fields, return state update dict.
- Forced hop step: `title: "Next: {route}"`, `result: "forced: only one valid route"`, `evidence: {attempts: 0, ...}`.
- Update `status`, `flagged`, `priority`, `confidence` as a model hop would; leave `summary` unchanged.

**Contracts:**

- Forced hops produce steps with `attempts: 0` and `result: "forced: only one valid route"`.
- Forced hops update `status`, `priority`, `confidence`, `flagged` consistently with model hops.
- Forced hops leave `summary` unchanged.

---

#### `src/agents/skills/client.py`

**Changes:**

- After each `ModelGateway.complete` call in `run_skill` (including JSON retry calls), extract `usage` from the response.
- Modify return type to `tuple[dict, dict]` where the second element is `usage_data: {calls: int, input_tokens: int, ...}`.
- All calling nodes (identified by searching `src/agents/nodes/` for `run_skill` calls) must be updated to unpack `(result, usage_data) = run_skill(...)` and include `{"usage": {node_name: usage_data}}` in their state updates.

**Contracts:**

- `run_skill` includes usage in its returned value for every model call (including retries).
- Return type change is coordinated with all calling nodes (breaking change — requires updating every caller).

---

#### `src/agents/tools/bindings.py`

**Changes:**

- In the MCP tool loop, after each round of tool calls (including the final no-tools call), extract `usage` from the response.
- Include usage in the node's returned state update dict with the calling node's name and the usage counts.

**Contracts:**

- MCP tool loop includes usage in the returned state update per round, including the final model call.

---

#### `src/agents/model/gateway.py`

**Changes:**

- Add optional `node_name: str | None = None` parameter to `complete` method.
- When `node_name` is provided, return `(response, usage_data)` tuple where `usage_data` is `{node_name: {calls: 1, input_tokens: ..., ...}}`.
- When `node_name` is `None`, return only `response` (backward compatible).
- **`run_skill` and MCP tool loop must NOT pass `node_name`** — they extract usage themselves to avoid double-dispatch.

**Contracts:**

- `ModelGateway.complete` logs usage per call (already implemented, preserve).
- Direct callers (bypassing `run_skill`/MCP loop) can opt into usage return by passing `node_name`.

---

#### `src/agents/runtime/runner.py`

**Changes:**

- In the graph execution function, after a run ends or pauses, sum `State.usage` across all nodes.
- Detect run end: next step is terminal state marker (determined from `build.py` edges).
- Detect run pause: `result["next"]` is empty and `state.get("status") != "complete"`, or `result.get("interrupted") == True`.
- Log totals: `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`.

**Contracts:**

- The graph execution function logs aggregated usage totals on run completion or pause.

---

#### `src/agents/runtime/backend.py` or `server.py`

**Changes:**

- Ensure `GET /runs/{thread_id}` endpoint includes `State.usage` in the returned `values` dict, keyed by node name.
- If `values` is filtered, add `usage` to the allowlist.

**Contracts:**

- API returns `usage` as `{node_name: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`.

---

### New Components

None. All changes are modifications to existing modules.

---

## Data Flow

### Usage Accumulation Flow

1. A node makes a model call (via `run_skill`, MCP tool loop, or `ModelGateway.complete` directly).
2. The call site extracts `usage` from the response: `{input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}`.
3. The node includes usage in its returned state update dict: `{"usage": {node_name: {calls: 1, input_tokens: <val>, ...}}}`.
4. LangGraph merges the state update into `State` using the `usage` reducer, which sums the new counts into `State.usage[node_name]` (initialize missing keys to zero).
5. When the run ends or pauses, the graph execution function sums `State.usage` across all nodes and logs totals.
6. `GET /runs/{thread_id}` returns `State.usage` in the response.

---

### Forced Routing Flow

1. The orchestrator receives control (start or after an agent/fetch/review).
2. Extract flag IDs from the state's flag field and return them in the state update: `{"orchestrator_seen_flag_ids": {flag_id_1, flag_id_2, ...}}`.
3. Call `forced_route(state)`:
   - Enumerate candidates (each node, plus all 7 non-empty subsets of stage agents).
   - For each candidate, call `broken_rules(state, candidate)`.
   - If exactly one candidate is valid, return it; otherwise return `None`.
4. If `forced_route` returns `None`, call `_decide` (existing behavior).
5. If `forced_route` returns a route:
   - Check model work guards (pause detection, unseen flags).
   - If model work required, call `_decide`.
   - Otherwise, skip `_decide`, use the forced route, write a forced hop step, update state fields, return the state update dict.

---

## API Contracts

### `forced_route(state: State) -> list[str] | None`

**Module:** `src/agents/nodes/orchestrator/route_rules.py`

**Input:**

- `state: State` — current graph state.

**Output:**

- `list[str]` — single valid route (e.g., `["planner"]`, `["__end__"]`, `["compliance", "validation"]`), or
- `None` — zero or multiple valid routes.

**Behavior:**

- Enumerate candidates: each node (from `NODES` or `src/agents/nodes/` directory), plus all 7 non-empty subsets of stage agents (compliance, payroll_calculation, validation).
- For each candidate, call `broken_rules(state, candidate)`.
- Return the candidate if exactly one is valid; otherwise return `None`.

**Contract:**

- No routing logic duplication — all validity checks delegate to `broken_rules`.
- Exhaustive candidate enumeration ensures no valid route is missed.

---

### `State.usage` Schema

**Type:** `dict[str, dict[str, int]]`

**Structure:**

```python
{
  "node_name": {
    "calls": int,  # Total model calls made by this node
    "input_tokens": int,
    "output_tokens": int,
    "cache_creation_input_tokens": int,
    "cache_read_input_tokens": int
  },
  ...
}
```

**Reducer:**

- For each node in the incoming update, sum `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` into `State.usage[node]` (initialize missing keys to zero).

**Backward Compatibility:**

- If `usage` is absent in a loaded checkpoint, initialize as an empty dict `{}`.

---

### `State.orchestrator_seen_flag_ids` Schema

**Type:** `set[str]`

**Structure:**

```python
{"flag_id_1", "flag_id_2", ...}
```

**Reducer:**

- Union incoming set into existing set.

**Default:**

- Empty set `set()`.

---

### Forced Hop Step Schema

**Fields:**

- `title: str` — `"Next: {route}"` (e.g., `"Next: ['planner']"`).
- `result: str` — `"forced: only one valid route"`.
- `evidence: dict` — includes `attempts: 0`, plus any other evidence fields from existing step structure in `orchestrator/node.py`.
- `status`, `flagged`, `priority`, `confidence` — updated as a model hop would (reuse existing logic or compute deterministically).
- `summary` — unchanged from prior step.

---

## Edge Cases and Error Handling

### Edge Case 1: Multiple stage agents valid, but only one route passes

**Scenario:** Two stage agents are unfinished, but one has a blocking flag. `forced_route` enumerates both individually and together; only one individual route passes `broken_rules`.

**Handling:** `forced_route` returns the single valid route. Forced hop proceeds.

---

### Edge Case 2: Unseen flag during forced route

**Scenario:** `forced_route` returns a route to terminal state, but a new flag was just added by a prior agent and the orchestrator hasn't seen it yet.

**Handling:** The orchestrator checks if any unresolved flag ID is missing from `orchestrator_seen_flag_ids`. If so, model work is required (flag merge); `_decide` is called instead of forcing.

---

### Edge Case 3: Run saved before `usage` field existed

**Scenario:** A checkpoint from before this feature has no `usage` field.

**Handling:** On load, `State.usage` initializes as an empty dict `{}`. The run resumes without error, and usage tracking starts from that point forward.

---

### Edge Case 4: JSON retry calls in `run_skill`

**Scenario:** The model's first response is not valid JSON; `run_skill` retries.

**Handling:** Each retry extracts and includes usage in the returned value. The `usage` reducer sums all retry counts into the node's totals.

---

### Edge Case 5: MCP tool loop with multiple rounds

**Scenario:** The MCP loop makes three tool-call rounds before the final no-tools call.

**Handling:** Each round extracts and includes usage in the returned state update. The `usage` reducer sums all rounds into the node's totals.

---

### Edge Case 6: Direct `ModelGateway.complete` call

**Scenario:** A node calls `ModelGateway.complete` directly, not via `run_skill` or MCP loop.

**Handling:** The calling node extracts usage from the response and includes it in its returned state update dict. If the node doesn't include usage, that call is not tracked in `State.usage` (logged by gateway but not aggregated).

---

### Edge Case 7: Pause is interrupt-based (no separate pause nodes)

**Scenario:** Pause is handled via `interrupt_before` in `interrupts.py`, not routing to a pause node (repo tree confirms no `review/` or `flag-review/` directories).

**Handling:** Guard 1 in orchestrator checks for interrupt conditions in state (e.g., `state.flagged`, `state.needs_review`), not route matching. The guard logic is adjusted based on pause mechanism.

---

### Edge Case 8: `"__end__"` sentinel in candidate enumeration

**Scenario:** LangGraph uses `"__end__"` as a sentinel value (not a real node), and `broken_rules` may reject it or handle it specially.

**Handling:** Include `["__end__"]` as a candidate if graph edges in `build.py` show it's used as a terminal target. If `broken_rules` rejects sentinels, exclude it and rely on `broken_rules` to enforce terminal logic.

---

### Edge Case 9: `run_skill` calling nodes not all identified

**Scenario:** Grep search for `run_skill` calls misses a calling node (e.g., indirect call, dynamic import).

**Handling:** That node crashes with `TypeError` when it tries to use the result as a dict. The crash is caught in testing (integration tests exercise all nodes).

**Mitigation:** Story 4 implementation checklist requires: "Search `src/agents/nodes/` for all `run_skill` calls, document calling nodes, update each to unpack tuple." Code review verifies the list is complete.

---

### Edge Case 10: Double-counting usage if `run_skill` passes `node_name`

**Scenario:** `run_skill` passes `node_name` to `gateway.complete`, causing both `run_skill` and the gateway to return usage.

**Handling:** Both usage dicts are included in the state update, causing double-counting. This is incorrect behavior.

**Mitigation:** Story 4 and Story 5 implementations must NOT pass `node_name` from `run_skill` or MCP tool loop to `gateway.complete`. Story 5 documents this in the `complete` docstring. Code review verifies no `node_name` is passed.

---

### Edge Case 11: Pause detection diverges between orchestrator and runner

**Scenario:** Orchestrator's Guard 1 checks `state.flagged`, but runner's pause detection checks `result["interrupted"]`.

**Handling:** If `flagged` is true but `interrupted` is false, orchestrator writes a summary for a pause that never happens, wasting tokens. If `flagged` is false but `interrupted` is true, runner logs usage for a pause without a summary.

**Mitigation:** Guard 1 and runner pause detection must use the same condition. Implementation verifies from `interrupts.py` what condition triggers pause, and both use it. If conditions differ, they must be documented and justified (e.g., Guard 1 is preventive, runner is reactive).

---

## Verification and Testing Strategy

### Verification Steps (Per Story)

**Story 1 (State schema):**

- Read `src/agents/graph/states.py` to determine actual flag field name and dict structure.
- Document flag schema in Story 1 LLD before coding: field name, ID field name, resolved field name, example flag dict.

**Story 2 (Forced route logic):**

- Read `route_rules.py` to determine `broken_rules` signature and return type.
- Read `__init__.py` to determine if `NODES` exists and its contents.
- Read `build.py` to determine terminal state marker.
- Document findings in Story 2 LLD before coding: `broken_rules` contract, `NODES` list, terminal marker, decision on `"__end__"` inclusion.

**Story 3 (Orchestrator guards):**

- Read Story 1's documented flag schema (field name, ID field, resolved field).
- Read `interrupts.py` to determine pause detection mechanism.
- Read `node.py` to determine orchestrator function structure.
- Document findings in Story 3 LLD before coding: pause detection pattern, flag extraction pattern, Guard 1 condition, Guard 2 condition.

**Story 4 (Usage in run_skill/MCP):**

- Search `src/agents/nodes/` for all `run_skill` calls.
- Document all calling nodes in Story 4 LLD before coding.
- Read `client.py` to determine `run_skill` signature and JSON retry logic.
- Read `bindings.py` to determine MCP tool loop structure.
- Read `gateway.py` to determine response object structure.

**Story 5 (Usage in gateway):**

- Verify Story 4 implementations do NOT pass `node_name` to `gateway.complete`.
- Read `gateway.py` to determine `complete` signature.

**Story 6 (Logging and API):**

- Read `runner.py` to determine graph execution function and end/pause detection.
- Read `backend.py` or `server.py` to determine endpoint response structure.

---

### Test Coverage

**Acceptance criteria:** 19 new tests total (6 for AC1, 6 for AC2, 7 for AC3 from requirements).

**Per-story tests:**

- Story 1: 7 tests (usage reducer, seen-flags reducer, backward compatibility)
- Story 2: 6 tests (forced_route returns correct route or None)
- Story 3: 6 tests (orchestrator skips model on forced hop, guards work)
- Story 4: 4 tests (usage returned per call, JSON retries, tool loop rounds)
- Story 5: 3 tests (gateway returns usage when node_name provided)
- Story 6: 4 tests (runner logs usage on end/pause, API returns usage)

**Integration test:** Full service order with mocked gateway, verify forced hops (`attempts: 0`), non-forced hops (`attempts > 0`), usage aggregation, and final logs (AC5 from requirements).

**Existing tests:** Must pass unchanged (`uv run pytest -q` in `tests/test_backend.py`, `tests/test_evaluate.py`, others).

---

## Out of Scope

- Changing models, `policy.yaml` limits, skill or prompt content
- Shrinking `_brief`, or removing the skill container and code-execution tool from orchestrator calls
- The MCP server, the GraphQL service, and fastapi-backend (beyond the `GET /runs/{thread_id}` endpoint modification in Story 6)
- Graph routing rules themselves: `broken_rules` behavior in `src/agents/nodes/orchestrator/route_rules.py` remains unchanged

---

## Success Criteria

**From requirements AC5 (PR evidence):**

After Story 6 is complete, run a full service order (e.g., `run_so.py` with SO-1003) and measure before/after token usage from the logs. Include in PR description:

```
Before: X calls, Y input tokens, Z output tokens
After: X' calls, Y' input tokens, Z' output tokens
Savings: (X-X') calls (%), (Y-Y') input tokens (%), (Z-Z') output tokens (%)
```

**From requirements AC4 (test coverage):**

All new tests (19 total: 6 for AC1, 6 for AC2, 7 for AC3) pass with a fake gateway (no live services), existing tests in `tests/test_backend.py` and `tests/test_evaluate.py` pass unchanged, and `uv run pytest -q` passes.

---

**End of HLD — PER-128 v1.2**
