# High-Level Design — PER-234

**Reviewers can set a case's priority**

---

## Overview

This feature adds the ability for reviewers to override a case's system-managed priority with their own judgment, storing the override persistently and displaying it throughout the case lifecycle. The reviewer-set priority persists across service restarts, flag raises, and re-planning events, and is tracked in the case history and activity feed.

## AD-1: Priority Override Model

**Binds:** The data structure distinguishing reviewer-set priority from system-managed priority.

**Prevents:** Confusion between planner-assigned priority and reviewer override; loss of reviewer intent when flags raise or planning reruns.

**Rule:**

- Add `priority_override` field to `State` (TypedDict in `src/agents/graph/states.py`):
  - `priority_override: NotRequired[dict[str, Any]]` — present when a reviewer has set the priority, absent otherwise.
  - Structure: `{"priority": "high" | "medium" | "low", "reason": str, "set_at": str (ISO 8601), "set_by": str | None}`
  - `set_by` is the reviewer identifier when available (from request context or activity feed logic); `None` when not provided.
- The existing `priority` field remains the **effective priority** — the value the system shows and uses for any priority-based logic.
  - When `priority_override` is absent, `priority` is system-managed (planner sets it, orchestrator raises it on flags).
  - When `priority_override` is present, `priority` is set to `priority_override["priority"]` and remains fixed.
- The system distinguishes the two cases by testing `"priority_override" in state` or `state.get("priority_override") is not None`.

**Rationale:** A separate `priority_override` dict cleanly separates reviewer intent from the effective priority value, avoids ambiguity when a reviewer sets priority to the same value the system chose, and stores the full audit trail (reason, timestamp, actor) without polluting the main state fields.

## AD-2: API Endpoint Contract

**Binds:** The HTTP interface for setting priority.

**Prevents:** Ambiguity in request shape, validation behavior, and error responses.

**Rule:**

- **Endpoint:** `POST /runs/{thread_id}/priority`
- **Request body:**
  ```json
  {
    "priority": "high" | "medium" | "low",
    "reason": "string (1-300 characters, non-whitespace)"
  }
  ```
- **Success response (200):**
  ```json
  {
    "status": "ok",
    "priority": "high" | "medium" | "low",
    "reason": "...",
    "set_at": "2026-01-15T10:30:00Z"
  }
  ```
- **Error responses:**
  - 400 if `priority` is not one of `high`, `medium`, `low` (message: `"Priority must be one of: high, medium, low"`)
  - 400 if `reason` is empty or only whitespace (message: `"Reason is required and cannot be blank"`)
  - 400 if `reason` exceeds 300 characters (message: `"Reason must be 300 characters or less"`)
  - 409 if case `status` is `running` (message: `"Case is still running. Wait until it pauses."`)
  - 409 if case `status` is `done`, `rejected`, or `failed` (message: `"Case is <status> and cannot be changed."`)
  - 404 if `thread_id` does not exist (message: `"Case not found"`)
- **Allowed statuses:** `needs_input`, `needs_review` — only cases paused for reviewer action can be changed.
- The endpoint retrieves the latest checkpoint for `thread_id`, validates the request, updates `state["priority_override"]` and `state["priority"]`, appends a history entry, writes the checkpoint back, and triggers an activity feed event.

**Implementation location:** `src/agents/runtime/server.py` (new route handler, parallel to `/runs/{thread_id}/resume`).

## AD-3: State Validation and Transition Rules

**Binds:** The state machine rules governing when priority can be changed.

**Prevents:** Priority changes during agent execution or on terminal cases; inconsistent error messages.

**Rule:**

- **Allowed transitions:**
  - From `needs_input` or `needs_review` → priority can be set.
- **Blocked transitions:**
  - From `running` → reject with 409 and "Case is still running. Wait until it pauses."
  - From `done`, `rejected`, `failed` → reject with 409 and "Case is <status> and cannot be changed."
  - Non-existent `thread_id` → reject with 404 and "Case not found."
