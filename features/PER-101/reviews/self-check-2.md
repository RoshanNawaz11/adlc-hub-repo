# Consistency Check — PER-101

**FAIL**

## Concrete Mismatches

### 1. Test Count Misalignment

**Breakdown verification approach:**
- Story 1 (PER-111): **11 tests** in `tests/test_idempotency.py`
- Story 2 (PER-112): **9 tests** (8 in `tests/test_runner.py`, 1 in `tests/test_orders.py`)

**Story LLD:**
- Story 01-agent (PER-111): **11 tests** ✓ (matches)
- Story 02-agent (PER-112): **9 tests** ✓ (matches)

**Requirements AC11:**
> Test suite passes, covering: [...] runner integration (claim outcomes, pause/exception/cancellation release), and worker status write guard for "already running"

**Issue:** Requirements AC11 states the test suite must cover all these areas but does not pin the exact test count. The breakdown pins 11 + 9 = 20 tests, but no artifact validates that these 20 tests collectively satisfy AC11's coverage mandate. The specs and stories list the test cases but do not cross-reference AC11's coverage requirements.

**Fix needed:** Add a traceability note in each story LLD confirming which AC11 coverage areas each test satisfies, or add a summary in breakdown stating "AC11 verification: 20 tests collectively cover [list areas]."

---

### 2. HLD Outcome vs. LLD Implementation Gap

**HLD AD-3 (Runner Integration Points):**
> Resume flows do not claim. A resumed run calls `complete` when it finishes.

**Story 02-agent LLD (Resume Flow and `complete`):**
> A resumed run is the continuation of a run that paused earlier. The original run already claimed the key when it started. When the run is resumed:
> - Skipping `claim` avoids a spurious `"running"` return (the key is already `running` from the original claim).
> - Calling `complete` when the resumed run finishes marks the run done (transitions the store entry from `running` to `completed`).

**Issue:** The HLD specifies the behavior but does not explain the **outcome** for the user (what happens if a resumed run completes vs. pauses again). The LLD explains the state transition but does not describe the observable result (e.g., "The resumed run's status is written as `done` by the worker"). The outcome is implicit (inferred from Story 2's worker status logic) but not explicit in HLD AD-3.

**Fix needed:** HLD AD-3 should add: "Resumed runs that complete mark the store entry as `completed` and the worker writes `done` status. Resumed runs that pause again call `release` (same as fresh runs)."

---

### 3. File Path Inconsistency (Minor)

**Breakdown Story 1 (PER-111) verification approach:**
> 11 unit tests in `tests/test_idempotency.py` (no live Redis; use `unittest.mock.AsyncMock`)

**Story 01-agent LLD Tests section:**
> Test File: `tests/test_idempotency.py`

**Breakdown Story 2 (PER-112) verification approach:**
> 9 integration/worker tests (8 in `tests/test_runner.py`, 1 in `tests/test_orders.py`)

**Story 02-agent LLD Tests section:**
> Test File: `tests/test_runner.py` [...] Test File: `tests/test_orders.py`

**Repo tree:**
- `tests/test_backend.py` exists
- `tests/test_decide.py` exists
- `tests/test_evaluate.py` exists
- `tests/test_idempotency.py` does NOT exist (new file per Story 1)
- `tests/test_runner.py` does NOT exist (new file per Story 2)
- `tests/test_orders.py` does NOT exist (new file per Story 2)

**Issue:** None—all test file paths are consistent and correctly marked as new files. However, the repo tree shows existing test files use snake_case (e.g., `test_backend.py`), and the new files follow the same convention. No mismatch.

**Resolution:** No fix needed. This is consistent.

---

### 4. Missing Cross-Reference in Breakdown → Stories

**Breakdown Story 1 (PER-111) "Covers (Requirements)":**
> AC4 (released claim re-claimable), AC5 (TTL expires stale claim), AC9 (completed entry TTL), AC10 (claim atomicity and TTL). Implements contracts validated in Story 2 by AC1, AC2, AC3, AC6, AC7.

**Breakdown Story 2 (PER-112) "Covers (Requirements)":**
> AC8 (worker skips status write for already running). Validates contracts from Story 1: AC1 (in-memory replay skips), AC2 (concurrent claims), AC3 (completed run skips), AC6 (claim failure raises), AC7 (complete/release failure logged).

**Story 01-agent LLD Acceptance Criteria:**
Lists AC4, AC5, AC6, AC7, AC9, AC10. **Does not list AC1, AC2, AC3** (which breakdown says are "validated in Story 2").

**Story 02-agent LLD Acceptance Criteria:**
Lists **only AC8**. Does not list AC1, AC2, AC3, AC6, AC7 (which breakdown says are "validated" here).

**Issue:** Breakdown states Story 2 "validates" AC1/AC2/AC3/AC6/AC7, but Story 2's LLD only lists AC8 in its Acceptance Criteria section. The tests in Story 2 LLD do cover these ACs (e.g., test 1 validates AC3, test 2 validates AC2), but the AC section does not list them.

**Fix needed:** Story 02-agent LLD should list AC1, AC2, AC3, AC6, AC7 in its Acceptance Criteria section (copied from requirements.md, marked as "validated via integration tests"). Or, breakdown should clarify "validates" means "tests the contracts" not "re-states the ACs."

---

### 5. Stack Mismatch (Redis Client Dependency)

**HLD AD-2 (Redis Key Contract):**
> Redis operations:
> - Claim: `SET run-idempotency:{key} <json_value> NX EX {claim_ttl}`

**Story 01-agent LLD Stack:**
> Redis Client: `redis.asyncio` (add to `pyproject.toml`)

**Repo tree `pyproject.toml`:**
> (Not shown, but breakdown Story 1 says: `pyproject.toml` (add `redis>=5.0.0` to dependencies))

**Issue:** The HLD does not specify the Redis client library version or Python package name. Story 1 spec says `redis>=5.0.0`, Story 1 LLD says `redis.asyncio` (the async submodule of the `redis` package). This is consistent within Story 1 artifacts, but the HLD should have specified the library to avoid ambiguity.

**Fix needed:** HLD should add a "Dependencies" section or note in AD-2: "Redis client: `redis.asyncio.Redis` from `redis>=5.0.0` Python package."

---

### 6. change_requests.md Not Present

**Task directive:**
> If change_requests.md is present, FAIL when artifacts still contradict it.

**Finding:** `change_requests.md` is not present in the input. No contradiction to check.

**Resolution:** No issue.

---

### 7. merged_plan.md Not Present

**Task directive:**
> Check consistency across [...] merged_plan.md when present.

**Finding:** `merged_plan.md` is not present. Plan step 7 (`merge_pack (pack)`) has not run yet.

**Resolution:** No issue (step 7 is pending per plan.md).

---

## Summary

**4 blockers:**

1. **Test count vs. AC11 coverage**: Breakdown pins 20 tests but does not map them to AC11's coverage requirements.
2. **HLD outcome gap**: AD-3 does not describe the observable result for resumed runs (status write behavior).
3. **Story 2 AC list incomplete**: Story 02-agent LLD does not list AC1/AC2/AC3/AC6/AC7 in its Acceptance Criteria section, though its tests cover them.
4. **HLD missing Redis library spec**: AD-2 does not name the Redis client library or version.

**0 non-blockers:**

- File paths are consistent (all new test files follow existing snake_case convention).
- `change_requests.md` and `merged_plan.md` are not present (no contradiction possible).
