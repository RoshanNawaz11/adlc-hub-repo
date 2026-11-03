# Low-Level Design — Story 03-agent

**Type:** Story  
**Ticket ID:** PER-113 (Story 03-agent)  
**Title:** Document idempotency configuration and update module docstring  
**Repo:** `trinet-adlc-agents`  
**Status:** Draft

---

## Acceptance Criteria

From **requirements.md** (unchanged, copied verbatim):

### AC1: `.env.example` documents `IDEMPOTENCY_BACKEND`
**Given** a developer setting up the project  
**When** they open `.env.example`  
**Then** they find an entry for `IDEMPOTENCY_BACKEND` with allowed values `memory` or `redis`, default `memory`, and inline comment: "Idempotency store backend. Use 'memory' for local dev (no Redis required), 'redis' for production (shared state across workers)."

### AC2: `.env.example` documents `IDEMPOTENCY_CLAIM_TTL_SECONDS`
**Given** a developer configuring TTLs  
**When** they open `.env.example`  
**Then** they find an entry for `IDEMPOTENCY_CLAIM_TTL_SECONDS` with default `3600` and inline comment: "Claim TTL in seconds. Must be >= ORDER_JOB_TIMEOUT to avoid duplicate execution. Default matches ORDER_JOB_TIMEOUT (1 hour)."

### AC3: `.env.example` documents `IDEMPOTENCY_COMPLETED_TTL_SECONDS`
**Given** a developer configuring TTLs  
**When** they open `.env.example`  
**Then** they find an entry for `IDEMPOTENCY_COMPLETED_TTL_SECONDS` with default `604800` and inline comment: "Completed run TTL in seconds. Bounds memory usage for historical run keys. Default matches STATUS_TTL_SECONDS (7 days)."

---

## File Plan

### Modified Files

| Path | Change Summary | Lines (est.) |
|------|----------------|--------------|
| `.env.example` | Add three new environment variables with inline comments. Place near existing `REDIS_URL` / `ORDER_JOB_TIMEOUT` / `STATUS_TTL_SECONDS` entries if they exist. | +12 |
| `src/agents/runtime/idempotency.py` | Validate module docstring (no code changes; docstring already rewritten in Story 1). Ensure it includes all required sections per requirements. | ~60 (docstring lines, validation only) |

**Total estimated LOC:** ~72 (12 new lines in `.env.example`, 60 lines validated in `idempotency.py` docstring).

---

## Interfaces

No new interfaces. This story validates existing contracts from Stories 1 and 2.

**Referenced contracts:**

- **Redis key format:** `run-idempotency:{idempotency_key}` (from HLD AD-2, implemented in Story 1).
- **Redis value schema:** `{"state": "running"|"completed", "thread_id": "<uuid>", "claimed_at": "<iso8601>", "completed_at": "<iso8601>"}` (from HLD AD-2, implemented in Story 1).
- **Config variables:** `IDEMPOTENCY_BACKEND`, `IDEMPOTENCY_CLAIM_TTL_SECONDS`, `IDEMPOTENCY_COMPLETED_TTL_SECONDS` (from HLD AD-5, implemented in Story 1, documented here).

---

## Implementation Details

### `.env.example` Changes

**Goal:** Document the three new environment variables with inline comments that explain their purpose, defaults, and relationship to existing config (`ORDER_JOB_TIMEOUT`, `STATUS_TTL_SECONDS`).

**Insertion point:** After `REDIS_URL` (or near other queue/worker config if `REDIS_URL` is not present in the file). If no natural grouping exists, create an "Idempotency" section with a comment header.

**New entries:**

```env
# Idempotency store backend. Use 'memory' for local dev (no Redis required), 'redis' for production (shared state across workers).
IDEMPOTENCY_BACKEND=memory

# Claim TTL in seconds. Must be >= ORDER_JOB_TIMEOUT to avoid duplicate execution. Default matches ORDER_JOB_TIMEOUT (1 hour).
IDEMPOTENCY_CLAIM_TTL_SECONDS=3600

# Completed run TTL in seconds. Bounds memory usage for historical run keys. Default matches STATUS_TTL_SECONDS (7 days).
IDEMPOTENCY_COMPLETED_TTL_SECONDS=604800
```

**Verification steps (manual):**

1. Open `.env.example`.
2. Locate the three new variables.
3. Verify each has:
   - The correct default value (`memory`, `3600`, `604800`).
   - An inline comment (starting with `#`) on the line above or same line.
   - A comment that references the related config variable (`ORDER_JOB_TIMEOUT` for claim TTL, `STATUS_TTL_SECONDS` for completed TTL).
4. Verify the placement follows existing conventions (e.g. grouped with other queue/worker config, not randomly inserted).

**Edge cases:**