- **Validation order:**
  1. Check `thread_id` exists (404 if not).
  2. Retrieve latest checkpoint snapshot.
  3. Check `state["status"]` is in `{"needs_input", "needs_review"}` (409 if not).
  4. Validate `priority` in `{"high", "medium", "low"}` (400 if not).
  5. Validate `reason` is non-empty after stripping whitespace (400 if not).
  6. Validate `reason` length ≤ 300 characters (400 if not).
  7. Apply the change.
- Validation logic resides in the route handler; Pydantic models enforce types and length constraints.

## AD-4: Priority Override Persistence and Checkpoint Schema

**Binds:** How `priority_override` is stored in the checkpoint.

**Prevents:** Loss of override on service restart; drift between in-memory state and persisted state.

**Rule:**

- `priority_override` is a top-level field in `State` (already specified in AD-1).
- The checkpointer (`src/agents/runtime/checkpointer.py`) writes `State` as-is; no special handling needed — `priority_override` is persisted like any other `State` field.
- On service restart, `State` is loaded from the checkpoint; `priority_override` presence governs priority behavior from that point forward.
- The existing checkpoint backends (`memory`, `sqlite`) already handle arbitrary `State` fields — no schema migration needed unless the checkpoint format changes (verify in LLD).

**Note:** If the checkpoint is stored in a database with a fixed schema (e.g. a `state` JSON column), this is already handled. If a migration is needed (e.g. adding a new column), mark that in the breakdown.

## AD-5: Orchestrator Ignores Reviewer-Set Priority

**Binds:** The orchestrator's priority-raising logic when `priority_override` is present.

**Prevents:** The orchestrator undoing the reviewer's decision when a flag is raised.

**Rule:**

- In `src/agents/nodes/orchestrator/node.py`, `orchestrator_node` currently raises priority when `state["flagged"]` is true (via `_RAISED` mapping).
- When `priority_override` is present, the orchestrator MUST NOT modify `state["priority"]` on flag raises or any other event.
- Implementation:
  ```python
  if "priority_override" not in state and state.get("flagged"):
      state["priority"] = _RAISED.get(state["priority"], "high")
  ```
- The orchestrator never writes to `priority_override` — only the `/priority` endpoint does.

**Rationale:** The reviewer's override is the source of truth once set. The orchestrator's auto-raise logic applies only to system-managed priority.

## AD-6: Planner Does Not Overwrite Reviewer-Set Priority

**Binds:** The planner's priority assignment when replanning a case.

**Prevents:** The planner erasing the reviewer's decision on a replan.

**Rule:**

- In `src/agents/nodes/planner/node.py`, `planner_node` sets `state["priority"]` from the planner's output.
- When `priority_override` is present, the planner MUST NOT modify `state["priority"]`.
- Implementation:
  ```python
  if "priority_override" not in state:
      state["priority"] = priority_from_plan
  ```
- The planner never writes to `priority_override`.

**Rationale:** Same as AD-5 — the reviewer override is binding.

## AD-7: History Tracking Schema

**Binds:** The history entry structure for priority changes.

**Prevents:** Ambiguity in what changed and why; loss of audit trail.

**Rule:**

- Priority changes are recorded in `state["steps"]` (existing history mechanism).
- Each priority change appends one `StepRecord`:
  ```python
  {
      "stage": "review",  # or a new stage "priority_change" — decide in LLD
      "agent": "reviewer",
      "title": "Priority changed from <old> to <new>",
      "source": "manual override",
      "at": "<ISO 8601 timestamp>",
      "result": "changed",
      "evidence": {
          "old_priority": "medium",
          "new_priority": "high",
          "reason": "Final pay due today",
          "set_by": "<reviewer_id or None>"
      },
      "feed": "attention"  # tagged for the activity feed
  }
  ```
- The `old_priority` is `state["priority"]` before the change (system-managed if no prior override, or the previous override value).
- The `new_priority` is the requested priority.
- The `reason` is the user-supplied reason.
- The `set_by` field is the reviewer identifier when available (see AD-2).
- The `feed: "attention"` tag ensures the activity feed processes it (AC-8).

