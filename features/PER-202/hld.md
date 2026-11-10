# HLD — PER-202

**Feature:** Reviewers can claim, hand over and track their cases

**Source:** PER-202 ticket  
**Date:** 2025-06-10  
**Status:** Draft

---

## Overview

When agents pause a payroll case for review, reviewers can now claim it, hand it over to another reviewer, view their active queue, and allow leads to monitor unclaimed cases that have waited too long. This introduces **claim state tracking**, **persistent handover history**, and **configurable timeout monitoring** to the existing review workflow.

The design preserves the current execution model (LangGraph state machine with Redis checkpointer) and adds claim metadata, persistence, and query endpoints without changing how cases run or how teams that don't claim cases operate (AC12).

---

## Components

### 1. Claim State Model

**Purpose:** Track which reviewer owns a paused case, when they claimed it, and the handover history.

**State fields** (added to LangGraph `State` TypedDict in `src/agents/graph/states.py`):

```python
class Claim(TypedDict):
    reviewer_id: str
    claimed_at: str  # ISO 8601 timestamp
    reason: NotRequired[str]  # populated on handover only

class State(TypedDict):
    # ... existing fields ...
    claim: NotRequired[Claim | None]
    claim_history: NotRequired[list[Claim]]
```

- **`claim`**: the current claim (reviewer, timestamp, optional handover reason), or `None` when unclaimed
- **`claim_history`**: append-only log of every claim, release, and handover (each entry carries `reviewer_id`, `claimed_at`, and `reason` when applicable)

**Transitions:**

- **Unclaimed → Claimed:** reviewer calls claim endpoint → state gets `{"reviewer_id": "<id>", "claimed_at": "<timestamp>"}`, appended to history
- **Claimed → Released:** owning reviewer calls release endpoint → `claim` set to `None`, release event appended to history with reason `"released"`
- **Claimed by A → Claimed by B (handover):** reviewer A calls handover endpoint with `{"to_reviewer": "B", "reason": "<text>"}` → `claim` updated to B's ID and timestamp, history appended with A's handover entry carrying the reason

**Validation rules:**

- Only cases in `status: "needs_review"` can be claimed (AC10)
- Only the owning reviewer can answer flags, approve, reject, release, or hand over (AC2, AC3, AC4)
- Handover requires a non-empty `reason` (AC4)
- Claims, releases, and handovers are immutable history entries (AC5)

---

### 2. Persistence Layer

**Purpose:** Survive service restarts (AC11).

**Approach:** LangGraph checkpointer already persists the full `State` to Redis (or SQLite in `db_path` mode). Claim fields added to `State` are automatically persisted and restored by the existing `get_checkpointer()` flow in `src/agents/runtime/checkpointer.py`.

**No new persistence code required** — claim state lives in the same checkpoint as the rest of the case state and is restored when `resume_service_order` reloads the thread.

**Trade-off:** Claim history grows unbounded in the state. For a case with 50 handovers, the checkpoint size increases proportionally. Acceptable for MVP (cases rarely hand over more than a few times); future optimization could archive history to a separate store after case completion.

---

### 3. API Contracts

**Purpose:** Expose claim, release, handover, and query operations via REST endpoints.

**New routes** (added to `src/agents/runtime/server.py`):

#### 3.1. Claim a case

```
POST /runs/{thread_id}/claim
Body: {"reviewer_id": "<id>"}
Response: 200 {"status": "claimed", "claim": {"reviewer_id": "<id>", "claimed_at": "<timestamp>"}}
          409 {"error": "already_claimed", "claim": {"reviewer_id": "<other>", "claimed_at": "<when>"}}
          400 {"error": "not_claimable", "reason": "case is <status>"}
```

**Logic:**

1. Load snapshot via `_snapshot(thread_id)`
2. Validate `state["status"] == "needs_review"` (else 400)
3. If `state.get("claim")` exists, return 409 with current claim
4. Update state: `{"claim": {"reviewer_id": <id>, "claimed_at": <now>}, "claim_history": [...history, same_claim]}`
5. Persist checkpoint and return 200

#### 3.2. Release a claim

