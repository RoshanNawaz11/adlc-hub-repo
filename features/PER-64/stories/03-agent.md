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

**Covers tickets:** PER-67 (03-agent: primary adds tests)

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

**Covers tickets:** PER-67 (03-agent: primary rewrites tests)

---

### AC-3: All automated tests pass

**Given** the changes to gateway, config, and tests  
**When** running `uv run pytest -q`  
**Then** all tests pass with no failures or errors  
**And** 9 new tests are added (6 in `test_gateway.py` + 3 in `test_tracing.py`)

**Maps to:** Requirements AC-6, Breakdown Story 3 AC-3

**Covers tickets:** PER-67 (03-agent: primary validates all stories integrated correctly)

---

### AC-4: Zero LiteLLM/Cloudflare references remain

**Given** the completed changes  
**When** running `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns no matches

**Maps to:** Requirements AC-3, Breakdown Story 3 AC-4

**Covers tickets:** PER-67 (03-agent: primary final validation)

---

### AC-5: Manual validation of direct Anthropic integration

**Given** a real `ANTHROPIC_API_KEY` and no LiteLLM process running  
**When** running `run_so.py` to complete one service order  
**Then** the service order completes successfully  
**And when** a skill upload is triggered via `src/agents/skills/client.py`  
**Then** the upload succeeds  
**And** results are documented in PR description

**Maps to:** Requirements AC-7, Breakdown Story 3 AC-5

**Covers tickets:** PER-67 (03-agent: primary documents validation)

---

## File Plan

### New Files

**`tests/test_gateway.py`** (path: `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/tests/test_gateway.py`)
- Create new test module for gateway client behavior validation
- 6 test functions covering error cases, base URL handling, header validation, and singleton pattern
- Uses `pytest`, `unittest.mock.patch`, environment variable patching

---

### Modified Files

**`tests/test_tracing.py`** (path: `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/tests/test_tracing.py`)
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

# Reset singleton between tests (in pytest fixture)
gateway._client = None
```

---

## Algorithm Notes

### Gateway Module Singleton Reset Strategy

**Problem:** Gateway module initializes singleton `_client` at import time. Testing different environment states requires singleton reset between tests.

**Solution:** Use pytest fixture-based reset pattern (safer than `importlib.reload` for test isolation):

```python
import pytest
from src.agents.model import gateway

@pytest.fixture(autouse=True)
def reset_gateway_client():
    """Reset gateway module singleton before/after each test."""
    gateway._client = None  # Reset before test
    yield
    gateway._client = None  # Clean up after test
```

**Rationale:**
- Avoids import order issues with `importlib.reload`
- Ensures clean state between tests
- Autouse fixture applies to all tests in module automatically

---

### Environment Variable Patching Pattern

**Goal:** Test gateway behavior under different environment configurations without polluting system environment.

**Pattern:**
```python
from unittest.mock import patch
import os

# Test with environment variable unset
with patch.dict(os.environ, {}, clear=True):
    # Code under test runs with empty environment
    gateway.require_client()  # Should raise MissingAPIKeyError

# Test with environment variable set
with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}):
    # Code under test runs with API key present
    client = gateway.require_client()  # Should succeed
```

**Key points:**
- `clear=True` empties environment completely for strict isolation
- Patches only affect code inside `with` block
- Original environment restored automatically after block

---

### Gateway Client Base URL Validation (Tests 3 & 4)

**Approach:** Direct inspection of `client.base_url` attribute.

**Test 3 logic (default base URL):**
```python
with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}):
    # Ensure ANTHROPIC_BASE_URL not set (explicitly clear it)
    env = dict(os.environ)
    env.pop("ANTHROPIC_BASE_URL", None)
    
    with patch.dict(os.environ, env, clear=True):
        client = gateway.require_client()
        
        # Anthropic SDK default
        assert client.base_url == "https://api.anthropic.com"
```

**Test 4 logic (custom base URL):**
```python
custom_url = "https://custom.endpoint.example.com"
with patch.dict(os.environ, {
    "ANTHROPIC_API_KEY": "test-key",
    "ANTHROPIC_BASE_URL": custom_url
}, clear=True):
    client = gateway.require_client()
    assert client.base_url == custom_url
```

**Fallback approach (if `base_url` attribute not accessible):** Mock `AsyncAnthropic` constructor and inspect kwargs:

```python
from unittest.mock import patch, MagicMock
import anthropic

with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}, clear=True):
    with patch("anthropic.AsyncAnthropic") as mock_constructor:
        mock_client = MagicMock()
        mock_constructor.return_value = mock_client
        
        gateway.require_client()
        
        # Verify base_url kwarg not passed (defaults to SDK default)
        call_kwargs = mock_constructor.call_args.kwargs
        assert "base_url" not in call_kwargs
```

---

### Header Validation Strategy (Test 5)

**Goal:** Verify no custom auth headers are added to client.

**Approach:** Mock `AsyncAnthropic` constructor and verify only `api_key` (and optionally `base_url`) are passed as kwargs.

```python
from unittest.mock import patch, MagicMock
import anthropic

with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}, clear=True):
    with patch("anthropic.AsyncAnthropic") as mock_constructor:
        mock_client = MagicMock()
        mock_constructor.return_value = mock_client
        
        gateway.require_client()
        
        # Verify no custom HTTP client or event hooks passed
        call_kwargs = mock_constructor.call_args.kwargs
        assert "httpx_client" not in call_kwargs
        assert "event_hooks" not in call_kwargs
        assert "default_headers" not in call_kwargs
        
        # Only api_key (and optionally base_url) should be present
        allowed_keys = {"api_key", "base_url"}
        assert set(call_kwargs.keys()).issubset(allowed_keys)
```

**Rationale:**
- Story 1 removed custom header logic
- Custom headers would be injected via `httpx_client`, `event_hooks`, or `default_headers` kwargs
- Absence of these kwargs confirms no custom header injection

---

### Singleton Validation Logic (Test 6)

**Goal:** Verify `require_client()` and `require_skills_client()` return the same object instance.

**Approach:** Direct object identity comparison using `is` operator:

```python
with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}, clear=True):
    client1 = gateway.require_client()
    client2 = gateway.require_skills_client()
    
    # Same object in memory (singleton pattern)
    assert client1 is client2
    
    # Also verify both are AsyncAnthropic instances
    import anthropic
    assert isinstance(client1, anthropic.AsyncAnthropic)
    assert isinstance(client2, anthropic.AsyncAnthropic)
```

**Why `is` not `==`:**
- `is` checks object identity (same memory address)
- `==` checks value equality (could be different objects with same state)
- Singleton pattern requires same instance, so `is` is correct

---

### Tracing Test 3 Worker Context Validation

**Goal:** Verify worker executes inside its run's `session_id` context.

**Approach (threading-based):**
```python
import threading
from src.agents.model import gateway

def test_worker_runs_in_session_context():
    # Set session in main thread
    gateway.session_id.set("main-session")
    
    # Capture session from worker thread
    captured_session = None
    def worker():
        nonlocal captured_session
        captured_session = gateway.session_id.get()
    
    # Execute worker in thread
    thread = threading.Thread(target=worker)
    thread.start()
    thread.join()
    
    # Verify worker saw same session
    assert captured_session == "main-session"
```

**Note:** ContextVars propagate correctly to threads by default in Python 3.7+. If worker pool uses `multiprocessing`, manual propagation logic would be needed, but per repo-tree `src/agents/queue/pool.py` exists and likely uses threading.

---

### `tests/test_tracing.py` Rewrite Scope

**Remove:**
- All assertions checking for `x-litellm-session-id` header
- All assertions checking for `cf-aig-authorization` header
- Any imports of LiteLLM-specific modules
- Any test fixtures setting up LiteLLM mock servers

**Keep:**
- Test function structure for session tracking validation
- Imports of `gateway` module and `ContextVar` utilities
- Assertions checking `traceparent()` stability and variance
- Worker context propagation tests

**Update:**
- Test docstrings to remove LiteLLM references
- Test function names if they include "litellm" in the name
- Any comments explaining LiteLLM-specific behavior

---

## Tests

### Test Count

**Total automated tests:** 9 tests (6 gateway + 3 tracing)  
**Maps to:** Requirements AC-4 (6 gateway tests), AC-5 (3 tracing tests), AC-6 (9 total new tests); Breakdown Story 3 verification approach

---

### Tests in `tests/test_gateway.py` (6 tests)

#### Test 1: `test_require_client_raises_when_api_key_unset`

**Maps to:** Requirements AC-4 (case 1), Breakdown Story 3 AC-1  
**Covers tickets:** PER-67 (03-agent: adds test)

**Given** `ANTHROPIC_API_KEY` is not set in environment  
**When** calling `gateway.require_client()`  
**Then** `MissingAPIKeyError` is raised  
**And** exception message contains "ANTHROPIC_API_KEY environment variable required"