**Implementation location:** The `/priority` route handler appends the step before writing the checkpoint.

## AD-8: Activity Feed Integration

**Binds:** The activity feed message format for priority changes.

**Prevents:** Missing or inconsistent activity feed entries for priority changes.

**Rule:**

- Priority change events are reported to the activity feed via `src/agents/runtime/backend.py:report_progress`.
- The activity feed message format:
  - Text: `"Priority set to <priority>"` (e.g. `"Priority set to high"`)
  - Employee name: included when `state["service_order"]` has an employee (via `employee_name` helper).
  - Event kind: derived from `feed: "attention"` in the step record (see AD-7).
- The `/priority` endpoint calls `report_progress` after appending the history step and writing the checkpoint:
  ```python
  await report_progress(
      thread_id,
      update=state,
      employee=employee_name(state.get("service_order"))
  )
  ```
- `events_from_update` in `backend.py` already processes steps with `feed` tags — it will extract the priority change step and format the activity feed entry.

**Note:** Verify `events_from_update` handles the step schema from AD-7; adjust if needed (LLD).

## AD-9: Display Logic for Reviewer-Set Priority

**Binds:** How the UI and logs distinguish reviewer-set priority from system priority.

**Prevents:** User confusion about where the priority came from.

**Rule:**

- Any component displaying `state["priority"]` MUST check `state.get("priority_override")`:
  - When `priority_override` is present: display `"Priority: <priority> (set by reviewer)"` or similar, and show the reason when space allows (e.g. tooltip, detail view).
  - When `priority_override` is absent: display `"Priority: <priority>"` (system-managed).
- The reason is `state["priority_override"]["reason"]`.
- The setter is `state["priority_override"].get("set_by", "a reviewer")`.
- The timestamp is `state["priority_override"]["set_at"]`.

**Implementation locations:**
- Backend: `src/agents/runtime/backend.py:events_from_update` (activity feed entries already carry the reason via the step evidence).
- Frontend: (out of scope for this repo; the `State` shape is the contract).
- Logs: `src/agents/runtime/run_dir.py` or wherever logs are formatted (verify in LLD).

**Rationale:** Transparency — the user must always know whether the priority is reviewer-set or system-managed.

## Component Diagram

```mermaid
graph TD
    API[POST /runs/:id/priority<br/>FastAPI endpoint]
    Checkpoint[(Checkpoint<br/>sqlite/memory)]
    State[State TypedDict<br/>priority_override: {...}]
    Orchestrator[Orchestrator Node<br/>ignores override on flag-raise]
    Planner[Planner Node<br/>ignores override on replan]
    Backend[Backend<br/>report_progress]
    Feed[Activity Feed]

    API -->|1. Validate request| API
    API -->|2. Load latest snapshot| Checkpoint
    Checkpoint -->|state| State
    API -->|3. Check status, priority, reason| State
    API -->|4. Set priority_override & priority| State
    API -->|5. Append history step| State
    API -->|6. Write checkpoint| Checkpoint
    API -->|7. Report to feed| Backend
    Backend -->|event| Feed

    Orchestrator -->|reads priority_override| State
    Orchestrator -->|does NOT modify priority when override present| State
    Planner -->|reads priority_override| State
    Planner -->|does NOT modify priority when override present| State
```

## Data Flow: Setting Priority

1. **Request:** Reviewer calls `POST /runs/{thread_id}/priority` with `{"priority": "high", "reason": "Final pay due today"}`.
2. **Load:** Endpoint retrieves the latest checkpoint snapshot for `thread_id`.
3. **Validate:**
   - Check `thread_id` exists (404 if not).
   - Check `state["status"]` is `needs_input` or `needs_review` (409 if not).
   - Check `priority` is `high`, `medium`, or `low` (400 if not).
   - Check `reason` is non-empty and ≤ 300 characters (400 if not).
4. **Update:**
   - Set `state["priority_override"] = {"priority": "high", "reason": "...", "set_at": "<now>", "set_by": "<reviewer_id or None>"}`.
   - Set `state["priority"] = "high"`.
