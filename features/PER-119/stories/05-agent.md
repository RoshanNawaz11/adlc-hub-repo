# Story 05-agent LLD

**Story:** Usage logging and API response in runner.py

**Repo:** `app`

**Epic:** PER-119

---

## Acceptance Criteria

From requirements and breakdown only:

1. **AC-1 (REQ-2 AC-2.5):** `run_query` in `src/agents/runtime/runner.py` logs the run's totals when the run ends or pauses
2. **AC-2 (REQ-2 AC-2.6):** `GET /runs/{thread_id}` returns usage totals in `values`

---

## HLD Binding Outcomes

From `hld-per-119.md`:

- **AD-7:** `run_query` in `src/agents/runtime/runner.py` sums `state.usage` across all nodes when the run ends or pauses, and logs the totals (total calls, total input tokens, total output tokens, total cache creation tokens, total cache read tokens). `GET /runs/{thread_id}` includes the same totals in the `values` field of the response. The format is:
  ```python
  {
    "usage": {
      "calls": <int>,
      "input_tokens": <int>,
      "output_tokens": <int>,
      "cache_creation_input_tokens": <int>,
      "cache_read_input_tokens": <int>
    }
  }
  ```

---

## File Plan

**Note:** Repo tree for `app` is not provided. The story specifies `src/agents/runtime/runner.py` as the file location; implementation must verify the actual path in the repo and use it.

### Modified Files

#### `src/agents/runtime/runner.py`

**Changes:**

1. **Add shared helper function `sum_usage`:**
   - Extract usage summing logic into a shared helper to avoid duplication between `run_query` (for logging) and API handler (for response)
   - Function signature: `def sum_usage(state_usage: dict[str, dict[str, int]]) -> dict[str, int]`
   - Iterates over `state_usage`, sums all five count fields across all nodes
   - Returns dict with total counts

2. **Add usage totaling logic to `run_query`:**
   - After the run ends or pauses, call `sum_usage(state.usage)` to get totals
   - Log the totals (total calls, input tokens, output tokens, cache creation tokens, cache read tokens)

3. **Modify API handler for `GET /runs/{thread_id}`:**
   - Load run state
   - Call `sum_usage(state.usage)` to get totals
   - Include totals in the `values` field of the response under key `usage`
   - Format as specified in AD-7 (see Interfaces section below)

**Algorithm notes:**

- **Shared helper `sum_usage`:**
  ```python
  def sum_usage(state_usage: dict[str, dict[str, int]]) -> dict[str, int]:
      """
      Sums usage counts across all nodes.
      
      Args:
          state_usage: dict mapping node names to usage dicts
                       (format: {node: {calls, input_tokens, ...}})
      
      Returns:
          dict with total usage across all nodes:
          {
              "calls": int,
              "input_tokens": int,
              "output_tokens": int,
              "cache_creation_input_tokens": int,
              "cache_read_input_tokens": int
          }
      """
      total_usage = {
          "calls": 0,
          "input_tokens": 0,
          "output_tokens": 0,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 0
      }
      
      for node_name, node_usage in state_usage.items():
          total_usage["calls"] += node_usage.get("calls", 0)
          total_usage["input_tokens"] += node_usage.get("input_tokens", 0)
          total_usage["output_tokens"] += node_usage.get("output_tokens", 0)
          total_usage["cache_creation_input_tokens"] += node_usage.get("cache_creation_input_tokens", 0)
          total_usage["cache_read_input_tokens"] += node_usage.get("cache_read_input_tokens", 0)
      
      return total_usage
  ```

- **Usage totaling in `run_query`:**
  ```python
  def run_query(thread_id, ...):
      # Existing run logic (execute graph, etc.)
      # ...
      
      # After run ends or pauses, sum usage
      if state.status in ["completed", "paused"]:  # (or whatever the end/pause statuses are)
          total_usage = sum_usage(state.usage)
          
          # Log totals
          logger.info(
              f"Run {thread_id} usage: "
              f"calls={total_usage['calls']}, "
              f"input_tokens={total_usage['input_tokens']}, "
              f"output_tokens={total_usage['output_tokens']}, "
              f"cache_creation_input_tokens={total_usage['cache_creation_input_tokens']}, "
              f"cache_read_input_tokens={total_usage['cache_read_input_tokens']}"
          )
          
          # Store totals for API response (if run_query returns state or stores it)
          # (Implementation depends on existing code structure)
  ```

