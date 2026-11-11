# ADLC Critic Review — PER-209

## What would go wrong

### C1
- file: hld.md
- section: ## Architectural Decisions
- quote: `status` values are `{'running', 'needs_input', 'needs_review', 'done', 'rejected', 'failed'}`
- match: requirements.md › ## Acceptance Criteria › AC1
- severity: blocking
- problem: The requirements say "running, waiting for input, waiting for review, done, rejected and failed" but the HLD normalizes two of them to `needs_input` and `needs_review`. Every document after the HLD uses the normalized names, so the developer would build a system where API consumers see `needs_review` but the ticket says "waiting for review". The mismatch is in how status values are named between the original ticket and the implementation.
- change: Either update the requirements to say the statuses are normalized (add a note explaining `waiting for input` becomes `needs_input`, `waiting for review` becomes `needs_review`), or change the HLD and all downstream documents to use `waiting_for_input` and `waiting_for_review` verbatim to match the ticket's wording.

### C2
- file: stories/02-agent.md
- section: ## Algorithm Notes › `save_case`
- quote: `employee_name` ← `backend.employee_name(state.get('service_order'))` (may return `None`)
- match: repo-outline/agent shows no `employee_name` function in `src/agents/runtime/backend.py`
- severity: blocking
- problem: The story says to call `backend.employee_name(service_order)` but the repo outline for `backend.py` lists only `ServiceOrder`, `next_service_order_id()` and `run_query()`—no `employee_name` function exists. The developer would get an `AttributeError` when the hook tries to extract the employee name.
- change: Either add `employee_name(service_order) -> str | None` to `backend.py` (and note it in the HLD's dependencies), or change the extraction logic to derive the name directly from the `service_order` dict structure (e.g. `service_order.get('employee', {}).get('name')`) and document what keys hold it.

### C3
- file: breakdown.md
- section: ## Stories (Build Order) › Story 2
- quote: `save_case` extracts fields from `State` as described in HLD AD-5.
- match: hld.md › AD-5 says the hook extracts the fields, not `save_case`
- severity: blocking
- problem: The breakdown says Story 2's `save_case` extracts fields "as described in HLD AD-5", but AD-5 describes the *persistence hook* (Story 4) doing the extraction and then calling `save_case`. Story 2's `save_case` is supposed to accept a `State` dict and extract what it needs from it, but the HLD puts that extraction logic in the hook. The developer working on Story 2 would read AD-5 for guidance and find hook-level logic instead of repository-level logic, so they would either duplicate the extraction or get confused about which component owns it.
- change: In Story 2's LLD, write the extraction logic directly in the `save_case` algorithm notes (don't just reference AD-5). In the HLD, clarify that AD-5 describes what the hook passes to `save_case`, and that `save_case` itself does its own extraction from the `State` dict it receives.

### C4
- file: stories/03-agent.md
- section: ## Algorithm Notes › Route: `GET /cases`
- quote: `_validate_sort(sort, statuses)` (helper checks `wait_time` requires `needs_review`).
- match: stories/03-agent.md › ## Algorithm Notes › Helper Functions › `_validate_sort` returns `str`, but the route algorithm doesn't use the return value
- severity: note
- problem: The helper `_validate_sort` is documented as returning `str` (the validated sort value), but the route algorithm shows `_validate_sort(sort, statuses)` without capturing the return value. The developer would write the helper to return `sort`, then never use it, which is harmless but odd. Not blocking because the route uses the original `sort` parameter afterward and it's already validated, so the code would work either way.
- change: Either change the helper signature to return `None` (it only raises on error), or change the route algorithm to `sort = _validate_sort(sort, statuses)` and use the returned value.

