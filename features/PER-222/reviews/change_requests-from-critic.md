# Open items from the critic for PER-222

Written by the critic, not approved by a person. Delete the items you disagree with and edit the rest; what is left is applied on the next run:
`adlc resume PER-222` (or `adlc start PER-222 --skip-dev`).

### C1 (gate_note)
- file: stories/01-agent.md
- section: ## File Plan
- quote: Add `type: NotRequired[str]`, `by: NotRequired[str]`, and `reason: NotRequired[str]` fields to `StepRecord` TypedDict (after line ~97, before `plan`):
- problem: The file plan says "after line ~97, before `plan`" but `StepRecord` ends at line 45 and the `plan` field is in `State` (line 97), not in `StepRecord`.
- change: Say "after line ~45, after the existing `pipeline` field" or "Add three new `NotRequired` fields to `StepRecord` TypedDict (after the existing fields, before the closing)".
---
- blocks because: The `StepRecord` TypedDict shown in `repo-file/agent/src/agents/graph/states.py` ends at line 45, and the fields listed come before `plan` at line 97, which is in the `State` TypedDict, not `StepRecord`. Dev would look for line 97 in the wrong class and either fail to find the right location or put the fields in the wrong TypedDict.
- why gate note: raised new in a re-check, about text no fix changed since it was read
- history:
  - round 2: raised (blocking)
  - round 2: gate note: raised new in a re-check, about text no fix changed since it was read

### C2 (gate_note)
- file: stories/01-agent.md
- section: ## File Plan
- quote: The modification to `StepRecord` is simpler and follows the pattern of other `NotRequired` fields already present (e.g. `result`, `evidence`, `rule`).
- problem: The LLD says the new fields follow the pattern of `NotRequired` fields like `result`, `evidence`, `rule`, but omits `feed` and `pipeline` from the list. The list is incomplete or unclear whether it's exhaustive.
- change: Say "follows the pattern of other `NotRequired` fields already present (`result`, `evidence`, `feed`, `rule`, `pipeline`)" or "follows the pattern of other `NotRequired` fields (e.g. `result`, `evidence`, `rule`)" with "e.g." to show it's not exhaustive.
---
- blocks because: The existing `StepRecord` fields shown in `repo-file` are `stage`, `agent`, `title`, `source`, `at` (all required), then `result`, `evidence`, `feed`, `rule`, `pipeline` (all `NotRequired`). Dev would add `type`, `by`, `reason` as additional `NotRequired` fields after these. However, the `feed` field (line 42) is also `NotRequired` but is not mentioned in the LLD's list of examples. If the pattern includes all `NotRequired` fields, the list should be complete; if it's just examples, it should say "e.g." more clearly.
- why gate note: raised new in a re-check, about text no fix changed since it was read
- history:
  - round 2: raised (blocking)
  - round 2: gate note: raised new in a re-check, about text no fix changed since it was read

### C6 (gate_note)
- file: stories/02-agent.md
- section: ## Tests Section
- quote: Test 1: Successful withdrawal from `needs_input`**
```python
@pytest.mark.asyncio
async def test_withdraw_from_needs_input_returns_200(client: TestClient, monkeypatch: pytest.MonkeyPatch) -> None
"""AC 1: A case waiting for flag answers can be withdrawn via API, returns 200 OK."""
thread_id = "test-api-1"
- why gate note: the critic gave no reason it blocks (no `blocks because`)
- history:
  - round 2: raised (blocking)
  - round 2: gate note: the critic gave no reason it blocks (no `blocks because`)

### C7 (gate_note)
- file: stories/07-agent.md
- section: ## File Plan
- quote: No changes to `resume_service_order`:**
Story 4 already added the withdrawn check to `resume_service_order` (around line 93). That check reads the state from the checkpointer before calling `_run`, and if `status == "withdrawn"`, it skips graph invocation and writes the queue status as `"withdrawn"` (not `"failed"`). The existing implementation from Story 4 handles resume operations on withdrawn cases; this story (Story 7) does not modify `resume_service_order`.
- match: stories/04-agent.md › ## File Plan › Modified Files › src/agents/queue/orders.py
- problem: Story 4's file plan adds the withdrawn check to `resume_service_order`, and Story 7 correctly states "no changes" to that function. The division of labor is clear: Story 4 handles resume (queue + API both delegate to the queue handler's check), Story 7 handles start/approve/reject. This is now consistent.
- change: (No change needed — the clarification was made.)
---
- severity: note
- blocks because: (Not blocking — this is consistent now, as requested in C3 from the last review.)
- why gate note: the critic marked it a note, not a blocker
- history:
  - round 1: raised (blocking)
  - round 1: applied: fix applied to `stories/07-agent.md`
  - round 2: raised again (note)
  - round 2: gate note: the critic marked it a note, not a blocker

### C8 (gate_note)
- file: stories/03-agent.md
- section: ## File Plan
- quote: Imports:**
```python
from __future__ import annotations
import argparse
import asyncio
import logging
import os
import sys
from pathlib import Path
- why gate note: raised new in a re-check, about text no fix changed since it was read
- history:
  - round 2: raised (blocking)
  - round 2: gate note: raised new in a re-check, about text no fix changed since it was read

