# Breakdown — PER-128

**Epic:** (standalone)  
**Story:** Skip the orchestrator's model call on hops with only one possible route, and record token usage per run  
**Version:** 1.2  
**Status:** Draft

---

## Overview

Break down the forced routing and usage telemetry feature into independently testable implementation slices, sequenced by dependency. State schema changes come first (usage accumulation, seen-flags tracking), then routing logic (forced_route function), then orchestrator integration (decision guards, forced hops), then usage collection at all call sites, and finally endpoint and logging.

Each story targets the `agent` repo and produces artifacts (code, tests) that can be verified in isolation before integration.

**Mandatory verification before each story:** The developer MUST read and document findings from the actual codebase files listed in each story's "Implementation must verify before coding" section. No code may be written until verification is complete and documented in the story's LLD under a "Verification Findings" section. The repo context shows these files exist, but their exact interfaces, field names, node names, and function signatures are unknown until inspected.

---

## Story Execution Order

### Story 1: State schema for usage tracking and seen-flags

**Repo:** `agent`

**Description:**

Add `State.usage` (per-node token aggregation) and `State.orchestrator_seen_flag_ids` (track which flags the orchestrator has processed) to `src/agents/graph/states.py`. Implement additive reducer for `usage` (sum incoming counts per node) and union reducer for `seen_flag_ids`. Handle backward compatibility: runs saved without `usage` field must load and resume with `usage` initialized as empty dict.

Produce a **Verification Findings** document (`stories/01-agent/verification-findings.md`) before any code is written, containing the actual flag schema extracted from `states.py`. Story 3 will read this document for flag schema details.

**Implementation must verify before coding:**

- Read `src/agents/graph/states.py` to determine:
  - Current `State` dataclass structure and existing fields
  - How reducers are registered (e.g., `Annotated[type, reducer_func]`, a `__reduce_key__` dict, or other pattern)
  - Whether checkpoint loading logic exists in `states.py` or elsewhere (e.g., `src/agents/runtime/checkpointer.py`)
  - **Actual flag field name** (is it `flags`, `open_flags`, `active_flags`, other?)
  - **Flag dict structure**: what fields exist and their exact names (`id` vs `flag_id`, `resolved` vs `is_resolved`, other?)
  - Whether flags are filtered (only unresolved stored) or all flags present with a status field

**Verification deliverable (MANDATORY — blocks code and tests):**

Before implementation, create `stories/01-agent/verification-findings.md` with:

```markdown
# Story 1 Verification Findings

## Flag Schema from states.py

**File inspected:** `src/agents/graph/states.py` (lines X-Y)

**Flag field name:** `state.<verified_name>` (e.g., `state.flags`)

**Flag storage type:** (list[dict] / dict[str, dict] / other)

**Flag dict structure (if list[dict]):**
```python
{
  "<verified_id_field>": str,  # e.g., "id" or "flag_id"
  "<verified_resolved_field>": bool,  # e.g., "resolved" or "is_resolved"
  # Other keys found: message, source, etc.
}
```

**Example flag from code/docstring:**
```python
{"id": "flag_abc", "resolved": False, "message": "...", "source": "compliance"}
# OR different structure if found
```

**How to check if a flag is unresolved:**
```python
# e.g., flag["resolved"] == False
# OR:  flag.get("is_resolved", True) == False
# OR:  other pattern found in code
```

## State Schema and Reducers

**Reducer registration pattern:** (describe with example from existing field)

**Checkpoint loading location:** (`states.py` / `src/agents/runtime/checkpointer.py` / other)

## Implementation Decisions

**`usage` field name:** `State.usage` (confirmed no conflict with existing fields)

**`orchestrator_seen_flag_ids` field name:** `State.orchestrator_seen_flag_ids` (confirmed no conflict)

**Reducer implementation approach:** (describe based on verified pattern)
```

**Covers:**

- Requirements R1 (usage schema), AD-1 (usage accumulation pattern), AD-3 (seen-flags tracking)

**Depends on:** (none — foundational state change)

**Allowed paths:**

