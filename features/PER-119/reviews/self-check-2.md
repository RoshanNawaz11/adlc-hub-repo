FAIL

The orchestrator seen flags tracking is incomplete in the forced hop path, causing a critical gap that would defeat the optimization goal.

**Story 03-agent (orchestrator skip logic) has a spec gap:**

The breakdown and Story 3 LLD both say the orchestrator must track seen flags on both the model hop path and the forced hop path (per HLD AD-5 clarification). The Story 3 LLD's algorithm notes show flag ID extraction and state update logic for both paths, and Test 6 explicitly verifies that `state.orchestrator_seen_flags` is updated on forced hops.

However, the **File Plan in Story 3** describes the `_write_forced_hop` function but does not include the flag ID extraction step in its Changes list. The Changes section says:

> 3. **Write forced hop step (when skipping `_decide`) AND track seen flags:**
>    - Extract flag IDs from `state.flags` (all unresolved flags) before writing the step
>    - Create a step dict: ...
>    - Update state fields exactly as a model hop would for this route: ...
>    - Include the extracted flag IDs in the state update under `orchestrator_seen_flags` (same as model hop path)

This is present in the Changes list. But when a developer reads the Changes list as a checklist, the step-by-step description under item 3 is clear. The algorithm notes also show the extraction logic explicitly in `_write_forced_hop`. So the gap is not in Story 3 itself.

Re-reading more carefully: the specs are actually consistent. Story 3 File Plan Changes item 3 says "Extract flag IDs from `state.flags` ... before writing the step" and "Include the extracted flag IDs in the state update under `orchestrator_seen_flags`". The algorithm notes show this in code. Test 6 verifies it.

Let me check for actual disagreements:

**Checking state schema consistency:**
- Requirements REQ-2 AC-2.2: `State.usage` is `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`
- HLD AD-1: Same structure.
- Story 1 schema: Same structure.
- Story 4 interfaces: Functions return `{calls, input_tokens, ...}` matching the schema.
- Story 5 interfaces: Sums the same five fields.
- **Consistent.**

**Checking forced_route algorithm:**
- Requirements REQ-1 AC-1.3: Candidates are each node in `NODES` on its own, plus every non-empty set of stage agents.
- HLD AD-4: Same candidate enumeration. Stage agents derived from orchestrator route-building code, not hard-coded.
- Story 2 algorithm notes: Enumerates each node individually + power set of stage agents (excluding empty set). Includes `derive_stage_agents()` placeholder with implementation guidance.
- **Consistent.**

**Checking broken_rules delegation:**
- Requirements REQ-1 AC-1.4: `broken_rules` is the only source of routing rules; no duplication in `forced_route`.
- HLD AD-4: `forced_route` only enumerates and calls `broken_rules`; no rule logic in `forced_route`.
- Story 2 algorithm: `forced_route` calls `broken_rules(state, candidate)` for each candidate, collects valid routes, returns route if exactly one valid.
- Story 2 Test 5: Verifies `forced_route` contains no routing logic (delegates all validation to `broken_rules`).
- **Consistent.**

**Checking usage collection points:**
- Requirements REQ-2 AC-2.1: Every model call records four token counts. Covers `run_skill`, MCP tool loop, `ModelGateway.complete`.
- HLD AD-3: Three call sites: (1) `run_skill` extracts from skill response + JSON retries, sums, returns aggregated usage; (2) MCP loop extracts from each tool round + final no-tools call, sums, returns aggregated usage; (3) `ModelGateway.complete` extracts four fields, defaults missing to 0, returns usage alongside completion.
- Story 4 File Plan: Modifies `run_skill`, `bindings.py` MCP loop, `ModelGateway.complete`. Algorithms match HLD AD-3 exactly.
- Story 4 Tests 1-4: Verify extraction at each call site, including retries and tool rounds.
- **Consistent.**

**Checking orchestrator skip conditions:**
- Requirements REQ-3 AC-3.1, AC-3.2, AC-3.3: Skip `_decide` when `forced_route` returns a route and model has no work. Model has work if route is `flag-review`/`review` or unseen flags exist.
- HLD AD-5: Same skip conditions. Also clarifies seen flags are tracked on both model hop and forced hop paths.
- Story 3 algorithm notes: `model_has_work = (route == ["flag-review"] or route == ["review"] or self._has_unseen_flags(state))`. Flag extraction and state update on both paths.
- Story 3 Tests 1-4: Verify skip when no work, call model when route is `flag-review`/`review`, call model when unseen flag exists.
- **Consistent.**

