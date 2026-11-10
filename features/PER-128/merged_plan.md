# Merged plan — PER-128

Execution order (6 ticket(s)):

1. `PER-130` (story `01-agent`)
2. `PER-131` (story `02-agent`)
3. `PER-132` (story `03-agent`)
4. `PER-147` (story `04-agent`)
5. `PER-148` (story `05-agent`)
6. `PER-149` (story `06-agent`)

Specs (1): `specs/01.md`

---

## Spec `specs/01.md`

# Tech Stub — PER-128-01

**Story:** Skip the orchestrator's model call on hops with only one possible route, and record token usage per run  
**Epic:** (standalone)  
**Parent Spec:** `specs/01.md`  
**Version:** 1.2  
**Altitude:** Story

---

## Overview

This stub seeds the implementation of forced routing and usage telemetry from repo context only. It establishes the stack, schema contracts, file touch points, and risks without inventing details beyond what the HLD and repo tree already show.

**Critical verification required before implementation:** Every story LLD must read the actual existing files listed in its "Implementation must verify" section before writing new code. The breakdown and HLD make assumptions about signatures, schemas, and node names that cannot be confirmed until the developer inspects the repo. This stub documents what must be verified, not what is assumed.

**MANDATORY PRE-CODING GATE:** No story implementation may begin until its verification checklist is complete and documented in a `verification-findings.md` file in the story folder. Each story's first deliverable is the verification findings document, not code.

---

## Stack

**Language:** Python 3.11+  
**Framework:** LangGraph (graph-based agent orchestration)  
**Model Gateway:** `src/agents/model/gateway.py` (`ModelGateway.complete`)  
**State Management:** `src/agents/graph/states.py` (`State` dataclass with reducers)  
**Routing Logic:** `src/agents/nodes/orchestrator/route_rules.py` (`broken_rules`)  
**Orchestrator:** `src/agents/nodes/orchestrator/node.py`  
**Skills:** `src/agents/skills/client.py` (`run_skill`)  
**Tools:** `src/agents/tools/bindings.py` (MCP tool loop)  
**Runtime:** `src/agents/runtime/runner.py` (`run_query` or equivalent)  
**Backend:** `src/agents/runtime/backend.py` or `server.py` (API endpoint)  
**Testing:** `pytest` (existing test files in `tests/`)  
**Dependency Management:** `pyproject.toml` (uv/pip)

---

## Schema and API Contracts

**Important:** All schemas below are design intentions from requirements and HLD. Actual field names, types, and structures must be verified by reading the existing code before implementation. Implementation sketches are deliberately vague and use placeholders to force verification.

### `State.usage` Schema (Design Intent from HLD AD-1)

**Type:** `dict[str, dict[str, int]]`

**Structure (unverified):**

```python
{
  "node_name": {
    "calls": int,
    "input_tokens": int,
    "output_tokens": int,
    "cache_creation_input_tokens": int,
    "cache_read_input_tokens": int
  }
}
```

**Reducer (unverified):**

- Additive: incoming counts sum into existing counts per node.
- Missing keys initialize to zero.

**Backward Compatibility (unverified):**

- If `usage` is absent in a loaded checkpoint, initialize as empty dict `{}`.

**Must verify before Story 1 implementation:**

- [ ] Read `src/agents/graph/states.py` lines 1-200 (or full file if shorter)
- [ ] Document current `State` dataclass structure: all existing fields, their types, and default values
- [ ] Document reducer registration pattern: Is it `Annotated[type, reducer_func]`, a `__reduce_key__` dict, or other?
- [ ] Locate checkpoint loading: Is it in `states.py`, `src/agents/runtime/checkpointer.py`, or elsewhere?
- [ ] Document actual flag field name: Search for "flag" in `states.py` — is it `flags`, `open_flags`, `active_flags`, `flagged`, or other?
- [ ] Document actual flag dict schema: If flags are a list, what keys exist in each dict? (`id`, `flag_id`, `resolved`, `is_resolved`, `message`, other?)
- [ ] Confirm `usage` and `orchestrator_seen_flag_ids` field names don't conflict with existing fields

**Blocking question for Story 1:** If reducer registration uses `Annotated`, what import is required? If it uses a dict, what is the dict's name and where is it defined?

---

