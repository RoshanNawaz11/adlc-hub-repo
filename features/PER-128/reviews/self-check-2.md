## What would go wrong

### 1. Flag schema inconsistency between specs

**Requirements R1** says:
> `State.usage` gets `usage`: `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`, with a reducer that adds a node's new counts to its existing ones.

**Requirements R3** says:
> Track which flags the orchestrator has seen in a new state field, set from the flag ids in each brief it receives.

**Breakdown Story 1** says:
> **Before implementation**, verify from `src/agents/graph/states.py`:
>   - Actual flag field name and dict structure in `src/agents/graph/states.py` (field name, dict structure with `id` and `resolved` fields, or different names)

**Story 03-agent verification checklist Step 1** says:
> Document actual flag field name (search for "flag" in the file—is it `flags`, `open_flags`, `active_flags`, `flagged`, other?)
> Document flag dict schema if flags are a list: what keys exist? (e.g., `id`/`flag_id`, `resolved`/`is_resolved`, `message`, other?)

**Problem:** Requirements, HLD, breakdown, and Story 1 all defer flag schema verification to implementation time. Story 3 Step 1 requires the developer to "re-confirm" from Story 1 findings, but Story 1 doesn't produce those findings—it only says "verify before coding" without documenting what was verified. The developer implementing Story 3 would have to re-read `states.py` themselves because Story 1's LLD contains no concrete findings.

**What the developer would build wrong:** Story 3's Guard 2 (unseen flags check) would use placeholder code like `flag.get("<verified_resolved_field>")` because Story 1 never documented what `<verified_resolved_field>` actually is. The code wouldn't compile, or would use wrong field names and fail at runtime.

---

### 2. Pause detection mechanism unknown until Story 3, but Story 2 tests depend on it

**HLD AD-3** says:
> **Guard 1 (pause routes require summary):**
>   - **Before implementation**, verify from `src/agents/graph/interrupts.py` and `src/agents/nodes/` directory:
>     - How pauses are triggered (via `interrupt_before` condition in graph builder, state flags, or routing to a pause node)

**Story 02-agent (Forced route logic) verification checklist Step 4** says:
> Search `src/agents/nodes/` directory for any review-related node directories (e.g., `review/`, `flag-review/`, `human_review/`)
> Read `src/agents/graph/interrupts.py` (search for pause or interrupt logic)
> Document how pause is triggered (routing to a pause node vs `interrupt_before` condition)

**Story 02-agent Test 3** says:
> **Given** the graph has reached terminal state (detected by next step being LangGraph's `"__end__"` sentinel, verified from `src/agents/graph/build.py` edge logic—not a real node directory)
> **When** `forced_route(state)` is called
> **Then** it returns `["__end__"]` if `"__end__"` is included as a candidate, OR returns `None` if `"__end__"` is excluded from enumeration (implementation decision based on whether `broken_rules` accepts sentinel values).

**Problem:** Story 2's Test 3 depends on a decision about `"__end__"` enumeration, which depends on whether `broken_rules` accepts it. Story 2's Step 1 verification says "document whether `broken_rules` accepts sentinel values," but Story 2 doesn't **make** the decision—it just says "decision based on whether `broken_rules` accepts sentinel values." The test assertion changes depending on the decision, but the LLD doesn't say which assertion to use.

Story 2's Step 4 verification investigates pause mechanism (interrupt vs routing), but the **HLD says Guard 1 is in Story 3**. If Story 2 discovers that pause is interrupt-based (no pause route nodes), it would document "no pause routes exist," but then Story 2's candidate enumeration wouldn't include them anyway. The verification is in the wrong story.

**What the developer would build wrong:** Story 2's Test 3 would have two conflicting assertions (`assert result == ["__end__"]` vs `assert result is None`), with a comment saying "depends on decision." The developer wouldn't know which to keep. Story 3's Guard 1 would check `route == ["review"]`, but if Story 2's findings show no review node exists, Guard 1's check would never trigger.

---

### 3. `run_skill` return type change breaks calling nodes, but Story 4 doesn't list them all

**Story 04-agent (Usage in run_skill/MCP) File Plan** says:
> **Identification required (MANDATORY PRE-IMPLEMENTATION):**
> 1. **Search for `run_skill` calls:** Run `grep -r "run_skill" src/agents/nodes/ --include="*.py"` to find all files that call `run_skill`.

**Story 04-agent AC5** says:
> **Given** `run_skill` or `run_tools` return type changes to `tuple[dict, dict]`
> **When** all nodes in `src/agents/nodes/` that call these functions are identified
> **Then** each calling node is updated to unpack `(result, usage_data) = run_skill(...)` and include `{"usage": {node_name: usage_data}}` in its returned state update dict.

**Story 04-agent "Calling Nodes" section** says:
> Based on repo tree structure, likely calling nodes include:
> - `src/agents/nodes/planner/node.py` (confirmed: planner skill exists)
> - `src/agents/nodes/orchestrator/node.py` (confirmed: orchestrator skill exists)
> - … (list of 7 nodes)

**Problem:** Story 4's LLD lists "likely calling nodes" from the repo tree, but says "MUST be confirmed by grep." The grep command would reveal the actual calling nodes, but the LLD already wrote implementation sketches for 7 specific nodes. If the grep finds an 8th node (or finds that one of the 7 doesn't actually call `run_skill`), the implementation sketches are wrong.