5. **History:** Append a `StepRecord` to `state["steps"]` (see AD-7).
6. **Persist:** Write the updated `state` back to the checkpoint.
7. **Notify:** Call `report_progress(thread_id, update=state, employee=...)` to send the activity feed event.
8. **Response:** Return 200 with the new priority, reason, and timestamp.

## Data Flow: Subsequent Flag or Replan

1. **Flag raised:** A worker raises a new flag; orchestrator runs.
2. **Orchestrator checks:** `if "priority_override" not in state and state["flagged"]: state["priority"] = _RAISED[state["priority"]]`.
3. **Result:** Because `priority_override` is present, `state["priority"]` is NOT modified — it stays `"high"`.
4. **Planner reruns:** Planner sets a new plan.
5. **Planner checks:** `if "priority_override" not in state: state["priority"] = priority_from_plan`.
6. **Result:** Because `priority_override` is present, `state["priority"]` is NOT modified.

The reviewer-set priority persists unchanged.

## Data Flow: Service Restart

1. **Before restart:** `state["priority_override"]` is written to the checkpoint.
2. **Restart:** Service restarts; checkpoint is reloaded.
3. **After restart:** `state["priority_override"]` is present in the loaded `State`; all logic respecting it (orchestrator, planner, display) works as before.

The reviewer-set priority is retained.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Reviewer sets priority while case is running, causing a race condition | Medium | Validate `status` before writing; reject with 409 if `running`. The checkpoint write is atomic (single transaction for sqlite, single write for memory). |
| Priority override is lost on checkpoint write failure | Medium | Checkpoint writes are transactional (verify in LLD); if write fails, return 500 and do not send activity feed event. User retries. |
| Activity feed event is sent but checkpoint write fails | Low | Send activity feed event AFTER successful checkpoint write, not before. If send fails, log error but do not block the 200 response (activity feed is not critical path). |
| Orchestrator or planner code does not check `priority_override` | High | Add explicit test cases (breakdown) covering flag-raise and replan with `priority_override` present. Code review enforces the check. |
| Checkpoint schema does not support arbitrary `State` fields | Medium | Verify checkpointer in LLD. If schema is fixed, add migration to support `priority_override`. If not needed, document why. |
| Reviewer identifier (`set_by`) is not available in the request context | Low | Accept `None` for `set_by`; display "a reviewer" instead of a name. Future: extract from auth token or request header. |

## Open Questions

1. **Checkpoint migration:** Does the sqlite checkpointer store `State` as JSON (flexible) or fixed columns? If columns, do we need a migration?
   - **Answer:** Verify in `src/agents/runtime/checkpointer.py`. If JSON, no migration. If columns, add migration script in the breakdown.
2. **Reviewer identifier source:** Where does `set_by` come from — auth token, request header, hardcoded test value?
   - **Answer:** LLD to specify. For now, accept `None` or a test value; extraction logic is a future enhancement unless the auth context is already available.
3. **History step `stage` value:** Is `"review"` appropriate, or should we add `"priority_change"`?
   - **Answer:** LLD to decide. `"review"` is semantically close (reviewer action); a new stage may clutter the pipeline diagram. Lean toward `"review"` unless there's a strong separation reason.

## Out of Scope

- **Command-line interface** for setting priority (AC explicitly excludes).
- **Changing planner logic** for initial priority assignment (AC-12: planner logic unchanged for non-overridden cases).
- **Ordering cases or queues by priority** (AC explicitly excludes).
- **Notifications** beyond the activity feed entry (AC-9 covers activity feed; no other notifications).
- **Frontend implementation** (this is backend-only; `State` shape is the contract for frontend).

## Deferred

- **Role-based permissions:** Who can set priority? Today: any caller of the endpoint. Future: integrate with auth/RBAC.
- **Priority history pagination:** If a case has many priority changes, the full history may be large. Today: all steps are in `state["steps"]`. Future: paginate or summarize old entries.
- **Bulk priority changes:** Setting priority for multiple cases at once. Today: one endpoint call per case. Future: batch endpoint.

---

**End of High-Level Design**
