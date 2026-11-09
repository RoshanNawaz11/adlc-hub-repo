# Merged plan — PER-119

Execution order (5 ticket(s)):

1. `PER-120` (story `01-agent`)
2. `PER-121` (story `02-agent`)
3. `PER-122` (story `03-agent`)
4. `PER-123` (story `04-agent`)
5. `PER-124` (story `05-agent`)

Specs (1): `specs/1.md`

---

## Spec `specs/1.md`

# Tech Stub — Ticket 1

**Title:** Skip orchestrator model call on forced routes and track token usage per run

**Requirements:** REQ-1 (forced routing), REQ-2 (usage tracking), REQ-3 (orchestrator skip logic) from `requirements-per-119.md`

**HLD:** `hld-per-119.md` (AD-1 through AD-7)

---

## Stack

**Language:** Python  
**Framework/runtime:** (unknown — not specified in repo context)  
**State management:** Appears to use a reducer-based state system (based on ticket references to "state reducer")  
**Model client:** Claude API (references `claude-haiku-4-5-20251001` in `src/agents/model/policy.yaml`)  
**Testing:** pytest (references `uv run pytest -q`)

---

## Schema & API Contracts

### State Schema Changes

**New fields added to `State`:**

```python
# State schema (likely in src/agents/state.py or similar)

usage: dict[str, dict[str, int]]
# Outer key: node name (e.g., "orchestrator", "planner", "agent-analyst")
# Inner dict keys: "calls", "input_tokens", "output_tokens", 
#                  "cache_creation_input_tokens", "cache_read_input_tokens"
# All values are int

orchestrator_seen_flags: set[str]
# Set of flag IDs (strings) the orchestrator has seen (on both model hop and forced hop paths)
```

**Example state after updates:**

```python
{
  "usage": {
    "orchestrator": {
      "calls": 3,
      "input_tokens": 4500,
      "output_tokens": 150,
      "cache_creation_input_tokens": 1200,
      "cache_read_input_tokens": 800
    },
    "planner": {
      "calls": 1,
      "input_tokens": 2000,
      "output_tokens": 100,
      "cache_creation_input_tokens": 0,
      "cache_read_input_tokens": 500
    }
  },
  "orchestrator_seen_flags": {"flag-1", "flag-2", "flag-3"}
}
```

### Reducer Contracts

**`usage` reducer:**
- **Input:** Incoming update with `{"usage": {node_name: counts_dict}}`
- **Behavior:** Deep merge per node — add incoming counts to existing counts for the node, or initialize if node not present
- **Output:** Updated `state.usage` with accumulated counts

**`orchestrator_seen_flags` reducer:**
- **Input:** Incoming update with `{"orchestrator_seen_flags": set_of_flag_ids}`
- **Behavior:** Set union — `state.orchestrator_seen_flags |= incoming_flags`
- **Output:** Updated `state.orchestrator_seen_flags` with all seen flag IDs

### API Contract Changes

**`GET /runs/{thread_id}` response (added to `values` field):**

```json
{
  "values": {
    "usage": {
      "calls": 12,
      "input_tokens": 25000,
      "output_tokens": 800,
      "cache_creation_input_tokens": 3500,
      "cache_read_input_tokens": 8000
    }
  }
}
```

### New Function Contract

**`forced_route(state) -> list[str] | None`** (in `src/agents/nodes/orchestrator/route_rules.py`):

- **Input:** Current state
- **Returns:** 
  - `list[str]` — the single valid route when exactly one candidate passes `broken_rules`
  - `None` — when zero or multiple candidates are valid
