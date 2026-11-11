## What would go wrong

### C1
- file: specs/1.md
- section: ## File Plan
- quote: Add `POST /runs/{thread_id}/priority` route handler that: … Updates `state["priority_override"]` to `{"priority": req.priority, "reason": req.reason.strip(), "set_at": datetime.now(UTC).isoformat(), "set_by": None}` … Writes checkpoint back via `graph.aupdate_state(config, state)` (within `open_graph` context)
- match: repo-file/agent/src/agents/runtime/server.py › open_graph function
- severity: blocking
- blocks because: Dev would write a route handler that calls `graph.aupdate_state(config, state)` to write the checkpoint, but that method does not exist in the LangGraph graph API—they would get an AttributeError at runtime and the endpoint would never work.
- problem: The spec prescribes `graph.aupdate_state(config, state)` to write the modified state back to the checkpoint, but LangGraph graphs do not expose an `aupdate_state` method. The existing code in `server.py` shows `_snapshot` reading state via `graph.aget_state(config)`, but no write method.
- change: Replace `graph.aupdate_state(config, state)` with the correct LangGraph checkpoint write API. If no such method exists, mark this as a blocker and note that the endpoint cannot be implemented until the checkpoint write mechanism is determined (either via a LangGraph API to be discovered, or by writing directly to the checkpointer's storage, which the spec explicitly calls fragile and asks to avoid).

### C2
- file: specs/1.md
- section: ## Algorithm Notes
- quote: Investigation needed: How does the graph's checkpointer API support updating a paused run's state without resuming it? … Likely approach (verify in LangGraph docs or `open_graph` implementation): … `await graph.aupdate_state(config, state)` writes it back without resuming
- match: hld.md › AD-2: API Endpoint Contract
- severity: blocking
- blocks because: Dev would attempt to implement the endpoint following the "likely approach" in the spec, but the spec itself marks this as unverified and needing investigation. If `aupdate_state` does not exist or behaves differently, the endpoint would fail or corrupt the checkpoint, and the spec gives no fallback beyond "defer the endpoint."
- problem: The spec acknowledges uncertainty about how to write the checkpoint without resuming the run ("Investigation needed … verify in LangGraph docs"), prescribes a method (`aupdate_state`) that may not exist, and offers only "defer the endpoint" as a fallback. Dev cannot implement the endpoint without resolving this.
- change: State how to write the checkpoint back: either confirm `graph.aupdate_state(config, state)` exists and works as described, or document the correct API call, or mark the endpoint as blocked pending discovery of the checkpoint write mechanism. Remove the "Investigation needed" note once resolved, or escalate it as a dependency blocker if it cannot be resolved within the spec.

### C3
- file: stories/01-agent.md
- section: ## File Plan
- quote: Updates `state["priority_override"]` to `{"priority": req.priority, "reason": req.reason.strip(), "set_at": datetime.now(UTC).isoformat(), "set_by": None}`
- match: hld.md › AD-1: Priority Override Model
- severity: blocking
- blocks because: Dev would write the timestamp as `datetime.now(UTC).isoformat()`, but the HLD specifies the timestamp format as "ISO 8601" and the existing code in `states.py` uses `datetime.now(UTC).isoformat()`, which produces ISO 8601 with microseconds (e.g. `2026-01-15T10:30:00.123456+00:00`). The story and HLD agree on ISO 8601, but neither specifies whether microseconds should be included or stripped, so Dev might produce inconsistent timestamp formats across the codebase if other code strips them.
- problem: The story prescribes `datetime.now(UTC).isoformat()` for the timestamp, but does not specify whether to strip microseconds. The HLD says "ISO 8601 UTC" and the existing `step_record` helper in `states.py` also uses `datetime.now(UTC).isoformat()`. If the rest of the codebase expects timestamps without microseconds, this would be inconsistent.
- change: Clarify whether `set_at` should include microseconds. If the codebase strips them elsewhere, add `.replace(microsecond=0)` before `.isoformat()`. If microseconds are fine, state that explicitly so Dev knows the format is intentional.

### C4
- file: stories/01-agent.md
- section: ## Tests
- quote: Test 6: Set priority twice — second overwrites first, both in history … `monke ypatch.setattr("agents.runtime.server.open_graph", fake_open_graph)`
- match: (test code block in stories/01-agent.md › ## Tests › Test 6)
- severity: blocking
- blocks because: Dev would copy the test code and get a NameError because `monke ypatch` is a typo for `monkeypatch`, and the test would never run. They would have to notice and fix the typo, but it blocks the test from passing as written.
- problem: The test code has a typo: `monke ypatch.setattr` instead of `monkeypatch.setattr`. This would cause a NameError when the test runs.
- change: Fix the typo to `monkeypatch.setattr("agents.runtime.server.open_graph", fake_open_graph)`.

---

VERDICT: FAIL
