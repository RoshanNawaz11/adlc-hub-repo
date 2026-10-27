# LLD — Story 03-agent

**Story:** Add gateway tests and rewrite tracing tests  
**Repo:** `agent`

---

## Acceptance Criteria

### AC-1: Gateway client behavior validated by automated tests

**Given** the new `tests/test_gateway.py`  
**When** running the test suite  
**Then** all 6 test cases pass, verifying:
- `require_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- `require_skills_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- Client uses default Anthropic base URL when `ANTHROPIC_BASE_URL` is not set
- Client uses `ANTHROPIC_BASE_URL` when set
- No custom auth headers (`cf-aig-authorization`, `x-litellm-session-id`) are added
- `require_client()` and `require_skills_client()` return the same client instance

**Maps to:** Requirements AC-4, Breakdown Story 3 AC-1

---

### AC-2: Tracing tests rewritten to verify session-based behavior

**Given** the rewritten `tests/test_tracing.py`  
**When** running the test suite  
**Then** all 3 test cases pass, verifying:
- `traceparent()` is stable per session id
- `traceparent()` differs between session ids
- A worker runs inside its run's `session_id`  
**And** no assertions for `x-litellm-session-id` or `cf-aig-authorization` headers remain

**Maps to:** Requirements AC-5, Breakdown Story 3 AC-2

---

### AC-3: All automated tests pass

**Given** the changes to gateway, config, and tests  
**When** running `uv run pytest -q`  
**Then** all tests pass with no failures or errors  
**And** 9 new tests are added (6 in `test_gateway.py` + 3 in `test_tracing.py`)

**Maps to:** Requirements AC-6, Breakdown Story 3 AC-3

---

### AC-4: Zero LiteLLM/Cloudflare references remain

**Given** the completed changes  
**When** running `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns no matches

**Maps to:** Requirements AC-3, Breakdown Story 3 AC-4

---

### AC-5: Manual validation of direct Anthropic integration

**Given** a real `ANTHROPIC_API_KEY` and no LiteLLM process running  
**When** running `run_so.py` to complete one service order  
**Then** the service order completes successfully  
**And when** a skill upload is triggered via `skills/client.py`  
**Then** the upload succeeds  
**And** results are documented in PR description

**Maps to:** Requirements AC-7, Breakdown Story 3 AC-5

---

## File Plan

### New Files

**`tests/test_gateway.py`** (path established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`)
- Create new test module for gateway client behavior validation
- 6 test functions covering error cases, base URL handling, header validation, and singleton pattern
- Uses `pytest`, `unittest.mock`, and environment variable patching

---

### Modified Files

**`tests/test_tracing.py`** (path established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`)
- Remove all header assertion logic (`x-litellm-session-id`, `cf-aig-authorization`)
- Rewrite to focus on session tracking behavior only
- 3 test functions validating `traceparent()` stability, variance, and worker context propagation

---

## Interfaces

### Test Module: `tests/test_gateway.py`

#### Test Function Signatures

```python
def test_require_client_raises_when_api_key_unset() -> None:
    """Verify require_client() raises MissingAPIKeyError when ANTHROPIC_API_KEY unset."""
```

```python
def test_require_skills_client_raises_when_api_key_unset() -> None:
    """Verify require_skills_client() raises MissingAPIKeyError when ANTHROPIC_API_KEY unset."""
```

```python
def test_client_uses_default_base_url() -> None:
    """Verify client uses Anthropic SDK default base URL when ANTHROPIC_BASE_URL not set."""
```

```python
def test_client_uses_custom_base_url() -> None:
    """Verify client uses ANTHROPIC_BASE_URL when set."""
```

```python
def test_no_custom_auth_headers() -> None:
    """Verify no cf-aig-authorization or x-litellm-session-id headers added."""
```

```python
def test_singleton_client_instance() -> None:
    """Verify require_client() and require_skills_client() return same instance."""
```

---

### Test Module: `tests/test_tracing.py` (Rewritten)

#### Test Function Signatures

```python
def test_traceparent_stable_per_session() -> None:
    """Verify traceparent() returns same value for same session_id."""
```

```python
def test_traceparent_differs_across_sessions() -> None:
    """Verify traceparent() returns different values for different session_ids."""
```

```python
def test_worker_runs_in_session_context() -> None:
    """Verify worker executes inside its run's session_id context."""
```

---

### Environment Variable Mocking Pattern

```python
from unittest.mock import patch
import os

# Unset environment variable
with patch.dict(os.environ, {}, clear=True):
    # Code under test runs with env var unset
    pass

# Set environment variable
with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}):
    # Code under test runs with env var set
    pass