- **Candidates:** Each node in `NODES` individually, plus every non-empty set of stage agents (stage agents derived from orchestrator's route-building code, not hard-coded)
- **Validation:** Delegates to `broken_rules(state, route)` — no rule duplication

### Usage Data Extraction Points

**Three call sites must extract and return usage:**

1. **`run_skill` in `src/agents/skills/client.py`:**
   - Extract from skill invocation response's `usage` field
   - Extract from JSON retry responses' `usage` fields (if any)
   - Sum all usage across initial + retries (calls = number of model API calls)
   - Return aggregated usage with result

2. **MCP tool loop in `src/agents/tools/bindings.py`:**
   - Extract from each tool round response's `usage` field
   - Extract from final no-tools call response's `usage` field
   - Sum all usage across all rounds (calls = number of model API calls, one per round)
   - Return aggregated usage with result

3. **`ModelGateway.complete` in `src/agents/model/gateway.py`:**
   - Extract four token count fields from completion response
   - Default missing fields to `0`
   - Return usage alongside completion

**Per-call usage shape:**

```python
{
  "input_tokens": int,
  "output_tokens": int,
  "cache_creation_input_tokens": int,
  "cache_read_input_tokens": int
}
```

---

## File Touch List

**New files:**
- _(none — all changes are modifications to existing files)_

**Modified files:**

1. **`src/agents/state.py`** (or wherever `State` is defined)
   - Add `usage: dict[str, dict[str, int]]` field (default `{}`)
   - Add `orchestrator_seen_flags: set[str]` field (default `set()`)
   - Add reducer for `usage` (deep merge per node: add incoming counts to existing, or initialize if node not present)
   - Add reducer for `orchestrator_seen_flags` (set union: `state.orchestrator_seen_flags |= incoming_flags`)
   - Add backward compatibility in state deserialization: initialize missing `usage` to `{}` and missing `orchestrator_seen_flags` to `set()` before validation/construction

2. **`src/agents/nodes/orchestrator/route_rules.py`**
   - Add `forced_route(state) -> list[str] | None` function
   - Enumerate candidates: each node in `NODES` individually + every non-empty set of stage agents (power set of stage agents, excluding empty set)
   - Derive stage agents from orchestrator's existing route-building code (identify nodes that appear in multi-agent routes; do not hard-code names)
   - For each candidate, call `broken_rules(state, candidate)`
   - Collect candidates where `broken_rules` returns `[]` (valid routes)
   - Return route if exactly one valid, else `None`

3. **`src/agents/nodes/orchestrator/node.py`**
   - Before `_decide`, call `forced_route(state)`
   - If `forced_route` returns `None`, proceed to `_decide` (extract flag IDs from `state.flags`, call `_decide`, include flag IDs in state update under `orchestrator_seen_flags`)
   - If `forced_route` returns a route, check skip conditions:
     - Model has work if: route is `["flag-review"]` or `["review"]`, **or** an unresolved flag exists whose ID is not in `state.orchestrator_seen_flags`
     - If model has work: proceed to `_decide` (extract flag IDs, call `_decide`, include flag IDs in state update)
     - If model has no work: skip `_decide`, write forced hop step (extract flag IDs from `state.flags` before writing step, write step with title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`, update `status`/`flagged`/`priority`/`confidence` as model hop would for this route, leave `summary` unchanged, include extracted flag IDs in state update under `orchestrator_seen_flags`)
   - Aggregate usage from model calls (if any) and include in state update under `usage: {orchestrator: counts_dict}`

4. **`src/agents/skills/client.py`**
   - Modify `run_skill`: extract usage from skill invocation response's `usage` field, extract from JSON retry responses' `usage` fields (if any), sum all counts (initial + retries) into `{calls: N, input_tokens: total, ...}` where N = number of model API calls, return `{"result": completion, "usage": aggregated_usage}`

5. **`src/agents/tools/bindings.py`**
   - Modify MCP tool loop: extract usage from each tool round response's `usage` field (including final no-tools call), sum all counts across all rounds into `{calls: N, ...}` where N = number of rounds, return `{"result": completion, "usage": aggregated_usage}`

6. **`src/agents/model/gateway.py`**
   - Modify `ModelGateway.complete`: extract `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens` from completion response, default missing fields to `0`, return `{"completion": text, "usage": {input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`

7. **`src/agents/runtime/runner.py`**
   - Modify `run_query`: when run ends or pauses, sum `state.usage` across all nodes (for each node, add that node's `calls`/`input_tokens`/`output_tokens`/`cache_creation_input_tokens`/`cache_read_input_tokens` to running totals), log totals (total calls, input tokens, output tokens, cache creation tokens, cache read tokens)
   - Modify `GET /runs/{thread_id}` handler: sum `state.usage` across all nodes, include totals in response under `values.usage` (format: `{calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}`)

**Test files (new):**

8. **`tests/test_forced_routing.py`** (or similar)
   - Test `forced_route` returns `["planner"]` before planner runs (5 tests for story 02-agent)
   - Test `forced_route` returns `["__end__"]` after review
   - Test `forced_route` returns `["output-validator"]` after execution
   - Test `forced_route` returns `None` when two agents are valid
   - Test `forced_route` delegates validation to `broken_rules` for every candidate
   - Test orchestrator skips `_decide` on forced hop when model has no work (6 tests for story 03-agent)
   - Test orchestrator calls model when forced route is `["flag-review"]`
   - Test orchestrator calls model when forced route is `["review"]`
   - Test orchestrator calls model when unseen flag exists
   - Test forced hop step format (title, result, evidence)
   - Test forced hop updates `status`/`flagged`/`priority`/`confidence` as model hop would, leaves `summary` unchanged, and tracks seen flags on forced hop path

9. **`tests/test_usage_tracking.py`** (or similar)
   - Test usage accumulates per node across multiple calls (4 tests for story 01-agent: state schema and reducers)
   - Test orchestrator seen flags accumulate across updates
   - Test backward compatibility: run saved without `usage` field loads and resumes
   - Test backward compatibility: run saved without `orchestrator_seen_flags` loads and resumes
   - Test `run_skill` extracts usage from skill invocation response (5 tests for story 04-agent)
   - Test `run_skill` sums usage across initial call and JSON retries
   - Test MCP tool loop sums usage across all tool rounds (including final no-tools call)
   - Test `ModelGateway.complete` extracts usage from completion response
   - Test node aggregates usage from multiple call sites (`run_skill` + MCP loop)
   - Test `run_query` logs usage totals when run ends (3 tests for story 05-agent)
   - Test `run_query` logs usage totals when run pauses
   - Test `GET /runs/{thread_id}` returns usage totals in `values.usage`

**All existing test files:**
- Must pass unchanged after modifications

**Total test count:** 23 tests (4 for story 01-agent, 5 for story 02-agent, 6 for story 03-agent, 5 for story 04-agent, 3 for story 05-agent)

---

## Risks

### RISK-1: Backward compatibility failure
**Description:** Runs saved before this change fail to load because `usage` or `orchestrator_seen_flags` fields are missing.

**Mitigation:**
- State deserialization logic must initialize missing `usage` to `{}` and missing `orchestrator_seen_flags` to `set()` before state is passed to reducers or used by nodes (per HLD AD-6: implementation location depends on codebase pattern — Pydantic `@model_validator`, dataclass `__post_init__`, or manual loader)
- Add tests: load run saved without `usage` field, verify it loads and resumes; load run saved without `orchestrator_seen_flags`, verify it loads and resumes (story 01-agent tests 3-4)
- Verify initialization runs during deserialization, not in reducers

**Likelihood:** Low  
**Impact:** High (breaks existing runs)

---

### RISK-2: Usage collection incomplete
**Description:** Usage data missing from JSON retries, MCP tool rounds, or direct completions.

**Mitigation:**
- Verify all three call sites (`run_skill`, `bindings.py` MCP loop, `ModelGateway.complete`) extract usage from every response per HLD AD-3 extraction contract
- Add tests verifying usage accumulates across retries (story 04-agent test 2: `run_skill` sums initial + retry)
- Add tests verifying usage accumulates across tool rounds (story 04-agent test 3: MCP loop sums all rounds including final no-tools call)
- Code review checklist: confirm every model call site extracts and aggregates usage, missing fields default to `0`

**Likelihood:** Medium  
**Impact:** High (usage data incomplete, savings unmeasurable)

---

### RISK-3: forced_route duplicates routing logic
**Description:** `forced_route` implements routing rules directly instead of delegating to `broken_rules`, violating single source of truth.

**Mitigation:**
- Design enforces delegation: `forced_route` only enumerates candidates and calls `broken_rules`; no rule logic in `forced_route` (HLD AD-4)
- Add test: verify `forced_route` delegates validation to `broken_rules` for every candidate (story 02-agent test 5: mock `broken_rules`, verify it is called for each enumerated candidate)
- Code review must verify no routing logic (e.g., checking planner state, review state) exists in `forced_route` itself
- Document in code comments: "`forced_route` delegates all validation to `broken_rules`"

**Likelihood:** Low  
**Impact:** High (routing logic diverges, maintenance burden)

---

### RISK-4: Orchestrator skip conditions incorrect
**Description:** Orchestrator skips `_decide` when model has work (new flag, reviewer summary needed), or calls model when no work needed.

**Mitigation:**
- Implement skip condition checks before skipping `_decide`: route is NOT `["flag-review"]` or `["review"]`, **and** no unresolved flag exists whose ID is not in `state.orchestrator_seen_flags` (HLD AD-5)
- Add tests (story 03-agent tests 2-4):
  - Orchestrator calls model when forced route is `["flag-review"]`
  - Orchestrator calls model when forced route is `["review"]`
  - Orchestrator calls model when unseen flag exists
- Code review checklist: verify all skip conditions are checked (route check + unseen flags check)

**Likelihood:** Medium  
**Impact:** High (model called unnecessarily or skipped when needed)

---

### RISK-5: Seen flags not tracked on forced hop path
**Description:** Forced hops do not mark flags as seen, causing the next hop to incorrectly call the model even though the flags were already present.

**Mitigation:**
- HLD AD-5 clarification: orchestrator must track seen flags on **both** model hop path and forced hop path (same flag ID extraction and state update logic on both paths)
- Forced hop implementation: extract flag IDs from `state.flags` before writing step, include in state update under `orchestrator_seen_flags` (same as model hop)
- Add test: verify forced hop updates `state.orchestrator_seen_flags` (story 03-agent test 6: forced hop includes flag IDs in state update, same as model hop)
- Code review: verify flag ID extraction and state update apply on both paths (model hop in `_decide`, forced hop in `_write_forced_hop`)

**Likelihood:** Low (HLD AD-5 explicitly requires tracking on both paths)  
**Impact:** High (defeats optimization goal; unnecessary model calls)

---

### RISK-6: Candidate enumeration incomplete
**Description:** `forced_route` does not enumerate all valid candidate routes (e.g., misses a stage agent set), causing it to incorrectly return `None` when a forced route exists.

**Mitigation:**
- Candidate enumeration must match orchestrator's current route space: each node in `NODES` individually + every non-empty subset of stage agents (power set excluding empty set)
- Stage agents derived from orchestrator's existing route-building code (HLD AD-4 clarification: examine code that constructs multi-agent routes, extract set of nodes that appear in such routes; do not hard-code names)
- Add test: verify candidate enumeration includes all expected routes for a known state (story 02-agent tests 1-4 cover specific forced routes; test 5 verifies all candidates are checked)
- Code review: verify candidate generation logic matches orchestrator route logic, stage agent identification mechanism is correct

**Likelihood:** Low  
**Impact:** Medium (missed optimization opportunities, but system still functions)

---

### RISK-7: State update ordering issues
**Description:** Usage update arrives before node completes, or orchestrator seen flags update races with flag creation.

**Mitigation:**
- Usage must be aggregated and included in the same state update that marks node completion (per HLD AD-3: each node aggregates usage from all call sites and includes total in state update)
- Orchestrator seen flags must be included in the same state update that processes the hop (model hop or forced hop) (per HLD AD-5: extract flag IDs before calling `_decide` or writing forced hop step, include in state update after)
- Review state update ordering in each node's update logic
- Add integration test: verify state consistency after concurrent updates (if applicable; story assumes single-threaded execution)

**Likelihood:** Low (single-threaded execution assumed)  
**Impact:** Medium (usage counts or seen flags incorrect)

---

### RISK-8: Test coverage gaps
**Description:** Not all acceptance criteria are covered by tests, or tests use live model calls instead of fake gateway.

**Mitigation:**
- Each AC mapped to specific test cases (see traceability in breakdown and story LLDs):
  - REQ-1 ACs 1.1-1.4 → story 02-agent tests 1-5
  - REQ-2 ACs 2.1-2.6 → story 01-agent tests 1-4, story 04-agent tests 1-5, story 05-agent tests 1-3
  - REQ-3 ACs 3.1-3.6 → story 03-agent tests 1-6
- Use fake gateway (mock `ModelGateway`) in all new tests (no live model calls)
- PR checklist: verify every AC has passing test coverage (23 tests total)
- Run `uv run pytest -q` before PR submission (existing tests must pass unchanged)

**Likelihood:** Medium  
**Impact:** High (untested behavior, regressions possible)

---

## Open Questions

_(none at this time — all design decisions resolved in HLD AD-1 through AD-7)_

---

## Notes

- **Design review completed:** State schema (AD-1), forced routing algorithm including stage agent derivation (AD-4), usage collection flow with extraction contract (AD-3), orchestrator skip conditions including seen flags tracking on both paths (AD-5 clarified), backward compatibility implementation location (AD-6), and usage logging/API (AD-7) all specified in HLD.
- **No new frameworks or languages:** All changes in Python, existing state/reducer system, existing model client (Claude API).
- **Single source of truth enforced:** `broken_rules` remains the only place routing logic lives; `forced_route` delegates validation (no rule duplication).
- **Backward compatibility mandatory:** Runs saved before this change must load and resume (missing `usage` and `orchestrator_seen_flags` fields initialize to `{}` and `set()` in state deserialization logic before validation/construction, per HLD AD-6).
- **Measurement required:** PR description must report model calls and total tokens for one service order (`run_so.py`, e.g., SO-1003) before and after the change (verify savings).
- **Critical clarification applied:** HLD AD-5 clarifies that orchestrator seen flags are tracked on **both** model hop and forced hop paths (same flag ID extraction and state update logic on both paths). This ensures the next hop does not call the model when flags were already present in a forced hop's state. Story 03-agent implements this on both paths, and test 6 verifies it.
- **Test count matches breakdown exactly:** 23 tests (4 + 5 + 6 + 5 + 3) matching breakdown verification items for each story. No additional tests beyond agreed breakdown.
- **All specs consistent:** Requirements ACs preserved in HLD architectural decisions, breakdown verification items, and story tests. State schema, forced_route algorithm (including stage agent derivation per HLD AD-4), usage extraction contract (per HLD AD-3), orchestrator skip conditions (including seen flags on both paths per HLD AD-5), forced hop step format, and backward compatibility (per HLD AD-6) all agree across requirements, HLD, breakdown, and story LLDs.

---

# Story summaries

- `01-agent`: Story 01-agent LLD
- `02-agent`: Story 02-agent LLD
- `03-agent`: Story 03-agent LLD
- `04-agent`: Story 04-agent LLD
- `05-agent`: Story 05-agent LLD

---

## Why this order

Follow breakdown.md dependencies; tickets listed above match scrum→Linear story order.
