# Requirements — PER-64

**Ticket:** Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly

**Goal:** Direct Anthropic API calls for model traffic and skill uploads, removing LiteLLM and Cloudflare intermediaries.

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

---

### AC-2: Configuration and documentation updated

**Given** the updated configuration and documentation files  
**When** reviewing `.env.example`, `runtime/runner.py`, `model/policy.yaml`, `README.md`, and `docs/`  
**Then** all of the following are true:
- `.env.example` contains required `ANTHROPIC_API_KEY=` and optional (commented) `ANTHROPIC_BASE_URL=`, with `LANGWATCH_DISABLED=false` and `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL` documented
- `.env.example` does not contain `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, or Cloudflare gateway URL references
- `runtime/runner.py` `_config()` comment references LangWatch instead of LiteLLM as the tracer
- `model/policy.yaml` header comment describes direct Anthropic configuration
- `docs/claude-via-litellm-cloudflare.md` is deleted
- `README.md` LiteLLM/Cloudflare sections are removed or rewritten to describe direct Anthropic setup

---

### AC-3: No LiteLLM or Cloudflare references remain in codebase

**Given** the completed changes  
**When** running `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns no matches

---

### AC-4: Gateway client behavior validated by automated tests

**Given** the new `tests/test_gateway.py`  
**When** running the test suite  
**Then** all 6 test cases pass, verifying:
- `require_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- `require_skills_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- Client uses default Anthropic base URL when `ANTHROPIC_BASE_URL` is not set
- Client uses `ANTHROPIC_BASE_URL` when set
- No custom auth headers (`cf-aig-authorization`, `x-litellm-session-id`) are added
- `require_client()` and `require_skills_client()` return the same client instance

---

### AC-5: Tracing tests rewritten to verify session-based behavior

**Given** the rewritten `tests/test_tracing.py`  
**When** running the test suite  
**Then** all 3 test cases pass, verifying:
- `traceparent()` is stable per session id
- `traceparent()` differs between session ids
- A worker runs inside its run's `session_id`  
**And** no assertions for `x-litellm-session-id` or `cf-aig-authorization` headers remain

---

### AC-6: All existing and new automated tests pass

**Given** the changes to gateway, config, and tests  
**When** running `uv run pytest -q`  
**Then** all tests pass with no failures or errors  
**And** the test suite includes 9 new tests: 6 in `tests/test_gateway.py` and 3 rewritten in `tests/test_tracing.py`

---

### AC-7: Manual validation of direct Anthropic integration

**Given** a real `ANTHROPIC_API_KEY` and no LiteLLM process running  
**When** running `run_so.py` to complete one service order  
**Then** the service order completes successfully  
**And when** a skill upload is triggered via `skills/client.py`  
**Then** the upload succeeds  
**And** results are documented in PR description

---

## Non-Goals

- Changing models, max_tokens, or prompts
- Modifying MCP server, GraphQL, backend, or frontend repositories
- Replacing LangWatch with another tracer

---

## References

- Ticket context: All model traffic currently routes through LiteLLM → Cloudflare AI Gateway → Anthropic
- Config files: `src/agents/model/gateway.py` (referenced in ticket), `.env.example`, `model/policy.yaml`
- Documentation: `docs/claude-via-litellm-cloudflare.md` (to be deleted), `README.md` (to be updated)
- Test files: `tests/test_gateway.py` (new), `tests/test_tracing.py` (rewritten)
- Repo structure: Paths established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`; repo-tree incomplete

---

## Notes

- **AC structure maps to breakdown**: AC-1 covers Story 1 (gateway refactor), AC-2 covers Story 2 (config/docs), AC-3 is final validation, AC-4 and AC-5 cover Story 3 tests, AC-6 is regression check, AC-7 is manual validation
- **Test count**: 9 automated tests (6 gateway + 3 tracing) align with breakdown Story 3 verification approach and LLD test sections
- **File paths**: All paths (`src/agents/model/gateway.py`, `runtime/runner.py`, `model/policy.yaml`, `tests/test_gateway.py`, `tests/test_tracing.py`, `.env.example`, `README.md`, `docs/claude-via-litellm-cloudflare.md`) verified against ticket context and breakdown; repo-tree shows empty existing files list but paths are established by ticket
- **Concrete verification**: AC-3 grep command is executable validation of zero references remaining; AC-6 test count matches breakdown and story LLD test sections
