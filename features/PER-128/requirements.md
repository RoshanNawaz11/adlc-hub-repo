# requirements.md

## Overview

This document formalizes the acceptance criteria for PER-128, which adds forced routing and per-run token usage tracking to reduce orchestrator model calls.

---

## Given/When/Then Acceptance Criteria

### AC1: Token usage tracking per run

**AC1.1: Record usage per call**

**Given** a run is executing  
**When** any model call completes (including `run_skill`, MCP tool rounds in `bindings.py`, JSON retries, and `ModelGateway.complete`)  
**Then** the call's `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens` are recorded in `State.usage` under the current node's key

**AC1.2: Accumulate usage per node**

**Given** multiple calls occur for the same node  
**When** usage is updated  
**Then** the new counts are added to the node's existing counts (not replaced)

**AC1.3: State migration for old runs**

**Given** a run was saved before this change (no `usage` field)  
**When** the run is loaded and resumed  
**Then** it loads successfully and continues execution without error

**AC1.4: Log usage at run end**

**Given** a run completes or pauses  
**When** `run_query` in `runner.py` finishes  
**Then** the total usage across all nodes is logged

**AC1.5: API returns usage totals**

**Given** a client queries `GET /runs/{thread_id}`  
**When** the response is returned  
**Then** `values` includes the run's total token usage

---

### AC2: Forced routing logic

**AC2.1: Return forced route when exactly one candidate passes**

**Given** the orchestrator is at a decision point  
**When** `forced_route(state)` is called  
**Then** it returns a single-element list `[route]` if exactly one candidate route passes `broken_rules`, or `None` if zero or multiple candidates pass

**AC2.2: Pre-planner forced route**

**Given** the system state before the planner has run  
**When** `forced_route(state)` is called  
**Then** it returns `["planner"]`

**AC2.3: Post-review forced route**

**Given** the system state after a review hop  
**When** `forced_route(state)` is called  
**Then** it returns `["__end__"]`

**AC2.4: Post-execution forced route**

**Given** execution has finished  
**When** `forced_route(state)` is called  
**Then** it returns `["output-validator"]`

**AC2.5: Multiple valid routes**

**Given** two stage agents are both unfinished and nothing is waiting  
**When** `forced_route(state)` is called  
**Then** it returns `None`

**AC2.6: Delegate to broken_rules**

**Given** candidate routes are generated  
**When** `forced_route` checks them  
**Then** it uses `broken_rules` as the sole source of routing rules (no duplication)

---

### AC3: Orchestrator integration

**AC3.1: Skip model call when forced route and no model work needed**

**Given** the orchestrator is at a hop and `forced_route` returns a route  
**When** the route does NOT pause for a person (`flag-review` or `review`) AND no unresolved flag exists that the orchestrator's model has not seen  
**Then** the orchestrator skips `_decide` and uses the forced route without calling the model

**AC3.2: Forced hop step structure**

**Given** a forced hop is taken  
**When** the step is recorded  
**Then** it has title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`, and updates `status`, `flagged`, `priority`, and `confidence` as a model hop would, leaving `summary` unchanged

**AC3.3: Model called for reviewer summary**

**Given** the orchestrator is at a hop and `forced_route` returns `["flag-review"]` or `["review"]`  
**When** the model must write the reviewer's `summary`  
**Then** the orchestrator calls the model even though the route is forced

**AC3.4: Model called for unseen flags**

**Given** an unresolved flag exists  
**When** the flag's id is NOT in the set of flags the orchestrator's model has seen  
**Then** the orchestrator calls the model to merge or address the flag, even if the route is forced

**AC3.5: Track seen flags**

**Given** the orchestrator's model sees a brief containing flags  
**When** the model call completes  
**Then** the flag ids from that brief are added to the state's set of seen flags

---

### AC4: Automated test coverage

**AC4.1: All new tests pass**

**Given** the test suite includes new tests with a fake gateway and no live services  
**When** `uv run pytest -q` is executed  
**Then** all tests pass

**AC4.2: Tests verify forced_route deterministic routes**

**Given** a test scenario where `forced_route` should return a deterministic route  
**When** the state matches pre-planner, post-review, or post-execution conditions  
**Then** the test asserts the expected forced route (`["planner"]`, `["__end__"]`, or `["output-validator"]`)

**AC4.3: Tests verify forced_route with multiple valid routes**

**Given** a test scenario where multiple routes are valid  
**When** `forced_route` is called  
**Then** the test asserts it returns `None`

**AC4.4: Tests verify no model call on forced hops**

**Given** a test scenario where a forced hop occurs  
**When** the orchestrator processes the hop  
**Then** the test asserts no model call was made (via gateway spy/mock)

**AC4.5: Tests verify model called when needed**

**Given** a test scenario where the forced route requires model work (pause or unseen flag)  
**When** the orchestrator processes the hop  
**Then** the test asserts the model was called

**AC4.6: Tests verify forced hop step structure**

**Given** a test scenario where a forced hop is recorded  
**When** the step is written  
**Then** the test asserts `status`, `priority`, `confidence`, and step structure match expectations

**AC4.7: Tests verify usage accumulation**

**Given** a test scenario where multiple model calls occur for a node  
**When** usage is recorded  
**Then** the test asserts counts accumulate correctly across calls

**AC4.8: Tests verify state migration**

**Given** a test scenario where a run is loaded without a `usage` field  
**When** the run resumes  
**Then** the test asserts it continues without error

**AC4.9: Existing tests pass unchanged**

**Given** existing tests that cover current orchestrator and state behavior  
**When** `uv run pytest -q` is executed after the changes  
**Then** all existing tests pass unchanged

---

### AC5: Demonstrable savings

**AC5.1: PR description includes before/after metrics**

**Given** the PR description includes before/after measurements  
**When** one service order is run (e.g. SO-1003 via `run_so.py`)  
**Then** the description reports model call count and total token usage before the change, and the same metrics after the change

---

## Test Count by Story

- **Story 1 (PER-128-1):** 5 tests covering AC1.1, AC1.2, AC1.3, AC4.7, AC4.8
- **Story 2 (PER-128-2):** 7 tests covering AC2.1, AC2.2, AC2.3, AC2.4, AC2.5, AC2.6, AC4.2, AC4.3
- **Story 3 (PER-128-3):** 10 tests covering AC3.1, AC3.2, AC3.3, AC3.4, AC3.5, AC1.4, AC1.5, AC4.1, AC4.4, AC4.5, AC4.6, AC4.9, AC5.1

**Total:** 22 new tests (matches breakdown.md Verification Summary)

---

## Out of Scope

- Changing models, `policy.yaml` limits, skill or prompt content
- Shrinking `_brief`, or removing the skill container and code-execution tool from orchestrator calls
- The MCP server, the GraphQL service, and fastapi-backend
- Graph routing rules themselves: `broken_rules` behaves as today
