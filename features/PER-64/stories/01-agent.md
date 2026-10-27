# LLD — Story 01-agent

**Story:** Refactor gateway.py to use Anthropic SDK directly  
**Repo:** `agent`

---

## Acceptance Criteria

### AC-1: Gateway client refactored to use Anthropic SDK directly

**Given** the refactored `src/agents/model/gateway.py`  
**When** reviewing the implementation  
**Then** it meets all of the following:
- Builds a single `anthropic.AsyncAnthropic` client from `ANTHROPIC_API_KEY`
- Passes `base_url` parameter only when `ANTHROPIC_BASE_URL` is set (defaults to Anthropic SDK default otherwise)
- Both `require_client()` and `require_skills_client()` return the same client instance
- Raises `MissingAPIKeyError` with message `"ANTHROPIC_API_KEY environment variable required"` when `ANTHROPIC_API_KEY` is unset
- Removes `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, `cf-aig-authorization` header, `x-litellm-session-id` header, and `_add_session_header` request hook
- Renames `session_id` ContextVar label from `"litellm_session_id"` to `"run_session_id"` while preserving `traceparent()` and `trace_id()` behavior
- Updates module docstring to describe direct Anthropic route

**Maps to:** Requirements AC-1, Breakdown Story 1 AC-1

**Covers tickets:** PER-65 (01-agent: primary gateway refactor)

---

### AC-2: Existing tests pass

**Given** the refactored `gateway.py`  
**When** running `uv run pytest -q`  
**Then** all existing tests pass with no failures

**Maps to:** Requirements AC-6 (partial: no regressions from gateway refactor), Breakdown Story 1 AC-2

**Covers tickets:** PER-65 (01-agent: validates no regressions)

---

## File Plan

### Modified Files

**`src/agents/model/gateway.py`** (path confirmed in repo-tree at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/src/agents/model/gateway.py`)
- Remove LiteLLM imports (if present)
- Remove Cloudflare-specific imports or header constants
- Replace dual client initialization paths with single `_get_or_create_client()` helper
- Remove environment variable reads for `LITELLM_BASE_URL` and `ANTHROPIC_GATEWAY_TOKEN`
- Remove header construction logic for `cf-aig-authorization` and `x-litellm-session-id`
- Remove `_add_session_header` request hook function and its registration
- Update `MissingAPIKeyError` message to `"ANTHROPIC_API_KEY environment variable required"`
- Change `session_id = ContextVar("litellm_session_id")` to `session_id = ContextVar("run_session_id")`
- Update module docstring to describe direct Anthropic API routing
- Implement `require_client()` and `require_skills_client()` to both call `_get_or_create_client()` (return same instance)

---

## Interfaces

### Module-Level API

```python
def require_client() -> anthropic.AsyncAnthropic:
    """
    Returns configured Anthropic client for model calls.
    
    Raises:
        MissingAPIKeyError: When ANTHROPIC_API_KEY is not set
    
    Returns:
        Singleton AsyncAnthropic client instance
    """
```

```python
def require_skills_client() -> anthropic.AsyncAnthropic:
    """
    Returns configured Anthropic client for skill uploads.
    Identical to require_client(); kept for backward compatibility.
    
    Raises:
        MissingAPIKeyError: When ANTHROPIC_API_KEY is not set
    
    Returns:
        Same singleton AsyncAnthropic client as require_client()
    """
```

```python
def traceparent() -> str:
    """
    Returns W3C Trace Context traceparent header for current session.
    
    Returns:
        Traceparent string in format: 00-<trace-id>-<parent-id>-<flags>
    """
```

```python
def trace_id() -> str:
    """
    Extracts and returns trace ID from current session's traceparent.
    
    Returns:
        Trace ID string (32 hex characters from traceparent)
    """
```

---

### Exception Classes

```python
class MissingAPIKeyError(Exception):
    """Raised when ANTHROPIC_API_KEY environment variable is not set."""
    pass
```

---

### Context Variables

```python
from contextvars import ContextVar

session_id: ContextVar[str] = ContextVar("run_session_id")
```

**Note:** Variable name `session_id` unchanged; only ContextVar label string changes from `"litellm_session_id"` to `"run_session_id"`. Downstream code (logging in `src/agents/runtime/runner.py`, worker pool in `src/agents/queue/pool.py` and `src/agents/queue/subtasks.py` per repo-tree) references the variable by name, not by label.