```
POST /runs/{thread_id}/release
Body: {"reviewer_id": "<id>"}
Response: 200 {"status": "released"}
          403 {"error": "not_owner", "claim": {"reviewer_id": "<other>", ...}}
          400 {"error": "not_claimed"}
```

**Logic:**

1. Load snapshot
2. If no claim, return 400
3. If `claim["reviewer_id"] != <id>`, return 403
4. Update state: `{"claim": None, "claim_history": [...history, {"reviewer_id": <id>, "claimed_at": <now>, "reason": "released"}]}`
5. Persist and return 200

#### 3.3. Hand over a claim

```
POST /runs/{thread_id}/handover
Body: {"from_reviewer": "<id>", "to_reviewer": "<other_id>", "reason": "<text>"}
Response: 200 {"status": "handed_over", "claim": {"reviewer_id": "<other_id>", "claimed_at": "<timestamp>"}}
          403 {"error": "not_owner", ...}
          400 {"error": "reason_required"}
```

**Logic:**

1. Load snapshot
2. Validate `state["claim"]["reviewer_id"] == from_reviewer` (else 403)
3. Validate `reason` is non-empty (else 400)
4. Update state: `{"claim": {"reviewer_id": to_reviewer, "claimed_at": <now>}, "claim_history": [...history, {"reviewer_id": from_reviewer, "claimed_at": <now>, "reason": <reason>}]}`
5. Persist and return 200

#### 3.4. Reviewer's queue

```
GET /reviewers/{reviewer_id}/queue
Response: 200 {"cases": [{"thread_id": "<id>", "employee": "<name>", "case_type": "<type>", "waiting_seconds": <int>}, ...]}
```

**Logic:**

1. Query all threads with `status: "needs_review"` and `claim.reviewer_id == <id>` from Redis (or SQLite)
2. For each, compute `waiting_seconds = now - claim.claimed_at`
3. Sort by `waiting_seconds` descending (oldest first, AC6)
4. Return list with `employee` from `state["service_order"]["employee"]`, `case_type` from `state["service_order"]["caseType"]`

**Data source:** Checkpointer already indexes threads by namespace; this query scans `status` and `claim` fields. **Trade-off:** Full scan for reviewers with many cases is O(n); acceptable for MVP (dozens of cases per reviewer). Future optimization: maintain a Redis sorted set keyed by reviewer, scored by claim timestamp.

#### 3.5. Lead's waiting list

```
GET /leads/waiting
Query: ?threshold_hours=4
Response: 200 {"cases": [{"thread_id": "<id>", "employee": "<name>", "case_type": "<type>", "waiting_seconds": <int>}, ...]}
```

**Logic:**

1. Load `threshold_hours` (default 4, from query param or config, AC9)
2. Query all threads with `status: "needs_review"` and `claim == None`
3. For each, compute `waiting_seconds = now - <timestamp when status became needs_review>`
4. Filter where `waiting_seconds > threshold_hours * 3600`
5. Sort by `waiting_seconds` descending (longest first, AC8)
6. Return list

**Timestamp source:** The timestamp when `status` became `"needs_review"` is not currently tracked. **Design choice:** Add a `needs_review_since` field to `State` (set when orchestrator transitions to review, preserved on resume). Alternative: derive from last step timestamp, but that's less precise.

---

### 4. Configurable Timeout Mechanism

**Purpose:** The 4-hour threshold can change without a release (AC9).

**Approach:** Store threshold in **environment variable** or **external config file** read at request time.

**Implementation:**

```python
# src/agents/runtime/server.py
import os

def _waiting_threshold_seconds() -> int:
    hours = float(os.getenv("CLAIM_WAITING_THRESHOLD_HOURS", "4"))
    return int(hours * 3600)
```

**Trade-off:** Environment variable requires service restart (not a code release, but still downtime). For zero-downtime updates, read from a Redis key or a mounted config file checked on every request.

**Recommendation:** Start with env var (simpler), migrate to Redis key if runtime updates are critical.

---

### 5. Authorization Check

**Purpose:** Only the owning reviewer can act on a claimed case (AC2).

