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

**Covered by:** Story 1 (primary: gateway refactor), Story 3 (validates via 6 gateway tests)

---

### AC-2: Configuration and documentation updated

**Given** the updated configuration and documentation files  
**When** reviewing `.env.example`, `src/agents/runtime/runner.py`, `src/agents/model/policy.yaml`, `README.md`, and `docs/`  
**Then** all of the following are true:
- `.env.example` contains required `ANTHROPIC_API_KEY=` and optional (commented) `ANTHROPIC_BASE_URL=`, with `LANGWATCH_DISABLED=false` and `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL` documented
- `.env.example` does not contain `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, or Cloudflare gateway URL references
- `src/agents/runtime/runner.py` `_config()` comment references LangWatch instead of LiteLLM as the tracer
- `src/agents/model/policy.yaml` header comment describes direct Anthropic configuration
- `docs/claude-via-litellm-cloudflare.md` is deleted
- `README.md` LiteLLM/Cloudflare sections are removed or rewritten to describe direct Anthropic setup

**Covered by:** Story 2 (primary: config/docs updates), Story 3 (validates via grep check)

---

### AC-3: No LiteLLM or Cloudflare references remain in codebase

**Given** the completed changes  
**When** running `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns no matches

**Covered by:** Story 1 (partial: gateway cleanup), Story 2 (partial: config/docs cleanup), Story 3 (primary: final grep validation)

---

### AC-4: Gateway client behavior validated by automated tests

**Given** the new `tests/test_gateway.py`  
**When** running the test suite  
**Then** all 6 test cases pass, verifying:
- Test 1: `require_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- Test 2: `require_skills_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- Test 3: Client uses default Anthropic base URL when `ANTHROPIC_BASE_URL` is not set
- Test 4: Client uses `ANTHROPIC_BASE_URL` when set
- Test 5: No custom auth headers (`cf-aig-authorization`, `x-litellm-session-id`) are added
- Test 6: `require_client()` and `require_skills_client()` return the same client instance

**Covered by:** Story 1 (partial: provides gateway implementation to test), Story 3 (primary: adds 6 gateway tests)

---

### AC-5: Tracing tests rewritten to verify session-based behavior

**Given** the rewritten `tests/test_tracing.py`  
**When** running the test suite  
**Then** all 3 test cases pass, verifying:
- Test 1: `traceparent()` is stable per session id
- Test 2: `traceparent()` differs between session ids
- Test 3: A worker runs inside its run's `session_id`  
**And** no assertions for `x-litellm-session-id` or `cf-aig-authorization` headers remain

**Covered by:** Story 1 (partial: preserves session tracking for tests to validate), Story 3 (primary: rewrites 3 tracing tests)

---

### AC-6: All existing and new automated tests pass

**Given** the changes to gateway, config, and tests  
**When** running `uv run pytest -q`  
**Then** all tests pass with no failures or errors  
**And** the test suite includes 9 new tests: 6 in `tests/test_gateway.py` and 3 rewritten in `tests/test_tracing.py`

**Covered by:** Story 1 (validates no regressions from gateway refactor), Story 2 (validates no regressions from config/docs changes), Story 3 (adds 9 new tests and validates full suite passes)

---

### AC-7: Manual validation of direct Anthropic integration

**Given** a real `ANTHROPIC_API_KEY` and no LiteLLM process running  
**When** running `run_so.py` to complete one service order  
**Then** the service order completes successfully  
**And when** a skill upload is triggered via `src/agents/skills/client.py`  
**Then** the upload succeeds  
**And** results are documented in PR description

**Covered by:** Story 1 (partial: implements gateway for validation), Story 2 (partial: updates config for validation), Story 3 (primary: documents manual validation in PR)

---

## Non-Goals

- Changing models, max_tokens, or prompts
- Modifying MCP server, GraphQL, backend, or frontend repositories
- Replacing LangWatch with another tracer

---

## References

- Ticket context: All model traffic currently routes through LiteLLM → Cloudflare AI Gateway → Anthropic
- Config files: `src/agents/model/gateway.py` (repo-tree confirmed), `.env.example`, `src/agents/model/policy.yaml` (repo-tree confirmed)
- Documentation: `docs/claude-via-litellm-cloudflare.md` (to be deleted), `README.md` (repo-tree confirmed)
- Test files: `tests/test_gateway.py` (new), `tests/test_tracing.py` (rewritten)
- Repo structure: Root at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents` per repo-tree
- Real file scan (per change_requests.md): `src/agents/model/gateway.py` (repo-tree confirmed), `src/agents/skills/client.py` (repo-tree confirmed), `src/agents/runtime/runner.py` (repo-tree confirmed), `tests/test_tracing.py` (not shown in repo-tree but referenced in ticket)

---

## Notes

- **AC-to-story mapping**: AC-1 → Story 1 (primary) + Story 3 (validation); AC-2 → Story 2 (primary) + Story 3 (validation); AC-3 → Stories 1+2 (partial cleanup) + Story 3 (final validation); AC-4 → Story 1 (provides implementation) + Story 3 (adds 6 tests); AC-5 → Story 1 (preserves tracking) + Story 3 (adds 3 tests); AC-6 → Stories 1+2 (regression checks) + Story 3 (adds 9 tests, validates suite); AC-7 → Stories 1+2 (implement/configure) + Story 3 (documents validation)
- **Test count alignment**: 9 automated tests total (6 gateway in AC-4 + 3 tracing in AC-5 = 9 in AC-6); Story 1 adds 0 tests, Story 2 adds 0 tests, Story 3 adds 9 tests (sum: 0+0+9=9 ✓)
- **File paths verification**: All modified paths (`src/agents/model/gateway.py`, `src/agents/runtime/runner.py`, `src/agents/skills/client.py`, `README.md`, `src/agents/model/policy.yaml`) confirmed in repo-tree at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents`; new/deleted paths (`tests/test_gateway.py`, `tests/test_tracing.py`, `.env.example`, `docs/claude-via-litellm-cloudflare.md`) established by ticket context
- **Concrete verification**: AC-3 grep command (`grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`) is executable validation returning no matches; AC-4 and AC-5 enumerate 6+3=9 specific test cases matching Story 3 LLD Tests sections; AC-6 test count (9 new) matches breakdown Story 3 test count
- **Real file details**: Per change_requests.md requirement to re-read real files — `src/agents/model/gateway.py` exists in repo-tree, current implementation details (singleton pattern, ContextVar label, LiteLLM/Cloudflare setup) described in HLD/breakdown based on ticket context stating "wired in `src/agents/model/gateway.py`"; `tests/test_tracing.py` not shown in repo-tree but ticket context describes header assertions to be removed