- If `ORDER_JOB_TIMEOUT` or `STATUS_TTL_SECONDS` do not exist in `.env.example`, omit the cross-reference in the comment. Instead, state the constraint directly: "Must be >= typical job duration" or "Controls how long completed runs are retained."
- If `.env.example` does not exist, create it with these three entries only (document this in the PR if it happens).

---

### Module Docstring Validation

**File:** `src/agents/runtime/idempotency.py`

**Goal:** Validate that the module docstring (written in Story 1) includes all required sections per AC4 and aligns with HLD AD-1, AD-2, AD-5, AD-6.

**Expected docstring structure (from Story 1 spec):**

```python
"""
Durable idempotency for agent runs across worker restarts and processes.

This module prevents replayed queue messages from executing the graph multiple times
by tracking run keys in a shared store. The store abstraction (`IdempotencyStore`)
supports two backends:

- `MemoryIdempotencyStore`: In-process dict-backed store for local development.
  No Redis required. State is lost on restart (acceptable for dev).
- `RedisIdempotencyStore`: Durable shared state backed by Redis. Required for
  production deployments with multiple workers.

Configuration
-------------
Set `IDEMPOTENCY_BACKEND=memory|redis` to select the store (default: memory).
Factory function `get_idempotency_store()` reads the variable and validates it.

Store Operations
----------------
- `claim(key, thread_id)`: Atomically claim a run key. Returns:
  - `"claimed"`: This worker now owns the run. Proceed to execute the graph.
  - `"running"`: Another worker currently owns the run. Skip execution.
  - `"completed"`: The run already finished. Skip execution.
  
- `complete(key, thread_id)`: Mark the run as completed. Idempotent (safe to call
  multiple times). Sets a TTL to bound memory usage.
  
- `release(key, thread_id)`: Drop the claim (called when a run pauses, fails, or
  is cancelled). Allows the run to be retried later.

Redis Key Contract
------------------
Key format: `run-idempotency:{idempotency_key}`
Value: JSON `{"state": "running"|"completed", "thread_id": "<uuid>", "claimed_at": "<iso8601>", ...}`

TTLs:
- Claim TTL (`IDEMPOTENCY_CLAIM_TTL_SECONDS`, default 3600): Expires stale claims
  from crashed workers. Must be >= ORDER_JOB_TIMEOUT to prevent duplicate execution
  of long-running jobs.
- Completed TTL (`IDEMPOTENCY_COMPLETED_TTL_SECONDS`, default 604800): Expires old
  completed runs to bound Redis memory usage. Matches STATUS_TTL_SECONDS.

Atomicity: Claims use Redis `SET ... NX EX` (atomic test-and-set). Only one worker
can claim a given key at a time.

Error Handling
--------------
- Claim failures (Redis unreachable): Raise `IdempotencyUnavailable`. The caller
  (runner) must not start the graph when idempotency cannot be guaranteed.
- Complete/release failures (Redis unreachable): Log an error but do not raise.
  The run result is preserved. The claim TTL cleans up stale entries eventually.

Integration
-----------
The runner (`src/agents/runtime/runner.py`) calls `claim` before starting the graph,
`complete` when the run finishes, and `release` when the run pauses, fails, or is
cancelled. The worker (`src/agents/queue/orders.py`) skips status writes when
another worker owns the run (claim returned `"running"`).

See Also
--------
- PER-101 HLD: Full architecture and Redis key contract.
- `src/agents/runtime/runner.py`: Integration into `run_query`.
- `src/agents/queue/orders.py`: Worker status write guard.
"""
```

**Verification steps (manual):**

1. Open `src/agents/runtime/idempotency.py`.
2. Verify the module docstring (first multiline string after any initial imports) matches the expected structure above.
3. Check each required section is present:
   - **Overview paragraph:** Describes purpose (prevent duplicate execution), store abstraction, two backends.
   - **Configuration:** `IDEMPOTENCY_BACKEND`, factory function.
   - **Store Operations:** `claim`, `complete`, `release` with return values and semantics.
   - **Redis Key Contract:** Key format, JSON value schema, TTLs (both claim and completed), atomicity guarantee.
   - **Error Handling:** Claim failures raise, complete/release failures log.
   - **Integration:** References runner and worker integration points.
   - **See Also:** Links to HLD, runner, worker files.
4. Cross-check against HLD:
   - **AD-1 (Store Abstraction):** Docstring describes both `MemoryIdempotencyStore` and `RedisIdempotencyStore` with same semantics.
   - **AD-2 (Redis Key Contract):** Docstring states key format `run-idempotency:{key}`, JSON schema, TTLs (3600 for claim, 604800 for completed), atomicity via `SET NX EX`.
   - **AD-5 (Configuration):** Docstring documents `IDEMPOTENCY_BACKEND`, `IDEMPOTENCY_CLAIM_TTL_SECONDS`, `IDEMPOTENCY_COMPLETED_TTL_SECONDS` with defaults and constraints.
   - **AD-6 (Error Handling):** Docstring states claim failures raise `IdempotencyUnavailable`, complete/release failures log but do not raise.
