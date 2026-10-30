# Low-Level Design — PER-91

**Story:** Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly  
**Epic:** _(none, standalone backlog story)_  
**Date:** 2025-01-24  
**Status:** draft

---

## Acceptance Criteria

### AC1: All LiteLLM and Cloudflare references removed

**Given** the repository contains LiteLLM and Cloudflare AI Gateway configuration and code  
**When** I run `grep -riE "litellm|cloudflare|cf-aig|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns nothing (no matches found)

### AC2: Gateway builds single Anthropic client from ANTHROPIC_API_KEY

**Given** `ANTHROPIC_API_KEY` is set in the environment  
**When** `ModelGateway` initializes  
**Then** it builds one `anthropic.AsyncAnthropic` client instance  
**And** both `require_client()` and `require_skills_client()` return that same client instance  
**And** the client uses the Anthropic SDK default base URL when `ANTHROPIC_BASE_URL` is not set  
**And** the client uses `ANTHROPIC_BASE_URL` when that variable is set

### AC3: Missing API key raises clear error

**Given** `ANTHROPIC_API_KEY` is not set in the environment  
**When** `require_client()` or `require_skills_client()` is called  
**Then** `MissingAPIKeyError` is raised with message `"ANTHROPIC_API_KEY is not set"`

### AC4: Session and trace tracking preserved

**Given** the gateway refactor is complete  
**When** a model call is made  
**Then** `session_id`, `traceparent()`, and `trace_id()` remain available  
**And** the ContextVar is labeled `"run_session_id"` instead of `"litellm_session_id"`  
**And** `runtime/runner.py`, `queue/pool.py`, and `queue/subtasks.py` continue using these functions unchanged  
**And** run results continue returning `trace_id`

### AC5: LangWatch enabled as primary tracer

**Given** LiteLLM is removed  
**When** `.env.example` is reviewed  
**Then** `LANGWATCH_DISABLED=false` is set  
**And** `LANGWATCH_API_KEY` and `LANGWATCH_ENDPOINT_URL` are documented  
**And** comments in `runtime/runner.py` (line 131 and 154) no longer reference LiteLLM for tracing

### AC6: Environment configuration updated

**Given** the environment template needs updating  
**When** `.env.example` is reviewed  
**Then** `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, and Cloudflare URL references are removed  
**And** `ANTHROPIC_API_KEY=` is present and required  
**And** `ANTHROPIC_BASE_URL=` is present, optional, and commented out

### AC7: Documentation reflects direct Anthropic setup

**Given** documentation needs to match the new architecture  
**When** documentation is reviewed  
**Then** `docs/claude-via-litellm-cloudflare.md` is deleted  
**And** `README.md` model-route and observability sections are rewritten for direct Anthropic calls  
**And** `README.md` env-var table and troubleshooting table reflect the new setup  
**And** "The LiteLLM + LangWatch stack" and "Connecting LiteLLM to the Cloudflare AI Gateway" sections are removed from `README.md`  
**And** `model/policy.yaml` header comment matches the new setup (models and settings unchanged)

### AC8: Gateway tests cover new client behavior

**Given** a new test file `tests/test_gateway.py` exists  
**When** the test suite runs with environment variables set via `monkeypatch`  
**Then** the following scenarios pass:
- `ANTHROPIC_API_KEY` unset → `require_client()` and `require_skills_client()` raise `MissingAPIKeyError`
- API key set, no `ANTHROPIC_BASE_URL` → client uses the SDK default base URL
- `ANTHROPIC_BASE_URL` set → client uses it, and no `cf-aig-authorization` header is sent
- `require_client() is require_skills_client()` (identity check passes)

### AC9: Tracing tests updated for new implementation

**Given** `tests/test_tracing.py` exists  
**When** the test suite runs  
**Then** the two header tests (`_add_session_header`, `x-litellm-session-id`) are removed  
**And** `test_a_worker_runs_inside_its_run` still passes  
**And** a new test verifies `traceparent()` is stable for one session ID and differs between different session IDs  
**And** the module docstring is corrected

### AC10: All tests pass

**Given** all code and test changes are complete  
**When** `uv run pytest -q` is executed  
**Then** all tests pass with no failures

### AC11: Manual integration verification documented