The LLD says "Search for `run_skill` calls" is **MANDATORY PRE-IMPLEMENTATION**, but then provides concrete implementation code for specific nodes **before that search happens**. A developer following the LLD top-to-bottom would read the implementation sketches, start coding, and only later discover the grep results contradict the sketches.

**What the developer would build wrong:** They'd update the 7 nodes listed in the LLD, but miss an 8th node that the grep found. That 8th node would still call `run_skill` expecting a `dict` return, but receive a `tuple`, causing a `TypeError: 'tuple' object has no attribute 'get'` when it tries to access the result. The code would crash in production on the first hop to that node.

---

### 4. Story 5 and Story 4 could double-count usage if `run_skill` passes `node_name`

**Story 05-agent (Usage in gateway) Overview** says:
> When a caller provides a `node_name`, the gateway extracts usage metadata from the model response and returns it alongside the response, allowing the caller to include it in their state update. If `node_name` is not provided, usage is logged only (existing behavior).

**Story 05-agent Edge Case 5** says:
> **Scenario:** `run_skill` or `run_tools` (from Story 4) passes `node_name` to `gateway.complete`.
> **Handling:** Usage is returned in the tuple, but `run_skill`/`run_tools` also extract usage themselves. Both extracted usage dicts are included in the state update, causing double-counting. **This is incorrect behavior.**
> **Mitigation:** **Story 4 callers (`run_skill`, `run_tools`) must NOT pass `node_name` to `gateway.complete`.**

**Story 04-agent `run_skill` implementation sketch** says:
> ```python
> response = gateway.complete(
>     messages=[...],
>     **kwargs
> )
> ```

**Problem:** Story 5's Edge Case 5 says "Story 4 callers must NOT pass `node_name`," and Story 4's sketch doesn't pass it, but **nothing enforces this at the interface level**. Story 5 adds `node_name` as an optional parameter to `gateway.complete`, so `run_skill` **could** accidentally pass it (e.g., if a developer thinks "I'm tracking usage, so I should pass `node_name`"). The interface allows it, but the documented contract forbids it.

Story 5's mitigation is "Document this in the `complete` docstring," but documentation alone doesn't prevent the error. The developer implementing Story 4 could miss the note in Story 5's docstring (they're reading Story 4's LLD, not Story 5's), or a future refactor could add `node_name` without realizing it causes double-counting.