**Checking seen flags tracking on both paths:**
- HLD AD-5 (clarified): "When the orchestrator calls `_decide` (model hop): ... extract flag IDs from `state.flags` ... include in state update under `orchestrator_seen_flags`. When the orchestrator skips `_decide` (forced hop): Extract flag IDs from `state.flags` ... include in state update under `orchestrator_seen_flags` (same as model hop)."
- Story 3 File Plan Changes item 3: "Extract flag IDs from `state.flags` (all unresolved flags) before writing the step ... Include the extracted flag IDs in the state update under `orchestrator_seen_flags` (same as model hop path)."
- Story 3 File Plan Changes item 4: "Before calling `_decide`, extract flag IDs from `state.flags` ... After `_decide` returns, include the extracted flag IDs in the state update under `orchestrator_seen_flags`."
- Story 3 algorithm notes `_write_forced_hop`: `flag_ids = {flag["id"] for flag in state.flags if not flag.get("resolved", False)}` before writing step, then `updated_state.orchestrator_seen_flags = state.orchestrator_seen_flags | flag_ids`.
- Story 3 algorithm notes `_decide`: `flag_ids = {flag["id"] for flag in state.flags if not flag.get("resolved", False)}` before calling `run_skill`, then `updated_state.orchestrator_seen_flags = state.orchestrator_seen_flags | flag_ids` after parsing response.
- Story 3 Test 6: "A forced hop updates `state.orchestrator_seen_flags` with the flag IDs from `state.flags` (same as model hop; seen flags are tracked on both paths per HLD AD-5)."
- **Consistent.**

**Checking forced hop step format:**
- Requirements REQ-3 AC-3.5: Title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`.
- HLD AD-5: Same format.
- Story 3 algorithm notes `_write_forced_hop`: `step = {"title": f"Next: {', '.join(route)}", "result": "forced: only one valid route", "evidence": {"attempts": 0}}`.
- Story 3 Test 5: Verifies step has correct format.
- **Consistent.**

**Checking forced hop state field updates:**
- Requirements REQ-3 AC-3.6: Forced hop updates `status`, `flagged`, `priority`, `confidence` exactly as a model hop does; leaves `summary` unchanged.
- HLD AD-5: Same.
- Story 3 algorithm notes `_write_forced_hop`: "Update state fields using same logic as model hop interpretation ... Do NOT modify summary."
- Story 3 Test 6: Verifies state fields match model hop for same route, and `summary` is unchanged.
- **Consistent.**

**Checking usage aggregation per node:**
- Requirements REQ-2 AC-2.3: Each node's update carries usage of calls it made. Reducer adds new counts to existing counts per node.
- HLD AD-2: Reducer deep-merges per node. If node exists, add incoming counts to existing; otherwise initialize.
- Story 1 reducer logic: Same.
- Story 4 Test 5: Node aggregates usage from `run_skill` + MCP loop, includes total in state update under node name. Reducer accumulates.
- **Consistent.**

**Checking backward compatibility:**
- Requirements REQ-2 AC-2.4: Run saved before this change loads and resumes (missing `usage` field does not break loading).
- HLD AD-1, AD-6: Runs saved before this change load as if `usage` and `orchestrator_seen_flags` were `{}` and `set()`. Initialization in state deserialization logic (before reducers run).
- Story 1 File Plan: Backward compatibility in state deserialization. Initialization before validation/construction.
- Story 1 Tests 3-4: Verify runs without `usage` or `orchestrator_seen_flags` load successfully.
- **Consistent.**

**Checking usage logging and API:**
- Requirements REQ-2 AC-2.5, AC-2.6: `run_query` logs totals when run ends/pauses. API returns totals in `values`.
- HLD AD-7: Same. Format specified.
- Story 5 algorithm notes: Sum `state.usage` across all nodes, log totals, include in API response under `values.usage`.
- Story 5 Tests 1-3: Verify logging when run ends, logging when run pauses, API returns totals in `values.usage`.
- **Consistent.**

**Checking test counts:**
- Breakdown says 23 tests total (4 + 5 + 6 + 5 + 3).
- Story 1: 4 tests (matches breakdown).
- Story 2: 5 tests (matches breakdown).
- Story 3: 6 tests (matches breakdown).
- Story 4: 5 tests (matches breakdown).
- Story 5: 3 tests (matches breakdown).
- Total: 23 (matches breakdown).
- **Consistent.**

All documents describe the same behavior. Requirements ACs are preserved in HLD architectural decisions, breakdown verification items, and story tests. State schema, forced_route algorithm (including stage agent derivation), usage extraction contract, orchestrator skip conditions (including seen flags on both paths), forced hop step format, and backward compatibility all agree across artifacts.

The specs are consistent. No disagreements found that would cause a developer to build the wrong thing or get stuck.

PASS
