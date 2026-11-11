# Breakdown — PER-234

**Reviewers can set a case's priority**

---

## Overview

This breakdown implements reviewer-settable case priority with validation, persistence, history tracking, and activity feed integration. The feature is split into two stories:

1. **Data model and persistence** — Add `priority_override` to `State`, modify orchestrator and planner to respect it, ensure persistence across restarts.
2. **API endpoint and integrations** — Implement the `/priority` endpoint with validation, history tracking, and activity feed integration.

The split separates the core state logic (which agents rely on) from the HTTP interface (which reviewers use). Story 1 can be tested entirely via the checkpoint without running the server; Story 2 depends on Story 1's state changes.

---

## Dependencies and Prerequisites

- **Story 1** has no dependencies — it modifies the state schema and node logic.
- **Story 2** depends on Story 1 — it writes to `priority_override` and expects the orchestrator and planner to respect it.

Build order: Story 1, then Story 2.

---

## Story 1: Add priority_override to State and respect it in orchestrator and planner

**Repo:** `agent`

**Description:**

Add the `priority_override` field to the `State` TypedDict to distinguish reviewer-set priority from system-managed priority. Modify the orchestrator's priority-raising logic and the planner's priority assignment to skip `state["priority"]` updates when `priority_override` is present. Verify that priority persists across service restarts (no checkpoint migration needed — SQLite already stores `State` as JSON).

**Covers:**

- AC-6: Reviewer priority persists (planner and orchestrator respect it)
- AC-13: Persistence across restarts
- AC-14: System-managed priority for other cases (baseline: existing behavior when `priority_override` is absent)

**Prerequisites:** None (`after: []`)

**Allowed paths:**

- `src/agents/graph/states.py` — add `priority_override: NotRequired[dict[str, Any]]` to `State`
- `src/agents/nodes/orchestrator/node.py` — guard priority-raise logic with `if "priority_override" not in state`
- `src/agents/nodes/planner/node.py` — guard priority assignment with `if "priority_override" not in state`

**Verification Approach:**

- **3 unit tests** (one per AC):
  1. Orchestrator does not raise priority when `priority_override` is present (flag raised, priority stays as reviewer set it).
  2. Planner does not overwrite priority when `priority_override` is present (replan, priority stays as reviewer set it).
  3. Checkpoint round-trip: write `State` with `priority_override`, restart (close and reopen the checkpointer), read it back — `priority_override` is intact.

---

## Story 2: Implement POST /priority endpoint with validation, history, and activity feed

**Repo:** `agent`

**Description:**

Add the `POST /runs/{thread_id}/priority` endpoint to `server.py`. Validate the request (priority value, reason presence and length, case status), update `state["priority_override"]` and `state["priority"]`, append a history `StepRecord`, write the checkpoint, and call `report_progress` to send the activity feed event. Handle all error cases per the requirements: invalid priority, blank or long reason, running/terminal cases, non-existent thread.

**Covers:**

- AC-1: Allowed priority values (high, medium, low)
- AC-2: Priority requires a reason (non-blank)
- AC-3: Reason length limit (≤ 300 characters)
- AC-4: Invalid priority values rejected
- AC-5: Priority display (reviewer-set flag, reason shown — state contract for frontend)
- AC-7: History tracking (each change recorded in `steps`)
- AC-8: Updating a reviewer-set priority (replaces prior override, both in history)
- AC-9: Activity feed notification ("Priority set to <priority>")
- AC-10: Running cases cannot be changed
- AC-11: Terminal-state cases cannot be changed (done, rejected, failed)
- AC-12: Non-existent cases

**Prerequisites:** Story 1 (`after: [1]`)

**Allowed paths:**

- `src/agents/runtime/server.py` — add `SetPriority` Pydantic model, `POST /runs/{thread_id}/priority` route handler
- `src/agents/graph/states.py` — use `step_record` helper for history entry (already exists)
- `src/agents/runtime/backend.py` — verify `events_from_update` processes priority change steps (no code change needed if step schema from Story 1 is followed)

**Verification Approach:**

- **12 integration tests** (one per AC):
  1. Set priority to `high`, `medium`, `low` on a `needs_input` case — 200, `priority_override` written, `state["priority"]` updated.
  2. Blank reason rejected (400, priority unchanged).
  3. Reason of 300 characters accepted; 301 rejected (400).
  4. Priority `urgent` rejected with allowed-values message (400).
  5. Case displays `priority_override` fields (verify state shape; display logic is out of scope for backend).
  6. Set priority twice — second overwrites first, both changes in `steps`.
  7. History `StepRecord` includes old priority, new priority, reason (verify step evidence).
  8. Activity feed event posted with "Priority set to <priority>" and employee name (mock `report_progress`, verify call args).
  9. Set priority on `running` case — 409, "wait until it pauses".
  10. Set priority on `done`, `rejected`, `failed` case — 409, "cannot be changed".
  11. Set priority on non-existent `thread_id` — 404, "not found".
  12. Set priority on `needs_review` case — 200 (allowed status).

---

## Notes

- **Reviewer identifier (`set_by`):** Story 2 accepts `None` for `set_by` in the `priority_override` dict. Future: extract from auth token or request header. Display logic shows "a reviewer" when `set_by` is `None`.
- **History step `stage`:** Story 2 uses `"review"` for the priority change step's `stage` field (semantically close to reviewer action). A new `"priority_change"` stage is not added unless the user requests it.
- **Checkpoint migration:** Not needed — `src/agents/runtime/checkpointer.py` stores `State` as JSON in SQLite's `checkpoints` table (verified in `get_checkpointer`). `priority_override` is a top-level `State` field and persists like any other.
- **Activity feed event order:** Story 2 sends the activity feed event AFTER the checkpoint write succeeds, not before. If the feed send fails, log the error but return 200 (activity feed is not critical path).
- **State transitions:** Only `needs_input` and `needs_review` statuses allow priority changes. `running`, `done`, `rejected`, `failed` are blocked. Story 2 validates `state["status"]` before writing.

---

## Out of Scope

- Command-line interface for setting priority (AC explicitly excludes)
- Changing planner logic for initial priority assignment (AC-14: baseline unchanged)
- Ordering cases or queues by priority (AC explicitly excludes)
- Notifications beyond activity feed entry (AC-9 covers activity feed only)
- Frontend implementation (backend-only; `State` shape is contract for frontend)
- Role-based permissions (who can set priority — today: any caller; future: auth/RBAC)

---

**End of Breakdown**