### `State.orchestrator_seen_flag_ids` Schema (Design Intent from HLD AD-1)

**Type:** `set[str]`

**Structure (unverified):**

```python
{"flag_id_1", "flag_id_2", ...}
```

**Reducer (unverified):**

- Union incoming set into existing set.

**Default:**

- Empty set `set()`.

**Must verify before Story 1 implementation:**

- [ ] Confirm reducer registration pattern matches `usage` field's pattern in same file

---

### Flag Schema (Must Verify Before Story 1 and Story 3)

**Design assumption from requirements R3 (UNVERIFIED):**

- Field name: `State.flags` (could be `open_flags`, `active_flags`, `flagged`, etc.)
- Structure: `list[dict]` where each dict has `id: str` and `resolved: bool` fields (field names unconfirmed)

**Story 1 requires (for orchestrator_seen_flag_ids extraction):**

- [ ] Actual field name on `State` that holds flags
- [ ] Whether flags are a list, dict, or other type
- [ ] If a list, are all flags present or only unresolved? If a dict, what are the keys?

**Story 3 requires (for unseen flag guard):**

- [ ] Actual field names within each flag dict: `id` vs `flag_id`, `resolved` vs `is_resolved`, other?
- [ ] How to check if a flag is unresolved: `flag["resolved"] == False`, `flag.get("is_resolved", True) == False`, `flag["status"] == "open"`, or other?

**Blocking question for Story 3:** If flags don't have an `id` field, how are flags uniquely identified? If there's no unique ID, how can `orchestrator_seen_flag_ids` track which flags have been seen?

---

### `forced_route(state: State) -> list[str] | None` (Design Intent from HLD AD-2)

**Module:** `src/agents/nodes/orchestrator/route_rules.py`

**Signature (unverified — actual signature TBD by Story 2 verification):**

```python
def forced_route(state: State) -> list[str] | None:
    """
    Returns a single valid route if exactly one candidate passes broken_rules.
    Otherwise returns None.
    
    Candidates:
    - Each node in NODES individually
    - Every non-empty set of stage agents
    
    Delegates all validity checks to broken_rules.
    
    ACTUAL IMPLEMENTATION DEPENDS ON VERIFIED broken_rules CONTRACT.
    """
```

**Input:**

- `state: State` — current graph state

**Output (design intent only):**

- `list[str]` — single valid route (e.g., `["planner"]`, terminal node name verified from repo)
- `None` — zero or multiple valid routes

**Contract (from HLD AD-2, unverified in code):**

- No routing logic duplication — all validity checks delegate to `broken_rules`.
- Exhaustive candidate enumeration (all nodes + all stage agent sets).

**Must verify before Story 2 implementation:**

- [ ] Read `src/agents/nodes/orchestrator/route_rules.py` lines containing `broken_rules` definition (search for `def broken_rules`)
- [ ] Document exact signature: `def broken_rules(state, next_step) -> ???` — what type is `next_step`? Single string, list of strings, tuple, other?
- [ ] Document return type: Does it return `list[str]` (error messages), `dict` (with errors key), `bool` (True = invalid), or other?
- [ ] Document validation convention: Empty list = valid? `False` = valid? No errors in dict = valid? Other?
- [ ] Document what `broken_rules` expects for `next_step` when validating parallel agents: Does it accept `["compliance", "validation"]` or does it require a different format?
- [ ] Read `src/agents/nodes/__init__.py` lines 1-50 (or full file)
- [ ] Document whether `NODES` list exists: Search for `NODES =` or `__all__` containing node names
- [ ] If `NODES` exists, document its contents exactly (copy the list)
- [ ] If `NODES` doesn't exist, derive it: List all imported node names from the file
- [ ] Document correct node name format: Run `ls src/agents/nodes/` and confirm underscore vs hyphen (is it `output_validator/` or `output-validator/`?)
- [ ] Document terminal node: Search `src/agents/graph/build.py` for `"__end__"` or `END` or terminal state marker — is `"__end__"` used? Is it in `NODES`? Is it a valid candidate for `broken_rules`?
- [ ] Document pause mechanism: Search `src/agents/graph/interrupts.py` for `interrupt_before` — is pause triggered by interrupt condition or by routing to a pause node?
- [ ] Search `src/agents/nodes/` directory for `review`, `flag-review`, `human_review`, or similar: Do any pause route node directories exist?