```

---

### Gateway Module Import Pattern

```python
# Import after environment patching to ensure fresh module load
from src.agents.model import gateway

# Or reload module if already imported
import importlib
importlib.reload(gateway)
```

---

## Algorithm Notes

### `tests/test_gateway.py` Test Logic

#### Test 1: `require_client()` raises when API key unset

1. Clear `ANTHROPIC_API_KEY` from environment (use `patch.dict(os.environ, {}, clear=True)`)
2. Force reload of `gateway` module to pick up environment change
3. Call `gateway.require_client()`
4. Assert `MissingAPIKeyError` raised
5. Assert exception message contains "ANTHROPIC_API_KEY environment variable required"

#### Test 2: `require_skills_client()` raises when API key unset

1. Clear `ANTHROPIC_API_KEY` from environment
2. Force reload of `gateway` module
3. Call `gateway.require_skills_client()`
4. Assert `MissingAPIKeyError` raised
5. Assert exception message contains "ANTHROPIC_API_KEY environment variable required"

#### Test 3: Client uses default base URL

1. Set `ANTHROPIC_API_KEY` in environment
2. Ensure `ANTHROPIC_BASE_URL` is **not** set (clear it explicitly)
3. Force reload of `gateway` module
4. Call `gateway.require_client()`
5. Assert returned client's `base_url` attribute matches Anthropic SDK default
   - **Note:** Anthropic SDK default is `"https://api.anthropic.com"` (verify in SDK docs if needed)
   - Alternative: mock `AsyncAnthropic` constructor and verify `base_url` kwarg was **not** passed

#### Test 4: Client uses custom base URL

1. Set `ANTHROPIC_API_KEY` in environment
2. Set `ANTHROPIC_BASE_URL="https://custom.endpoint.example.com"` in environment
3. Force reload of `gateway` module
4. Call `gateway.require_client()`
5. Assert returned client's `base_url` attribute equals `"https://custom.endpoint.example.com"`
   - Alternative: mock `AsyncAnthropic` constructor and verify `base_url` kwarg was passed with custom value

#### Test 5: No custom auth headers added

1. Set `ANTHROPIC_API_KEY` in environment
2. Force reload of `gateway` module
3. Call `gateway.require_client()` to get client instance
4. Inspect client's HTTP configuration (if accessible) or mock HTTP request and verify headers
   - **Approach A (if client exposes headers):** Assert `cf-aig-authorization` and `x-litellm-session-id` not in client's default headers
   - **Approach B (mock-based):** Mock `httpx.AsyncClient` (Anthropic SDK's underlying HTTP client) and verify no custom headers injected beyond SDK defaults
   - **Simplest approach:** Since Stories 1-2 already removed header injection logic, this test can verify absence of header-related code by checking no custom `httpx_client` or request hooks were passed to `AsyncAnthropic` constructor

#### Test 6: Singleton client instance

1. Set `ANTHROPIC_API_KEY` in environment
2. Force reload of `gateway` module
3. Call `gateway.require_client()` → store as `client1`
4. Call `gateway.require_skills_client()` → store as `client2`
5. Assert `client1 is client2` (same object identity)

---

### `tests/test_tracing.py` Rewrite Logic

#### Test 1: `traceparent()` stable per session

1. Import `gateway` module
2. Set `gateway.session_id.set("test-session-123")`
3. Call `gateway.traceparent()` → store as `tp1`
4. Call `gateway.traceparent()` again → store as `tp2`
5. Assert `tp1 == tp2` (same traceparent string for same session ID)

#### Test 2: `traceparent()` differs across sessions

1. Import `gateway` module
2. Set `gateway.session_id.set("session-A")`
3. Call `gateway.traceparent()` → store as `tp_a`
4. Set `gateway.session_id.set("session-B")`
5. Call `gateway.traceparent()` → store as `tp_b`
6. Assert `tp_a != tp_b` (different traceparent strings for different session IDs)

#### Test 3: Worker runs in session context

1. Import `gateway` module and worker execution utilities (e.g., `queue/pool.py` or `queue/subtasks.py`)
2. Set `gateway.session_id.set("main-session")`
3. Define a worker function that:
   - Captures the current `gateway.session_id.get()` inside the worker
   - Returns the captured session ID
4. Submit worker to worker pool (or execute in subprocess if pool testing is complex)
5. Assert worker-returned session ID equals `"main-session"`
   - **Note:** This validates that `session_id` ContextVar propagates correctly to worker threads/processes
   - **Caveat:** If worker pool uses multiprocessing (not threading), ContextVars don't propagate automatically — test may need to verify explicit propagation logic in `queue/pool.py` or `queue/subtasks.py`

---

### Module Reload Strategy

**Why reload?**  
The `gateway` module may initialize the singleton client at import time. To test different environment variable states, the module must be reloaded after each environment patch.

**How to reload:**
```python
import importlib
from src.agents.model import gateway

