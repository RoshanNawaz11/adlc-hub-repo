# LLD — Story 06-agent: Aggregate usage logging and API endpoint

**Story:** Aggregate usage logging and API endpoint  
**Epic:** PER-128 (Skip orchestrator model calls on deterministic hops and track token usage)  
**Repo:** `agent`  
**Version:** 1.0  
**Status:** Draft

---

## Overview

Modify `src/agents/runtime/runner.py` to sum `State.usage` across all nodes when a run ends (`"__end__"`) or pauses (detected by `next` being empty and `status != "complete"`, or by `interrupted` flag), and log the totals. Modify `src/agents/runtime/backend.py` (or `server.py`) to include `State.usage` in the `GET /runs/{thread_id}` response under `values`.

This story completes the **usage telemetry feature** (Stories 1, 4, 5, 6 together) by making the collected data visible to operators and clients. Stories 1, 4, and 5 already populate `State.usage` with per-node counts; this story aggregates and reports them.

**Key deliverables:**

1. **Logging on run end/pause:** `run_query` sums `State.usage` across all nodes and logs totals (calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens).
2. **API response:** `GET /runs/{thread_id}` includes `State.usage` in the returned `values` dict, keyed by node name.

---

## Acceptance Criteria

### AC1: `run_query` logs summed usage totals on run end

**Given** a run reaches the `"__end__"` state  
**When** `run_query` completes  
**Then** it logs the total token counts summed across all nodes (`calls`, `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`).

---

### AC2: `run_query` logs summed usage totals on run pause

**Given** a run pauses (detected by `next` being empty and `status != "complete"`, or by `interrupted` flag being true)  
**When** `run_query` returns control  
**Then** it logs the total token counts summed across all nodes.

---

### AC3: `GET /runs/{thread_id}` returns `usage` in `values`

**Given** a client requests run state via `GET /runs/{thread_id}`  
**When** the endpoint responds  
**Then** the response includes `State.usage` under `values.usage`, with per-node counts.

---

### AC4: Summed totals match individual node counts

**Given** `State.usage` contains counts for multiple nodes  
**When** `run_query` sums the totals  
**Then** the logged totals equal the sum of each node's counts (verified in a unit test).

---

## File Plan

### Modified Files

#### `src/agents/runtime/runner.py`

**Current state (inferred from repo tree):**

- Contains `run_query` function (or similar) that executes the graph and returns results.
- Likely handles run completion (`"__end__"` state) and pauses.
- May already log some run metadata (e.g., final state, elapsed time).

**Changes:**

1. **Import logging:**

   ```python
   import logging
   logger = logging.getLogger(__name__)
   ```

2. **Add usage aggregation logic:**

   - After the run ends (`next == ["__end__"]`) or pauses (detected by `next` being empty and `state.get("status") != "complete"`, or by `result.get("interrupted") == True`), extract `State.usage` from the final state.
   - Sum counts across all nodes: for each token field, sum the values from all nodes.
   - Log the totals.