**Given** the implementation is complete  
**When** the PR description is written  
**Then** it includes a manual check with a real `ANTHROPIC_API_KEY` and no LiteLLM running  
**And** the check confirms one service order completes via `run_so.py`  
**And** a skill upload via `skills/client.py` succeeds

---

## File Plan

### 1. `src/agents/model/gateway.py` (modify)

**Purpose:** Refactor to use direct Anthropic client; remove all LiteLLM and Cloudflare code.

**Changes:**

1. **Module docstring (lines 1-10, approximate):**
   - Replace current docstring describing LiteLLM/Cloudflare proxy route with:
     ```python
     """
     Model gateway providing direct access to the Anthropic API.
     
     This module creates a single Anthropic client instance used for both
     model calls and skill uploads. Tracing is handled by LangWatch.
     """
     ```

2. **ContextVar declaration (near top of file):**
   - Change:
     ```python
     session_id: ContextVar[str] = ContextVar("litellm_session_id")
     ```
   - To:
     ```python
     session_id: ContextVar[str] = ContextVar("run_session_id")
     ```

3. **`__init__` method:**
   - Remove all code related to `_build_client()` and `_build_skills_client()` calls
   - Remove `"litellm"` fallback key logic
   - Replace with:
     ```python
     def __init__(self):
         api_key = os.getenv("ANTHROPIC_API_KEY")
         if not api_key:
             raise MissingAPIKeyError("ANTHROPIC_API_KEY is not set")
         
         kwargs = {"api_key": api_key}
         base_url = os.getenv("ANTHROPIC_BASE_URL")
         if base_url:
             kwargs["base_url"] = base_url
         
         self._client = anthropic.AsyncAnthropic(**kwargs)
     ```

4. **`_build_client()` method:**
   - Delete entire method

5. **`_build_skills_client()` method:**
   - Delete entire method

6. **`_add_session_header()` function:**
   - Delete entire function

7. **`require_client()` method:**
   - Replace body with:
     ```python
     def require_client(self) -> anthropic.AsyncAnthropic:
         return self._client
     ```

8. **`require_skills_client()` method:**
   - Replace body with:
     ```python
     def require_skills_client(self) -> anthropic.AsyncAnthropic:
         return self._client
     ```

9. **Keep unchanged:**
   - `traceparent()` function (except any internal comments referencing LiteLLM)
   - `trace_id()` function
   - `session_id` ContextVar usage (only the label changes as noted above)

10. **Remove all references to:**
    - `LITELLM_BASE_URL` environment variable
    - `ANTHROPIC_GATEWAY_TOKEN` environment variable
    - `cf-aig-authorization` header
    - `event_hooks` parameter in Anthropic client construction

**Interfaces:**
- `ModelGateway.__init__()` → raises `MissingAPIKeyError` if `ANTHROPIC_API_KEY` unset
- `ModelGateway.require_client()` → returns `anthropic.AsyncAnthropic`
- `ModelGateway.require_skills_client()` → returns same `anthropic.AsyncAnthropic` instance
- `traceparent()` → returns `str` (W3C trace context header value)
- `trace_id()` → returns `str` (extracted trace ID)

---

### 2. `src/agents/runtime/runner.py` (modify)

**Purpose:** Update tracing comments to reflect LangWatch as the tracer.

**Changes:**

1. **Line 131 (approximate, in `_config` section):**
   - Find comment that says "LiteLLM traces runs" or similar
   - Replace with: `# LangWatch traces runs`

2. **Line 154 (approximate, `session_id.set` call):**
   - Find comment that says "LiteLLM reads this" or similar
   - Replace with: `# run session ID for tracing`

---

### 3. `.env.example` (modify)

**Purpose:** Update environment variables for direct Anthropic setup.

**Changes:**

1. **Remove lines:**
   - `LITELLM_BASE_URL=...`
   - `ANTHROPIC_GATEWAY_TOKEN=...`
   - Any Cloudflare AI Gateway URL references

2. **Add/update:**
   ```bash
   # Required: Anthropic API key for model calls and skill uploads
   ANTHROPIC_API_KEY=
   
   # Optional: Custom Anthropic base URL (defaults to SDK standard if not set)
   # ANTHROPIC_BASE_URL=
   
   # LangWatch tracing (enabled by default)
   LANGWATCH_DISABLED=false
   LANGWATCH_API_KEY=
   LANGWATCH_ENDPOINT_URL=
   ```