# After patching environment
importlib.reload(gateway)
```

**Alternative (safer for pytest):**  
Use `pytest` fixtures to reset the gateway module state between tests:

```python
import pytest
from src.agents.model import gateway

@pytest.fixture(autouse=True)
def reset_gateway_client():
    """Reset gateway module singleton before each test."""
    gateway._client = None  # Reset singleton (assumes _client is module-level variable)
    yield
    gateway._client = None  # Clean up after test
```

**Recommended approach:** Use fixture-based reset to avoid import order issues.

---

### Header Validation Strategy (Test 5)

Since header injection logic was removed in Story 1, Test 5 validates **absence of custom behavior** rather than presence of headers.

**Approach 1: Inspect AsyncAnthropic constructor calls (mock-based)**

```python
from unittest.mock import patch, MagicMock
import anthropic

with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}):
    with patch("anthropic.AsyncAnthropic") as mock_constructor:
        mock_client = MagicMock()
        mock_constructor.return_value = mock_client
        
        importlib.reload(gateway)
        gateway.require_client()
        
        # Verify constructor called with minimal args (no custom httpx_client, no event_hooks)
        call_kwargs = mock_constructor.call_args.kwargs
        assert "httpx_client" not in call_kwargs
        assert "event_hooks" not in call_kwargs
        # Only api_key (and optionally base_url) should be present
```

**Approach 2: Check actual client default headers (if accessible)**

```python
with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}):
    importlib.reload(gateway)
    client = gateway.require_client()
    
    # Access client's internal HTTP client (if exposed)
    # Anthropic SDK uses httpx.AsyncClient internally
    # May not be directly accessible; this approach is fragile
    
    # Safer: just verify no custom headers in any real request (requires integration test)
```

**Recommended:** Use **Approach 1** (mock-based validation) to avoid depending on Anthropic SDK internals.

---

## Tests

### Test Count

**Total automated tests:** 9 tests (6 gateway + 3 tracing)  
**Maps to:** Requirements AC-4 (6 gateway tests), AC-5 (3 tracing tests), AC-6 (9 total new tests); Breakdown Story 3 verification approach

---

### Tests in `tests/test_gateway.py` (6 tests)

#### Test 1: `test_require_client_raises_when_api_key_unset`
**Maps to:** Requirements AC-4 (case 1), Breakdown Story 3 AC-1

**Given** `ANTHROPIC_API_KEY` is not set in environment  
**When** calling `gateway.require_client()`  
**Then** `MissingAPIKeyError` is raised  
**And** exception message contains "ANTHROPIC_API_KEY environment variable required"

**Verification:**
1. Use `patch.dict(os.environ, {}, clear=True)` to clear environment
2. Force reload of `gateway` module
3. Call `gateway.require_client()` within `pytest.raises(MissingAPIKeyError)` context
4. Assert exception message matches expected text

---

#### Test 2: `test_require_skills_client_raises_when_api_key_unset`
**Maps to:** Requirements AC-4 (case 2), Breakdown Story 3 AC-1

**Given** `ANTHROPIC_API_KEY` is not set in environment  
**When** calling `gateway.require_skills_client()`  
**Then** `MissingAPIKeyError` is raised  
**And** exception message contains "ANTHROPIC_API_KEY environment variable required"

**Verification:**
1. Use `patch.dict(os.environ, {}, clear=True)` to clear environment
2. Force reload of `gateway` module
3. Call `gateway.require_skills_client()` within `pytest.raises(MissingAPIKeyError)` context
4. Assert exception message matches expected text

---

#### Test 3: `test_client_uses_default_base_url`
**Maps to:** Requirements AC-4 (case 3), Breakdown Story 3 AC-1

**Given** `ANTHROPIC_API_KEY` is set and `ANTHROPIC_BASE_URL` is not set  
**When** calling `gateway.require_client()`  
**Then** the returned client uses the Anthropic SDK default base URL (`https://api.anthropic.com`)

