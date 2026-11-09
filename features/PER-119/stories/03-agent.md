# Story 03-agent LLD

**Story:** Orchestrator skip logic and seen flags tracking

**Repo:** `app`

**Epic:** PER-119

---

## Acceptance Criteria

From requirements and breakdown only:

1. **AC-1 (REQ-3 AC-3.1):** When `forced_route` returns a route and the model has no work to do, the orchestrator skips `_decide` and uses that route
2. **AC-2 (REQ-3 AC-3.2):** When `forced_route` returns a route and the route pauses for a person (`flag-review` or `review`), the orchestrator calls `_decide` (the model writes the reviewer's `summary` before a pause)
3. **AC-3 (REQ-3 AC-3.3):** When `forced_route` returns a route and an unresolved flag exists that the orchestrator's model has not seen yet, the orchestrator calls `_decide` (merging duplicate flags needs the model)
4. **AC-4 (REQ-3 AC-3.4):** The orchestrator updates a new state field with the flag IDs from each hop (clarification from HLD AD-5: seen flags are tracked on both model hop and forced hop paths)
5. **AC-5 (REQ-3 AC-3.5):** A forced hop's step has title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`
6. **AC-6 (REQ-3 AC-3.6):** A forced hop updates `status`, `flagged`, `priority`, and `confidence` exactly as a model hop does, and leaves `summary` unchanged

---

## HLD Binding Outcomes

From `hld-per-119.md`:

- **AD-5 (CLARIFIED):** The orchestrator skips `_decide` (and thus the model call) when: (1) `forced_route(state)` returns a route (not `None`), **and** (2) The model has no work to do. The model has work to do when: the forced route is `["flag-review"]` or `["review"]` (the model writes the reviewer's `summary` before a pause), **or** an unresolved flag exists whose ID is not in `state.orchestrator_seen_flags` (merging duplicate flags needs the model). 
  
  **Seen flags tracking on both paths (critical clarification):**
  - **When the orchestrator calls `_decide` (model hop):** Before calling `_decide`, extract flag IDs from `state.flags` (all unresolved flags). After `_decide` returns, include the extracted flag IDs in the state update under `orchestrator_seen_flags`. The reducer merges these IDs with existing `state.orchestrator_seen_flags` via set union.
  - **When the orchestrator skips `_decide` (forced hop):** Extract flag IDs from `state.flags` (all unresolved flags) before writing the forced hop step. Include the extracted flag IDs in the state update under `orchestrator_seen_flags` (same as model hop). The reducer merges these IDs with existing `state.orchestrator_seen_flags` via set union.
  
  This ensures that flags are marked as seen on both the model hop path and the forced hop path. Without this, a forced hop would never mark flags as seen, causing the next hop to incorrectly call the model.

  When the orchestrator skips `_decide`, it writes a step with title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`. It updates `status`, `flagged`, `priority`, and `confidence` exactly as a model hop does (using the same logic that interprets the model's route choice), and leaves `summary` unchanged.

---

## File Plan

**Note:** Repo tree for `app` is not provided. The story specifies `src/agents/nodes/orchestrator/node.py` as the likely location; implementation must verify the actual path in the repo and use it.

### Modified Files

#### `src/agents/nodes/orchestrator/node.py`

**Changes:**

1. **Add forced routing check before `_decide`:**
   - At the point where the orchestrator chooses a route (before calling `_decide`), call `forced_route(state)`
   - If `forced_route` returns `None`, proceed to `_decide` as today (extract flag IDs from `state.flags`, call `_decide`, include extracted flag IDs in state update under `orchestrator_seen_flags`)
   - If `forced_route` returns a route, check skip conditions (see below)

2. **Implement skip conditions:**
   - Model has work if:
     - Forced route is `["flag-review"]` or `["review"]`, **or**
     - An unresolved flag exists whose ID is not in `state.orchestrator_seen_flags`
   - If model has work: call `_decide` as today (extract flag IDs from `state.flags`, call `_decide`, include extracted flag IDs in state update under `orchestrator_seen_flags`)
   - If model has no work: skip `_decide`, use forced route, write forced hop step (see step 3)

3. **Write forced hop step (when skipping `_decide`) AND track seen flags:**
   - Extract flag IDs from `state.flags` (all unresolved flags) before writing the step
   - Create a step dict:
     - `title`: `f"Next: {', '.join(route)}"` (e.g., `"Next: planner"`, `"Next: __end__"`)
     - `result`: `"forced: only one valid route"`
     - `evidence`: `{"attempts": 0}`
   - Update state fields exactly as a model hop would for this route:
     - `status`: same logic as model hop interpretation for the route
     - `flagged`: same logic as model hop interpretation for the route
     - `priority`: same logic as model hop interpretation for the route
     - `confidence`: same logic as model hop interpretation for the route
     - `summary`: leave unchanged (do not set or modify)
   - Include the extracted flag IDs in the state update under `orchestrator_seen_flags` (same as model hop path)
   - Append the step to `state.steps` (or equivalent step history)
   - Return the updated state with the forced route

4. **Track seen flags on model hop path (when calling `_decide`):**
   - Before calling `_decide`, extract flag IDs from `state.flags` (all unresolved flags: `{flag["id"] for flag in state.flags if not flag.get("resolved", False)}`)
   - After `_decide` returns, include the extracted flag IDs in the state update under `orchestrator_seen_flags`
   - The reducer (from Story 1) will merge these IDs with existing `state.orchestrator_seen_flags` via set union

**Algorithm notes:**

- **Forced routing check:**
  ```python
  route = forced_route(state)
  
  if route is None:
      # No forced route; proceed to _decide as today
      # Extract flag IDs and track in state update (step 4)
      flag_ids = {flag["id"] for flag in state.flags if not flag.get("resolved", False)}
      result = self._decide(state)
      result.orchestrator_seen_flags = state.orchestrator_seen_flags | flag_ids
      return result
  
  # Forced route exists; check skip conditions
  model_has_work = (
      route == ["flag-review"] or 
      route == ["review"] or 
      self._has_unseen_flags(state)
  )
  
  if model_has_work:
      # Model needs to run; proceed to _decide
      # Extract flag IDs and track in state update (step 4)
      flag_ids = {flag["id"] for flag in state.flags if not flag.get("resolved", False)}
      result = self._decide(state)
      result.orchestrator_seen_flags = state.orchestrator_seen_flags | flag_ids
      return result
  else:
      # Skip _decide; write forced hop AND track seen flags (step 3)
      return self._write_forced_hop(state, route)
  ```

- **Unseen flags check:**
  ```python
  def _has_unseen_flags(self, state) -> bool:
      """
      Returns True if any unresolved flag exists whose ID is not in
      state.orchestrator_seen_flags.
      """
      unresolved_flags = [
          flag for flag in state.flags 
          if not flag.get("resolved", False)
      ]
      unseen_flags = [
          flag for flag in unresolved_flags
          if flag["id"] not in state.orchestrator_seen_flags
      ]
      return len(unseen_flags) > 0
  ```

- **Write forced hop (WITH seen flags tracking):**
  ```python
  def _write_forced_hop(self, state, route: list[str]):
      """
      Writes a forced hop step, updates state fields as a model hop would,
      AND tracks seen flags (critical: flags must be marked as seen on both paths).
      """
      # Extract flag IDs BEFORE writing the step (same extraction as model hop path)
      flag_ids = {flag["id"] for flag in state.flags if not flag.get("resolved", False)}
      
      step = {
          "title": f"Next: {', '.join(route)}",
          "result": "forced: only one valid route",
          "evidence": {"attempts": 0}
      }
      
      # Update state fields using same logic as model hop interpretation
      # (This logic already exists in the code that interprets model route choices;
      #  refactor into a shared helper if needed)
      updated_state = self._apply_route_effects(state, route)
      
      # Do NOT modify summary
      # (Ensure _apply_route_effects does not touch summary, or explicitly preserve it)
      
      # Append step
      updated_state.steps.append(step)
      
      # Track seen flags (same as model hop path; reducer merges via set union)
      updated_state.orchestrator_seen_flags = state.orchestrator_seen_flags | flag_ids
      
      return updated_state
  ```

- **Track seen flags (in `_decide`):**
  ```python
  def _decide(self, state):
      # Extract flag IDs from state.flags BEFORE calling run_skill (all unresolved flags)
      # (Note: flag extraction happens in the caller now, but _decide can also extract
      #  internally if needed; the caller's extraction is for the state update after _decide)
      
      # Build brief (as today)
      brief = self._brief(state, ...)
      
      # Call run_skill (as today)
      response = run_skill(...)
      
      # Parse response, interpret route, etc. (as today)
      route = self._parse_route(response)
      updated_state = self._apply_route_effects(state, route)
      
      # Caller will include seen flag IDs in state update (reducer merges via set union)
      # (Flag extraction happens before _decide call in the routing check logic)
      
      return updated_state
  ```

- **Refactoring note:** The logic that interprets a route choice and updates `status`, `flagged`, `priority`, `confidence` likely already exists in the code that processes model hop responses. Extract this into a shared helper (e.g., `_apply_route_effects(state, route)`) to avoid duplication between model hops and forced hops. Ensure this helper does not modify `summary` (or explicitly preserve `summary` when calling it for forced hops).

---

## Interfaces

### Internal Helper Functions

**`_has_unseen_flags(state) -> bool`:**
- **Input:** Current state
- **Returns:** `True` if any unresolved flag exists whose ID is not in `state.orchestrator_seen_flags`, else `False`
- **Used by:** Skip condition check

**`_write_forced_hop(state, route: list[str])`:**
- **Input:** Current state, forced route (list of node names)
- **Returns:** Updated state with forced hop step appended, state fields updated as a model hop would (except `summary` unchanged), AND `orchestrator_seen_flags` updated with flag IDs from `state.flags`
- **Used by:** Orchestrator when skipping `_decide`

**`_apply_route_effects(state, route: list[str])`:**
- **Input:** Current state, route (list of node names)
- **Returns:** Updated state with `status`, `flagged`, `priority`, `confidence` set according to the route
- **Used by:** Both model hops (existing) and forced hops (new)
- **Note:** This is a refactor of existing logic that interprets route choices; may already exist under a different name

### State Update (output of forced hop)

**Forced hop state update (WITH seen flags tracking):**

```python
{
  "steps": [...previous steps, {
      "title": "Next: planner",
      "result": "forced: only one valid route",
      "evidence": {"attempts": 0}
  }],
  "status": "routing",  # (or whatever status a hop to "planner" would set)
  "flagged": False,     # (or whatever flagged value a hop to "planner" would set)
  "priority": 1,        # (or whatever priority a hop to "planner" would set)
  "confidence": 0.95,   # (or whatever confidence a hop to "planner" would set)
  "orchestrator_seen_flags": state.orchestrator_seen_flags | {extracted flag IDs from state.flags},
  # summary: UNCHANGED (not included in update, so reducer leaves it as-is)
}
```

**Model hop state update (when model has work):**

```python
{
  "steps": [...previous steps, {model hop step from _decide}],
  "orchestrator_seen_flags": state.orchestrator_seen_flags | {extracted flag IDs from state.flags},
  # (plus other fields from model hop interpretation)
}
```

---

## Tests

From breakdown Verification Approach (6 tests, exact count matching breakdown for story 03-agent):

### Test 1: Orchestrator skips _decide when forced route exists and model has no work

**Mapped from:** REQ-3 AC-3.1, AC-3.5, AC-3.6, AC-3.4 (Breakdown verification item 1 for story 03-agent)

**Setup:**
- Create a fake state where:
  - `forced_route(state)` returns `["planner"]` (mocked)
  - Route is not `["flag-review"]` or `["review"]`
  - No unresolved flags exist, or all unresolved flags have IDs in `state.orchestrator_seen_flags`
- Mock `_decide` to track whether it is called

**Expected outcome:**
- Orchestrator does NOT call `_decide` (mock verifies zero calls)
- Orchestrator returns updated state with forced hop step:
  - `step.title == "Next: planner"`
  - `step.result == "forced: only one valid route"`
  - `step.evidence == {"attempts": 0}`
- State fields `status`, `flagged`, `priority`, `confidence` are updated as a model hop to `["planner"]` would update them
- `summary` is unchanged
- `state.orchestrator_seen_flags` is updated with flag IDs from `state.flags` (even though no flags exist or all are already seen, the extraction and update still runs; the result is the same set or empty set)

**Verification:**
- AC-1: Model has no work → skip `_decide`
- AC-5: Forced hop step format correct
- AC-6: State fields updated as model hop would (except `summary`)
- AC-4: Orchestrator updates `orchestrator_seen_flags` on forced hop path (seen flags tracking applies to both paths)

---

### Test 2: Orchestrator calls _decide when forced route is ["flag-review"]

**Mapped from:** REQ-3 AC-3.2 (Breakdown verification item 2 for story 03-agent)

**Setup:**
- Create a fake state where:
  - `forced_route(state)` returns `["flag-review"]` (mocked)
  - No unresolved unseen flags exist
- Mock `_decide` to track whether it is called and return a fake response

**Expected outcome:**
- Orchestrator DOES call `_decide` (mock verifies one call)
- Orchestrator returns the result of `_decide` (not a forced hop)

**Verification:**
- AC-2: Route pauses for a person (`flag-review`) → model writes reviewer summary → call `_decide`

---

### Test 3: Orchestrator calls _decide when forced route is ["review"]

**Mapped from:** REQ-3 AC-3.2 (Breakdown verification item 3 for story 03-agent)

**Setup:**
- Create a fake state where:
  - `forced_route(state)` returns `["review"]` (mocked)
  - No unresolved unseen flags exist
- Mock `_decide` to track whether it is called and return a fake response

**Expected outcome:**
- Orchestrator DOES call `_decide` (mock verifies one call)
- Orchestrator returns the result of `_decide` (not a forced hop)

**Verification:**
- AC-2: Route pauses for a person (`review`) → model writes reviewer summary → call `_decide`

---

### Test 4: Orchestrator calls _decide when unseen flag exists

**Mapped from:** REQ-3 AC-3.3, AC-3.4 (Breakdown verification item 4 for story 03-agent)

**Setup:**
- Create a fake state where:
  - `forced_route(state)` returns `["planner"]` (mocked, route is not `flag-review` or `review`)
  - An unresolved flag exists with ID `"flag-new"`, not in `state.orchestrator_seen_flags`
  - `state.orchestrator_seen_flags = {"flag-1", "flag-2"}` (does not include `"flag-new"`)
- Mock `_decide` to track whether it is called and return a fake response

**Expected outcome:**
- Orchestrator DOES call `_decide` (mock verifies one call)
- Orchestrator returns the result of `_decide` (not a forced hop)
- After `_decide`, state update includes `orchestrator_seen_flags` with `"flag-new"` added (reducer merges via set union)

**Verification:**
- AC-3: Unseen flag exists → model needs to merge duplicate flags → call `_decide`
- AC-4: Orchestrator updates `state.orchestrator_seen_flags` with flag IDs from `state.flags` (model hop path)

---

### Test 5: Forced hop step has correct format

**Mapped from:** REQ-3 AC-3.5 (Breakdown verification item 5 for story 03-agent)

**Setup:**
- Create a fake state where:
  - `forced_route(state)` returns `["output-validator"]` (mocked)
  - Route is not `["flag-review"]` or `["review"]`
  - No unresolved unseen flags exist
- Run orchestrator hop

**Expected outcome:**
- Step appended to `state.steps` has:
  - `title == "Next: output-validator"`
  - `result == "forced: only one valid route"`
  - `evidence == {"attempts": 0}`

**Verification:**
- AC-5: Forced hop step format correct

---

### Test 6: Forced hop updates state fields as model hop would, leaves summary unchanged, and tracks seen flags

**Mapped from:** REQ-3 AC-3.6, AC-3.4 (Breakdown verification item 6 for story 03-agent)

**Setup:**
- Create a fake state where:
  - `forced_route(state)` returns `["__end__"]` (mocked)
  - Route is not `["flag-review"]` or `["review"]`
  - No unresolved unseen flags exist (or all are already in `state.orchestrator_seen_flags`)
  - `state.summary = "Previous summary text"`
  - `state.flags = [{"id": "flag-1", "resolved": False}, {"id": "flag-2", "resolved": False}]`
  - `state.orchestrator_seen_flags = set()` (empty, so the forced hop will add `{"flag-1", "flag-2"}`)
- Run orchestrator hop
- Compare the forced hop's state updates to what a model hop to `["__end__"]` would produce (mock or reference a known model hop result for this route)

**Expected outcome:**
- `state.status`, `state.flagged`, `state.priority`, `state.confidence` match what a model hop to `["__end__"]` would set
- `state.summary == "Previous summary text"` (unchanged)
- `state.orchestrator_seen_flags == {"flag-1", "flag-2"}` (flags marked as seen on forced hop path)

**Verification:**
- AC-6: Forced hop updates `status`, `flagged`, `priority`, `confidence` exactly as a model hop does, and leaves `summary` unchanged
- AC-4: Orchestrator updates `state.orchestrator_seen_flags` on forced hop path (critical: seen flags tracking applies to both paths, not just model hop path)

---

## Notes

- **No repo tree provided:** Implementation must verify the actual path of `node.py` in the `app` repo. The story specifies `src/agents/nodes/orchestrator/node.py` as the likely location.
- **Critical clarification (seen flags on forced hop path):** HLD AD-5 clarifies that seen flags are tracked on **both** the model hop path and the forced hop path. The same flag ID extraction and state update logic applies on both paths. This prevents the orchestrator from calling the model on the next hop when flags were already present in a forced hop's state.
- **Refactoring opportunity:** The logic that interprets a route and updates `status`, `flagged`, `priority`, `confidence` likely already exists in the orchestrator's model hop handling code. Extract this into a shared helper (`_apply_route_effects` or similar) to avoid duplication between model hops and forced hops. Ensure this helper does not modify `summary` when called for forced hops.
- **Flag ID extraction:** The story assumes flag IDs can be extracted from `state.flags` (a list or dict of flag objects, each with an `"id"` field). Implementation must verify the actual structure of `state.flags` in the codebase. The extraction logic is the same on both paths: `{flag["id"] for flag in state.flags if not flag.get("resolved", False)}`.
- **Extraction timing:** Flag IDs are extracted **before** calling `_decide` (on the model hop path) or **before** writing the forced hop step (on the forced hop path), and included in the state update **after** the hop completes. This ensures the orchestrator tracks which flags were present at the time of the hop, regardless of which path it took.
- **No live runs in tests:** All tests use fake states, mock `forced_route`, mock `_decide`, and fake gateway (no live model calls). Existing tests pass unchanged; `uv run pytest -q` passes.
- **Dependencies clear:** Story 1 provides `state.orchestrator_seen_flags` schema and reducer; Story 2 provides `forced_route` function. This story integrates them into orchestrator logic.
- **Test count matches breakdown:** 6 tests exactly as specified in breakdown Verification Approach for story 03-agent. No additional tests beyond those agreed in the breakdown.
- **Alignment with requirements AC-4:** Requirements REQ-3 AC-3.4 says "The orchestrator updates a new state field with the flag IDs from each brief it receives." The clarification in this LLD (from HLD AD-5) specifies that this update happens on **both** the model hop path (when calling `_decide`) **and** the forced hop path (when skipping `_decide`). The same extraction logic applies on both paths. This ensures the orchestrator never loses track of flags, regardless of which path the hop takes.