5. Verify no contradictions:
   - TTL defaults match `.env.example` entries (3600, 604800).
   - Key format matches HLD (`run-idempotency:{key}`, not a different prefix).
   - Error handling matches Story 1 implementation (claim raises, complete/release log).

**Edge cases:**

- If the docstring is missing any section (e.g. "TTLs" subsection), flag it as incomplete.
- If the docstring contradicts the HLD (e.g. claims are not atomic, different key format), flag as a blocker.
- If the docstring uses placeholder text (e.g. `TODO: document TTLs`), flag as incomplete.

---

## Algorithm Notes

No algorithms (documentation-only story). This section validates that the documented contracts are consistent with the implementation in Stories 1 and 2.

**Consistency checks:**

1. **`.env.example` TTL defaults match Story 1 code:**
   - `IDEMPOTENCY_CLAIM_TTL_SECONDS=3600` must match the default in `RedisIdempotencyStore.__init__` (from Story 1: `claim_ttl = int(os.getenv("IDEMPOTENCY_CLAIM_TTL_SECONDS", "3600"))`).
   - `IDEMPOTENCY_COMPLETED_TTL_SECONDS=604800` must match the default in `RedisIdempotencyStore.__init__` (from Story 1: `completed_ttl = int(os.getenv("IDEMPOTENCY_COMPLETED_TTL_SECONDS", "604800"))`).

2. **Module docstring key format matches Story 1 code:**
   - Docstring: `run-idempotency:{idempotency_key}`.
   - Code (from Story 1): `def _redis_key(self, key: str) -> str: return f"run-idempotency:{key}"`.

3. **Module docstring claim semantics match Story 1 code:**
   - Docstring: Returns `"claimed"`, `"running"`, or `"completed"`.
   - Code (from Story 1): `claim` method returns `Literal["claimed", "running", "completed"]`.

4. **Module docstring error handling matches Story 1 code:**
   - Docstring: Claim failures raise `IdempotencyUnavailable`, complete/release failures log.
   - Code (from Story 1): `claim` raises `IdempotencyUnavailable` on `redis.exceptions.ConnectionError`; `complete` and `release` log errors but do not raise.

If any inconsistency is found during validation, it is a **blocker** for this story (requires fixing Story 1 code or docstring to align).

---

## Tests

### Verification Method

**Automated:** 0 tests (documentation-only story). Human review via PR.

**Manual checks (PR review checklist):**

1. **AC1-AC3 (`.env.example` validation):**
   - [ ] Open `.env.example`.
   - [ ] Verify `IDEMPOTENCY_BACKEND=memory` is present with inline comment: "Idempotency store backend. Use 'memory' for local dev (no Redis required), 'redis' for production (shared state across workers)."
   - [ ] Verify `IDEMPOTENCY_CLAIM_TTL_SECONDS=3600` is present with inline comment referencing `ORDER_JOB_TIMEOUT`: "Claim TTL in seconds. Must be >= ORDER_JOB_TIMEOUT to avoid duplicate execution. Default matches ORDER_JOB_TIMEOUT (1 hour)."
   - [ ] Verify `IDEMPOTENCY_COMPLETED_TTL_SECONDS=604800` is present with inline comment referencing `STATUS_TTL_SECONDS`: "Completed run TTL in seconds. Bounds memory usage for historical run keys. Default matches STATUS_TTL_SECONDS (7 days)."
   - [ ] Verify the three variables are placed near other queue/worker config (e.g. `REDIS_URL`, `ORDER_JOB_TIMEOUT`), not randomly inserted.

2. **Module docstring validation:**
   - [ ] Open `src/agents/runtime/idempotency.py`.
   - [ ] Verify the module docstring includes:
     - [ ] Overview: Purpose (prevent duplicate execution), store abstraction, two backends.
     - [ ] Configuration: `IDEMPOTENCY_BACKEND`, factory function.
     - [ ] Store Operations: `claim`, `complete`, `release` with return values.
     - [ ] Redis Key Contract: Key format, JSON schema, TTLs (claim 3600, completed 604800), atomicity.
     - [ ] Error Handling: Claim failures raise, complete/release failures log.
     - [ ] Integration: References runner and worker.
     - [ ] See Also: Links to HLD, runner, worker files.

3. **HLD alignment:**
   - [ ] Cross-check docstring against HLD AD-1, AD-2, AD-5, AD-6 (see "Verification steps (manual)" above).
   - [ ] Verify no contradictions (key format, TTL defaults, error semantics, atomicity).