**Verification:**
1. Use `patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"})` (no `ANTHROPIC_BASE_URL`)
2. Force reload of `gateway` module
3. Call `gateway.require_client()`
4. Assert `client.base_url` equals `"https://api.anthropic.com"`  
   - Alternative: Mock `AsyncAnthropic` constructor and verify `base_url` kwarg was **not** passed

---

#### Test 4: `test_client_uses_custom_base_url`
**Maps to:** Requirements AC-4 (case 4), Breakdown Story 3 AC-1

**Given** both `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL` are set  
**When** calling `gateway.require_client()`  
**Then** the returned client uses the value from `ANTHROPIC_BASE_URL`

**Verification:**
1. Use `patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key", "ANTHROPIC_BASE_URL": "https://custom.endpoint.example.com"})`
2. Force reload of `gateway` module
3. Call `gateway.require_client()`
4. Assert `client.base_url` equals `"https://custom.endpoint.example.com"`

---

#### Test 5: `test_no_custom_auth_headers`
**Maps to:** Requirements AC-4 (case 5), Breakdown Story 3 AC-1

**Given** `ANTHROPIC_API_KEY` is set  
**When** the gateway client is instantiated  
**Then** no custom auth headers (`cf-aig-authorization`, `x-litellm-session-id`) are added

**Verification:**
1. Use `patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"})`
2. Mock `anthropic.AsyncAnthropic` constructor
3. Force reload of `gateway` module
4. Call `gateway.require_client()`
5. Assert mock constructor called without `httpx_client` or `event_hooks` kwargs
6. Verify only `api_key` (and optionally `base_url`) were passed

---

#### Test 6: `test_singleton_client_instance`
**Maps to:** Requirements AC-4 (case 6), Breakdown Story 3 AC-1

**Given** `ANTHROPIC_API_KEY` is set  
**When** calling both `gateway.require_client()` and `gateway.require_skills_client()`  
**Then** both return the same client instance (same object identity)

**Verification:**
1. Use `patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"})`
2. Force reload of `gateway` module
3. Call `gateway.require_client()` → store as `client1`
4. Call `gateway.require_skills_client()` → store as `client2`
5. Assert `client1 is client2` (object identity check)

---

### Tests in `tests/test_tracing.py` (3 tests, rewritten)

#### Test 1: `test_traceparent_stable_per_session`
**Maps to:** Requirements AC-5 (case 1), Breakdown Story 3 AC-2

**Given** a session ID is set  
**When** calling `gateway.traceparent()` multiple times  
**Then** it returns the same traceparent string each time

**Verification:**
1. Import `gateway` module
2. Set `gateway.session_id.set("test-session-123")`
3. Call `gateway.traceparent()` → store as `tp1`
4. Call `gateway.traceparent()` again → store as `tp2`
5. Assert `tp1 == tp2`

---

#### Test 2: `test_traceparent_differs_across_sessions`
**Maps to:** Requirements AC-5 (case 2), Breakdown Story 3 AC-2

**Given** two different session IDs are set sequentially  
**When** calling `gateway.traceparent()` for each session  
**Then** the traceparent strings differ