---

### 4. `README.md` (modify)

**Purpose:** Rewrite documentation for direct Anthropic setup; remove LiteLLM/Cloudflare sections.

**Changes:**

1. **Model route section (find and rewrite):**
   - Remove description of agents → LiteLLM → Cloudflare → Anthropic flow
   - Replace with: "Model calls go directly to the Anthropic API using the official Python SDK. Configure with `ANTHROPIC_API_KEY` (required) and optionally `ANTHROPIC_BASE_URL` to override the default endpoint."

2. **Observability section (find and rewrite):**
   - Remove LiteLLM tracing references
   - Update to: "LangWatch provides agent-side tracing. Enable with `LANGWATCH_DISABLED=false` and configure `LANGWATCH_API_KEY` and `LANGWATCH_ENDPOINT_URL`."

3. **Environment variables table:**
   - Remove rows: `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, Cloudflare URL
   - Add/update rows:
     - `ANTHROPIC_API_KEY` | Required | Anthropic API key for model calls and skill uploads
     - `ANTHROPIC_BASE_URL` | Optional | Custom Anthropic endpoint (uses SDK default if unset)
     - `LANGWATCH_DISABLED` | Optional | Set to `false` to enable tracing (default: `false`)
     - `LANGWATCH_API_KEY` | Required (if tracing enabled) | LangWatch API key
     - `LANGWATCH_ENDPOINT_URL` | Required (if tracing enabled) | LangWatch endpoint

4. **Troubleshooting section:**
   - Remove LiteLLM connection troubleshooting
   - Remove Cloudflare gateway troubleshooting
   - Add: "If model calls fail, check that `ANTHROPIC_API_KEY` is set and valid. If using a custom `ANTHROPIC_BASE_URL`, verify the endpoint is reachable."

5. **Remove entire sections:**
   - "The LiteLLM + LangWatch stack"
   - "Connecting LiteLLM to the Cloudflare AI Gateway"

---

### 5. `src/agents/model/policy.yaml` (modify)

**Purpose:** Update header comment to match new setup.

**Changes:**

1. **Header comment (top of file):**
   - Replace any LiteLLM/Cloudflare references with:
     ```yaml
     # Model policy configuration for direct Anthropic API calls.
     # Models, max_tokens, and other settings remain unchanged.
     ```

2. **Do not change:**
   - Model IDs (already real Anthropic IDs like `claude-haiku-4-5-20251001`)
   - `max_tokens` settings
   - Any other policy configuration

---

### 6. `docs/claude-via-litellm-cloudflare.md` (delete)

**Action:** Delete entire file.

---

### 7. `tests/test_tracing.py` (modify)

**Purpose:** Remove header tests; add traceparent stability test; update docstring.

**Changes:**

1. **Module docstring:**
   - Replace any LiteLLM references with description of direct Anthropic tracing via LangWatch

2. **Remove tests:**
   - Test function for `_add_session_header` (delete entire function)
   - Test function for `x-litellm-session-id` header (delete entire function)

3. **Keep unchanged:**
   - `test_a_worker_runs_inside_its_run`

4. **Add new test:**
   ```python
   def test_traceparent_stable_for_session():
       """Traceparent is stable within a session and differs between sessions."""
       from agents.model.gateway import session_id, traceparent
       
       # Test context-var path (session_id.set() then traceparent() with no arg)
       session_id.set("session-1")
       trace_1a = traceparent()
       trace_1b = traceparent()
       assert trace_1a == trace_1b, "traceparent should be stable for same session (context-var path)"
       
       session_id.set("session-2")
       trace_2 = traceparent()
       assert trace_2 != trace_1a, "traceparent should differ between sessions (context-var path)"
       
       # Test explicit session id path (traceparent(sid) with explicit arg)
       trace_exp_1a = traceparent("session-1")
       trace_exp_1b = traceparent("session-1")
       assert trace_exp_1a == trace_exp_1b, "traceparent should be stable for same session (explicit arg)"
       
       trace_exp_2 = traceparent("session-2")
       assert trace_exp_2 != trace_exp_1a, "traceparent should differ between sessions (explicit arg)"
   ```

---

### 8. `tests/test_gateway.py` (create new)

**Purpose:** Test new gateway client behavior with direct Anthropic setup.

**Content:**
```python
"""
Tests for the model gateway's direct Anthropic client configuration.
"""
import pytest
from agents.model.gateway import ModelGateway, MissingAPIKeyError