- **API response modification:**
  ```python
  @app.get("/runs/{thread_id}")
  def get_run(thread_id: str):
      # Existing code to load run state
      state = load_run_state(thread_id)
      
      # Sum usage (call shared helper)
      total_usage = sum_usage(state.usage)
      
      # Include in response
      return {
          "values": {
              "usage": total_usage,
              # (other existing values fields)
          },
          # (other response fields)
      }
  ```

- **Note on run status:** The story assumes runs have a status field (e.g., `"completed"`, `"paused"`) that indicates when the run has ended or paused. Implementation must verify the actual status values in the codebase and use them in the condition that triggers usage logging.

- **Note on API framework:** The algorithm above assumes a REST API using a framework like FastAPI or Flask. Implementation must verify the actual API framework and adjust the handler code accordingly (e.g., Flask uses `@app.route`, FastAPI uses `@app.get`).

---

## Interfaces

### Function Signature (new helper)

```python
def sum_usage(state_usage: dict[str, dict[str, int]]) -> dict[str, int]:
    """
    Sums usage counts across all nodes.
    
    Args:
        state_usage: dict mapping node names to usage dicts
                     (format: {node: {calls, input_tokens, ...}})
    
    Returns:
        dict with total usage across all nodes:
        {
            "calls": int,
            "input_tokens": int,
            "output_tokens": int,
            "cache_creation_input_tokens": int,
            "cache_read_input_tokens": int
        }
    """
```

### Inputs

**`state.usage` (from state):**

```python
{
  "orchestrator": {
    "calls": 3,
    "input_tokens": 4500,
    "output_tokens": 150,
    "cache_creation_input_tokens": 1200,
    "cache_read_input_tokens": 800
  },
  "planner": {
    "calls": 1,
    "input_tokens": 2000,
    "output_tokens": 100,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 500
  },
  "agent-analyst": {
    "calls": 2,
    "input_tokens": 3000,
    "output_tokens": 120,
    "cache_creation_input_tokens": 500,
    "cache_read_input_tokens": 600
  }
}
```

### Outputs

**Summed usage (logged and returned in API):**

```python
{
  "calls": 6,  # 3 + 1 + 2
  "input_tokens": 9500,  # 4500 + 2000 + 3000
  "output_tokens": 370,  # 150 + 100 + 120
  "cache_creation_input_tokens": 1700,  # 1200 + 0 + 500
  "cache_read_input_tokens": 1900  # 800 + 500 + 600
}
```

**Log message (example):**

```
INFO: Run thread-123 usage: calls=6, input_tokens=9500, output_tokens=370, cache_creation_input_tokens=1700, cache_read_input_tokens=1900
```

**API response (example):**

```json
{
  "thread_id": "thread-123",
  "status": "completed",
  "values": {
    "usage": {
      "calls": 6,
      "input_tokens": 9500,
      "output_tokens": 370,
      "cache_creation_input_tokens": 1700,
      "cache_read_input_tokens": 1900
    }
  }
}
```

---

## Tests

From breakdown Verification Approach (3 tests, exact count matching breakdown for story 05-agent):

### Test 1: run_query sums usage and logs totals when run ends

**Mapped from:** REQ-2 AC-2.5 (Breakdown verification item 1 for story 05-agent)

**Setup:**
- Create a fake state with pre-populated `usage` data:
  ```python
  state.usage = {
      "orchestrator": {
          "calls": 3,
          "input_tokens": 4500,
          "output_tokens": 150,
          "cache_creation_input_tokens": 1200,
          "cache_read_input_tokens": 800
      },
      "planner": {
          "calls": 1,
          "input_tokens": 2000,
          "output_tokens": 100,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 500
      }
  }
  state.status = "completed"  # (or whatever the end status is)
  ```
- Mock the logger to capture log messages
- Call `run_query` with the fake state (or run a fake execution that ends)

**Expected outcome:**
- Logger receives a message containing:
  - `calls=4` (3 + 1)
  - `input_tokens=6500` (4500 + 2000)
  - `output_tokens=250` (150 + 100)
  - `cache_creation_input_tokens=1200` (1200 + 0)
  - `cache_read_input_tokens=1300` (800 + 500)