### C9 (open)
- file: stories/04-agent.md
- section: ## File Plan
- quote: async def resume_service_order(ctx: dict, job: dict) -> dict[str, Any]:
- problem: Story 4's LLD adds a withdrawn status check to `resume_service_order` (lines 93-98 in the existing code become a much larger block with checkpointer reads and status checking). Story 7's breakdown says it "modifies `resume_service_order`" but Story 7's LLD says "**No changes to `resume_service_order`:** Story 4 already added the withdrawn check" in the "Modified Files" section. The division is unclear: if Story 4 owns the resume withdrawn check, Story 7 should not list `resume_service_order` in its breakdown; if Story 7 owns preventing all operations, Story 4 should only handle the queue status write when already withdrawn, not the operation prevention.
- change: Clarify in Story 4's LLD whether it adds the full withdrawn check to `resume_service_order` (as the code shows) or only the queue status write (deferring the check to Story 7). Update Story 7's breakdown and LLD to match: either remove `resume_service_order` from Story 7's file list (if Story 4 owns it) or move the check from Story 4 to Story 7 (if Story 7 owns operation prevention).
---
- blocks because: Dev would add withdrawal checking code to `resume_service_order` when Story 4's purpose is to add that check, but the breakdown says Story 7 handles "prevent operations on withdrawn cases" and its LLD modifies `resume_service_order` again. Two stories would modify the same function for overlapping purposes, so Dev would either apply both changes (duplicating the check) or skip one, leaving gaps.
- why open: raised in the last read; no fix round was left
- history:
  - round 1: raised (blocking)
  - round 1: stale: the quoted text is in no spec file (not in `stories/04-agent.md` now)
  - round 3: raised again (blocking)
  - round 3: open: raised in the last read; no fix round was left

### C10 (gate_note)
- file: breakdown.md
- section: ## Summary
- quote: Total test count:** 31 tests (10 + 7 + 5 + 2 + 2 + 1 + 4 + 5 = 36 tests across all stories)
- match: breakdown.md › ## Summary
- problem: The summary line says "31 tests" but the addition "(10 + 7 + 5 + 2 + 2 + 1 + 4 + 5 = 36 tests)" gives 36. One is wrong.
- change: Recalculate the test count from the stories' test sections and update the summary to match. If the stories deliver 36 tests, say "36 tests (10 + 7 + 5 + 2 + 2 + 1 + 4 + 5)". If they deliver 31, correct the addition.
---
- blocks because: The breakdown says 31 tests in the summary sentence but the addition gives 36, so Dev would write 31 tests (trusting the summary) and miss 5, or write 36 and wonder which 5 to drop. Either way the test count would not match what the stories deliver.
- why gate note: raised new in a re-check, about text no fix changed since it was read
- history:
  - round 3: raised (blocking)
  - round 3: gate note: raised new in a re-check, about text no fix changed since it was read

### C11 (gate_note)
- file: stories/06-agent.md
- section: ## Tests Section
- quote: async def test_withdrawn_case_persists_across_restart() -> None:
- match: stories/06-agent.md › ## File Plan
- problem: The LLD repeats the full test implementation twice: once in the "File Plan" section (under `tests/test_withdrawal_persistence.py`) and again in the "Tests Section". The text is identical. A developer following the LLD would see the test body in the File Plan, write it, then reach the Tests Section and see the same test quoted in full with `# (full implementation above in File Plan)` in the docstring but the body still present below it.
- change: In the "Tests Section", replace the full test implementation with a reference: `(See File Plan for full implementation)` and remove the test body. The File Plan already shows the complete test; the Tests Section should only describe coverage and assertions, not repeat the code.
---
- blocks because: The story's purpose is "Persistence across restarts" and it creates a new test file `tests/test_withdrawal_persistence.py`, but the test implementation in the "Tests Section" shows the full test body while the "File Plan" section says the same test file contains the same test. Dev would write the test file once (in the File Plan section) and then see the test body again in the Tests Section and either duplicate it or skip the Tests Section, not knowing which is the source of truth.
- why gate note: raised new in a re-check, about text no fix changed since it was read
- history:
  - round 3: raised (blocking)
  - round 3: gate note: raised new in a re-check, about text no fix changed since it was read

### C12 (gate_note)
- file: stories/08-agent.md
- section: ## File Plan
- quote: async def test_example_1_duplicate_case_withdrawn_flags_not_answered() -> None:
- match: stories/08-agent.md › ## Tests Section
- problem: The File Plan section of Story 8 contains the full implementations of all 5 tests inside the test file `tests/test_withdrawal_e2e.py`. The Tests Section then repeats the same 5 tests in full (with `# (full implementation above in File Plan)` in each docstring but the entire test body still shown). This is the same pattern as C11.
- change: In the "Tests Section", replace each full test implementation with a reference: `(See File Plan for full implementation)` and remove the test bodies. Only the coverage and assertions descriptions belong in the Tests Section; the code is already in the File Plan.
---
- blocks because: Same issue as C11: the LLD shows each test's full implementation in the File Plan section, then repeats the full implementation in the Tests Section with a comment `# (full implementation above in File Plan)`. Dev would write the tests once in the File Plan, then see the same code in the Tests Section and either duplicate them or skip the section, not knowing the Tests Section is only documentation.
- why gate note: raised new in a re-check, about text no fix changed since it was read
- history:
  - round 3: raised (blocking)
  - round 3: gate note: raised new in a re-check, about text no fix changed since it was read