**Blocking questions for Story 2:**

1. If `broken_rules` expects `next_step` as a single string, how should `forced_route` call it for parallel agent sets like `["compliance", "validation"]`? Should it skip parallel candidates, or pass them as a joined string, or call `broken_rules` once per agent in the set?
2. If `"__end__"` is not in `NODES` (because it's a sentinel, not a runnable node), should `forced_route` enumerate it as a candidate anyway? If yes, will `broken_rules` accept it without crashing?
3. If no pause route nodes exist (pause is via `interrupt_before`), how should Story 3's Guard 1 detect pause conditions? Should it check `route` or check `state` for an interrupt flag?

---

### Forced Hop Step Schema (Design Intent from HLD AD-3)

**Fields (unverified — actual step structure TBD by Story 3 verification):**

- `title: str` — `"Next: {route}"` (e.g., `"Next: ['planner']"`)
- `result: str` — `"forced: only one valid route"`
- `evidence: dict` — includes `attempts: 0` (other fields TBD)
- `status`, `flagged`, `priority`, `confidence` — updated as model hop would (field names and update pattern TBD)
- `summary` — unchanged from prior step (field name TBD)

**Must verify before Story 3 implementation:**

- [ ] Read `src/agents/nodes/orchestrator/node.py` lines containing step creation (search for `"title"` or `"result"` or `steps`)
- [ ] Document existing step dict structure: What keys exist? (`title`, `result`, `evidence`, `timestamp`, `node`, other?)
- [ ] Document evidence structure: Is it a dict with `attempts` key? What other keys exist?
- [ ] Document where steps are written: Does the orchestrator return `{"steps": [step]}` (list) or `{"step": step}` (single)?
- [ ] Document state field names: Search for `state.status`, `state.priority`, `state.confidence`, `state.flagged`, `state.summary` — do these fields exist? Are they named differently?
- [ ] Document state update pattern: Does the orchestrator return a dict with updated fields, or call a method like `state.update(...)`?

**Blocking question for Story 3:** If the orchestrator doesn't update `priority` or `confidence` (fields don't exist), should forced hops skip those updates or initialize them?

---

### API Response Extension (Design Intent from requirements R1)

**Endpoint:** `GET /runs/{thread_id}`

**Added to Response (unverified structure):**

```json
{
  "values": {
    "usage": {
      "node_name": {
        "calls": 5,
        "input_tokens": 12500,
        "output_tokens": 850,
        "cache_creation_input_tokens": 3200,
        "cache_read_input_tokens": 9300
      }
    }
  }
}
```

**Must verify before Story 6 implementation:**

- [ ] Read `src/agents/runtime/backend.py` lines containing `/runs/` route (search for `@app.get` or `@router.get` or `"/runs/"`)
- [ ] If not in `backend.py`, read `src/agents/runtime/server.py` for same
- [ ] Document which file implements `GET /runs/{thread_id}`: Is it `backend.py`, `server.py`, or both?
- [ ] Document response structure: Is `values` the raw `State` dict, or is it filtered/transformed?
- [ ] Document whether `State.usage` is automatically included: Test by adding a field to `State` and checking if it appears in the response
- [ ] If filtering is applied, document the allowlist: Where is it defined? How to add `usage`?

**Blocking question for Story 6:** If the endpoint doesn't return `values` but returns individual state fields (`status`, `next`, etc.), how should `usage` be added to the response?

---

## File Touch List

**Note:** All modifications depend on verifying existing code structure first. The lists below are planning aids, not prescriptive implementations. NO CODE MAY BE WRITTEN until verification is complete for that story.

### Modified Files

1. **`src/agents/graph/states.py`** (Story 1)
   - Add `usage: dict[str, dict[str, int]]` field (exact syntax TBD by verification: `Annotated[...]`, plain field, other?)
   - Add `orchestrator_seen_flag_ids: set[str]` field
   - Implement additive reducer for `usage` (registration pattern TBD by verification)
   - Implement union reducer for `orchestrator_seen_flag_ids`
   - Handle missing `usage` on checkpoint load (location TBD by verification: inline, `checkpointer.py`, other?)
   - **Verify first:** Current `State` structure, reducer registration pattern, checkpoint loading mechanism, actual flag schema