**Approach:** Middleware function `_require_owner(state, reviewer_id)` shared by `/runs/{thread_id}/answer_flags`, `/runs/{thread_id}/resume` (approve/reject), release, and handover endpoints.

**Logic:**

```python
def _require_owner(state: dict, reviewer_id: str) -> tuple[bool, dict | None]:
    claim = state.get("claim")
    if claim and claim["reviewer_id"] != reviewer_id:
        return False, {"error": "not_owner", "claim": claim}
    return True, None
```

Called before processing flag answers or review decisions; returns 403 if unauthorized.

**Existing endpoints to modify:**

- `POST /runs/{thread_id}/resume` (flag answers and review decision) — add `reviewer_id` to request body, check ownership
- Future flag answer endpoint (if split from resume) — same check

---

### 6. Backward Compatibility

**Purpose:** Teams that never claim cases see no change (AC12).

**Approach:**

- `claim` and `claim_history` fields are `NotRequired` — existing cases have them as `None` or absent
- Claim endpoints are opt-in — if a team never calls them, the workflow is unchanged
- Review flow (`/runs/{thread_id}/resume`) works with or without a claim: if no claim, no ownership check is applied

**Test coverage:** Run a case end-to-end without claiming it; verify no errors and history is clean.

---

## Data Flow

**Claim flow:**

1. Reviewer opens case details UI → sees `state["claim"]` (null or owner info)
2. Clicks "Claim" → POST `/runs/{thread_id}/claim` with `reviewer_id`
3. Server validates state, updates claim, persists checkpoint
4. UI refreshes, shows "Claimed by <you>, <timestamp>"

**Handover flow:**

1. Reviewer A clicks "Hand Over" → UI prompts for recipient and reason
2. POST `/runs/{thread_id}/handover` with `from_reviewer`, `to_reviewer`, `reason`
3. Server validates ownership, updates claim to B, appends handover to history
4. UI shows "Handed over to <B>" and case leaves A's queue, enters B's queue

**Queue view:**

1. Reviewer clicks "My Cases" → GET `/reviewers/{id}/queue`
2. Server scans threads, filters by `claim.reviewer_id`, sorts by wait time
3. UI renders list with employee, case type, waiting duration

**Lead view:**

1. Lead clicks "Waiting Cases" → GET `/leads/waiting?threshold_hours=4`
2. Server scans unclaimed threads, filters by wait time > threshold
3. UI renders list, sorted longest first

---

## Error Handling

| Error                          | HTTP Code | Response                                                              |
|--------------------------------|-----------|-----------------------------------------------------------------------|
| Case not in `needs_review`     | 400       | `{"error": "not_claimable", "reason": "case is <status>"}`            |
| Already claimed by someone else| 409       | `{"error": "already_claimed", "claim": {"reviewer_id": "<id>", ...}}` |
| Claim action by non-owner      | 403       | `{"error": "not_owner", "claim": {"reviewer_id": "<id>", ...}}`       |
| Handover without reason        | 400       | `{"error": "reason_required"}`                                        |
| Thread not found               | 404       | `{"error": "thread_not_found"}`                                       |

---

## File Modifications

| File                             | Change                                                                                     |
|----------------------------------|--------------------------------------------------------------------------------------------|
| `src/agents/graph/states.py`    | Add `Claim` TypedDict, `claim` and `claim_history` fields to `State`, add `needs_review_since` |
| `src/agents/runtime/server.py`  | Add `/claim`, `/release`, `/handover`, `/reviewers/{id}/queue`, `/leads/waiting` endpoints |
| `src/agents/runtime/server.py`  | Add `_require_owner()` middleware, modify `/resume` to check ownership                     |
| `src/agents/runtime/server.py`  | Add `_waiting_threshold_seconds()` config reader                                           |
| `src/agents/graph/edges.py`     | Set `needs_review_since` when transitioning to `needs_review` status                       |

**No new files required** — all changes extend existing modules.

---

## Risks

### 1. Concurrent claims

**Risk:** Two reviewers claim the same case simultaneously.

**Mitigation:** Redis checkpointer uses optimistic locking (checkpoint version increment). The second claim write fails and retries, sees the case is already claimed, returns 409. **No data corruption**, but one reviewer sees a stale "available" state briefly.

