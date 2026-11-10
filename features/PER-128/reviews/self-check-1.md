## What would go wrong

### 1. Inconsistent forced_route contract between HLD and Story 2 LLD

**HLD AD-2** states `broken_rules(state, next_step)` where `next_step` is the route parameter, but the **Story 2 LLD implementation sketch** shows:

```python
valid_candidates = [
    candidate for candidate in candidates
    if len(broken_rules(state, candidate)) == 0
]
```

This assumes `broken_rules` returns a list of error strings (empty = valid). However, the HLD also shows examples like `broken_rules(state, candidate)` without specifying what `candidate` is (single node string vs list).

**Problem:** Story 2's LLD cannot be implemented until the actual `broken_rules` signature is verified. The sketch assumes `candidate` is passed as-is (could be `list[str]` like `["planner"]` or `["compliance", "validation"]`), but if `broken_rules` expects a different format (e.g., single string for individual nodes, or requires different handling for parallel sets), the enumeration logic breaks.

**What the developer would get wrong:** They'd write `forced_route` based on the HLD's assumption, then discover at runtime that `broken_rules` rejects the format (e.g., `TypeError: expected str, got list`) or returns a different structure (e.g., dict instead of list), forcing a rewrite. The Story 2 LLD's verification checklist requires reading `broken_rules` first, but the HLD doesn't enforce this order—it just assumes the signature.

---

### 2. Flag schema assumptions conflict between requirements, HLD, and Story 3 LLD

**Requirements R3** says "an unresolved flag exists whose ID is not in `orchestrator_seen_flag_ids`" and states "implementation must verify actual flag schema in `src/agents/graph/states.py` (field name, structure of flag dicts: `id`, `resolved` fields)."

**HLD AD-3** says "Implementation must verify the actual flag schema in `src/agents/graph/states.py`: field name (is it `flags`, `open_flags`, other?), and structure of flag dicts (do they have `id: str` and `resolved: bool` fields, or different names?)."

**Story 3 LLD** shows this guard:

```python
current_flags = getattr(state, 'flags', []) or []  # PLACEHOLDER
flag_ids = {flag.get("id") for flag in current_flags if "id" in flag}
unseen_flags = [f for f in unresolved_flags if f.get("id") not in state.orchestrator_seen_flag_ids]
```

**Problem:** All three documents assume flags are in a field (name unknown) as a list of dicts with keys `id` and `resolved` (names unknown). But the repo tree shows `src/agents/graph/states.py` exists—the actual schema could be completely different (e.g., flags might be a dict keyed by ID, or have different field names like `flag_id`/`is_resolved`, or be stored in a separate table). The Story 3 LLD's implementation sketch uses `getattr(state, 'flags', [])` as a placeholder, but if the actual field is `state.active_flags` or `state.flag_list`, the code breaks. Worse, if flags don't have an `id` key but use `flag_id`, the set comprehension `{flag.get("id") for flag in ...}` produces a set of `None` values, bypassing the unseen flag guard entirely (all flags appear "seen").

**What the developer would get wrong:** They'd copy the Story 3 sketch without verifying the schema, write tests using the assumed field names, and all tests would pass (because test mocks match the assumptions). Then in integration, the orchestrator would either crash with `AttributeError: 'State' object has no attribute 'flags'` or silently skip the unseen flag guard (wrong field accessed, guard never triggers), allowing forced hops when the model should have been called to merge flags. This breaks AC5 from requirements ("model call proceeds on unseen flags").

---

### 3. Pause route detection assumes nodes that may not exist

**HLD AD-3** says "The route is a pause route. Implementation must determine actual pause route node names by inspecting `src/agents/nodes/` directory (not assumed to be `["flag-review"]` or `["review"]`—verify actual node names)."

**Story 3 LLD** shows this guard:

```python
PAUSE_ROUTES = ["VERIFIED_PAUSE_ROUTE_1", "VERIFIED_PAUSE_ROUTE_2"]  # Replace with verified names
if route in [[r] for r in PAUSE_ROUTES]:
    decide_result = _decide(state)
```

**But** the repo tree lists these node directories:

```
- src/agents/nodes/compliance/node.py
- src/agents/nodes/execution/fetch.py
- src/agents/nodes/execution/node.py
- src/agents/nodes/orchestrator/node.py
- src/agents/nodes/output_validator/__init__.py
- src/agents/nodes/payroll_calculation/node.py
- src/agents/nodes/planner/node.py
- src/agents/nodes/validation/node.py
```

**No `review/` or `flag-review/` directories exist.** The repo tree shows `src/agents/graph/interrupts.py`, suggesting pause logic is handled via LangGraph's `interrupt_before` mechanism, not separate nodes.

**Problem:** Guard 1 in Story 3 checks `if route == ["review"]`, but no such route exists. The guard never triggers because the enumerated candidates from Story 2 (based on `NODES` from `__init__.py`) won't include a non-existent `review` node. If pause is handled by `interrupt_before` in the graph builder (not routing), the guard is checking the wrong thing.

**What the developer would get wrong:** They'd implement Guard 1 as written, test it with a mocked `["review"]` route (test passes because the mock matches the code), then deploy. In production, when the orchestrator actually needs to pause (e.g., flagged for human review), `forced_route` returns the next valid route (say `["__end__"]`), Guard 1 doesn't match, the forced hop proceeds without calling the model to write a summary, and the pause occurs with a missing or stale summary. This breaks AC4 from requirements ("model call proceeds on pause routes").

---

### 4. Story 4's `run_skill` return type change breaks all calling nodes, but the story doesn't identify them all