4. **`.env.example` format consistency:**
   - [ ] Verify the new entries follow the same comment style as existing entries (e.g. `#` on the line above, or same line if that's the convention).
   - [ ] Verify no trailing whitespace or inconsistent indentation.

**Test count:** 0 automated tests (manual verification only, per breakdown verification approach).

---

## Dependencies

### External Dependencies

None (documentation-only story).

### Internal Dependencies

**Blocked by Story 1 (PER-111) and Story 2 (PER-112):**

This story validates the documentation written in Story 1 (module docstring) and aligns it with the integration implemented in Story 2. It cannot complete until both stories are merged, because:

- The module docstring is written as part of Story 1's file changes.
- The `.env.example` entries must align with the config variables referenced in Stories 1 and 2 (`IDEMPOTENCY_BACKEND`, `IDEMPOTENCY_CLAIM_TTL_SECONDS`, `IDEMPOTENCY_COMPLETED_TTL_SECONDS`).

**This story is the last in the sequence** (Story 1 → Story 2 → Story 3).

---

## Risks & Mitigations

### Risk 1: Documentation Drift

**Impact:** If the docstring or `.env.example` comments are not updated when the code changes (e.g. new config variable, altered TTL defaults), developers will be misled.

**Mitigation:** This story completes after Stories 1 and 2 are merged, validating the documentation against the final code. Any code changes that alter the config or contract must update the documentation in the same PR. PR template includes a checklist item: "Documentation updated (module docstrings, `.env.example`)."

**Residual risk:** Low. The documentation is validated in this story; future drift is a maintenance concern (out of scope for this ticket).

---

### Risk 2: Unclear Relationships Between Config Variables

**Impact:** If the inline comments do not explain the relationship between `IDEMPOTENCY_CLAIM_TTL_SECONDS` and `ORDER_JOB_TIMEOUT` (or between `IDEMPOTENCY_COMPLETED_TTL_SECONDS` and `STATUS_TTL_SECONDS`), operators may set them incorrectly, causing duplicate execution or unbounded memory growth.

**Mitigation:** The `.env.example` comments explicitly reference the related variables and state the constraint ("Must be >= ORDER_JOB_TIMEOUT"). The module docstring reinforces this ("Claim TTL must be >= ORDER_JOB_TIMEOUT to prevent duplicate execution of long-running jobs"). Future work could add a startup validation warning (per HLD Risk 2), but that is out of scope for this ticket.

**Residual risk:** Medium. Documentation reduces the risk but does not eliminate it (operators can still ignore the comments). A validation warning (future ticket) would catch misconfigurations at startup.

---

### Risk 3: Missing Cross-References

**Impact:** If `ORDER_JOB_TIMEOUT` or `STATUS_TTL_SECONDS` do not exist in `.env.example`, the cross-reference in the comment will confuse developers.

**Mitigation:** The implementation notes include a fallback: "If `ORDER_JOB_TIMEOUT` or `STATUS_TTL_SECONDS` do not exist in `.env.example`, omit the cross-reference in the comment. Instead, state the constraint directly: 'Must be >= typical job duration' or 'Controls how long completed runs are retained.'"

**Residual risk:** Low. The fallback ensures the comment is still load-bearing.

---

## Out of Scope

- Code changes to `src/agents/runtime/idempotency.py` (Story 1 / PER-111).
- Integration into `src/agents/runtime/runner.py` or `src/agents/queue/orders.py` (Story 2 / PER-112).
- Test implementation (Stories 1 and 2).
- Startup validation warning for TTL misconfiguration (future work per HLD Risk 2).
- Changes to the `order-run:{thread_id}` status contract or fastapi-backend.
- The checkpointer and its Redis TODO.
- Changing how the idempotency key is built (`idempotency_key(task)` remains unchanged).
- Model gateway, tracing, node logic, prompts.

---

## Definition of Done

- [ ] `.env.example` updated with three new variables (`IDEMPOTENCY_BACKEND`, `IDEMPOTENCY_CLAIM_TTL_SECONDS`, `IDEMPOTENCY_COMPLETED_TTL_SECONDS`) and inline comments (AC1-AC3).
- [ ] `src/agents/runtime/idempotency.py` module docstring validated (all required sections present, no contradictions with HLD).
- [ ] PR description includes manual verification steps (checklist in "Tests" section).
- [ ] No contradictions or omissions relative to the HLD (AD-1, AD-2, AD-5, AD-6) or requirements (AC1-AC3).
- [ ] New `.env.example` entries follow existing format and placement conventions.
- [ ] Stories 1 (PER-111) and 2 (PER-112) are merged (this story cannot complete without them).

---

**End of LLD — Story 03-agent (PER-113)**