**Residual risk:** Low. UI should poll or use server-sent events to reflect live claim state.

### 2. Unbounded claim history

**Risk:** A case handed over 100 times balloons the checkpoint.

**Mitigation:** Accept for MVP (rare). Future: archive `claim_history` to a separate store after case completion, keep only last N entries in state.

### 3. Query performance

**Risk:** Scanning all threads for `/queue` and `/waiting` is slow with 10,000 paused cases.

**Mitigation:** Acceptable for MVP (dozens to hundreds of cases). Future: maintain Redis sorted sets indexed by reviewer and wait time, updated on claim/release.

### 4. Threshold change lag

**Risk:** Changing `CLAIM_WAITING_THRESHOLD_HOURS` env var requires restart.

**Mitigation:** For MVP, restart is acceptable (no data loss, just brief downtime). Future: read from Redis key polled every request.

---

## Open Questions

*None at this time. All design choices are made and documented above.*

---

## Architecture Decisions

### AD-1: Claim state lives in LangGraph State

**Binds:** Claim ownership, history, and timestamps are fields in the existing `State` TypedDict.

**Prevents:** Claim data diverging from case state or surviving in a separate store when the case is deleted.

**Rule:** All claim mutations update `State` and rely on the checkpointer for persistence and atomicity.

**Status:** [ADOPTED] — the repo already uses LangGraph state for all case metadata.

---

### AD-2: Persistence via existing checkpointer

**Binds:** Claims are persisted by the Redis (or SQLite) checkpointer that already saves the full `State`.

**Prevents:** Adding a second persistence layer (database, separate Redis keys) that could drift or fail independently.

**Rule:** No new persistence code. Claim fields are automatically saved and restored with the rest of the state.

**Status:** [ADOPTED] — the repo uses `src/agents/runtime/checkpointer.py` for all state.

---

### AD-3: Timeout threshold via environment variable

**Binds:** The 4-hour (default) waiting threshold is read from `CLAIM_WAITING_THRESHOLD_HOURS` env var.

**Prevents:** Hardcoding the threshold in code, requiring a release to change it.

**Rule:** The threshold is read at request time (GET `/leads/waiting`). Changing it requires a service restart, not a code deploy.

**Status:** [ADOPTED] — balances simplicity (no new config system) and flexibility (no code change).

---

### AD-4: Authorization via claim ownership check

**Binds:** Endpoints that modify a case (answer flags, approve, reject, release, handover) check `state["claim"]["reviewer_id"]` against the request's `reviewer_id`.

**Prevents:** Unauthorized reviewers modifying a claimed case.

**Rule:** All mutation endpoints call `_require_owner(state, reviewer_id)` before proceeding. Returns 403 if unauthorized.

**Status:** [ADOPTED] — enforces AC2 with a single shared check.

---

### AD-5: Handover requires a reason

**Binds:** The handover endpoint rejects requests without a non-empty `reason` field.

**Prevents:** Handovers without context, making history unhelpful.

**Rule:** Validate `reason` is present and non-empty before updating state. Return 400 if missing.

**Status:** [ADOPTED] — enforces AC4.

---

### AD-6: Backward compatibility via optional fields

**Binds:** `claim` and `claim_history` are `NotRequired` fields in `State`.

**Prevents:** Breaking existing cases that don't use claims.

**Rule:** Claim logic is opt-in. If `claim` is absent or `None`, the case behaves as today. No ownership checks are applied to unclaimed cases.

**Status:** [ADOPTED] — enforces AC12.

---

## Deferred

- **Notifications:** Push/email alerts when a case is claimed, handed over, or waiting too long (out of scope per ticket).
- **Automatic assignment:** Routing cases to specific reviewers or teams (out of scope).
- **Reviewer roles:** Distinguishing reviewer permissions beyond "lead" (out of scope).
- **Claim history archival:** Moving completed cases' claim history to a separate store (defer to future optimization).
- **Queue indexes:** Redis sorted sets for faster `/queue` and `/waiting` queries (defer to future optimization).
- **Zero-downtime threshold updates:** Reading threshold from Redis key instead of env var (defer to future enhancement).