**Implementation:**
```python
import pytest
from unittest.mock import patch
import os
from src.agents.model import gateway
from src.agents.model.gateway import MissingAPIKeyError

def test_require_client_raises_when_api_key_unset():
    """Verify require_client() raises MissingAPIKeyError when ANTHROPIC_API_KEY unset."""
    with patch.dict(os.environ, {}, clear=True):
        with pytest.raises(MissingAPIKeyError) as exc_info:
            gateway.require_client()
        
        assert "ANTHROPIC_API_KEY environment variable required" in str(exc_info.value)
```

---

#### Test 2: `test_require_skills_client_raises_when_api_key_unset`

**Maps to:** Requirements AC-4 (case 2), Breakdown Story 3 AC-1  
**Covers tickets:** PER-67 (03-agent: adds test)

**Given** `ANTHROPIC_API_KEY` is not set in environment  
**When** calling `gateway.require_skills_client()`  
**Then** `MissingAPIKeyError` is raised  
**And** exception message contains "ANTHROPIC_API_KEY environment variable required"

**Implementation:**
```python
def test_require_skills_client_raises_when_api_key_unset():
    """Verify require_skills_client() raises MissingAPIKeyError when ANTHROPIC_API_KEY unset."""
    with patch.dict(os.environ, {}, clear=True):
        with pytest.raises(MissingAPIKeyError) as exc_info:
            gateway.require_skills_client()
        
        assert "ANTHROPIC_API_KEY environment variable required" in str(exc_info.value)
```

---

#### Test 3: `test_client_uses_default_base_url`

**Maps to:** Requirements AC-4 (case 3), Breakdown Story 3 AC-1  
**Covers tickets:** PER-67 (03-agent: adds test)

**Given** `ANTHROPIC_API_KEY` is set and `ANTHROPIC_BASE_URL` is not set  
**When** calling `gateway.require_client()`  
**Then** the returned client uses the Anthropic SDK default base URL

**Implementation:**
```python
import anthropic

def test_client_uses_default_base_url():
    """Verify client uses Anthropic SDK default base URL when ANTHROPIC_BASE_URL not set."""
    env = {"ANTHROPIC_API_KEY": "test-key"}
    # Explicitly remove ANTHROPIC_BASE_URL if present
    env_clean = {k: v for k, v in env.items()}
    
    with patch.dict(os.environ, env_clean, clear=True):
        client = gateway.require_client()
        
        # Verify client is AsyncAnthropic instance
        assert isinstance(client, anthropic.AsyncAnthropic)
        
        # Verify default base URL
        assert client.base_url == "https://api.anthropic.com"
```

---

#### Test 4: `test_client_uses_custom_base_url`

**Maps to:** Requirements AC-4 (case 4), Breakdown Story 3 AC-1  
**Covers tickets:** PER-67 (03-agent: adds test)

**Given** both `ANTHROPIC_API_KEY` and `ANTHROPIC_BASE_URL` are set  
**When** calling `gateway.require_client()`  
**Then** the returned client uses the value from `ANTHROPIC_BASE_URL`

**Implementation:**
```python
def test_client_uses_custom_base_url():
    """Verify client uses ANTHROPIC_BASE_URL when set."""
    custom_url = "https://custom.endpoint.example.com"
    env = {
        "ANTHROPIC_API_KEY": "test-key",
        "ANTHROPIC_BASE_URL": custom_url
    }
    
    with patch.dict(os.environ, env, clear=True):
        client = gateway.require_client()
        
        # Verify custom base URL
        assert client.base_url == custom_url
```

---

#### Test 5: `test_no_custom_auth_headers`

**Maps to:** Requirements AC-4 (case 5), Breakdown Story 3 AC-1  
**Covers tickets:** PER-67 (03-agent: adds test)

**Given** `ANTHROPIC_API_KEY` is set  
**When** the gateway client is instantiated  
**Then** no custom auth headers are added

**Implementation:**
```python
from unittest.mock import MagicMock

def test_no_custom_auth_headers():
    """Verify no cf-aig-authorization or x-litellm-session-id headers added."""
    with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}, clear=True):
        with patch("anthropic.AsyncAnthropic") as mock_constructor:
            mock_client = MagicMock()
            mock_constructor.return_value = mock_client
            
            # Reset singleton to force new client creation
            gateway._client = None
            gateway.require_client()
            
            # Verify constructor called without custom HTTP config
            call_kwargs = mock_constructor.call_args.kwargs
            assert "httpx_client" not in call_kwargs, "Custom httpx_client passed (may inject headers)"
            assert "event_hooks" not in call_kwargs, "Event hooks passed (may inject headers)"
            assert "default_headers" not in call_kwargs, "Custom headers passed"
            
            # Only api_key and optionally base_url should be present
            allowed_keys = {"api_key", "base_url"}
            actual_keys = set(call_kwargs.keys())
            assert actual_keys.issubset(allowed_keys), f"Unexpected kwargs: {actual_keys - allowed_keys}"
```