### C5
- file: stories/04-agent.md
- section: ## Algorithm Notes › Checkpointer Hook Registration
- quote: LangGraph's `AsyncSqliteSaver` and `InMemorySaver` do not natively support write hooks as of LangGraph 1.2.11. Instead, wrap the checkpointer's `aput` method
- match: stories/04-agent.md › ## Interfaces › `get_checkpointer_with_hook` docstring says "attach on_checkpoint_write hook" but the algorithm notes say wrap `aput` instead
- severity: blocking
- problem: The interface docstring for `get_checkpointer_with_hook` says it attaches an `on_checkpoint_write` hook, which sounds like a native LangGraph feature, but the algorithm notes explain that LangGraph 1.2.11 doesn't have that feature and you have to wrap the `aput` method instead. The developer would read the interface and think they can call `saver.on_checkpoint_write = _hook`, but the algorithm notes contradict that by saying to wrap `aput`. They would try the native-looking approach first, fail, then read the algorithm notes and realize the interface was misleading.
- change: Rewrite the `get_checkpointer_with_hook` docstring to say "Wraps the checkpointer's `aput` method to call `repository.save_case` after each write" instead of "attaches hook". The algorithm notes already explain the wrapping approach, so the interface should match it.

### C6
- file: stories/05-agent.md
- section: ## Algorithm Notes › Modified `_snapshot` Function
- quote: `async with open_graph(thread_id) as graph:`
- match: repo-outline/agent shows `open_graph(thread_id: str, checkpointer_kind: str = "memory")` in `src/agents/runtime/checkpointer.py`, but the story's algorithm never passes `checkpointer_kind`
- severity: blocking
- problem: The existing `open_graph` function takes an optional `checkpointer_kind` parameter (default `"memory"`), but the story's `_snapshot` implementation calls `open_graph(thread_id)` without specifying which checkpointer to use. If the server is running with `checkpointer_kind="sqlite"` (which the HLD suggests as an option), the fallback would open a different checkpointer than the one the run is actually using, so it wouldn't find the run's state and would incorrectly fall back to SQLite even when the checkpointer has the data.
- change: In Story 5's `_snapshot` algorithm notes, pass the same `checkpointer_kind` that the server is configured to use. Either add a global variable `CHECKPOINTER_KIND` that `_snapshot` reads, or pass `checkpointer_kind` as a parameter to `_snapshot` (which the route handlers already know from the server's startup config). Document how `_snapshot` knows which checkpointer the run is using.

---

## Notes for the gate

- **Open Question (HLD OQ1): case_type extraction.** The HLD, breakdown, Story 2 and Story 4 all note that `case_type` derivation is unresolved and uses placeholder logic (`service_order.get('caseType')`). The documents are consistent about it being a placeholder, but the developer would ship code that might extract the wrong value. Gate review should clarify whether `service_order['caseType']` is correct or whether a mapping is needed.

- **Open Question (HLD OQ4): wait_time calculation.** The schema has no `waiting_since` field, so Story 3's `sort=wait_time` uses `last_changed_at` as a proxy. The documents agree on this, but AC10 says "how long they have been waiting" and that's ambiguous. If "waiting" means "time since status became `needs_review`", the current schema can't answer it accurately (because `last_changed_at` updates on every state change, not just status transitions). Gate review should decide whether the proxy is acceptable or whether a schema amendment is needed.

- **Repo context assumption: `backend.employee_name` missing.** C2 flags that the function doesn't exist in the current code. If it's supposed to be added as part of this feature, the HLD should list it under "New modules/functions" instead of "Existing modules" in Dependencies. If it already exists but isn't shown in the outline, the outline is incomplete and the gate reviewer should verify the real code before approving.

- **Checkpointer kind configuration.** C6 notes that `_snapshot` doesn't pass `checkpointer_kind` to `open_graph`. The documents don't say how the server knows which checkpointer to use (is it hardcoded? an environment variable? a startup argument?). The gate reviewer should verify that the server has a single, consistent `checkpointer_kind` setting and that all code paths use it.

- **Status name normalization (C1).** If the gate decides to keep `needs_input` and `needs_review` instead of the ticket's "waiting for input" and "waiting for review", the requirements document should explain the normalization so it's clear to everyone reading the spec. If the ticket's exact wording is required, all documents after the HLD need to change.

VERDICT: FAIL
