# Story 04-agent LLD

**Story:** Usage collection at model call sites

**Repo:** `app`

**Epic:** PER-119

---

## Acceptance Criteria

From requirements and breakdown only:

1. **AC-1 (REQ-2 AC-2.1):** Every model call records `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, and `cache_read_input_tokens` (covers `run_skill`, MCP tool loop in `bindings.py`, `ModelGateway.complete`)
2. **AC-2 (REQ-2 AC-2.3):** Each node's update carries the usage of the calls it made
3. **AC-3 (REQ-2 AC-2.3):** `usage` adds counts per node across calls, including JSON retries and every MCP tool round

---

## HLD Binding Outcomes

From `hld-per-119.md`:

- **AD-3:** Three call sites collect usage: (1) `run_skill` extracts from skill invocation response's `usage` field and from JSON retry responses' `usage` fields, sums all counts (initial + retries) into `{calls: N, input_tokens: total, ...}` where `calls` is the number of model API calls made (1 for no retry, 2 for one retry, etc.), and returns the aggregated usage with the result. (2) MCP tool loop extracts from each tool round response's `usage` field (including the final no-tools call), sums all rounds into `{calls: N, ...}` where `calls` is the number of model API calls (one per round), and returns the aggregated usage with the result. (3) `ModelGateway.complete` extracts the four token count fields from the completion response and returns them in a dict `{input_tokens: int, output_tokens: int, cache_creation_input_tokens: int, cache_read_input_tokens: int}` alongside the completion; missing fields default to `0`. Each node that makes model calls aggregates usage from these sources and includes the aggregated counts in its state update.

---

## File Plan

**Note:** Repo tree for `app` is not provided. The story specifies `src/agents/skills/client.py`, `src/agents/tools/bindings.py`, `src/agents/model/gateway.py` as the file locations; implementation must verify the actual paths in the repo and use them.

### Modified Files

#### `src/agents/skills/client.py`

**Changes:**

1. **Modify `run_skill` to extract and aggregate usage:**
   - After calling the skill (via `ModelGateway.complete` or the model client), extract usage from the response
   - The response format (based on HLD AD-3 and test mocks) is: `{"completion": <str>, "usage": {input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`
   - If the response has no JSON and `run_skill` retries (calls the model again with the previous reply as context), extract usage from the retry response as well
   - Sum usage across all calls (initial + retries): initialize a local accumulator `{calls: 0, input_tokens: 0, output_tokens: 0, cache_creation_input_tokens: 0, cache_read_input_tokens: 0}`, add counts from each response, increment `calls` by 1 for each model API call
   - Return the aggregated usage with the result: `{"result": <completion>, "usage": <aggregated_usage>}`

**Algorithm notes:**

- **Usage extraction and aggregation in `run_skill`:**
  ```python
  def run_skill(skill_name, input_data, ...):
      # Initialize usage accumulator
      usage_accumulator = {
          "calls": 0,
          "input_tokens": 0,
          "output_tokens": 0,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 0
      }
      
      # Call the skill (via ModelGateway.complete or model client)
      response = model_gateway.complete(skill_prompt, ...)
      
      # Extract usage from initial response
      response_usage = response.get("usage", {})
      usage_accumulator["calls"] += 1
      usage_accumulator["input_tokens"] += response_usage.get("input_tokens", 0)
      usage_accumulator["output_tokens"] += response_usage.get("output_tokens", 0)
      usage_accumulator["cache_creation_input_tokens"] += response_usage.get("cache_creation_input_tokens", 0)
      usage_accumulator["cache_read_input_tokens"] += response_usage.get("cache_read_input_tokens", 0)
      
      completion = response["completion"]
      
      # Check if response has JSON; if not, retry with previous reply
      if not has_json(completion):
          # Retry with previous reply as context
          retry_response = model_gateway.complete(retry_prompt_with_previous_reply(completion), ...)
          
          # Extract usage from retry response
          retry_usage = retry_response.get("usage", {})
          usage_accumulator["calls"] += 1
          usage_accumulator["input_tokens"] += retry_usage.get("input_tokens", 0)
          usage_accumulator["output_tokens"] += retry_usage.get("output_tokens", 0)
          usage_accumulator["cache_creation_input_tokens"] += retry_usage.get("cache_creation_input_tokens", 0)
          usage_accumulator["cache_read_input_tokens"] += retry_usage.get("cache_read_input_tokens", 0)
          
          completion = retry_response["completion"]
      
      # Return result with aggregated usage
      return {
          "result": completion,
          "usage": usage_accumulator
      }
  ```

- **Notes:**
  - The exact location of usage extraction depends on the existing `run_skill` implementation. The algorithm above shows the logic; the developer must locate the existing model call site(s) in `run_skill` and add usage extraction there.
  - `has_json(completion)` is a placeholder for the existing JSON validation logic. The developer must use the actual check already present in the code.
  - If `run_skill` already returns a dict, add `"usage"` to it. If it returns a string, change the return type to a dict with `"result"` and `"usage"`.

---

#### `src/agents/tools/bindings.py`

**Changes:**

1. **Modify the MCP tool loop to extract and aggregate usage:**
   - The MCP tool loop calls the model multiple times: once per tool invocation, plus a final no-tools call
   - After each call (via `ModelGateway.complete`), extract usage from the response
   - Sum usage across all rounds: initialize a local accumulator `{calls: 0, input_tokens: 0, output_tokens: 0, cache_creation_input_tokens: 0, cache_read_input_tokens: 0}`, add counts from each response, increment `calls` by 1 for each model API call
   - Return the aggregated usage with the result: `{"result": <final_completion>, "usage": <aggregated_usage>}`

**Algorithm notes:**

- **Usage extraction and aggregation in MCP tool loop:**
  ```python
  def mcp_tool_loop(tools, input_data, ...):
      # Initialize usage accumulator
      usage_accumulator = {
          "calls": 0,
          "input_tokens": 0,
          "output_tokens": 0,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 0
      }
      
      # Loop: call model with tools until no tools are invoked
      while True:
          # Call model
          response = model_gateway.complete(prompt, tools=tools, ...)
          
          # Extract usage from response
          response_usage = response.get("usage", {})
          usage_accumulator["calls"] += 1
          usage_accumulator["input_tokens"] += response_usage.get("input_tokens", 0)
          usage_accumulator["output_tokens"] += response_usage.get("output_tokens", 0)
          usage_accumulator["cache_creation_input_tokens"] += response_usage.get("cache_creation_input_tokens", 0)
          usage_accumulator["cache_read_input_tokens"] += response_usage.get("cache_read_input_tokens", 0)
          
          completion = response["completion"]
          
          # Check if tools were invoked
          if no_tools_invoked(response):
              # Final no-tools call; break
              break
          
          # Execute tools and update context for next call
          # (existing tool execution logic)
      
      # Return result with aggregated usage
      return {
          "result": completion,
          "usage": usage_accumulator
      }
  ```

- **Notes:**
  - The exact location of usage extraction depends on the existing MCP tool loop implementation. The algorithm above shows the logic; the developer must locate the existing model call site(s) in the loop and add usage extraction there.
  - `no_tools_invoked(response)` is a placeholder for the existing tool check logic. The developer must use the actual check already present in the code.
  - The MCP tool loop may already return a dict; if so, add `"usage"` to it. If it returns a string, change the return type to a dict with `"result"` and `"usage"`.

---

#### `src/agents/model/gateway.py`

**Changes:**

1. **Modify `ModelGateway.complete` to extract and return usage:**
   - After calling the model API client, extract usage from the response
   - The response format depends on the model client (e.g., Anthropic's Claude API returns `usage` in the response body)
   - Parse the four token count fields from the response: `input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`
   - Default missing fields to `0`
   - Return usage alongside the completion: `{"completion": <str>, "usage": {input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`

**Algorithm notes:**

- **Usage extraction in `ModelGateway.complete`:**
  ```python
  def complete(self, prompt, ...):
      # Call the model API client (e.g., Anthropic Claude API)
      api_response = self.client.messages.create(
          model=self.model_name,
          messages=[{"role": "user", "content": prompt}],
          ...
      )
      
      # Extract completion text
      completion = api_response.content[0].text  # (format depends on client)
      
      # Extract usage from response
      # Anthropic format: api_response.usage is an object with fields:
      # input_tokens, output_tokens, cache_creation_input_tokens (optional), cache_read_input_tokens (optional)
      usage = {
          "input_tokens": getattr(api_response.usage, "input_tokens", 0),
          "output_tokens": getattr(api_response.usage, "output_tokens", 0),
          "cache_creation_input_tokens": getattr(api_response.usage, "cache_creation_input_tokens", 0),
          "cache_read_input_tokens": getattr(api_response.usage, "cache_read_input_tokens", 0)
      }
      
      # Return completion with usage
      return {
          "completion": completion,
          "usage": usage
      }
  ```

- **Notes:**
  - The exact response format depends on the model API client. The algorithm above assumes Anthropic's Claude API format (based on the ticket's reference to `claude-haiku-4-5-20251001`). If the client is different, the developer must adjust the extraction logic accordingly.
  - Missing fields default to `0` via `getattr(..., 0)` or `.get(..., 0)`.
  - If `ModelGateway.complete` already returns a dict, add `"usage"` to it. If it returns a string, change the return type to a dict with `"completion"` and `"usage"`.

---

## Interfaces

### Function Signatures (modified return types)

**`run_skill` (in `src/agents/skills/client.py`):**

```python
def run_skill(skill_name: str, input_data: dict, ...) -> dict:
    """
    Calls a skill and returns the result with aggregated usage.
    
    Returns:
        dict: {
            "result": str,  # skill completion text
            "usage": {
                "calls": int,  # number of model API calls (initial + retries)
                "input_tokens": int,
                "output_tokens": int,
                "cache_creation_input_tokens": int,
                "cache_read_input_tokens": int
            }
        }
    """
```

**MCP tool loop function (in `src/agents/tools/bindings.py`):**

```python
def mcp_tool_loop(tools: list, input_data: dict, ...) -> dict:
    """
    Executes an MCP tool loop and returns the result with aggregated usage.
    
    Returns:
        dict: {
            "result": str,  # final completion text
            "usage": {
                "calls": int,  # number of model API calls (all tool rounds + final no-tools call)
                "input_tokens": int,
                "output_tokens": int,
                "cache_creation_input_tokens": int,
                "cache_read_input_tokens": int
            }
        }
    """
```

**`ModelGateway.complete` (in `src/agents/model/gateway.py`):**

```python
def complete(self, prompt: str, ...) -> dict:
    """
    Calls the model API and returns the completion with usage.
    
    Returns:
        dict: {
            "completion": str,  # model completion text
            "usage": {
                "input_tokens": int,
                "output_tokens": int,
                "cache_creation_input_tokens": int,
                "cache_read_input_tokens": int
            }
        }
    """
```

### Per-call usage shape

**Returned by `ModelGateway.complete`:**

```python
{
  "completion": "model completion text",
  "usage": {
    "input_tokens": 2000,
    "output_tokens": 100,
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 500
  }
}
```

### Aggregated usage shape

**Returned by `run_skill` (after initial + retry calls):**

```python
{
  "result": '{"answer": "json result"}',
  "usage": {
    "calls": 2,  # initial + retry
    "input_tokens": 1500,  # sum across both calls
    "output_tokens": 75,
    "cache_creation_input_tokens": 500,
    "cache_read_input_tokens": 300
  }
}
```

**Returned by MCP tool loop (after all tool rounds + final no-tools call):**

```python
{
  "result": "final result",
  "usage": {
    "calls": 3,  # three rounds
    "input_tokens": 2400,  # sum across all rounds
    "output_tokens": 120,
    "cache_creation_input_tokens": 500,
    "cache_read_input_tokens": 600
  }
}
```

---

## Tests

From breakdown Verification Approach (5 tests, exact count matching breakdown for story 04-agent):

### Test 1: run_skill extracts usage from skill invocation response

**Mapped from:** REQ-2 AC-2.1 (Breakdown verification item 1 for story 04-agent)

**Setup:**
- Mock `ModelGateway.complete` to return a fake response with usage data:
  ```python
  {
      "completion": "skill result text",
      "usage": {
          "input_tokens": 1000,
          "output_tokens": 50,
          "cache_creation_input_tokens": 500,
          "cache_read_input_tokens": 200
      }
  }
  ```
- Call `run_skill` with a skill name and input data (no JSON retry needed)

**Expected outcome:**
- `run_skill` returns:
  ```python
  {
      "result": "skill result text",
      "usage": {
          "calls": 1,
          "input_tokens": 1000,
          "output_tokens": 50,
          "cache_creation_input_tokens": 500,
          "cache_read_input_tokens": 200
      }
  }
  ```

**Verification:**
- AC-1 (REQ-2 AC-2.1): `run_skill` extracts usage from skill invocation response
- Usage dict contains all four token count fields
- `calls` count is 1 (single model API call)

---

### Test 2: run_skill sums usage across initial call and JSON retries

**Mapped from:** REQ-2 AC-2.3 (Breakdown verification item 2 for story 04-agent)

**Setup:**
- Mock `ModelGateway.complete` to return:
  - First call (no JSON in response):
    ```python
    {
        "completion": "no json here",
        "usage": {
            "input_tokens": 1000,
            "output_tokens": 50,
            "cache_creation_input_tokens": 500,
            "cache_read_input_tokens": 200
        }
    }
    ```
  - Second call (retry, with JSON):
    ```python
    {
        "completion": '{"answer": "json result"}',
        "usage": {
            "input_tokens": 500,
            "output_tokens": 25,
            "cache_creation_input_tokens": 0,
            "cache_read_input_tokens": 100
        }
    }
    ```
- Call `run_skill` with a skill name and input data

**Expected outcome:**
- `run_skill` returns:
  ```python
  {
      "result": '{"answer": "json result"}',
      "usage": {
          "calls": 2,  # initial + retry
          "input_tokens": 1500,  # 1000 + 500
          "output_tokens": 75,   # 50 + 25
          "cache_creation_input_tokens": 500,  # 500 + 0
          "cache_read_input_tokens": 300   # 200 + 100
      }
  }
  ```

**Verification:**
- AC-3 (REQ-2 AC-2.3): `usage` includes JSON retries in `run_skill`
- Counts are summed across initial + retry calls (2 model API calls)
- All four token count fields are aggregated correctly

---

### Test 3: MCP tool loop sums usage across all tool rounds

**Mapped from:** REQ-2 AC-2.3 (Breakdown verification item 3 for story 04-agent)

**Setup:**
- Mock `ModelGateway.complete` to return usage for three rounds:
  - Round 1 (tool call):
    ```python
    {
        "completion": "tool call 1",
        "usage": {"input_tokens": 1000, "output_tokens": 50, "cache_creation_input_tokens": 500, "cache_read_input_tokens": 200}
    }
    ```
  - Round 2 (tool call):
    ```python
    {
        "completion": "tool call 2",
        "usage": {"input_tokens": 800, "output_tokens": 40, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 300}
    }
    ```
  - Round 3 (final no-tools call):
    ```python
    {
        "completion": "final result",
        "usage": {"input_tokens": 600, "output_tokens": 30, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 100}
    }
    ```
- Call MCP tool loop function with tools and input data

**Expected outcome:**
- MCP tool loop returns:
  ```python
  {
      "result": "final result",
      "usage": {
          "calls": 3,  # three rounds
          "input_tokens": 2400,  # 1000 + 800 + 600
          "output_tokens": 120,  # 50 + 40 + 30
          "cache_creation_input_tokens": 500,  # 500 + 0 + 0
          "cache_read_input_tokens": 600   # 200 + 300 + 100
      }
  }
  ```

**Verification:**
- AC-3 (REQ-2 AC-2.3): `usage` includes every MCP tool round in `bindings.py` (including final no-tools call)
- Counts are summed across all rounds (3 model API calls)
- All four token count fields are aggregated correctly

---

### Test 4: ModelGateway.complete extracts usage from completion response

**Mapped from:** REQ-2 AC-2.1 (Breakdown verification item 4 for story 04-agent)

**Setup:**
- Mock the model API client to return a fake response:
  ```python
  {
      "completion": "model completion text",
      "usage": {
          "input_tokens": 2000,
          "output_tokens": 100,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 500
      }
  }
  ```
- Call `ModelGateway.complete` with a prompt

**Expected outcome:**
- `ModelGateway.complete` returns:
  ```python
  {
      "completion": "model completion text",
      "usage": {
          "input_tokens": 2000,
          "output_tokens": 100,
          "cache_creation_input_tokens": 0,
          "cache_read_input_tokens": 500
      }
  }
  ```

**Verification:**
- AC-1 (REQ-2 AC-2.1): `ModelGateway.complete` extracts usage from completion response
- Usage dict contains all four token count fields (`input_tokens`, `output_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`)
- Usage data is returned alongside completion

---

### Test 5: Node aggregates usage from multiple call sites

**Mapped from:** REQ-2 AC-2.3 (Breakdown verification item 5 for story 04-agent)

**Setup:**
- Mock `run_skill` to return:
  ```python
  {
      "result": "skill result",
      "usage": {
          "calls": 2,  # initial + retry
          "input_tokens": 1500,
          "output_tokens": 75,
          "cache_creation_input_tokens": 500,
          "cache_read_input_tokens": 300
      }
  }
  ```
- Mock MCP tool loop to return:
  ```python
  {
      "result": "tool result",
      "usage": {
          "calls": 3,  # three rounds
          "input_tokens": 2400,
          "output_tokens": 120,
          "cache_creation_input_tokens": 500,
          "cache_read_input_tokens": 600
      }
  }
  ```
- Run a fake node (e.g., orchestrator) that calls both `run_skill` and the MCP tool loop

**Expected outcome:**
- Node's state update includes:
  ```python
  {
      "usage": {
          "orchestrator": {
              "calls": 5,  # 2 (skill) + 3 (tools)
              "input_tokens": 3900,  # 1500 + 2400
              "output_tokens": 195,  # 75 + 120
              "cache_creation_input_tokens": 1000,  # 500 + 500
              "cache_read_input_tokens": 900   # 300 + 600
          }
      },
      # (other state update fields)
  }
  ```

**Verification:**
- AC-2 (REQ-2 AC-2.3): Each node's update carries the usage of the calls it made
- Usage is aggregated from multiple call sites (`run_skill` + MCP tool loop)
- All counts are summed correctly (5 total calls, all four token count fields aggregated)
- Per-node aggregation works (usage under `"orchestrator"` key in state update)

---

## Notes

- **No repo tree provided:** Implementation must verify the actual paths of `client.py`, `bindings.py`, and `gateway.py` in the `app` repo. The story specifies `src/agents/skills/client.py`, `src/agents/tools/bindings.py`, `src/agents/model/gateway.py` as the likely locations.
- **Usage extraction contract enforced:** All three call sites (`run_skill`, MCP tool loop, `ModelGateway.complete`) extract usage per the contract in HLD AD-3. Missing fields default to `0` via `.get(..., 0)` or `getattr(..., 0)`.
- **`calls` is number of model API calls:** In `run_skill`, `calls = 1` for no retry, `calls = 2` for one retry (initial + retry). In the MCP tool loop, `calls` is the number of rounds (including the final no-tools call). This matches the HLD AD-3 specification and test expectations.
- **Return type changes:** All three functions now return dicts with `"result"` (or `"completion"`) and `"usage"` fields. Callers must be updated to extract the result from the dict (e.g., `response["result"]` instead of `response`). This change may affect other nodes that call these functions; those changes are out of scope for this story but must be coordinated.
- **Model API client format:** The `ModelGateway.complete` algorithm assumes Anthropic's Claude API format (based on the ticket's reference to `claude-haiku-4-5-20251001`). If the actual client is different, the developer must adjust the extraction logic to match the client's response format.
- **No live model calls in tests:** All tests use mock `ModelGateway.complete`, mock `run_skill`, mock MCP tool loop, or fake responses. No live services.
- **Test count matches breakdown exactly:** 5 tests as specified in breakdown Verification Approach for story 04-agent. No additional tests beyond those agreed in the breakdown.
- **Existing tests unchanged:** After this story, `uv run pytest -q` must pass, and existing tests must pass unchanged. Existing tests that call `run_skill`, the MCP tool loop, or `ModelGateway.complete` may need minor updates to handle the new dict return type (extract `["result"]` or `["completion"]`), but their assertions should remain unchanged.
- **Dependencies clear:** Story 1 provides `state.usage` schema and reducer. This story collects usage at call sites. Story 3 (orchestrator) and other nodes aggregate usage from these call sites and include it in state updates.