- `src/agents/graph/states.py` — add fields, reducers, migration logic
- `src/agents/runtime/checkpointer.py` — if checkpoint loading is separate, add migration there

**Verification Approach:**

1. Unit test: `usage` reducer sums counts across multiple updates for same node (3 tests: single node multiple calls, multiple nodes, missing keys initialize to zero)
2. Unit test: `orchestrator_seen_flag_ids` reducer unions incoming sets (2 tests: empty initial set, existing set extended)
3. Unit test: load checkpoint without `usage` field → state initializes with empty dict, no error (1 test)
4. Unit test: load checkpoint with `usage` → state preserves existing counts (1 test)

**Total tests:** 7

**Acceptance Criteria (from requirements R1):**

- `State.usage` schema: `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`
- Reducer sums new counts into existing counts per node; missing keys initialize to zero
- A run saved without `usage` loads and resumes without error, `usage` initialized as empty dict
- `State.orchestrator_seen_flag_ids` is a set, reducer unions incoming IDs into existing set
- Verification findings document produced and contains actual flag schema

---

### Story 2: Forced route logic in route_rules

**Repo:** `agent`

**Description:**

Add `forced_route(state: State) -> list[str] | None` to `src/agents/nodes/orchestrator/route_rules.py`. Enumerate candidates (each node in `NODES` individually, plus every non-empty set of stage agents verified from repo tree: `compliance`, `payroll_calculation`, `validation`). For each candidate, call `broken_rules(state, candidate)` using its verified signature. Return the candidate if exactly one passes validation; otherwise return `None`. Delegate all routing validity to `broken_rules` — no logic duplication.

Produce a **Verification Findings** document (`stories/02-agent/verification-findings.md`) before any code is written, making the decision on `"__end__"` enumeration and documenting the final candidate list. Tests will use this documented decision.

**Implementation must verify before coding:**

- Read `src/agents/nodes/orchestrator/route_rules.py` to determine:
  - **Exact signature of `broken_rules(state, route)`**: what type is `route` parameter (single node name as `str`, list of node names as `list[str]`, other)?
  - **What `broken_rules` returns**: list of error strings (empty = valid), dict with errors, boolean (True = invalid), or other?
  - How errors are represented (empty list = valid, non-empty list = errors, or different convention?)
- Read `src/agents/nodes/__init__.py` to determine:
  - Whether `NODES` list exists and what it contains
  - If `NODES` doesn't exist, which nodes are imported/registered (derive list from imports)
  - **Correct node name format**: underscores vs hyphens (is it `output_validator` or `output-validator`?)
- Read `src/agents/nodes/` directory structure (from repo tree) to verify:
  - All node directory names: `compliance/`, `execution/`, `orchestrator/`, `output_validator/`, `payroll_calculation/`, `planner/`, `validation/` confirmed
  - No `review/` or `flag-review/` directories exist (pause is not a separate node)
  - Stage agents: `compliance`, `payroll_calculation`, `validation` (confirmed)
