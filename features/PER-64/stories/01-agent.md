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

---

### AC-2: Existing tests pass

**Given** the refactored `gateway.py`  
**When** running `uv run pytest -q`  
**Then** all existing tests pass with no failures

**Maps to:** Requirements AC-6 (partial: no regressions), Breakdown Story 1 AC-2

---

## File Plan

### Modified Files

**`src/agents/model/gateway.py`**
- Remove imports/references to LiteLLM and Cloudflare components
- Replace dual client paths with single `AsyncAnthropic` instance
- Remove `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN` env var reads
- Remove `cf-aig-authorization` and `x-litellm-session-id` header logic
- Remove `_add_session_header` request hook
- Update `MissingAPIKeyError` message text
- Rename `session_id` ContextVar label to `"run_session_id"`
- Update module docstring
- Ensure `require_client()` and `require_skills_client()` return same instance

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
        Trace ID string
    """
```

---

### Exception Classes

```python
class MissingAPIKeyError(Exception):
    """Raised when ANTHROPIC_API_KEY environment variable is not set."""
```

---

### Context Variables

```python
session_id: ContextVar[str] = ContextVar("run_session_id")
```

---

### Client Initialization Pattern

```python
# Internal: singleton client instance
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
```

---

## Algorithm Notes

### Client Instantiation Logic

1. **Singleton pattern:**
   - Module-level `_client` variable initialized to `None`
   - First call to `require_client()` or `require_skills_client()` triggers creation
   - Subsequent calls return cached instance
   - Both methods delegate to internal `_get_or_create_client()` helper

2. **Environment variable validation:**
   - Check for `ANTHROPIC_API_KEY` presence before client creation
   - Raise `MissingAPIKeyError` immediately if unset (fail-fast)
   - Do not validate `ANTHROPIC_BASE_URL` (optional override)

3. **Base URL handling:**
   - Read `ANTHROPIC_BASE_URL` from environment
   - Only pass `base_url` kwarg to `AsyncAnthropic()` if value is non-empty
   - When unset/empty, Anthropic SDK uses its default endpoint

4. **No custom HTTP configuration:**
   - Remove all custom request hooks (e.g., `_add_session_header`)
   - Remove all custom header injection logic
   - Rely entirely on Anthropic SDK's default HTTP client behavior

---

### Session Tracking Preservation

1. **ContextVar rename:**
   - Change `ContextVar("litellm_session_id")` → `ContextVar("run_session_id")`
   - Variable name `session_id` stays unchanged (only label string changes)
   - Downstream code (logging, worker pool) references variable by name, not label

2. **Traceparent stability:**
   - `traceparent()` function logic unchanged
   - Still generates W3C-compliant header from current `session_id` value
   - Still deterministic: same session ID → same traceparent on repeated calls

3. **Trace ID extraction:**
   - `trace_id()` function logic unchanged
   - Still extracts trace ID component from `traceparent()` output
   - Format: `00-<trace-id>-<parent-id>-<flags>` → extract `<trace-id>` segment

---

### Removed Logic

1. **LiteLLM routing:**
   - Delete `LITELLM_BASE_URL` environment variable reads
   - Delete any LiteLLM-specific client initialization code
   - Delete any LiteLLM SDK imports

2. **Cloudflare gateway:**
   - Delete `ANTHROPIC_GATEWAY_TOKEN` environment variable reads
   - Delete `cf-aig-authorization` header construction
   - Delete any Cloudflare-specific URL/endpoint logic

3. **Session header injection:**
   - Delete `x-litellm-session-id` header logic
   - Delete `_add_session_header` request hook function
   - Delete any event listener registration for request hooks

---

## Tests

**Test count:** 0 new automated tests in this story (Story 3 adds 6 gateway tests + 3 tracing tests = 9 total automated tests per breakdown)

### Verification Approach

This story does not add new automated tests. Story 3 will add comprehensive test coverage in `tests/test_gateway.py` (6 tests) and rewrite `tests/test_tracing.py` (3 tests). Verification here is manual and regression-based:

#### 1. Manual REPL Testing
- Set `ANTHROPIC_API_KEY`, unset `ANTHROPIC_BASE_URL`
- Import and call `require_client()` and `require_skills_client()`
- Verify same object returned (check `id(client1) == id(client2)`)
- Verify client uses default Anthropic base URL (inspect `client.base_url`)
- Set `ANTHROPIC_BASE_URL` to custom value
- Re-run and verify client uses custom URL
- Unset `ANTHROPIC_API_KEY`
- Verify `MissingAPIKeyError` raised with correct message

#### 2. Code Review
- Manual `grep` for `litellm`, `cloudflare`, `cf-aig`, `ANTHROPIC_GATEWAY_TOKEN`
- Verify no matches in `src/agents/model/gateway.py`
- Verify `session_id` ContextVar label is `"run_session_id"`
- Verify `traceparent()` and `trace_id()` logic unchanged

#### 3. Regression Validation
- Run `uv run pytest -q` (full test suite)
- Verify exit code 0 (all tests pass)
- Review any test failures/warnings for gateway-related issues

**Mapping to ACs:**
- Manual REPL testing validates gateway client instantiation, singleton behavior, base URL handling, and error handling (Requirements AC-1 partial)
- Code review validates removal of LiteLLM/Cloudflare artifacts, session tracking preservation (Requirements AC-1 partial)
- Regression validation ensures no existing test breakage (Requirements AC-6 partial, story AC-2)

**Total verification steps:** 3 approaches (REPL, code review, regression tests)

---

## Notes

- **AC count alignment:** Story has 2 ACs matching breakdown Story 1 structure (AC-1 consolidates all Requirements AC-1 elements, AC-2 covers regression check); Requirements AC-1 is a single grouped AC, not fragmented
- **Test count alignment:** 0 new tests in this story; Story 3 adds 9 automated tests (6 gateway + 3 tracing) per breakdown and Requirements AC-4/AC-5/AC-6
- **File path verification:** `src/agents/model/gateway.py` established by ticket context (ticket says "wired in `src/agents/model/gateway.py`"); repo-tree shows empty `existing files:` list indicating tree is incomplete but path is specified in ticket at `/home/regularinsaan/projects/trinet-adlc-agents`
- **No new dependencies:** Story uses existing `anthropic` SDK; no package additions required
- **Backward compatibility:** `require_skills_client()` method name preserved; callers in `skills/client.py` and `tools/bindings.py` unchanged per ticket context
- **Session tracking continuity:** `session_id` ContextVar behavior unchanged; only label text updated for clarity
- **Import cleanup:** Expect to remove imports like `from litellm import ...` or Cloudflare-specific modules
- **Docstring scope:** Module docstring update describes high-level routing change; function docstrings stay focused on individual method contracts
- **Error message precision:** `MissingAPIKeyError` message specifies `ANTHROPIC_API_KEY` (not generic "API key") to guide user to correct env var