**Verification:**
1. Import `gateway` module
2. Set `gateway.session_id.set("session-A")`
3. Call `gateway.traceparent()` → store as `tp_a`
4. Set `gateway.session_id.set("session-B")`
5. Call `gateway.traceparent()` → store as `tp_b`
6. Assert `tp_a != tp_b`

---

#### Test 3: `test_worker_runs_in_session_context`
**Maps to:** Requirements AC-5 (case 3), Breakdown Story 3 AC-2

**Given** a session ID is set in the main thread  
**When** a worker executes  
**Then** the worker runs inside its run's `session_id` context

**Verification:**
1. Import `gateway` module and worker execution utilities (from `queue/pool.py` or `queue/subtasks.py`)
2. Set `gateway.session_id.set("main-session")`
3. Define a worker function that:
   - Captures `gateway.session_id.get()` inside the worker
   - Returns the captured session ID
4. Submit worker to worker pool (or execute in subprocess if pool testing is complex)
5. Assert worker-returned session ID equals `"main-session"`

**Note:** If worker pool uses multiprocessing (not threading), ContextVars don't propagate automatically — test validates explicit propagation logic in `queue/pool.py` or `queue/subtasks.py`

---

### Verification Steps

**Step 1: Run new gateway tests**
```bash
uv run pytest -q tests/test_gateway.py
```
- Expected: 6 tests pass, exit code 0
- Maps to: Requirements AC-4, Breakdown Story 3 AC-1

---

**Step 2: Run rewritten tracing tests**
```bash
uv run pytest -q tests/test_tracing.py
```
- Expected: 3 tests pass, exit code 0
- Maps to: Requirements AC-5, Breakdown Story 3 AC-2

---

**Step 3: Run full test suite**
```bash
uv run pytest -q
```
- Expected: All tests pass (including 9 new ones), exit code 0
- Maps to: Requirements AC-6, Breakdown Story 3 AC-3

---

**Step 4: Validate zero LiteLLM/Cloudflare references**
```bash
grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example
```
- Expected: No output (zero matches)
- Maps to: Requirements AC-3, Breakdown Story 3 AC-4

---

**Step 5: Manual validation (AC-5)**
1. Set real `ANTHROPIC_API_KEY` in environment
2. Ensure no LiteLLM process is running
3. Run `run_so.py` to complete one service order
   - Document: success or failure message, execution time
4. Trigger skill upload via `skills/client.py`
   - Document: upload status, any errors
5. Record results in PR description under "Manual Validation (AC-5)" section

- Maps to: Requirements AC-7, Breakdown Story 3 AC-5

---

## Notes

- **AC count alignment:** Story has 5 ACs matching breakdown Story 3 structure (AC-1 covers Requirements AC-4 six test cases, AC-2 covers Requirements AC-5 three test cases, AC-3 covers Requirements AC-6 regression check, AC-4 covers Requirements AC-3 grep validation, AC-5 covers Requirements AC-7 manual validation); Requirements has 7 ACs, breakdown Story 3 has 5 ACs, this story has 5 ACs — counts align
- **Test count alignment:** 9 automated tests (6 gateway + 3 tracing) matches Requirements AC-4 (6 tests), AC-5 (3 tests), AC-6 (9 total), breakdown Story 3 verification approach, and breakdown "Test Strategy" section
- **File path verification:** `tests/test_gateway.py` (new), `tests/test_tracing.py` (modified) — paths established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`; repo-tree shows empty `existing files:` list indicating tree is incomplete but paths are specified in ticket and breakdown
- **Module reload necessity:** Gateway module initializes singleton at import; tests must reload after environment patches
- **Header validation approach:** Mock-based constructor inspection preferred over inspecting client internals (fragile)
- **Worker context test caveat:** If worker pool uses multiprocessing (not threading), ContextVar propagation may not work automatically — test validates explicit propagation logic in `queue/pool.py` or `queue/subtasks.py`
- **Manual validation scope:** AC-5 is documented (not automated) — PR description template should include section for manual test results
- **No Story 1/2 dependencies in tests:** Tests assume Stories 1-2 already completed (gateway refactored, config updated)
- **Test isolation:** Use `pytest` fixtures to reset `gateway._client` singleton between tests to avoid state leakage
- **Grep validation timing:** AC-4 grep check is final validation step — run after all code changes complete, before opening PR