- Read `src/agents/graph/build.py` to determine:
  - **Terminal state marker**: is it `"__end__"` (LangGraph sentinel), or another name?
  - Whether `"__end__"` is included in edges as a target (it's a sentinel, not a runnable node, so likely not in `NODES`)

**Verification deliverable (MANDATORY — blocks code and tests):**

Before implementation, create `stories/02-agent/verification-findings.md` with:

```markdown
# Story 2 Verification Findings

## broken_rules Contract

**File inspected:** `src/agents/nodes/orchestrator/route_rules.py` (lines X-Y)

**Function signature (verbatim copy):**
```python
def broken_rules(state: State, next_step: <ACTUAL_TYPE>) -> <ACTUAL_RETURN_TYPE>:
```

**Parameter `next_step` expects:** (single node name string / list of node names / other — describe exactly)

**Return type:** (list of error strings / bool / dict / other — describe exactly)

**Validation convention:** (how to interpret the return value to determine if route is valid)
- Example: "Returns list[str]. Empty list means route is valid. Non-empty list contains error messages."

**Parallel route handling:** (how `broken_rules` treats multi-node routes like `["compliance", "validation"]`)

## NODES List

**File inspected:** `src/agents/nodes/__init__.py` (lines X-Y)

**NODES list exists?** (YES / NO)

**If YES, exact contents (verbatim copy):**
```python
NODES = [
    # Copy the exact list here
]
```

**If NO, derived node names from imports:**
- planner
- compliance
- payroll_calculation
- validation
- execution
- output_validator (exact spelling verified)

## Node Names and Terminal Marker

**Verified node names (match directory names exactly):**
- planner
- compliance
- payroll_calculation
- validation
- execution
- output_validator (or output-validator? — confirmed spelling)

**Verified stage agents:**
- compliance
- payroll_calculation
- validation

**File inspected for terminal marker:** `src/agents/graph/build.py` (lines X-Y)

**Terminal route marker:**
- Is `"__end__"` used? (YES / NO)
- If YES, is `"__end__"` a runnable node? (YES / NO)
- How is terminal state represented in edges? (describe)

## Implementation Decision: __end__ Enumeration

**Decision:** Include `"__end__"` in `forced_route` candidates? (YES / NO)

**Reasoning:** (based on whether `broken_rules` accepts sentinel values, verified above)

**Final candidate list for enumeration:**
```python
# Individual nodes:
["planner"], ["compliance"], ["payroll_calculation"], ["validation"], ["execution"], ["output_validator"]
# Include ["__end__"] if decision is YES

# Stage agent subsets (7 non-empty subsets):
["compliance"], ["payroll_calculation"], ["validation"],
["compliance", "payroll_calculation"], ["compliance", "validation"], ["payroll_calculation", "validation"],
["compliance", "payroll_calculation", "validation"]
```

## Test Assertions (based on decision)

**Test 3 assertion (terminal state):**
```python
# If __end__ included:
assert result == ["__end__"]
# If __end__ excluded:
assert result is None  # (or whatever route is valid after terminal logic)
```
```

**Covers:**

- Requirements R2 (forced route selection), AD-2 (forced_route interface and contract)

**Depends on:** Story 1 (State schema must exist, though `forced_route` doesn't use new fields yet)

**Allowed paths:**

- `src/agents/nodes/orchestrator/route_rules.py` — add `forced_route` function
- `src/agents/nodes/__init__.py` — add `NODES` if not present (only if verification shows it's missing)

**Verification Approach:**

1. Unit test: before planner runs → `forced_route` returns verified planner route (1 test)
2. Unit test: after terminal state reached → `forced_route` returns verified terminal route or `None` (1 test; uses decision from findings doc)
3. Unit test: after execution finishes → `forced_route` returns verified output validator route (1 test)
4. Unit test: two stage agents unfinished, no flags/reviews waiting → `forced_route` returns `None` (1 test)
5. Unit test: only parallel stage agent set is valid → `forced_route` returns the set or `None` (1 test; based on verified `broken_rules` behavior)
6. Unit test: all candidates rejected by `broken_rules` → `forced_route` returns `None` (1 test)

**Total tests:** 6

**Acceptance Criteria (from requirements R2):**

- `forced_route(state)` returns a single route when exactly one candidate passes `broken_rules`, `None` otherwise
- Candidates include each verified node individually, plus every non-empty set of verified stage agents
- `broken_rules` remains sole source of routing validity logic (no duplication in `forced_route`)
- Contract with `broken_rules` matches its verified signature and return type
- Verification findings document produced and contains `__end__` enumeration decision and final candidate list
- Tests use assertions from verification findings document (no conflicting assertions)

---

### Story 3: Orchestrator decision guards and forced hops

**Repo:** `agent`

**Description:**

Modify `src/agents/nodes/orchestrator/node.py` to call `forced_route(state)` before the model decision function (verified name from codebase). Extract flag IDs from state using the flag schema from Story 1's verification findings document, and add them to `State.orchestrator_seen_flag_ids` before decision logic. When `forced_route` returns a route, check model work guards:

1. **Pause detection guard**: If pause is triggered (verified mechanism from codebase — may be via `interrupt_before` in `src/agents/graph/interrupts.py`, not a separate pause route node since no `review/` directory exists), call the model to write summary.
2. **Unseen flags guard**: If any unresolved flag (using flag schema from Story 1's findings) has an ID not in `orchestrator_seen_flag_ids`, call the model to merge flags.

If no guards trigger, skip the model call and write a forced hop step: `title: "Next: {route}"`, `result: "forced: only one valid route"`, `evidence: {attempts: 0}`, update `status`, `flagged`, `priority`, `confidence` using verified field update pattern, leave `summary` unchanged.

Produce a **Verification Findings** document (`stories/03-agent/verification-findings.md`) before any code is written, resolving the pause detection mechanism and orchestrator structure, using Story 1's flag schema findings.

**Implementation must verify before coding:**

- Read `stories/01-agent/verification-findings.md` to obtain flag schema (field name, dict structure, ID field, resolved field)
- Read `src/agents/nodes/` directory to verify:
  - No `review/` or `flag-review/` directories exist (confirmed from repo tree)
  - Pause is not a separate node route
- Read `src/agents/graph/interrupts.py` to determine:
  - How pause is triggered (likely `interrupt_before` condition in graph builder, not a route to a pause node)
  - What state condition or flag triggers the interrupt (e.g., `state.needs_review`, or presence of unresolved flags, or other)
- Read `src/agents/nodes/orchestrator/node.py` to determine:
  - **Main orchestrator function name** (is it `run`, `decide`, `orchestrator_node`, other?)
  - **Model decision function name** (is it `_decide`, `decide_route`, other?)
  - How steps are written (return dict structure, field names)
  - How state fields are updated (`status`, `priority`, `confidence`, `flagged`, `summary` — field names and update pattern)
  - Current flag extraction pattern (if any) to match existing code style

**Verification deliverable (MANDATORY — blocks code and tests):**

Before implementation, create `stories/03-agent/verification-findings.md` with:

```markdown
# Story 3 Verification Findings

## Flag Schema (from Story 1 findings)

**Source:** `stories/01-agent/verification-findings.md`

**Flag field name:** state.<verified_name> (copied from Story 1)

**Flag dict structure:** (copied from Story 1)

**ID field name:** <verified_id_field> (e.g., "id" or "flag_id")

**Resolved field name:** <verified_resolved_field> (e.g., "resolved" or "is_resolved")

**Unresolved flag check pattern:**
```python
# Pattern from Story 1 findings
```

## Pause Detection Mechanism

**Directories searched:** `src/agents/nodes/` (confirmed no `review/`, `flag-review/`, etc.)

**Pause route nodes exist?** NO

**File inspected:** `src/agents/graph/interrupts.py` (lines X-Y)

**Pause mechanism:**
- Triggered by: (interrupt_before condition / state flag / other)
- Condition code location: (show relevant snippet)

**State condition that triggers pause:**
```python
# e.g., state.flagged == True
# OR:  len(state.open_flags) > 0
# OR:  state.needs_review == True
# (verified condition)
```

## Orchestrator Structure

**File inspected:** `src/agents/nodes/orchestrator/node.py` (lines X-Y)

**Main orchestrator function:**
```python
def <verified_name>(state: State) -> dict:
```

**Model decision function:**
```python
def <verified_name>(state: State) -> dict:
```

**Step structure:**
```python
{
  "title": str,
  "result": str,
  "evidence": {
    "attempts": int,
    # Other evidence keys found
  },
  # Other step fields
}
```

**State field update pattern:**
```python
# How status, priority, confidence, flagged, summary are computed/updated
# (show code snippet)
```

## Implementation Decisions

**Guard 1 (pause detection) checks:**
```python
# Condition to check (based on verified pause mechanism)
if <verified_pause_condition>:
    # Call model decision function
```

**Guard 2 (unseen flags) checks:**
```python
# Using verified flag schema
current_flags = state.<verified_flag_field>
unresolved_flags = [f for f in current_flags if not f.get("<verified_resolved_field>")]
unseen_flags = [f for f in unresolved_flags if f.get("<verified_id_field>") not in state.orchestrator_seen_flag_ids]
if unseen_flags:
    # Call model decision function
```

**Forced hop step creation:**
```python
# Using verified step structure
step = {
    "title": f"Next: {route}",
    "result": "forced: only one valid route",
    "evidence": {
        "attempts": 0,
        # Other verified evidence fields
    },
    # Other verified step fields
}
```

**State update return:**
```python
# Using verified field names and update pattern
return {
    "steps": [step],
    "next": route,
    "status": <computed>,
    "flagged": <computed>,
    "priority": <computed>,
    "confidence": <computed>,
    "summary": state.<verified_summary_field>
}
```
```

**Covers:**

- Requirements R3 (orchestrator behavior on forced routes), AD-3 (orchestrator decision guards)

**Depends on:** Story 1 (State schema with `orchestrator_seen_flag_ids` and flag schema findings), Story 2 (forced_route function)

**Allowed paths:**

- `src/agents/nodes/orchestrator/node.py` — modify decision logic, add forced hop logic

**Verification Approach:**

1. Unit test: forced route returned, no model work → no model call, step has `attempts: 0` (1 test)
2. Unit test: forced route returned, pause condition detected (using verified pause mechanism) → model call proceeds (1 test)
3. Unit test: forced route returned, unseen flag exists (using verified flag schema) → model call proceeds (1 test)
4. Unit test: orchestrator receives control with flags → `orchestrator_seen_flag_ids` updated before decision (1 test)
5. Unit test: forced hop writes step with verified structure, updates verified state fields, leaves `summary` unchanged (1 test)
6. Unit test: forced hop evidence includes `attempts: 0` and verified evidence fields (1 test)

**Total tests:** 6

**Acceptance Criteria (from requirements R3):**

- Orchestrator skips model call on forced hop when no model work required (verified pause condition not met, no unseen flags per verified schema)
- Forced hop step: `title: "Next: {route}"`, `result: "forced: only one valid route"`, `attempts: 0` in evidence
- Forced hop updates verified state fields as model hop would; `summary` unchanged
- `orchestrator_seen_flag_ids` updated from state before decision (using verified flag extraction pattern)
- Model call proceeds when verified pause condition detected, or when unseen flag exists per verified schema
- Verification findings document produced and uses Story 1's flag schema (not re-derived)

---

### Story 4: Usage collection in run_skill and MCP tool loop

**Repo:** `agent`

**Description:**

Extract usage (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`) from every model call in `src/agents/skills/client.py` (`run_skill`, including JSON retries) and `src/agents/tools/bindings.py` (MCP tool loop, including final no-tools call). Modify return type of `run_skill` to `(result, usage_data)` and MCP tool loop to `(final_response, usage_data)`. All calling nodes (identified by searching `src/agents/nodes/` for `run_skill` imports/calls) must be updated to unpack the tuple and include usage in their state updates. The `usage` reducer (from Story 1) will sum these into `State.usage[node_name]`.

Produce a **Verification Findings** document (`stories/04-agent/verification-findings.md`) before any code is written, listing ALL calling nodes identified by grep (not a speculative list). Implementation sketches will be written only after this list is confirmed.

**Implementation must verify before coding:**

- Read `src/agents/skills/client.py` to determine:
  - **Current signature and return type of `run_skill`** (sync vs async? what does it return today?)
  - How `run_skill` calls `ModelGateway.complete`
  - Where JSON retry logic lives and how many retries happen
- Read `src/agents/tools/bindings.py` to determine:
  - **MCP tool loop function name and signature**
  - How tool rounds work and when the loop terminates
  - How the loop calls `ModelGateway.complete`
- Read `src/agents/model/gateway.py` to determine:
  - **Response object structure** from `complete`: does it have `response.usage.input_tokens` or `response["usage"]["input_tokens"]`?
  - What usage fields are included in responses (verify all four token fields exist)
  - Whether usage is already logged
- **RUN GREP to identify ALL calling nodes:**
  - `grep -r "run_skill" src/agents/nodes/ --include="*.py"` — document every result with file path and line number
  - `grep -r "run_tools" src/agents/nodes/ --include="*.py"` — document every result (if MCP tool loop is named `run_tools`)

**Verification deliverable (MANDATORY — blocks code and tests):**

Before implementation, create `stories/04-agent/verification-findings.md` with:

```markdown
# Story 4 Verification Findings

## run_skill Contract

**File inspected:** `src/agents/skills/client.py` (lines X-Y)

**Current signature:**
```python
def run_skill(...) -> <current_return_type>
```

**New signature:**
```python
def run_skill(...) -> tuple[dict, dict]
```

**How run_skill calls ModelGateway.complete:** (direct call / wrapper / other)

**JSON retry logic:** (location and number of retries)

## Response Structure

**File inspected:** `src/agents/model/gateway.py` (lines X-Y)

**Response usage access pattern:**
```python
response.usage.input_tokens  # (or response["usage"]["input_tokens"], or other)
```

**All four token fields verified:**
- input_tokens: (exists / missing)
- output_tokens: (exists / missing)
- cache_creation_input_tokens: (exists / missing / may be absent)
- cache_read_input_tokens: (exists / missing / may be absent)

## MCP Tool Loop

**File inspected:** `src/agents/tools/bindings.py` (lines X-Y)

**Tool loop function name:** (run_tools / mcp_loop / other)

**Tool loop structure:** (while loop / recursive / other)

**How loop calls gateway:** (direct / wrapper)

**Termination condition:** (no more tool_calls / other)

## Calling Nodes (from grep)

**grep command run:** `grep -r "run_skill" src/agents/nodes/ --include="*.py"`

**Results (file path : line number : snippet):**
- src/agents/nodes/planner/node.py : 42 : `result = run_skill(...)`
- src/agents/nodes/orchestrator/node.py : 67 : `result = run_skill(...)`
- (list ALL results from grep — no speculation)

**grep command run:** `grep -r "run_tools" src/agents/nodes/ --include="*.py"`

**Results (if any):**
- (list ALL results)

**Total calling nodes to update:** (count from grep results)

**Calling nodes list (file paths only):**
- src/agents/nodes/planner/node.py
- src/agents/nodes/orchestrator/node.py
- (complete list from grep, no additions)

## Implementation Notes

**Each calling node must:**
1. Change `result = run_skill(...)` to `(result, usage_data) = run_skill(...)`
2. Add to state update: `{"usage": {node_name: usage_data}}`

**Node names (from file paths):**
- planner/node.py → `"planner"`
- orchestrator/node.py → `"orchestrator"`
- (derive from file paths)
```

**Covers:**

- Requirements R1 (usage recording at call sites), AD-4 (usage collection integration points)

**Depends on:** Story 1 (State schema with `usage` reducer)

**Allowed paths:**

- `src/agents/skills/client.py` — modify `run_skill` to return `(result, usage_data)`
- `src/agents/tools/bindings.py` — modify MCP tool loop to return `(final_response, usage_data)`
- All verified calling nodes (from grep results, not speculative list) — update to unpack new return type

**Verification Approach:**

1. Unit test: `run_skill` makes one call → usage returned with correct counts (1 test)
2. Unit test: `run_skill` retries JSON → both calls' usage summed in returned data (1 test)
3. Unit test: MCP tool loop makes two tool rounds + final call → all three usage updates summed (1 test)
4. Unit test: verify returned usage matches response fields (all four token fields) (1 test)

**Total tests:** 4

**Acceptance Criteria (from requirements R1):**

- Every model call in `run_skill` (including JSON retries) returns usage data
- Every MCP tool loop round (including final no-tools call) returns usage data
- Returned usage includes `calls: 1`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
- All calling nodes identified by grep are updated to unpack tuple and include usage in state updates
- No calling nodes missed (grep results are complete and match implementation)
- `State.usage` aggregates counts per node across all calls
- Verification findings document lists actual grep results (not speculative list)

---

### Story 5: Usage collection in ModelGateway direct calls

**Repo:** `agent`

**Description:**

Modify `src/agents/model/gateway.py` to accept optional `node_name` parameter in `complete` method. When `node_name` is provided, return `(response, usage_data)` tuple. If `node_name` is not provided, return only response (backward compatible). Usage is logged in both cases.

**CRITICAL CONSTRAINT:** Verify that Story 4's `run_skill` and MCP tool loop implementations do NOT pass `node_name` to `gateway.complete`. Add a runtime assertion in `run_skill` and MCP tool loop to prevent accidental double-dispatch.

Produce a **Verification Findings** document (`stories/05-agent/verification-findings.md`) before any code is written, confirming Story 4 non-interference.

**Implementation must verify before coding:**

- Read `src/agents/model/gateway.py` to determine:
  - **Current signature of `complete` method**
  - **Response structure** (re-confirm from Story 4 findings)
  - Whether usage is already logged
- **Verify Story 4 implementations (after Story 4 is merged):**
  - Read merged `src/agents/skills/client.py` to confirm `run_skill` does NOT pass `node_name` to `gateway.complete`
  - Read merged `src/agents/tools/bindings.py` to confirm MCP tool loop does NOT pass `node_name` to `gateway.complete`

**Verification deliverable (MANDATORY — blocks code and tests):**

Before implementation, create `stories/05-agent/verification-findings.md` with:

```markdown
# Story 5 Verification Findings

## ModelGateway.complete Contract

**File inspected:** `src/agents/model/gateway.py` (lines X-Y)

**Current signature:**
```python
def complete(self, messages, tools=None, **kwargs) -> <current_return_type>
```

**New signature:**
```python
def complete(self, messages, tools=None, node_name: Optional[str] = None, **kwargs) -> Union[<response_type>, tuple[<response_type>, dict]]
```

**Response structure (re-confirmed from Story 4):**
```python
response.usage.input_tokens  # (or other pattern)
```

**Usage logging:** (existing logger.info calls at lines X-Y)

## Story 4 Non-Interference Verification

**File inspected:** `src/agents/skills/client.py` (merged Story 4 code, lines X-Y)

**run_skill calls gateway.complete:**
```python
response = gateway.complete(
    messages=[...],
    **kwargs  # NO node_name parameter
)
```

**Confirmed:** `run_skill` does NOT pass `node_name` to `gateway.complete`

**File inspected:** `src/agents/tools/bindings.py` (merged Story 4 code, lines X-Y)

**MCP loop calls gateway.complete:**
```python
response = gateway.complete(
    messages=messages,
    tools=tools,
    **kwargs  # NO node_name parameter
)
```

**Confirmed:** MCP tool loop does NOT pass `node_name` to `gateway.complete`

## Implementation Safeguard

**Runtime assertion added to run_skill:**
```python
assert 'node_name' not in kwargs, "run_skill must NOT pass node_name to gateway.complete (causes double-counting)"
```

**Runtime assertion added to MCP tool loop:**
```python
assert 'node_name' not in kwargs, "MCP loop must NOT pass node_name to gateway.complete (causes double-counting)"
```

**Docstring note added to gateway.complete:**
```
Note:
    If calling from run_skill or run_tools, do not pass node_name — those functions
    extract usage themselves. Only pass node_name for direct calls that need usage tracking.
```
```

**Covers:**

- Requirements R1 (usage recording at all call sites), AD-4 (direct gateway calls)

**Depends on:** Story 1 (State schema), Story 4 (usage dispatch pattern established, and must not conflict)

**Allowed paths:**

- `src/agents/model/gateway.py` — add `node_name` parameter, conditional return of usage data, add docstring note
- `src/agents/skills/client.py` — add runtime assertion (if Story 4 doesn't already have it)
- `src/agents/tools/bindings.py` — add runtime assertion (if Story 4 doesn't already have it)

**Verification Approach:**

1. Unit test: `complete` called with `node_name` → tuple returned with usage (1 test)
2. Unit test: `complete` called without `node_name` → response only returned