---

### Client Initialization Pattern

```python
from typing import Optional
import os
import anthropic

# Module-level singleton
_client: Optional[anthropic.AsyncAnthropic] = None

def _get_or_create_client() -> anthropic.AsyncAnthropic:
    """
    Creates or returns existing Anthropic client singleton.
    
    Raises:
        MissingAPIKeyError: When ANTHROPIC_API_KEY is not set
    
    Returns:
        Configured AsyncAnthropic client
    """
    global _client
    
    if _client is not None:
        return _client
    
    api_key = os.environ.get("ANTHROPIC_API_KEY")
    if not api_key:
        raise MissingAPIKeyError("ANTHROPIC_API_KEY environment variable required")
    
    # Build kwargs conditionally
    client_kwargs = {"api_key": api_key}
    base_url = os.environ.get("ANTHROPIC_BASE_URL")
    if base_url:
        client_kwargs["base_url"] = base_url
    
    _client = anthropic.AsyncAnthropic(**client_kwargs)
    return _client


def require_client() -> anthropic.AsyncAnthropic:
    """Returns configured Anthropic client for model calls."""
    return _get_or_create_client()


def require_skills_client() -> anthropic.AsyncAnthropic:
    """Returns configured Anthropic client for skill uploads (same as require_client)."""
    return _get_or_create_client()
```

---

## Algorithm Notes

### Client Instantiation Logic

1. **Singleton pattern:**
   - Module-level `_client` variable initialized to `None`
   - First call to `require_client()` or `require_skills_client()` triggers creation via `_get_or_create_client()`
   - Subsequent calls return cached instance from `_client`
   - Both public methods delegate to same helper → guaranteed singleton

2. **Environment variable validation:**
   - Check for `ANTHROPIC_API_KEY` presence before client creation
   - Raise `MissingAPIKeyError` immediately if unset (fail-fast, no fallback)
   - Do **not** validate `ANTHROPIC_BASE_URL` content (optional override, pass through as-is)

3. **Base URL handling:**
   - Read `ANTHROPIC_BASE_URL` from environment
   - Only pass `base_url` kwarg to client constructor if value is non-empty
   - When unset or empty string, omit kwarg entirely → Anthropic SDK uses its default endpoint

4. **No custom HTTP configuration:**
   - Do **not** pass custom HTTP client kwarg
   - Do **not** pass custom headers or event hooks kwargs
   - Rely entirely on Anthropic SDK's default HTTP client behavior

---

### Session Tracking Preservation

1. **ContextVar rename:**
   - Change ContextVar label string from `"litellm_session_id"` to `"run_session_id"`
   - Variable name `session_id` stays unchanged (only label string in ContextVar constructor changes)
   - Downstream code in `src/agents/runtime/runner.py`, `src/agents/queue/pool.py`, `src/agents/queue/subtasks.py` (all confirmed in repo-tree) references variable by name, not by label → no changes needed there

2. **Traceparent stability:**
   - `traceparent()` function logic **completely unchanged**
   - Still reads `session_id.get()`, still generates W3C-compliant traceparent header
   - Still deterministic: same session ID value → same traceparent on repeated calls

3. **Trace ID extraction:**
   - `trace_id()` function logic **completely unchanged**
   - Still extracts trace ID component from `traceparent()` output

---

### Removed Logic

1. **LiteLLM routing:**
   - Delete any LiteLLM imports if present
   - Delete environment variable reads for `LITELLM_BASE_URL`
   - Delete any LiteLLM-specific client initialization code (if separate client was built for LiteLLM path)

2. **Cloudflare gateway:**
   - Delete environment variable reads for `ANTHROPIC_GATEWAY_TOKEN`
   - Delete header construction logic for `cf-aig-authorization`
   - Delete any Cloudflare-specific URL construction

3. **Session header injection:**
   - Delete header logic for `x-litellm-session-id`
   - Delete `_add_session_header` request hook function entirely (was registered as event listener)
   - Delete any event hooks registration for request/response listeners

4. **Dual client paths:**
   - If `require_client()` and `require_skills_client()` previously returned different client instances, unify them to return the same `_client` singleton