---

#### Test 6: `test_singleton_client_instance`

**Maps to:** Requirements AC-4 (case 6), Breakdown Story 3 AC-1  
**Covers tickets:** PER-67 (03-agent: adds test)

**Given** `ANTHROPIC_API_KEY` is set  
**When** calling both `gateway.require_client()` and `gateway.require_skills_client()`  
**Then** both return the same client instance (same object identity)

**Implementation:**
```python
def test_singleton_client_instance():
    """Verify require_client() and require_skills_client() return same instance."""
    with patch.dict(os.environ, {"ANTHROPIC_API_KEY": "test-key"}, clear=True):
        client1 = gateway.require_client()
        client2 = gateway.require_skills_client()
        
        # Same object in memory (singleton pattern)
        assert client1 is client2, "Clients are not the same instance"
        
        # Both are AsyncAnthropic instances
        assert isinstance(client1, anthropic.AsyncAnthropic)
        assert isinstance(client2, anthropic.AsyncAnthropic)
```

---

### Tests in `tests/test_tracing.py` (3 tests, rewritten)

#### Test 1: `test_traceparent_stable_per_session`

**Maps to:** Requirements AC-5 (case 1), Breakdown Story 3 AC-2  
**Covers tickets:** PER-67 (03-agent: rewrites test)

**Given** a session ID is set  
**When** calling `gateway.traceparent()` multiple times  
**Then** it returns the same traceparent string each time

**Implementation:**
```python
from src.agents.model import gateway

def test_traceparent_stable_per_session():
    """Verify traceparent() returns same value for same session_id."""
    test_session = "test-session-123"
    gateway.session_id.set(test_session)
    
    tp1 = gateway.traceparent()
    tp2 = gateway.traceparent()
    
    assert tp1 == tp2, f"Traceparent changed: {tp1} != {tp2}"
    
    # Verify format (W3C Trace Context: 00-<trace-id>-<parent-id>-<flags>)
    assert tp1.startswith("00-"), f"Invalid traceparent format: {tp1}"
```

---

#### Test 2: `test_traceparent_differs_across_sessions`

**Maps to:** Requirements AC-5 (case 2), Breakdown Story 3 AC-2  
**Covers tickets:** PER-67 (03-agent: rewrites test)

**Given** two different session IDs are set sequentially  
**When** calling `gateway.traceparent()` for each session  
**Then** the traceparent strings differ

**Implementation:**
```python
def test_traceparent_differs_across_sessions():
    """Verify traceparent() returns different values for different session_ids."""
    gateway.session_id.set("session-A")
    tp_a = gateway.traceparent()
    
    gateway.session_id.set("session-B")
    tp_b = gateway.traceparent()
    
    assert tp_a != tp_b, f"Traceparent should differ: {tp_a} == {tp_b}"
    
    # Both should be valid format
    assert tp_a.startswith("00-"), f"Invalid traceparent A: {tp_a}"
    assert tp_b.startswith("00-"), f"Invalid traceparent B: {tp_b}"
```

---

#### Test 3: `test_worker_runs_in_session_context`

**Maps to:** Requirements AC-5 (case 3), Breakdown Story 3 AC-2  
**Covers tickets:** PER-67 (03-agent: rewrites test)

**Given** a session ID is set in the main thread  
**When** a worker executes  
**Then** the worker runs inside its run's `session_id` context

**Implementation:**
```python
import threading

def test_worker_runs_in_session_context():
    """Verify worker executes inside its run's session_id context."""
    main_session = "main-session-456"
    gateway.session_id.set(main_session)
    
    # Capture session from worker thread
    captured_session = None
    def worker():
        nonlocal captured_session
        captured_session = gateway.session_id.get()
    
    # Execute worker in thread
    thread = threading.Thread(target=worker)
    thread.start()
    thread.join()
    
    # Verify worker saw same session
    assert captured_session == main_session, \
        f"Worker session mismatch: expected {main_session}, got {captured_session}"
```

---

### Verification Steps

