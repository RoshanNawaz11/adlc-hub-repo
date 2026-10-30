# Requirements — PER-91

**Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly**

---

## Given/When/Then Acceptance Criteria

### AC1: All LiteLLM and Cloudflare references removed from codebase

**Given** the repository contains LiteLLM and Cloudflare AI Gateway configuration and code  
**When** I run `grep -riE "litellm|cloudflare|cf-aig|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns nothing (no matches found)

---

### AC2: Gateway builds single Anthropic client from ANTHROPIC_API_KEY

**Given** `ANTHROPIC_API_KEY` is set in the environment  
**When** `ModelGateway` initializes  
**Then** it builds one `anthropic.AsyncAnthropic` client instance  
**And** both `require_client()` and `require_skills_client()` return that same client instance  
**And** the client uses the Anthropic SDK default base URL when `ANTHROPIC_BASE_URL` is not set  
**And** the client uses `ANTHROPIC_BASE_URL` when that variable is set

---

### AC3: Missing API key raises clear error

**Given** `ANTHROPIC_API_KEY` is not set in the environment  
**When** `require_client()` or `require_skills_client()` is called  
**Then** `MissingAPIKeyError` is raised with message "ANTHROPIC_API_KEY is not set"

---

### AC4: Session and trace tracking preserved

**Given** the gateway refactor is complete  
**When** a model call is made  
**Then** `session_id`, `traceparent()`, and `trace_id()` remain available  
**And** the ContextVar is labeled `"run_session_id"` instead of `"litellm_session_id"`  
**And** `runtime/runner.py`, `queue/pool.py`, and `queue/subtasks.py` continue using these functions unchanged  
**And** run results continue returning `trace_id`

---

### AC5: LangWatch enabled as primary tracer

**Given** LiteLLM is removed  
**When** `.env.example` is reviewed  
**Then** `LANGWATCH_DISABLED=false` is set  
**And** `LANGWATCH_API_KEY` and `LANGWATCH_ENDPOINT_URL` are documented  
**And** comments in `runtime/runner.py` (line 131 and 154) no longer reference LiteLLM for tracing

---

### AC6: Environment configuration updated

**Given** the environment template needs updating  
**When** `.env.example` is reviewed  
**Then** `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, and Cloudflare URL references are removed  
**And** `ANTHROPIC_API_KEY=` is present and required  
**And** `ANTHROPIC_BASE_URL=` is present, optional, and commented out

---

### AC7: Documentation reflects direct Anthropic setup

**Given** documentation needs to match the new architecture  
**When** documentation is reviewed  
**Then** `docs/claude-via-litellm-cloudflare.md` is deleted  
**And** `README.md` model-route and observability sections are rewritten for direct Anthropic calls  
**And** `README.md` env-var table and troubleshooting table reflect the new setup  
**And** "The LiteLLM + LangWatch stack" and "Connecting LiteLLM to the Cloudflare AI Gateway" sections are removed from `README.md`  
**And** `model/policy.yaml` header comment matches the new setup (models and settings unchanged)

---

### AC8: Gateway tests cover new client behavior

**Given** a new test file `tests/test_gateway.py` exists  
**When** the test suite runs with environment variables set via `monkeypatch`  
**Then** the following scenarios pass:
- `ANTHROPIC_API_KEY` unset → `require_client()` and `require_skills_client()` raise `MissingAPIKeyError`
- API key set, no `ANTHROPIC_BASE_URL` → client uses the SDK default base URL
- `ANTHROPIC_BASE_URL` set → client uses it, and no `cf-aig-authorization` header is sent
- `require_client() is require_skills_client()` (identity check passes)

---

### AC9: Tracing tests updated for new implementation

**Given** `tests/test_tracing.py` exists  
**When** the test suite runs  
**Then** the two header tests (`_add_session_header`, `x-litellm-session-id`) are removed  
**And** `test_a_worker_runs_inside_its_run` still passes  
**And** a new test verifies `traceparent()` is stable for one session id and differs between different session ids  
**And** the module docstring is corrected

---

### AC10: All tests pass

**Given** all code and test changes are complete  
**When** `uv run pytest -q` is executed  
**Then** all tests pass with no failures

---

### AC11: Manual integration verification documented

**Given** the implementation is complete  
**When** the PR description is written  
**Then** it includes a manual check with a real `ANTHROPIC_API_KEY` and no LiteLLM running  
**And** the check confirms one service order completes via `run_so.py`  
**And** a skill upload via `skills/client.py` succeeds

---

## Implementation Notes

### Scope Boundaries

**In Scope:**
- `src/agents/model/gateway.py` refactor to direct Anthropic client
- Tracing variable renaming (`litellm_session_id` → `run_session_id`)
- `.env.example` updates for new configuration
- Documentation rewrite in `README.md` and deletion of Cloudflare doc
- `model/policy.yaml` header comment update
- Test file updates (`test_tracing.py` modifications, new `test_gateway.py`)

**Out of Scope:**
- Model IDs, `max_tokens`, prompts, or node logic changes
- MCP server, GraphQL, backend, frontend repos
- Redis or worker queue
- Replacing LangWatch with another tracer
- The separate `llm-gateway-observability-poc` repository

### Key Refactoring Points

1. **Single Client Instance:** `ModelGateway` creates one `anthropic.AsyncAnthropic` client, reused by both `require_client()` and `require_skills_client()`.

2. **Base URL Logic:** Pass `base_url` parameter only when `ANTHROPIC_BASE_URL` environment variable is set; otherwise omit it to use SDK default.

3. **Removed Components:**
   - `LITELLM_BASE_URL` configuration and client building
   - `ANTHROPIC_GATEWAY_TOKEN` and `cf-aig-authorization` header
   - `_add_session_header` hook and event_hooks setup
   - `_build_skills_client()` separate method
   - `"litellm"` fallback key in `__init__`

4. **Preserved Components:**
   - `session_id` ContextVar (relabeled)
   - `traceparent()` and `trace_id()` functions
   - Method names `require_client()` and `require_skills_client()` for backward compatibility
   - All existing callers in `skills/client.py`, `tools/bindings.py`, `runtime/runner.py`, `queue/pool.py`, and `queue/subtasks.py`

5. **Error Handling:** `MissingAPIKeyError` with explicit message when `ANTHROPIC_API_KEY` is unset; existing exception handling in callers remains unchanged.

6. **Tracing Comments:** Update inline comments in `runtime/runner.py` at lines 131 and 154 to reflect that LangWatch (not LiteLLM) handles tracing.

---

## Verification Approach

- **Static Analysis:** `grep` command verifies complete removal of LiteLLM/Cloudflare references
- **Unit Tests:** New `test_gateway.py` covers all client initialization scenarios with mocked environment variables
- **Existing Tests:** Modified `test_tracing.py` ensures tracing functions still work correctly
- **Integration Test:** Manual verification with real API key confirms end-to-end functionality
- **Test Suite:** `uv run pytest -q` must pass completely

---

## Dependencies

- **Anthropic SDK:** Already in use; no new dependencies required
- **LangWatch:** Already configured; becomes primary tracer with `LANGWATCH_DISABLED=false`
- **Environment Variables:** Requires `ANTHROPIC_API_KEY`; optionally accepts `ANTHROPIC_BASE_URL`