2. **`src/agents/nodes/orchestrator/route_rules.py`** (Story 2)
   - Add `forced_route(state: State) -> list[str] | None` function (exact implementation TBD by verification of `broken_rules` contract)
   - Import `NODES` from `src/agents/nodes/__init__.py` or define if missing (determined by verification)
   - Enumerate candidates: nodes + stage agent sets (format TBD by verification)
   - Delegate validity to `broken_rules` (calling convention TBD by verification)
   - **Verify first:** `broken_rules` signature and return type, `NODES` existence and content, actual node names, terminal node handling, pause mechanism

3. **`src/agents/nodes/orchestrator/node.py`** (Story 3)
   - Extract flag IDs from state before decision (field name TBD by verification)
   - Update `orchestrator_seen_flag_ids` in state (return pattern TBD by verification)
   - Call `forced_route(state)` before model decision function (function name TBD by verification)
   - Implement model work guards: pause detection (mechanism TBD by verification), unseen flags (schema TBD by verification)
   - Skip model decision function and write forced hop step when no model work (step structure TBD by verification)
   - Update `status`, `priority`, `confidence`, `flagged` on forced hops (field names and update pattern TBD by verification)
   - **Verify first:** Flag field name and schema, pause mechanism (interrupt vs routing), orchestrator decision logic structure, step write pattern, state update pattern

4. **`src/agents/skills/client.py`** (Story 4)
   - Extract usage from `ModelGateway.complete` response (response structure TBD by verification)
   - Return `(result, usage_data)` instead of just `result` (current return type TBD by verification: sync return, async return, generator?)
   - Update all calling nodes to unpack tuple (calling nodes TBD by verification: must search all files in `src/agents/nodes/`)
   - **Verify first:** Current `run_skill` signature and return type, how `run_skill` calls `ModelGateway.complete`, JSON retry logic location, all nodes that call `run_skill`

5. **`src/agents/tools/bindings.py`** (Story 4)
   - Extract usage from each MCP tool loop round (loop structure TBD by verification)
   - Return `(final_response, usage_data)` instead of just `final_response` (current return type TBD by verification)
   - Update calling nodes to unpack tuple (calling nodes TBD by verification)
   - **Verify first:** MCP tool loop structure and function signature, how loop calls `ModelGateway.complete`, calling nodes

6. **`src/agents/model/gateway.py`** (Story 5)
   - Accept optional `node_name` parameter in `complete` method (current signature TBD by verification)
   - Return `(response, usage_data)` when `node_name` provided, else return `response` only (current return type TBD by verification)
   - **Verify first:** Current `complete` signature, response object structure (where `usage` metadata lives), whether usage is already logged, whether `run_skill` and MCP loop pass `node_name` (they must NOT)