3. **Implementation sketch:**

   ```python
   def run_query(thread_id, input_data, **kwargs):
       """
       Executes the graph for the given thread_id.
       Logs aggregated usage totals when the run ends or pauses.
       """
       # Existing graph execution logic
       result = graph.invoke(input_data, config={"thread_id": thread_id}, **kwargs)
       
       # Extract final state
       state = result.get("values", {})
       
       # Check if run ended or paused
       next_node = result.get("next", [])
       if next_node == ["__end__"]:
           # Run ended
           _log_usage_totals(state, thread_id, "ended")
       elif not next_node and state.get("status") != "complete":
           # Run paused (no next node, not complete)
           _log_usage_totals(state, thread_id, "paused")
       elif result.get("interrupted"):
           # Run paused (interrupted flag)
           _log_usage_totals(state, thread_id, "paused")
       
       return result
   
   def _log_usage_totals(state, thread_id, status):
       """
       Sums State.usage across all nodes and logs totals.
       
       Args:
           state: Final state dict (containing 'usage' field)
           thread_id: Run identifier (for logging context)
           status: "ended" or "paused"
       """
       usage = state.get("usage", {})
       
       if not usage:
           logger.info(f"Run {thread_id} {status} with no usage data")
           return
       
       # Sum counts across all nodes
       totals = {
           "calls": 0,
           "input_tokens": 0,
           "output_tokens": 0,
           "cache_creation_input_tokens": 0,
           "cache_read_input_tokens": 0
       }
       
       for node_name, node_usage in usage.items():
           totals["calls"] += node_usage.get("calls", 0)
           totals["input_tokens"] += node_usage.get("input_tokens", 0)
           totals["output_tokens"] += node_usage.get("output_tokens", 0)
           totals["cache_creation_input_tokens"] += node_usage.get("cache_creation_input_tokens", 0)
           totals["cache_read_input_tokens"] += node_usage.get("cache_read_input_tokens", 0)
       
       logger.info(
           f"Run {thread_id} {status}. Total usage: "
           f"calls={totals['calls']}, "
           f"input_tokens={totals['input_tokens']}, "
           f"output_tokens={totals['output_tokens']}, "
           f"cache_creation_input_tokens={totals['cache_creation_input_tokens']}, "
           f"cache_read_input_tokens={totals['cache_read_input_tokens']}"
       )
   ```

**Notes:**

- Pause detection: `next` is empty AND `status != "complete"`, OR `interrupted` flag is true (as described in HLD AD-4).
- If `run_query` is async, make `_log_usage_totals` async or keep it sync (logging is typically sync).

---

#### `src/agents/runtime/backend.py` (or `server.py`)

**Current state (inferred from repo tree):**

- Contains the `GET /runs/{thread_id}` endpoint (likely using FastAPI or similar).
- Returns run state, including `values` (the current graph state dict).
- May already return some metadata (e.g., `next`, `status`, `steps`).

**Changes:**

1. **No structural changes needed** — `State.usage` is already part of the state dict returned by the graph (from Story 1).
2. **Verify that `values` includes `usage`:**
   - If `values` is serialized from `State`, and `State.usage` exists (from Story 1), then `usage` should already be included.
   - If `values` is filtered or transformed before returning, ensure `usage` is not excluded.

3. **Implementation sketch (FastAPI example):**

   ```python
   from fastapi import APIRouter, HTTPException
   from src.agents.runtime.runner import get_run_state  # Hypothetical function to fetch state
   
   router = APIRouter()
   
   @router.get("/runs/{thread_id}")
   async def get_run(thread_id: str):
       """
       Returns the current state of the run, including usage data.
       """
       state = get_run_state(thread_id)
       
       if not state:
           raise HTTPException(status_code=404, detail="Run not found")
       
       # Return state as-is (usage is already in state if it exists)
       return {
           "thread_id": thread_id,
           "values": state,  # Includes 'usage' if present
           "next": state.get("next", []),
           "status": state.get("status", "unknown")
       }
   ```

**Notes:**

- If `get_run_state` or the equivalent function already returns `State` as a dict, no changes are needed beyond verifying `usage` is included.
- If the endpoint currently filters fields (e.g., only returns specific keys from `values`), add `usage` to the allowlist.
- The exact endpoint implementation depends on whether `backend.py` or `server.py` is used. Inspect both files and modify the one that handles `GET /runs/{thread_id}`.

---

### New Files

None. All changes are modifications to existing files.

---

## Interfaces

### `_log_usage_totals(state, thread_id, status)`

**Module:** `src/agents/runtime/runner.py` (new helper function)

**Signature:**

```python
def _log_usage_totals(state: dict, thread_id: str, status: str) -> None:
    """
    Sums State.usage across all nodes and logs totals.
    
    Args:
        state: Final state dict (containing 'usage' field)
        thread_id: Run identifier (for logging context)
        status: "ended" or "paused"
    
    Returns:
        None
    """
```

**Behavior:**