**Story 4 LLD** says:

> "Return usage alongside skill result: Instead of returning only the skill result, return `(result, usage_data)`... All calling nodes (e.g., planner) includes this in its state update."

**And later:**

> "All calling nodes (must be identified by inspecting `src/agents/nodes/` directory) must be updated to handle the new return type."

**The repo tree shows these node files:**

```
- src/agents/nodes/compliance/node.py
- src/agents/nodes/execution/node.py
- src/agents/nodes/orchestrator/node.py
- src/agents/nodes/output_validator/node.py
- src/agents/nodes/payroll_calculation/node.py
- src/agents/nodes/planner/node.py
- src/agents/nodes/validation/node.py
```

**Problem:** Story 4 identifies only `planner` as an example calling node, but the orchestrator likely also calls `run_skill` (from `src/agents/skills/orchestrator/skill.md`), and possibly output_validator and others. The story's implementation checklist has "All calling nodes in `src/agents/nodes/` — update to unpack new return type and include usage in state updates," but it doesn't list *which* nodes those are. The developer must search the entire `nodes/` directory for `run_skill` calls before changing the return type.

**What the developer would get wrong:** They'd update `run_skill` to return a tuple, update `planner/node.py` (the example), run the Story 4 unit tests (which mock the calling nodes), see them pass, and merge. In integration, when the orchestrator's model hop calls `run_skill`, it does `result = run_skill(...)` (expecting a dict), gets a tuple, then tries `result["route"]` and crashes with `TypeError: tuple indices must be integers, not str`. The existing tests in `tests/test_backend.py` fail because the orchestrator is broken, but this isn't caught until after Story 4 is "complete."

---

### 5. Story 2's `NODES` list conflicts with HLD's terminal node assumption

**HLD AD-2** says "Terminal node name (not assumed to be `__end__`)."

**Story 2 LLD verification checklist** says "Identify terminal node name (search for end-state logic, likely `__end__` but verify)."

**Story 2 LLD implementation sketch** shows:

```python
NODES = ["planner", "compliance", "payroll_calculation", "validation", 
         "execution", "output_validator", "__end__"]  # PLACEHOLDER
```

**But** the repo tree shows no `__end__/` directory in `src/agents/nodes/`. LangGraph's convention is that `"__end__"` is a special marker (not a real node), used in edges to signal graph termination. It's not in `NODES` because it's not a runnable node—it's a sentinel value.

**Problem:** If `NODES` includes `"__end__"`, then `forced_route` will enumerate `["__end__"]` as a candidate and call `broken_rules(state, ["__end__"])`. If `broken_rules` is only designed to validate *runnable* nodes (not sentinel values), it might reject `["__end__"]` as invalid or crash. Conversely, if `NODES` excludes `"__end__"`, then after a review completes, `forced_route` enumerates all real nodes and finds multiple valid (e.g., review is done, but `broken_rules` allows re-running stage agents or going to execution). The HLD assumes `forced_route` returns `["__end__"]` after review, but if `"__end__"` isn't a candidate, it can't be returned.

**What the developer would get wrong:** They'd add `"__end__"` to `NODES` based on the HLD's examples, then `broken_rules(state, ["__end__"])` returns errors like `"__end__ is not a valid node"`, so `forced_route` never returns `["__end__"]` (always `None`), forcing the model to decide every end-of-run hop. This contradicts the HLD's example "After review, only `["__end__"]` is valid." Alternatively, if they exclude `"__end__"` from `NODES`, the test `test_forced_route_after_review` expects `forced_route` to return `["__end__"]`, but it returns `None` (because `"__end__"` wasn't enumerated), and the test fails. The Story 2 LLD says "ADJUST AFTER VERIFICATION," but the verification steps don't clarify whether to include sentinel values—they just say "document node names from directories," and `__end__` has no directory.

---

## Notes for the gate

**Verification mandate not enforced:** The HLD, breakdown, and story LLDs repeatedly say "implementation must verify" various things (flag schema, `broken_rules` signature, node names, pause routes), but the documents don't enforce that verification happens *before* the developer writes code. The LLDs provide implementation sketches with placeholders and say "ADJUST AFTER VERIFICATION," but a developer could skip verification, use the placeholders as-is (they're close enough to look plausible), write tests that mock the same assumptions, and get green tests without ever checking the real code. The specs should either remove the sketches (forcing the developer to verify first) or make verification a blocking step (e.g., "LLD cannot be marked complete until verification findings are documented").

**Pause detection ambiguity:** If pause is handled via `interrupt_before` (not a separate node), none of the documents explain how to detect it. Story 3's Guard 1 checks if `route` matches a pause route node name, but if there's no such node, the guard never triggers. The developer would need to check `state` for an interrupt flag or condition instead of checking `route`, but this isn't specified.

**`broken_rules` contract unknown:** The entire forced routing feature hinges on `broken_rules`' actual signature and behavior, yet none of the specs verify it before writing code that calls it. Story 2 can't be implemented without this info, and if the assumptions are wrong, Story 3's guards also break (they assume `broken_rules` accepts the same format `forced_route` uses).

**Double-dispatch risk in Story 4/5:** If a node calls `ModelGateway.complete` via `run_skill`, and `run_skill` extracts usage, and the node also passes `node_name` to `complete` (triggering Story 5's usage return), the usage is double-counted. The specs warn against this, but don't enforce it (e.g., by having `run_skill` explicitly pass `node_name=None` to `complete`). A developer could miss the warning and introduce the bug.

VERDICT: FAIL
