# Merged plan — PER-234

Execution order (1 ticket(s)):

1. `PER-236` (story `01-agent`)

Specs (1): `specs/1.md`

---

## Spec `specs/1.md`

# Tech Spec — Ticket 1

**Reviewer-settable case priority with validation and persistence**

---

## Stack

- **Language:** Python 3.11+
- **Framework:** FastAPI (HTTP server), LangGraph (agent orchestration)
- **Persistence:** SQLite (via `langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver`)
- **Type System:** TypedDict (runtime), Pydantic (API validation)
- **Dependencies:** `httpx` (backend reporting), `aiosqlite` (async sqlite)

---

## Schema / API Contracts

### State Schema Changes

**File:** `src/agents/graph/states.py`

Add `priority_override` field to the `State` TypedDict:

```python
class State(TypedDict):
    # ... existing fields ...
    priority_override: NotRequired[dict[str, Any]]
    # Structure:
    # {
    #     "priority": "high" | "medium" | "low",
    #     "reason": str,
    #     "set_at": str,  # ISO 8601
    #     "set_by": str | None
    # }
```

- **Rationale:** `NotRequired` — field is absent when no reviewer has set priority.
- **Effective priority:** `state["priority"]` remains the single source of truth for what the system shows and uses. When `priority_override` is present, `state["priority"]` is set to `priority_override["priority"]` and never modified by orchestrator or planner.

### History Schema

**File:** `src/agents/graph/states.py`

Priority changes append one `StepRecord` to `state["steps"]`:

```python
{
    "stage": "review",
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
    "feed": "attention"
}
```

### API Endpoint

**File:** `src/agents/runtime/server.py`

**New route:** `POST /runs/{thread_id}/priority`

**Request model:**

```python
class SetPriority(BaseModel):
    priority: Literal["high", "medium", "low"]
    reason: str

    @model_validator(mode="after")
    def _validate_reason(self) -> "SetPriority":
        stripped = self.reason.strip()
        if not stripped:
            raise ValueError("Reason is required and cannot be blank")
        if len(stripped) > 300:
            raise ValueError("Reason must be 300 characters or less")
        return self
```

**Success response (200):**

```json
{
  "status": "ok",
  "priority": "high",
  "reason": "...",
  "set_at": "2026-01-15T10:30:00Z"
}
```

**Error responses:**

- **400:** Invalid priority value, blank reason, or reason > 300 chars
- **404:** `thread_id` not found
- **409:** Case is `running`, or `done`/`rejected`/`failed`

**Allowed case statuses:** `needs_input`, `needs_review`

### Checkpoint Schema

**File:** `src/agents/runtime/checkpointer.py` (no changes needed)

- `priority_override` is a top-level field in `State` TypedDict
- SQLite checkpointer stores `State` as JSON blob in `checkpoints` table
- No migration required — existing column already holds arbitrary JSON

---

## File Touch List

### Modified Files

1. **`src/agents/graph/states.py`**
   - Add `priority_override: NotRequired[dict[str, Any]]` to `State` TypedDict
   - Document structure in inline comment

2. **`src/agents/runtime/server.py`**
   - Add `SetPriority` Pydantic model
   - Add `POST /runs/{thread_id}/priority` route handler:
     - Load checkpoint snapshot
     - Validate status, priority, reason
     - Update `state["priority_override"]` and `state["priority"]`
     - Append history `StepRecord`
     - Write checkpoint
     - Call `report_progress` with activity feed event
     - Return 200 or error response

3. **`src/agents/nodes/orchestrator/node.py`**
   - Modify priority-raising logic:
     ```python
     if "priority_override" not in state and state.get("flagged"):
         state["priority"] = _RAISED.get(state["priority"], "high")
     ```
   - Never write to `priority_override` (endpoint-only)

4. **`src/agents/nodes/planner/node.py`**
   - Modify priority assignment:
     ```python
     if "priority_override" not in state:
         state["priority"] = priority_from_plan
     ```
   - Never write to `priority_override`

5. **`src/agents/runtime/backend.py`**
   - Verify `events_from_update` processes priority change steps (already handles steps with `feed` tag)
   - No code changes needed if step schema from `states.py` is followed

### New Files

None. All changes are modifications to existing files.

---

## Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Checkpoint write failure loses override** | Medium | SQLite writes are atomic per transaction. If write fails, return 500 and do not send activity feed event. User retries. |
| **Race condition: reviewer sets priority while case is running** | Medium | Validate `status` before writing. Reject with 409 if `running`. Checkpoint write is single transaction. |
| **Orchestrator/planner code does not check `priority_override`** | High | Add explicit `if "priority_override" not in state` guard in both nodes. Add test cases covering flag-raise and replan with `priority_override` present. |
| **Activity feed event sent but checkpoint write fails** | Low | Send activity feed event AFTER successful checkpoint write, not before. If send fails, log error but return 200 (activity feed is not critical path). |
| **Reviewer identifier (`set_by`) not available** | Low | Accept `None` for `set_by`. Display "a reviewer" instead of name. Future: extract from auth token or request header. |
| **Checkpoint schema does not support arbitrary `State` fields** | Low | Verified: SQLite checkpointer stores `State` as JSON. No migration needed. |

---

## Open Questions

1. **Reviewer identifier source:** Where does `set_by` come from?
   - **Answer needed in LLD:** For now, accept `None` or hardcoded test value. Auth token extraction is future enhancement unless context is already available.

2. **History step `stage` value:** Use `"review"` or add `"priority_change"`?
   - **Answer needed in LLD:** Lean toward `"review"` (semantically close to reviewer action) unless strong separation reason exists.

3. **Activity feed message format:** Current text is `"Priority set to <priority>"`. Include reason in feed entry?
   - **Answer needed in LLD:** Reason is in step evidence. Feed entry can show shortened reason if space allows.

---

## Out of Scope

- Command-line interface for setting priority (AC explicitly excludes)
- Changing planner logic for initial priority assignment (AC-14: planner logic unchanged for non-overridden cases)
- Ordering cases or queues by priority (AC explicitly excludes)
- Notifications beyond activity feed entry (AC-9 covers activity feed only)
- Frontend implementation (backend-only; `State` shape is contract for frontend)
- Role-based permissions (who can set priority — today: any caller; future: integrate with auth/RBAC)

---

**End of Tech Spec**

---

# Story summaries

- `01-agent`: Story 01-agent: Reviewer-settable case priority with validation and persistence

---

_Single-ticket pack: order is trivial (no multi-split)._