**What the developer would build wrong:** A well-meaning developer implementing Story 4 reads that `gateway.complete` now has a `node_name` parameter (from Story 5's changes merged earlier), thinks "I'm calling this from a node, so I should provide `node_name` for better tracking," and writes:
```python
response = gateway.complete(
    messages=[...],
    node_name="planner",  # WRONG — causes double-counting
    **kwargs
)
```
The code compiles and runs, but every `run_skill` call now increments `State.usage[node_name]` **twice**: once from `gateway.complete`'s returned usage, and once from `run_skill`'s own extraction. The logged totals would be 2x the actual calls.

---

### 5. Story 6 pause detection uses different condition than Story 3 orchestrator pause detection

**Story 06-agent (Logging and API) "Run End/Pause Detection" algorithm notes** say:
> **Run pause detection (per HLD AD-4):**
> - `result["next"]` is empty AND `state.get("status") != "complete"` (no more nodes to execute, not yet complete)
> - OR `result.get("interrupted") == True` (graph paused due to `interrupt_before`)

**Story 03-agent (Orchestrator guards) Guard 1** says:
> **Guard 1 (Pause detection):**
>   - **Before implementation**, verify from `src/agents/graph/interrupts.py` and `src/agents/nodes/` directory:
>     - How pauses are triggered (via `interrupt_before` condition in graph builder, state flags, or routing to a pause node)
>   - **If pause routes exist as separate nodes:** check if `route` matches a verified pause route node name → call `_decide` (summary required).
>   - **If pause is interrupt-based (no separate nodes):** check if the next action triggers an interrupt condition (e.g., `state.needs_review == True`, or `route` leads to a state where `interrupt_before` applies) → call `_decide` (summary required).

**Problem:** Story 6's pause detection checks `result["next"]` (the graph runner's view of pause), but Story 3's Guard 1 checks state conditions **before routing** (the orchestrator's view of pause). These are different moments in time:
- **Story 3 (orchestrator):** Checks if forcing this route would **trigger** a pause (future state).
- **Story 6 (runner):** Checks if the graph **has already paused** (current state).

If the orchestrator's pause detection uses `state.needs_review`, but the graph runner's pause uses `interrupted == True`, they might disagree. For example:
- Orchestrator sees `state.needs_review == False`, forces route to `__end__`, skips model.
- But `__end__` sets `interrupted = True` (e.g., via `interrupt_before` checking a different condition).
- Graph pauses, Story 6 logs usage.
- But the orchestrator didn't write a summary because it thought no pause was needed.

The specs never say these two pause checks **must use the same condition**. Story 3's verification says "verify from `interrupts.py`," Story 6's algorithm says "per HLD AD-4," but HLD AD-4 doesn't say whether Guard 1's check and Story 6's check should match.

**What the developer would build wrong:** Story 3's Guard 1 checks `state.flagged == True` (from `interrupts.py` verification), but Story 6's pause detection checks `result["interrupted"] == True` (from HLD AD-4 wording). A run with `flagged = True` would trigger Guard 1 → model writes summary. But if the model's response clears the flag (`flagged = False` in the update), and `interrupted` was never set, the graph doesn't pause. The orchestrator wrote a summary for a pause that never happened, wasting tokens.

---

## Notes for the gate

- **Flag schema verification is circular:** Story 1 says "verify flag schema before coding," Story 3 says "re-confirm from Story 1 findings," but Story 1's LLD produces no findings document—it just repeats "verify before coding." Either Story 1 must produce a `verification-findings.md` with the actual schema (not just instructions to find it), or Story 3 must do the verification itself and not rely on Story 1.

- **`"__end__"` enumeration decision is unresolved:** Story 2's Test 3 has two conflicting assertions and says "depends on decision based on whether `broken_rules` accepts sentinel values." The LLD must make the decision and pick one assertion, or the test can't be written. The decision can't be deferred to "during implementation" because the test assertion is in the LLD.

- **`run_skill` calling nodes list is speculative:** Story 4 lists 7 "likely calling nodes" before running the grep that would confirm them. The implementation sketches assume those 7 are correct, so if the grep finds different nodes, the sketches are wrong. Either remove the sketches (too specific for an LLD), or run the grep before writing the LLD and document the **actual** list.

- **Double-counting mitigation is documentation-only:** Story 5's Edge Case 5 says "`run_skill` must NOT pass `node_name`," but the interface allows it. A runtime check (e.g., `assert 'node_name' not in kwargs` in `run_skill`) or a type system constraint (separate `complete` methods for direct vs skill calls) would prevent the error. Documentation alone relies on every developer reading both Story 4 and Story 5's docstrings.

- **Pause detection has two definitions:** Story 3's Guard 1 checks if forcing a route **would cause** a pause (preventive), Story 6's logging checks if a run **has paused** (reactive). These operate at different times and could use different conditions. The specs should say explicitly whether they must use the same condition, or if divergence is acceptable (e.g., Guard 1 is conservative, Story 6 is precise).

VERDICT: FAIL