1. Extract `usage` from `state` (defaults to empty dict if missing).
2. Initialize totals: `{calls: 0, input_tokens: 0, output_tokens: 0, cache_creation_input_tokens: 0, cache_read_input_tokens: 0}`.
3. For each node in `usage`, sum its counts into the totals.
4. Log the totals using `logger.info`, including `thread_id` and `status` for context.

**Contract:**

- If `state["usage"]` is missing or empty, log a message indicating no usage data (no error).
- Totals are always non-negative integers (sum of non-negative node counts).
- Log format is structured (key-value pairs) for easy parsing by log aggregation tools.

---

### `GET /runs/{thread_id}` Response Schema

**Endpoint:** `GET /runs/{thread_id}` in `src/agents/runtime/backend.py` or `server.py`

**Response structure (before changes):**

```json
{
  "thread_id": "run_123",
  "values": {
    "service_order": {...},
    "status": "in_progress",
    "steps": [...],
    ...
  },
  "next": ["planner"],
  "status": "in_progress"
}
```

**Response structure (after changes):**

```json
{
  "thread_id": "run_123",
  "values": {
    "service_order": {...},
    "status": "in_progress",
    "steps": [...],
    "usage": {
      "orchestrator": {
        "calls": 5,
        "input_tokens": 12500,
        "output_tokens": 850,
        "cache_creation_input_tokens": 3200,
        "cache_read_input_tokens": 9300
      },
      "planner": {
        "calls": 1,
        "input_tokens": 2000,
        "output_tokens": 300,
        "cache_creation_input_tokens": 0,
        "cache_read_input_tokens": 2000
      }
    }
  },
  "next": ["planner"],
  "status": "in_progress"
}
```

**Contract:**

- `values.usage` is a dict with node names as keys and per-node counts as values (same schema as `State.usage` from Story 1).
- If a run has no usage data (e.g., a run saved before Story 1), `values.usage` is an empty dict `{}` (not missing).
- The endpoint's existing fields (`thread_id`, `next`, `status`) are unchanged.

---

### Aggregated Usage Totals (Logged)

**Log format:**

```
INFO: Run run_123 ended. Total usage: calls=6, input_tokens=14500, output_tokens=1150, cache_creation_input_tokens=3200, cache_read_input_tokens=11300
```

**Fields:**

- `calls`: Total number of model calls across all nodes
- `input_tokens`: Total input tokens (including cached reads)
- `output_tokens`: Total output tokens
- `cache_creation_input_tokens`: Total tokens used to create prompt cache
- `cache_read_input_tokens`: Total tokens read from prompt cache

**Contract:**

- Each field is summed across all nodes in `State.usage`.
- Missing nodes (no usage data) contribute `0` to each total.
- Log includes `thread_id` and `status` ("ended" or "paused") for context.

---

## Algorithm Notes

### Usage Aggregation Logic

**Goal:** Sum per-node token counts to produce run-level totals.

**Inputs:**

- `State.usage`: `{node_name: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`

**Outputs:**

- `totals`: `{calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}` (summed across all nodes)

**Algorithm:**

1. Initialize `totals` with all fields set to `0`.
2. For each `(node_name, node_usage)` in `State.usage.items()`:
   - `totals["calls"] += node_usage.get("calls", 0)`
   - `totals["input_tokens"] += node_usage.get("input_tokens", 0)`
   - `totals["output_tokens"] += node_usage.get("output_tokens", 0)`
   - `totals["cache_creation_input_tokens"] += node_usage.get("cache_creation_input_tokens", 0)`
   - `totals["cache_read_input_tokens"] += node_usage.get("cache_read_input_tokens", 0)`
3. Return `totals`.

**Invariants:**

- All totals are non-negative integers (node counts are non-negative by construction in Stories 4, 5).
- Missing keys in `node_usage` default to `0` (via `.get(key, 0)`).
- Empty `State.usage` produces `totals` with all fields set to `0`.

**Performance:**