**Verification:**
- AC-1 (REQ-2 AC-2.5): `run_query` logs totals when run ends
- All 5 counts are logged correctly (summed across all nodes)

---

### Test 2: run_query sums usage and logs totals when run pauses

**Mapped from:** REQ-2 AC-2.5 (Breakdown verification item 2 for story 05-agent)

**Setup:**
- Create a fake state with pre-populated `usage` data:
  ```python
  state.usage = {
      "agent-analyst": {
          "calls": 2,
          "input_tokens": 3000,
          "output_tokens": 120,
          "cache_creation_input_tokens": 500,
          "cache_read_input_tokens": 600
      }
  }
  state.status = "paused"  # (or whatever the pause status is)
  ```
- Mock the logger to capture log messages
- Call `run_query` with the fake state (or run a fake execution that pauses)

**Expected outcome:**
- Logger receives a message containing:
  - `calls=2`
  - `input_tokens=3000`
  - `output_tokens=120`
  - `cache_creation_input_tokens=500`
  - `cache_read_input_tokens=600`

**Verification:**
- AC-1 (REQ-2 AC-2.5): `run_query` logs totals when run pauses
- All 5 counts are logged correctly

---

### Test 3: GET /runs/{thread_id} returns usage totals in values.usage

**Mapped from:** REQ-2 AC-2.6 (Breakdown verification item 3 for story 05-agent)

**Setup:**
- Create a fake run state with pre-populated `usage` data:
  ```python
  state.usage = {
      "orchestrator": {
          "calls": 3,
          "input_tokens": 4500,
          "output_tokens": 150,
          "cache_creation_input_tokens": 1200,
          "cache_read_input_tokens": 800
      },
      "planner": {
          "calls": 1,
          "input_tokens": 2000,
          "output_tokens": 100,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 500
      }
  }
  ```
- Mock the state loader to return this state for thread ID `"thread-123"`
- Call `GET /runs/thread-123` (via test client or direct function call)

**Expected outcome:**
- API response includes:
  ```json
  {
    "values": {
      "usage": {
        "calls": 4,
        "input_tokens": 6500,
        "output_tokens": 250,
        "cache_creation_input_tokens": 1200,
        "cache_read_input_tokens": 1300
      }
    }
  }
  ```

**Verification:**
- AC-2 (REQ-2 AC-2.6): `GET /runs/{thread_id}` returns usage totals in `values`
- Format is correct (all 5 counts present, correct values)

---

## Notes

- **No repo tree provided:** Implementation must verify the actual path of `runner.py` in the `app` repo. The story specifies `src/agents/runtime/runner.py` as the likely location.
- **Run status values:** The algorithm assumes runs have a status field with values like `"completed"` or `"paused"`. Implementation must verify the actual status values in the codebase and use them in the condition that triggers usage logging.
- **API framework:** The API handler example assumes a REST API framework (e.g., FastAPI, Flask). Implementation must verify the actual framework and adjust the handler code accordingly.
- **Logger location:** The algorithm assumes a logger is available (e.g., `logger = logging.getLogger(__name__)`). Implementation must verify the actual logging setup in `runner.py`.
- **Shared helper function:** The usage summing logic is extracted into `sum_usage` to avoid duplication between `run_query` (for logging) and the API handler (for response). This helper is defined in `runner.py` alongside the functions that use it.
- **Empty usage:** If `state.usage` is empty (e.g., a run that made no model calls, or a run saved before Story 1), the summed totals should be all zeros. The algorithm handles this correctly (iterating over an empty dict produces zero sums).
- **No live runs in tests:** All tests use fake states with pre-populated `usage` data. No live model calls, no live runs.
- **Existing tests unchanged:** After this story, `uv run pytest -q` must pass, and existing tests must pass unchanged.
- **Dependencies clear:** Stories 3 and 4 write usage to `state.usage`. This story reads and sums it. All three stories together complete the usage tracking feature.
- **Test count matches breakdown exactly:** 3 tests as specified in breakdown Verification Approach for story 05-agent. No additional tests beyond those agreed in the breakdown.
- **AC references added:** Each AC now includes the full requirement reference (REQ-2 AC-2.5 for logging, REQ-2 AC-2.6 for API) for precise traceability to requirements.
- **Test mappings updated:** Each test now includes the full requirement reference in its "Mapped from" line for precise traceability.