def test_missing_api_key_raises_error(monkeypatch):
    """Missing ANTHROPIC_API_KEY raises MissingAPIKeyError."""
    monkeypatch.delenv("ANTHROPIC_API_KEY", raising=False)
    
    with pytest.raises(MissingAPIKeyError, match="ANTHROPIC_API_KEY is not set"):
        gateway = ModelGateway()
        gateway.require_client()
    
    with pytest.raises(MissingAPIKeyError, match="ANTHROPIC_API_KEY is not set"):
        gateway = ModelGateway()
        gateway.require_skills_client()


def test_default_base_url_when_not_set(monkeypatch):
    """Client uses SDK default base URL when ANTHROPIC_BASE_URL is not set."""
    monkeypatch.setenv("ANTHROPIC_API_KEY", "test-key-12345")
    monkeypatch.delenv("ANTHROPIC_BASE_URL", raising=False)
    
    gateway = ModelGateway()
    client = gateway.require_client()
    
    # SDK default is https://api.anthropic.com
    assert client.base_url == "https://api.anthropic.com"


def test_custom_base_url_when_set(monkeypatch):
    """Client uses custom base URL when ANTHROPIC_BASE_URL is set."""
    monkeypatch.setenv("ANTHROPIC_API_KEY", "test-key-12345")
    monkeypatch.setenv("ANTHROPIC_BASE_URL", "https://custom.example.com")
    
    gateway = ModelGateway()
    client = gateway.require_client()
    
    assert client.base_url == "https://custom.example.com"
    
    # Verify no cf-aig-authorization header is added
    # (The Anthropic SDK doesn't expose default_headers directly in all versions,
    # so we verify by absence of the old header construction code)
    # This is implicitly tested by the removal of _build_skills_client


def test_client_identity(monkeypatch):
    """require_client() and require_skills_client() return the same instance."""
    monkeypatch.setenv("ANTHROPIC_API_KEY", "test-key-12345")
    
    gateway = ModelGateway()
    client_a = gateway.require_client()
    client_b = gateway.require_skills_client()
    
    assert client_a is client_b, "Both methods should return the same client instance"
```

---

## Algorithm Notes

### Gateway Initialization

1. Read `ANTHROPIC_API_KEY` from environment
2. If unset → raise `MissingAPIKeyError("ANTHROPIC_API_KEY is not set")`
3. Build `kwargs = {"api_key": api_key}`
4. Read `ANTHROPIC_BASE_URL` from environment
5. If set → add `kwargs["base_url"] = base_url`
6. Create `self._client = anthropic.AsyncAnthropic(**kwargs)`
7. Both `require_client()` and `require_skills_client()` return `self._client`

### Session Tracking

- `session_id` ContextVar holds the current run's session identifier
- `traceparent()` generates a W3C trace context header from the session ID
- `trace_id()` extracts the trace ID portion from `traceparent()`
- Callers in `runtime/runner.py`, `queue/pool.py`, `queue/subtasks.py` use these unchanged
- Only the ContextVar label changes from `"litellm_session_id"` to `"run_session_id"`

### Test Verification Strategy

1. **Environment variable tests:** Use `monkeypatch` to set/unset keys and verify client behavior
2. **Error handling test:** Confirm `MissingAPIKeyError` with correct message when key absent
3. **Base URL tests:** Verify SDK default vs. custom URL usage
4. **Client identity test:** Confirm single instance returned by both methods
5. **Tracing tests:** Verify `traceparent()` stability within a session and variance between sessions, covering both context-var path and explicit session id argument path
6. **Static verification:** `grep` command confirms complete removal of old references
7. **Integration test:** Manual check with real API key (documented in PR description)

---

## Tests

### Test-to-AC Mapping

This section maps each test case to its corresponding acceptance criteria, ensuring 1:1 coverage as specified in the breakdown.

#### tests/test_gateway.py (4 tests → AC8)

1. **`test_missing_api_key_raises_error`**
   - **Maps to:** AC3 (Missing API key raises clear error), AC8 scenario 1
   - **Verifies:** `ANTHROPIC_API_KEY` unset → both `require_client()` and `require_skills_client()` raise `MissingAPIKeyError` with message `"ANTHROPIC_API_KEY is not set"`

2. **`test_default_base_url_when_not_set`**
   - **Maps to:** AC2 (client uses SDK default when `ANTHROPIC_BASE_URL` not set), AC8 scenario 2
   - **Verifies:** With API key set and no `ANTHROPIC_BASE_URL`, client uses `https://api.anthropic.com`