- O(N) where N is the number of nodes with usage data (typically ~5-10 nodes per run).
- No I/O, pure arithmetic — negligible overhead.

---

### Run End/Pause Detection

**Goal:** Identify when to log usage totals.

**Run end detection:**

- `result["next"] == ["__end__"]` (graph reached terminal node)

**Run pause detection (per HLD AD-4):**

- `result["next"]` is empty AND `state.get("status") != "complete"` (no more nodes to execute, not yet complete)
- OR `result.get("interrupted") == True` (graph paused due to `interrupt_before`)

**Implementation:**

- Check `next == ["__end__"]` for run end.
- Check `not next and state.get("status") != "complete"` OR `result.get("interrupted")` for pause.
- Log usage totals in both cases.

---

## Edge Cases

### Edge Case 1: Run with no usage data (old checkpoint)

**Scenario:** A run saved before Story 1 (no `usage` field) completes.

**Handling:**

- `state.get("usage", {})` returns an empty dict.
- `_log_usage_totals` logs: `"Run {thread_id} ended with no usage data"`.
- API returns `values.usage = {}`.

**Test:** Load and complete a checkpoint without `usage`, verify log message and API response.

---

### Edge Case 2: Run with partial usage (some nodes have data, some don't)

**Scenario:** A run has usage data for `orchestrator` and `planner`, but not for `compliance` (e.g., compliance node didn't make any model calls).

**Handling:**

- Only nodes with usage data are in `State.usage`.
- Summing over existing nodes works as expected (missing nodes implicitly contribute `0`).

**Test:** Verify totals match the sum of existing nodes only.

---

### Edge Case 3: Run pauses mid-execution

**Scenario:** A run pauses after the `flag-review` node (before `__end__`).

**Handling:**

- `_log_usage_totals` is called with `status="paused"`.
- Usage totals reflect all calls up to the pause point.
- When the run resumes and completes, totals are logged again (may include additional calls from resumed nodes).

**Test:** Verify usage is logged at pause and again at completion, with increasing totals.

---

### Edge Case 4: Multiple nodes with high token counts

**Scenario:** A run has 10 nodes, each with 100k input tokens.

**Handling:**

- Totals sum correctly: `totals["input_tokens"] = 1,000,000`.
- No overflow (Python integers are arbitrary precision).

**Test:** Verify summing large counts works (no overflow, correct total).

---

### Edge Case 5: API request before any model calls

**Scenario:** A client requests `GET /runs/{thread_id}` immediately after starting a run, before any nodes execute.

**Handling:**

- `State.usage` is empty (no calls yet).
- API returns `values.usage = {}`.

**Test:** Verify API returns empty usage dict for a fresh run.

---

### Edge Case 6: Run fails or errors before completion

**Scenario:** A run crashes or raises an exception before reaching `__end__`.

**Handling:**

- `run_query` may not complete normally, so `_log_usage_totals` might not be called.
- Usage data up to the failure point is still in `State.usage` (if checkpointed).
- The API can still return partial usage if the run's state is saved.

**Mitigation:**

- Consider wrapping `run_query` in a try/finally to ensure usage is logged even on failure.
- Example:
  ```python
  try:
      result = graph.invoke(...)
  except Exception as e:
      logger.error(f"Run {thread_id} failed: {e}")
      state = result.get("values", {}) if result else {}
      _log_usage_totals(state, thread_id, "failed")
      raise
  ```

**Test:** Verify usage is logged if a run fails mid-execution (requires mocking a failure).

---

## Tests

### Test File: `tests/test_usage_logging.py`

**Setup:**

- Mock the graph execution in `run_query` to return canned `result` dicts with `State.usage`.
- Mock `logger.info` to capture log messages.
- Use real `_log_usage_totals` function (unit test the aggregation logic).

---

#### Test 1: Run ends → `run_query` logs summed usage totals

```python
from unittest.mock import patch, MagicMock
from src.agents.runtime.runner import run_query, _log_usage_totals

def test_run_query_logs_usage_on_end():
    # Mock graph result (run ended)
    mock_result = {
        "next": ["__end__"],
        "values": {
            "usage": {
                "orchestrator": {
                    "calls": 3,
                    "input_tokens": 1500,
                    "output_tokens": 300,
                    "cache_creation_input_tokens": 0,
                    "cache_read_input_tokens": 1500
                },
                "planner": {
                    "calls": 1,
                    "input_tokens": 500,
                    "output_tokens": 100,
                    "cache_creation_input_tokens": 0,
                    "cache_read_input_tokens": 500
                }
            }
        }
    }
    
    with patch('src.agents.runtime.runner.graph.invoke', return_value=mock_result):
        with patch('src.agents.runtime.runner.logger') as mock_logger:
            run_query("test_thread", input_data={})
    
    # Verify usage totals were logged
    mock_logger.info.assert_called_once()
    log_message = mock_logger.info.call_args[0][0]
    
    assert "Run test_thread ended" in log_message
    assert "calls=4" in log_message  # 3 + 1
    assert "input_tokens=2000" in log_message  # 1500 + 500
    assert "output_tokens=400" in log_message  # 300 + 100
    assert "cache_creation_input_tokens=0" in log_message
    assert "cache_read_input_tokens=2000" in log_message  # 1500 + 500
```

---

#### Test 2: Run pauses → `run_query` logs summed usage totals

```python
def test_run_query_logs_usage_on_pause():
    # Mock graph result (run paused via interrupted flag)
    mock_result = {
        "next": ["flag-review"],
        "interrupted": True,
        "values": {
            "usage": {
                "orchestrator": {
                    "calls": 2,
                    "input_tokens": 1000,
                    "output_tokens": 200,
                    "cache_creation_input_tokens": 0,
                    "cache_read_input_tokens": 1000
                }
            }
        }
    }
    
    with patch('src.agents.runtime.runner.graph.invoke', return_value=mock_result):
        with patch('src.agents.runtime.runner.logger') as mock_logger:
            run_query("test_thread", input_data={})
    
    # Verify usage totals were logged with "paused" status
    mock_logger.info.assert_called_once()
    log_message = mock_logger.info.call_args[0][0]
    
    assert "Run test_thread paused" in log_message
    assert "calls=2" in log_message
    assert "input_tokens=1000" in log_message
    assert "output_tokens=200" in log_message
```

---

#### Test 3: `GET /runs/{thread_id}` returns `usage` in `values`

```python
from fastapi.testclient import TestClient
from src.agents.runtime.backend import app  # Adjust import to match actual app

client = TestClient(app)

def test_get_run_returns_usage():
    # Mock get_run_state to return state with usage
    mock_state = {
        "service_order": {"id": "SO-1003"},
        "status": "in_progress",
        "usage": {
            "orchestrator": {
                "calls": 5,
                "input_tokens": 2500,
                "output_tokens": 500,
                "cache_creation_input_tokens": 0,
                "cache_read_input_tokens": 2500
            }
        }
    }
    
    with patch('src.agents.runtime.backend.get_run_state', return_value=mock_state):
        response = client.get("/runs/test_thread")
    
    assert response.status_code == 200
    data = response.json()
    
    assert "values" in data
    assert "usage" in data["values"]
    assert data["values"]["usage"]["orchestrator"]["calls"] == 5
    assert data["values"]["usage"]["orchestrator"]["input_tokens"] == 2500
```

---

#### Test 4: Verify summed totals match individual node counts

```python
def test_usage_totals_sum_correctly():
    state = {
        "usage": {
            "orchestrator": {
                "calls": 3,
                "input_tokens": 1500,
                "output_tokens": 300,
                "cache_creation_input_tokens": 100,
                "cache_read_input_tokens": 1400
            },
            "planner": {
                "calls": 1,
                "input_tokens": 500,
                "output_tokens": 100,
                "cache_creation_input_tokens": 0,
                "cache_read_input_tokens": 500
            },
            "compliance": {
                "calls": 2,
                "input_tokens": 800,
                "output_tokens": 150,
                "cache_creation_input_tokens": 50,
                "cache_read_input_tokens": 750
            }
        }
    }
    
    # Call _log_usage_totals and capture log output
    with patch('src.agents.runtime.runner.logger') as mock_logger:
        _log_usage_totals(state, "test_thread", "ended")
    
    log_message = mock_logger.info.call_args[0][0]
    
    # Verify totals
    expected_calls = 3 + 1 + 2  # 6
    expected_input = 1500 + 500 + 800  # 2800
    expected_output = 300 + 100 + 150  # 550
    expected_cache_creation = 100 + 0 + 50  # 150
    expected_cache_read = 1400 + 500 + 750  # 2650
    
    assert f"calls={expected_calls}" in log_message
    assert f"input_tokens={expected_input}" in log_message
    assert f"output_tokens={expected_output}" in log_message
    assert f"cache_creation_input_tokens={expected_cache_creation}" in log_message
    assert f"cache_read_input_tokens={expected_cache_read}" in log_message
```

---

## Implementation Checklist

- [ ] Modify `src/agents/runtime/runner.py`:
  - [ ] Import `logging` and create `logger`
  - [ ] Implement `_log_usage_totals(state, thread_id, status)` helper function
  - [ ] In `run_query`, detect run end (`next == ["__end__"]`)
  - [ ] In `run_query`, detect run pause (`not next and status != "complete"` OR `result.get("interrupted")`)
  - [ ] Call `_log_usage_totals` on end and pause
- [ ] Verify `src/agents/runtime/backend.py` or `server.py`:
  - [ ] Inspect `GET /runs/{thread_id}` endpoint
  - [ ] Verify `values` includes `State.usage` (no filtering)
  - [ ] If `values` is filtered, add `usage` to allowlist
  - [ ] Test endpoint returns `usage` in response
- [ ] Write unit tests in `tests/test_usage_logging.py` (4 tests):
  - [ ] Test 1: run ends → usage logged
  - [ ] Test 2: run pauses → usage logged
  - [ ] Test 3: API returns usage in response
  - [ ] Test 4: summed totals match node counts
- [ ] Run tests: `uv run pytest tests/test_usage_logging.py -v`
- [ ] Verify all 4 tests pass
- [ ] Verify existing tests pass: `uv run pytest -q`

---

## Dependencies

**Depends on:**

- Story 1 (State schema) — `State.usage` must exist with additive reducer
- Story 4 (Usage in run_skill/MCP) — usage data must be populated by call sites
- Story 5 (Usage in gateway) — direct calls also populate usage

**Blocks:**

None. This is the final story in the usage telemetry track.

**Integration:**

- This story completes the usage telemetry feature (Stories 1, 4, 5, 6 together).
- Combined with forced routing (Stories 2, 3), the entire PER-128 feature is complete.

---

## Notes

- **Log format:** Structured key-value pairs for easy parsing by log aggregation tools (e.g., Datadog, Splunk).
- **API response:** `values.usage` is included automatically if `State.usage` exists and isn't filtered out. Verify by inspecting the endpoint's serialization logic.
- **Pause detection (per HLD AD-4):** Check if `next` is empty AND `state.get("status") != "complete"`, OR if `result.get("interrupted") == True`.
- **Performance:** Aggregation is O(N) where N is the number of nodes (~5-10), negligible overhead.
- **Error handling:** Consider logging usage even if a run fails (wrap in try/finally). This is optional for the MVP but valuable for debugging.
- **PR evidence (AC6 from requirements):** After this story is complete, run a full service order (e.g., `run_so.py` with SO-1003) and measure before/after token usage from the logs. Include in PR description:
  ```
  Before: 25 calls, 125k input tokens, 15k output tokens
  After: 18 calls, 90k input tokens, 12k output tokens
  Savings: 7 calls (28%), 35k input tokens (28%), 3k output tokens (20%)
  ```

---

**End of LLD — Story 06-agent**