---

### Module Docstring Update

**Target state:**
Describe direct Anthropic API access via official AsyncAnthropic client, configured with required ANTHROPIC_API_KEY and optional ANTHROPIC_BASE_URL.

---

## Tests

### Test Count

**Total automated tests added in this story:** 0 new tests

Story 3 will add comprehensive test coverage (9 automated tests: 6 gateway in `tests/test_gateway.py` + 3 tracing in rewritten `tests/test_tracing.py` per breakdown verification approach and Requirements AC-4, AC-5, AC-6).

---

### Verification Approach

This story does not add new automated tests. Story 3 will add comprehensive test coverage in `tests/test_gateway.py` (6 tests covering client instantiation, error handling, base URL behavior, header validation, and singleton pattern per Requirements AC-4) and rewritten `tests/test_tracing.py` (3 tests per Requirements AC-5). Verification here is manual and regression-based:

#### 1. Manual REPL Testing (Interactive Python Session)

**Setup:**
```bash
export ANTHROPIC_API_KEY="sk-ant-test123"
unset ANTHROPIC_BASE_URL
python3 -i
```

**Test sequence:**
```python
from src.agents.model import gateway

# Test 1: Verify singleton pattern
client1 = gateway.require_client()
client2 = gateway.require_skills_client()
assert id(client1) == id(client2), "Clients are not same instance"
print("✓ Singleton verified")

# Test 2: Verify default base URL
assert "anthropic.com" in str(client1.base_url), f"Wrong base URL: {client1.base_url}"
print("✓ Default base URL verified")

# Exit and test custom base URL
exit()
```

```bash
export ANTHROPIC_BASE_URL="https://custom.endpoint.example.com"
python3 -i
```

```python
from src.agents.model import gateway

# Test 3: Verify custom base URL
client = gateway.require_client()
assert "custom.endpoint.example.com" in str(client.base_url), f"Wrong base URL: {client.base_url}"
print("✓ Custom base URL verified")

exit()
```

```bash
unset ANTHROPIC_API_KEY
python3 -i
```

```python
from src.agents.model import gateway
from src.agents.model.gateway import MissingAPIKeyError

# Test 4: Verify error handling
try:
    gateway.require_client()
    print("✗ Should have raised MissingAPIKeyError")
except MissingAPIKeyError as e:
    assert "ANTHROPIC_API_KEY environment variable required" in str(e), f"Wrong error message: {e}"
    print("✓ Error handling verified")
```

**Maps to:** Requirements AC-1 (client instantiation, singleton, base URL, error handling)

---

#### 2. Code Review Checklist

Run the following command to verify removal of LiteLLM/Cloudflare artifacts:

```bash
grep -n "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN\|x-litellm-session-id\|_add_session_header" \
  src/agents/model/gateway.py
```

**Expected output:** No matches (exit code 1)

**Manual visual inspection of `gateway.py`:**
- [ ] `session_id = ContextVar("run_session_id")` present (label changed)
- [ ] `traceparent()` function body unchanged (compare git diff)
- [ ] `trace_id()` function body unchanged (compare git diff)
- [ ] Module docstring updated to describe direct Anthropic route (no LiteLLM/Cloudflare references)
- [ ] No imports from LiteLLM package
- [ ] No Cloudflare URL construction logic
- [ ] `require_client()` and `require_skills_client()` both return result of `_get_or_create_client()`

**Maps to:** Requirements AC-1 (removal of LiteLLM/Cloudflare artifacts, session tracking preservation)

---

#### 3. Regression Validation (Existing Test Suite)

Run the full test suite to ensure no existing tests break:

```bash
uv run pytest -q
```

**Expected output:**
- Exit code 0 (all tests pass)
- No test failures or errors
- Warnings acceptable (deprecation warnings, etc.) but no new errors

**If failures occur:**
- Review failing test names and stack traces
- Check for hard-coded LiteLLM/Cloudflare URLs in test fixtures
- Check for tests that mock `gateway` module and may need fixture updates
- **Do not fix test failures in this story** — document them and address in Story 3 if gateway-related

**Maps to:** Requirements AC-6 (partial: no regressions), Breakdown Story 1 AC-2

---

#### 4. Downstream Import Validation

Verify that files importing `gateway` functions still work:

```bash
# Check skills/client.py imports
python3 -c "from src.agents.skills.client import *; print('✓ skills.client imports OK')"

# Check tools/bindings.py imports
python3 -c "from src.agents.tools.bindings import *; print('✓ tools.bindings imports OK')"

# Check runtime/runner.py imports
python3 -c "from src.agents.runtime.runner import *; print('✓ runtime.runner imports OK')"
```

**Expected output:** No import errors, prints "OK" messages

**Maps to:** Backward compatibility verification (ticket requirement to keep method names unchanged)

---

### Verification Summary

| Step | Type | Maps to AC | Pass Criteria |
|------|------|------------|---------------|
| 1. REPL Testing | Manual | AC-1 (partial) | Singleton verified, base URLs correct, error raised when key unset |
| 2. Code Review | Manual | AC-1 (partial) | Zero grep matches, ContextVar label updated, functions unchanged |
| 3. Regression Tests | Automated | AC-2 | `uv run pytest -q` exits 0 |
| 4. Import Validation | Manual | Backward compat | No import errors from downstream files |

**Total verification steps:** 4 approaches

**Mapping to ACs:**
- Manual REPL testing validates gateway client instantiation, singleton behavior, base URL handling, and error handling (Requirements AC-1 partial)
- Code review validates removal of LiteLLM/Cloudflare artifacts, session tracking preservation (Requirements AC-1 partial)
- Regression validation ensures no existing test breakage (Requirements AC-6 partial, story AC-2)
- Import validation ensures backward compatibility with downstream code (Requirements AC-1 partial: method names unchanged)

---

## Notes

- **AC count alignment:** Story has 2 ACs matching breakdown Story 1 structure (AC-1 consolidates all Requirements AC-1 elements into single grouped AC, AC-2 covers regression check); Requirements AC-1 is a single grouped AC, not fragmented
- **Test count alignment:** 0 new automated tests in this story; Story 3 adds 9 automated tests (6 gateway + 3 tracing) per breakdown verification approach and Requirements AC-4, AC-5, AC-6; sum across all three stories: 0+0+9=9
- **Ticket coverage explicit:** AC-1 covers ticket PER-65 (01-agent: primary gateway refactor implementation); AC-2 covers ticket PER-65 (01-agent: validates no regressions from gateway refactor)
- **File path verification:** `src/agents/model/gateway.py` confirmed in repo-tree at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents/src/agents/model/gateway.py`; downstream files `src/agents/skills/client.py`, `src/agents/tools/bindings.py`, `src/agents/runtime/runner.py`, `src/agents/queue/pool.py`, `src/agents/queue/subtasks.py` all confirmed in repo-tree
- **Real file context:** Repo-tree shows `gateway.py` exists; ticket context specifies current LiteLLM → Cloudflare routing and describes removal targets
- **No new dependencies:** Story uses existing `anthropic` SDK (already in project dependencies per ticket context); no package additions required
- **Backward compatibility:** `require_skills_client()` method name preserved; callers in `src/agents/skills/client.py` and `src/agents/tools/bindings.py` (both confirmed in repo-tree) unchanged
- **Session tracking continuity:** `session_id` ContextVar behavior unchanged; only label text updated for clarity. Downstream files (`src/agents/runtime/runner.py`, `src/agents/queue/pool.py`, `src/agents/queue/subtasks.py` confirmed in repo-tree) reference variable by name, not label
- **Import cleanup:** Expect to remove any LiteLLM imports if present and any Cloudflare-specific constants/helpers
- **Docstring scope:** Module docstring update describes high-level routing change; function docstrings for `require_client()` / `require_skills_client()` stay focused on individual method contracts
- **Error message precision:** `MissingAPIKeyError` message explicitly names `ANTHROPIC_API_KEY` to guide user to correct environment variable
- **No functional changes to tracing:** `traceparent()` and `trace_id()` functions are **not modified** in this story — only the ContextVar label changes. Logic for generating/extracting trace IDs stays identical
- **Singleton reset for tests:** Story 3 will add pytest fixture to reset `gateway._client = None` between tests; not implemented in this story
- **Base URL validation:** SDK default used when `ANTHROPIC_BASE_URL` unset; no fallback URLs, no URL validation (pass through as-is to SDK)