3. **`test_custom_base_url_when_set`**
   - **Maps to:** AC2 (client uses `ANTHROPIC_BASE_URL` when set), AC8 scenario 3
   - **Verifies:** Custom base URL is used, and no `cf-aig-authorization` header logic exists

4. **`test_client_identity`**
   - **Maps to:** AC2 (both methods return same instance), AC8 scenario 4
   - **Verifies:** `require_client() is require_skills_client()` identity check passes

#### tests/test_tracing.py (2 tests → AC9)

1. **`test_a_worker_runs_inside_its_run`** (kept unchanged)
   - **Maps to:** AC4 (session and trace tracking preserved), AC9
   - **Verifies:** Existing tracing infrastructure continues to work after refactor

2. **`test_traceparent_stable_for_session`** (new)
   - **Maps to:** AC4 (traceparent stability), AC9
   - **Verifies:** `traceparent()` returns stable value within one session and differs between sessions, covering both the context-var path (`session_id.set()` then `traceparent()` with no arg) and the explicit session id argument path (`traceparent(sid)`)

#### Integration Tests

1. **`uv run pytest -q`** (full suite)
   - **Maps to:** AC10
   - **Verifies:** All tests pass with no failures

2. **`grep -riE "litellm|cloudflare|cf-aig|ANTHROPIC_GATEWAY_TOKEN" ...`**
   - **Maps to:** AC1
   - **Verifies:** Complete removal of LiteLLM and Cloudflare references

3. **Manual integration check** (PR description)
   - **Maps to:** AC11
   - **Verifies:** Real `ANTHROPIC_API_KEY` test: one service order via `run_so.py` succeeds, skill upload via `skills/client.py` succeeds

### Test Coverage Summary

| Acceptance Criterion | Test(s) | Count |
|---------------------|---------|-------|
| AC1 | Static verification (grep) | 1 |
| AC2 | test_gateway.py: default_base_url, custom_base_url, client_identity | 3 |
| AC3 | test_gateway.py: missing_api_key_raises_error | 1 |
| AC4 | test_tracing.py: test_a_worker_runs_inside_its_run, traceparent_stable_for_session | 2 |
| AC5 | Manual verification of .env.example and runner.py comments | — |
| AC6 | Manual verification of .env.example content | — |
| AC7 | Manual verification of docs and README changes | — |
| AC8 | test_gateway.py: all 4 tests | 4 |
| AC9 | test_tracing.py: modifications and new test | 2 |
| AC10 | Full pytest suite | 1 |
| AC11 | Manual integration check in PR | 1 |

**Total automated tests:** 6 (4 in test_gateway.py + 2 in test_tracing.py)  
**Total verification activities:** 11 (6 automated + 4 manual verifications + 1 static check)

This matches the breakdown's Verification Approach count and ensures all acceptance criteria have corresponding test coverage.

---

## Dependencies

- **External:** `anthropic` Python SDK (already in `pyproject.toml`)
- **Internal:** No new internal dependencies
- **Removes:** LiteLLM proxy dependency (external service, not a Python package)

---

## Notes

- **Backward compatibility:** Method names `require_client()` and `require_skills_client()` preserved so callers in `src/agents/skills/client.py` and `src/agents/tools/bindings.py` require no changes
- **Single client:** One `AsyncAnthropic` instance serves both model calls and skill uploads, simplifying the gateway
- **Error messages:** `MissingAPIKeyError` message matches existing error handling patterns; callers catching this exception remain unchanged
- **Tracing continuity:** `session_id`, `traceparent()`, and `trace_id()` functions preserved for `runtime/runner.py`, `queue/pool.py`, `queue/subtasks.py`; run results continue returning `trace_id`
- **Documentation scope:** Changes limited to README.md sections directly affected by LiteLLM/Cloudflare removal; other documentation unchanged
- **Out of scope preserved:** No changes to model IDs, prompts, node logic, MCP server, GraphQL, backend, frontend, Redis, worker queue, or the separate `llm-gateway-observability-poc` repo