7. **`src/agents/runtime/runner.py`** (Story 6)
   - Sum `State.usage` across all nodes when run ends or pauses (end/pause detection TBD by verification)
   - Log totals: `calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
   - **Verify first:** Which function executes the graph, how run end is detected (actual terminal node name), how run pause is detected (field names and detection logic)

8. **`src/agents/runtime/backend.py` or `server.py`** (Story 6)
   - Include `State.usage` in `GET /runs/{thread_id}` response under `values` (response structure TBD by verification)
   - **Verify first:** Which file implements the endpoint, response structure, whether filtering is applied to `values`

---

### New Test Files

1. **`tests/test_forced_routing.py`** (Story 2)
   - Tests use actual verified node names, not placeholders
   - Tests call actual `broken_rules` with verified signature
   - Tests verify actual return value from `forced_route` matches actual `broken_rules` contract

2. **`tests/test_orchestrator_forced.py`** (Story 3)
   - Tests use actual verified flag schema
   - Tests use actual verified pause detection mechanism (if interrupt-based, tests check interrupt conditions, not route names)
   - Tests verify actual forced hop step structure matches actual model hop structure

3. **`tests/test_usage_tracking.py`** (Stories 4, 5, 6)
   - Tests use actual verified response structure from `ModelGateway.complete`
   - Tests verify actual calling nodes are updated (all nodes identified by verification, not assumed list)
   - Tests use actual verified state update pattern

4. **`tests/test_ac_per128.py`** (Integration)
   - Full acceptance test with mocked gateway and canned state sequence
   - Uses all verified node names, field names, and structures
   - Asserts forced hops, non-forced hops, usage aggregation, and final logs

---

### Modified Test Files

1. **`tests/test_backend.py`**
   - Verify no regressions in existing endpoint tests
   - Update assertions if tests inspect step counts or evidence (account for forced hops with `attempts: 0`)

2. **`tests/test_evaluate.py`**
   - Verify no regressions in existing evaluation tests
   - Update assertions if tests inspect call counts (expect lower counts with forced routing)

---

## Verification Checklist (PRE-IMPLEMENTATION GATE)

**MANDATORY FOR EACH STORY:** Complete verification BEFORE writing any implementation code. Document findings in `stories/<story-number>/verification-findings.md` BEFORE writing `stories/<story-number>/implementation.md` or any code.

**Story 1 (State schema):**

- [ ] Read `src/agents/graph/states.py` (full file or lines 1-200)
- [ ] Create `stories/01-agent/verification-findings.md` with:
  - [ ] Current `State` dataclass structure (list all fields with types and defaults)
  - [ ] Reducer registration pattern (show example from existing field)
  - [ ] Checkpoint loading location and mechanism (show relevant code snippet)
  - [ ] Actual flag field name (show line number and field definition)
  - [ ] Actual flag dict schema (show example flag dict from code or docstring)
  - [ ] Confirmation that `usage` and `orchestrator_seen_flag_ids` don't conflict with existing field names
- [ ] Answer blocking question: What is the reducer registration pattern? (copy the exact syntax from the file)
- [ ] Do NOT proceed to implementation until findings are documented

**Story 2 (Forced route logic):**

- [ ] Read `src/agents/nodes/orchestrator/route_rules.py` (search for `def broken_rules`)
- [ ] Read `src/agents/nodes/__init__.py` (full file)
- [ ] Read `src/agents/graph/build.py` (search for `"__end__"` or `END` or terminal marker)
- [ ] Read `src/agents/graph/interrupts.py` (search for `interrupt_before`)
- [ ] Run `ls src/agents/nodes/` and inspect directory names
- [ ] Create `stories/02-agent/verification-findings.md` with:
  - [ ] Exact `broken_rules` signature (copy line with function definition)
  - [ ] `broken_rules` return type and validation convention (show example or docstring)
  - [ ] What `broken_rules` expects for `next_step` (single string vs list vs other)
  - [ ] Whether `NODES` exists and its exact contents (copy the list)
  - [ ] Correct node name format (underscore or hyphen)
  - [ ] Terminal node name and whether it's in `NODES` and whether `broken_rules` accepts it
  - [ ] Pause mechanism (interrupt-based or routing-based)
  - [ ] Pause route node names (if any exist — search for directories)
- [ ] Answer blocking questions (listed in Schema section above)
- [ ] Do NOT proceed to implementation until findings are documented and blocking questions answered

**Story 3 (Orchestrator guards):**

- [ ] Read `src/agents/graph/states.py` (use findings from Story 1 verification)
- [ ] Read `src/agents/nodes/orchestrator/node.py` (search for model decision function, step creation, state updates)
- [ ] Create `stories/03-agent/verification-findings.md` with:
  - [ ] Actual flag field name and schema (from Story 1 findings, re-confirmed)
  - [ ] Pause mechanism (from Story 2 findings, re-confirmed)
  - [ ] Orchestrator main function name (show function definition line)
  - [ ] Model decision function name and signature (show function definition line)
  - [ ] Step dict structure (show example step creation code)
  - [ ] Evidence structure (show keys present in evidence dict)
  - [ ] State field names for `status`, `priority`, `confidence`, `flagged`, `summary` (confirm existence and types)
  - [ ] State update pattern (show how orchestrator returns updated state)
- [ ] Answer blocking questions (listed in Schema section above)
- [ ] Do NOT proceed to implementation until findings are documented and blocking questions answered

**Story 4 (Usage in run_skill/MCP):**

- [ ] Read `src/agents/skills/client.py` (search for `def run_skill`)
- [ ] Read `src/agents/model/gateway.py` (search for response structure)
- [ ] Search `src/agents/nodes/` for all `run_skill` imports and calls: Run `grep -r "run_skill" src/agents/nodes/`
- [ ] Read `src/agents/tools/bindings.py` (search for MCP tool loop)
- [ ] Create `stories/04-agent/verification-findings.md` with:
  - [ ] Current `run_skill` signature (copy function definition line)
  - [ ] Current `run_skill` return type (sync, async, generator, other)
  - [ ] How `run_skill` calls `ModelGateway.complete` (show code snippet)
  - [ ] JSON retry logic location (show relevant code)
  - [ ] Response object structure (how to access `usage` metadata — show example)
  - [ ] All four token field names (verify they exist in response)
  - [ ] All nodes that call `run_skill` (list file paths and line numbers from grep results)
  - [ ] MCP tool loop structure and function signature (copy function definition line)
  - [ ] How MCP loop calls `ModelGateway.complete` (show code snippet)
  - [ ] Calling nodes for MCP loop (list file paths)
- [ ] Answer blocking question: Are there nodes beyond those found by grep that call `run_skill` indirectly?
- [ ] Do NOT proceed to implementation until all calling nodes are identified and documented

**Story 5 (Usage in gateway):**

- [ ] Read `src/agents/model/gateway.py` (if not already read in Story 4)
- [ ] Verify Story 4 implementations DO NOT pass `node_name` parameter (inspect `run_skill` and MCP loop code after Story 4 is merged, or inspect current code if Story 4 not yet started)
- [ ] Create `stories/05-agent/verification-findings.md` with:
  - [ ] Current `complete` signature (copy method definition line)
  - [ ] Current `complete` return type
  - [ ] Response structure and usage metadata location (from Story 4 findings, re-confirmed)
  - [ ] Whether usage is already logged (show logger call if present)
  - [ ] Confirmation that Story 4 implementations do NOT pass `node_name` (show relevant code snippets)
- [ ] Do NOT proceed to implementation until Story 4 non-interference is confirmed

**Story 6 (Logging and API):**

- [ ] Read `src/agents/runtime/runner.py` (search for graph execution function)
- [ ] Read `src/agents/graph/build.py` (use findings from Story 2 verification for terminal node)
- [ ] Read `src/agents/graph/interrupts.py` or `states.py` (use findings from Story 2 for pause detection)
- [ ] Read `src/agents/runtime/backend.py` and `server.py` (search for `/runs/` endpoint)
- [ ] Create `stories/06-agent/verification-findings.md` with:
  - [ ] Graph execution function name (show function definition line)
  - [ ] How run end is detected (terminal node name from Story 2, show code snippet)
  - [ ] How run pause is detected (field names and logic, show code snippet)
  - [ ] Which file implements `GET /runs/{thread_id}` (show route definition line)
  - [ ] Response structure (show how `values` is built)
  - [ ] Whether `State.usage` is automatically included or needs explicit addition (show relevant code)
- [ ] Do NOT proceed to implementation until endpoint response pattern is confirmed

---

## Risks

### Risk 1: Verification not completed before implementation

**Impact:** Developer writes code based on HLD assumptions, discovers at runtime that assumptions are wrong, must rewrite implementation and tests

**Likelihood:** **HIGH** if verification checklist is not enforced as a blocking gate

**Mitigation:**

- **MANDATE:** No code may be written for a story until its `verification-findings.md` is complete and reviewed
- Code review rejects any PR where verification findings are missing or incomplete
- Integration tests surface mismatches between assumptions and actual code

**Detection:**

- PR lacks `stories/<story-number>/verification-findings.md` file
- Verification findings document says "TBD" or "assumed" instead of showing actual code snippets
- Tests fail with TypeErrors or AttributeErrors during integration

---

### Risk 2: `broken_rules` contract unknown until Story 2 verification

**Impact:** Story 2 cannot implement `forced_route` until `broken_rules` signature is known; if assumptions are wrong, HLD AD-2 contract is unimplementable

**Likelihood:** **CERTAIN** — `broken_rules` contract is currently unknown from specs

**Mitigation:**

- Story 2 verification checklist requires reading `broken_rules` definition and documenting signature, return type, and calling convention
- If `broken_rules` rejects the format `forced_route` needs (e.g., cannot accept `["compliance", "validation"]` for parallel agents), Story 2 findings must document this and propose a solution (e.g., call `broken_rules` once per agent in the set and require all to pass)
- HLD may need revision if `broken_rules` contract is incompatible with AD-2 design intent

**Detection:**

- Story 2 verification findings document a `broken_rules` signature that conflicts with HLD assumptions
- Integration tests fail when `forced_route` calls `broken_rules` with a format it doesn't accept

---

### Risk 3: Flag schema unknown until Story 1 verification

**Impact:** Story 3 cannot implement unseen flag guard until flag schema is known; if flag dicts don't have unique IDs, `orchestrator_seen_flag_ids` cannot track which flags have been seen

**Likelihood:** **HIGH** — flag schema is currently unknown from specs

**Mitigation:**

- Story 1 verification checklist requires reading `states.py` and documenting flag field name and dict schema
- If flags don't have unique IDs, Story 1 findings must propose a solution (e.g., generate ID from flag content hash, or track seen flags by content instead of ID)
- Story 3 verification must re-confirm flag schema and adjust unseen flag guard logic accordingly

**Detection:**

- Story 1 verification findings document a flag schema without unique IDs
- Integration tests fail when orchestrator tries to extract flag IDs that don't exist

---

### Risk 4: Pause detection mechanism unknown until Story 2 verification

**Impact:** Story 3 Guard 1 (pause routes require summary) cannot be implemented until pause mechanism is known; if pause is interrupt-based, checking `route` names will never trigger the guard

**Likelihood:** **HIGH** — repo tree shows `interrupts.py` but no `review/` node directory, suggesting interrupt-based pause

**Mitigation:**

- Story 2 verification checklist requires reading `interrupts.py` and documenting pause mechanism
- If pause is interrupt-based (no separate pause route nodes), Story 2 findings must document how to detect an impending pause (e.g., check `state` for interrupt condition)
- Story 3 verification must adjust Guard 1 to check the correct condition (interrupt flag or state condition, not route name)

**Detection:**

- Story 2 verification findings document interrupt-based pause with no pause route nodes
- Integration tests fail when orchestrator's pause guard never triggers (Guard 1 checks non-existent routes)

---

### Risk 5: All `run_skill` calling nodes unknown until Story 4 verification

**Impact:** Story 4 changes `run_skill` return type to tuple, but misses updating some calling nodes; those nodes crash with TypeError when they try to use the result as a dict

**Likelihood:** **MEDIUM** if grep search is incomplete (e.g., misses indirect calls)

**Mitigation:**

- Story 4 verification checklist requires running `grep -r "run_skill" src/agents/nodes/` and documenting all results
- Verification findings must list file paths and line numbers for every call
- Code review verifies that all identified calling nodes are updated in the same PR
- Integration tests exercise all nodes, surfacing crashes if any are missed

**Detection:**

- Story 4 verification findings list only `planner` but integration tests crash in `orchestrator` or other nodes
- Grep search finds more `run_skill` calls than are documented in verification findings

---

### Risk 6: Terminal node handling conflict between HLD and actual code

**Impact:** HLD assumes `forced_route` returns `["__end__"]` after review, but if `"__end__"` is a LangGraph sentinel (not a runnable node) and not in `NODES`, it cannot be enumerated or returned by `forced_route`

**Likelihood:** **HIGH** — repo tree shows no `__end__/` directory, suggesting `"__end__"` is a sentinel

**Mitigation:**

- Story 2 verification checklist requires reading `build.py` and documenting how terminal state is represented
- If `"__end__"` is a sentinel not in `NODES`, Story 2 findings must document whether to include it as a candidate (and whether `broken_rules` accepts it) or exclude it (and revise HLD example)
- If excluded, `forced_route` returns `None` after review (multiple valid routes), and the model decides to route to `"__end__"` — this contradicts HLD AD-2 assumption that after review only `["__end__"]` is valid

**Detection:**

- Story 2 verification findings document `"__end__"` as a sentinel,

---

# Story summaries

- `01-agent`: LLD — Story 01-agent: State schema for usage tracking and seen-flags
- `02-agent`: LLD — Story 02-agent: Forced route logic in route_rules
- `03-agent`: LLD — Story 03-agent: Orchestrator decision guards and forced hops
- `04-agent`: LLD — Story 04-agent: Usage collection in run_skill and MCP tool loop
- `05-agent`: LLD — Story 05-agent: Usage collection in ModelGateway direct calls
- `06-agent`: LLD — Story 06-agent: Aggregate usage logging and API endpoint

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