**Step 1: Run new gateway tests**
```bash
uv run pytest -q tests/test_gateway.py
```
- **Expected:** 6 tests pass, exit code 0
- **Maps to:** Requirements AC-4, Breakdown Story 3 AC-1

---

**Step 2: Run rewritten tracing tests**
```bash
uv run pytest -q tests/test_tracing.py
```
- **Expected:** 3 tests pass, exit code 0
- **Maps to:** Requirements AC-5, Breakdown Story 3 AC-2

---

**Step 3: Run full test suite**
```bash
uv run pytest -q
```
- **Expected:** All tests pass (including 9 new ones), exit code 0
- **Maps to:** Requirements AC-6, Breakdown Story 3 AC-3

---

**Step 4: Validate zero LiteLLM/Cloudflare references**
```bash
grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example
```
- **Expected:** No output (zero matches)
- **Maps to:** Requirements AC-3, Breakdown Story 3 AC-4

---

**Step 5: Manual validation**

1. Set real `ANTHROPIC_API_KEY` in environment:
   ```bash
   export ANTHROPIC_API_KEY="sk-ant-..."
   ```

2. Ensure no LiteLLM process is running:
   ```bash
   ps aux | grep litellm
   # Should return no matching processes
   ```

3. Run `run_so.py` to complete one service order:
   ```bash
   python run_so.py --order-id <test-order-id>
   ```
   - **Document:** Success/failure message, execution time, any errors

4. Trigger skill upload via `src/agents/skills/client.py`:
   ```bash
   # Exact command depends on skills/client.py interface
   # Example (adjust as needed):
   python -m src.agents.skills.client upload --skill <skill-name>
   ```
   - **Document:** Upload status, file size, any errors

5. Record results in PR description under "Manual Validation" section:
   ```markdown
   ## Manual Validation (AC-5)
   
   **Environment:**
   - `ANTHROPIC_API_KEY`: Set (real key)
   - `ANTHROPIC_BASE_URL`: Not set (using default)
   - LiteLLM process: Not running (confirmed via `ps aux`)
   
   **Test 1: Service order completion**
   - Command: `python run_so.py --order-id test-123`
   - Result: ✅ SUCCESS
   - Execution time: 4.2 seconds
   - Notes: Order completed successfully, no errors
   
   **Test 2: Skill upload**
   - Command: `python -m src.agents.skills.client upload --skill compliance`
   - Result: ✅ SUCCESS
   - Upload size: 24 KB
   - Notes: Skill uploaded successfully, no errors
   ```

- **Maps to:** Requirements AC-7, Breakdown Story 3 AC-5

---

## Notes

- **AC count alignment:** Story has 5 ACs matching Breakdown Story 3 structure (AC-1 covers Requirements AC-4 six test cases, AC-2 covers Requirements AC-5 three test cases, AC-3 covers Requirements AC-6 regression check, AC-4 covers Requirements AC-3 grep validation, AC-5 covers Requirements AC-7 manual validation); Requirements has 7 ACs, breakdown Story 3 has 5 ACs, this story has 5 ACs — counts align
- **Test count alignment:** 9 automated tests (6 gateway + 3 tracing) matches Requirements AC-4 (6 tests), AC-5 (3 tests), AC-6 (9 total), Breakdown Story 3 verification approach, and breakdown "Test Strategy" section
- **Ticket coverage explicit:** All ACs list ticket coverage per critic requirement: AC-1 covers PER-67 (adds tests); AC-2 covers PER-67 (rewrites); AC-3 covers PER-67 (validates integration); AC-4 covers PER-67 (final validation); AC-5 covers PER-67 (documents validation)
- **File path verification:** `tests/test_gateway.py` new at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/tests/test_gateway.py`; `tests/test_tracing.py` at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/tests/test_tracing.py`
- **Real file context:** Repo-tree confirms `src/agents/model/gateway.py` exists, `src/agents/skills/client.py` exists, tests folder exists; ticket context describes current LiteLLM routing and headers
- **No new dependencies:** Story uses existing `anthropic` SDK and `pytest`; no package additions required
- **Backward compatibility:** Tests validate `require_skills_client()` method preserved; callers in `src/agents/skills/client.py` unchanged
- **Session tracking continuity:** Tests validate `session_id` ContextVar behavior unchanged; only label text updated in Story 1
- **Singleton reset for tests:** Pytest fixture pattern documented for test isolation
- **Base URL validation:** Tests verify SDK default used when `ANTHROPIC_BASE_URL` unset; no fallback URLs, no URL validation
