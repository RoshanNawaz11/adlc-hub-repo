# Requirements — PER-64

**Ticket:** Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly

**Goal:** Direct Anthropic API calls for model traffic and skill uploads, removing LiteLLM and Cloudflare intermediaries.

---

## Acceptance Criteria

### AC-1: No LiteLLM or Cloudflare references remain

**Given** the codebase after this change  
**When** running `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`  
**Then** the command returns no matches

---

### AC-2: Tracing tests rewritten to verify session-based traceparent behavior

**Given** the rewritten `tests/test_tracing.py`  
**When** running the test suite  
**Then** all tests pass, verifying:
- `traceparent()` is stable per session id
- `traceparent()` differs between session ids
- A worker runs inside its run's `session_id`

---

### AC-3: Gateway client behavior validated

**Given** the new `tests/test_gateway.py`  
**When** running the test suite  
**Then** all tests pass, verifying:
- `require_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- `require_skills_client()` raises `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset
- Client uses default Anthropic base URL when `ANTHROPIC_BASE_URL` is not set
- Client uses `ANTHROPIC_BASE_URL` when set
- No custom auth headers are added
- `require_client()` and `require_skills_client()` return the same client instance

---

### AC-4: All automated tests pass

**Given** the changes to gateway, config, and tests  
**When** running `uv run pytest -q`  
**Then** all tests pass with no failures or errors

---

### AC-5: Manual validation of direct Anthropic integration

**Given** a real `ANTHROPIC_API_KEY` and no LiteLLM process running  
**When** running `run_so.py` to complete one service order  
**Then** the service order completes successfully  
**And when** a skill upload is triggered via `skills/client.py`  
**Then** the upload succeeds

---

## Non-Goals

- Changing models, max_tokens, or prompts
- Modifying MCP server, GraphQL, backend, or frontend repositories
- Replacing LangWatch with another tracer

---

## References

- Ticket context: All model traffic currently routes through LiteLLM → Cloudflare AI Gateway → Anthropic
- Config files: `src/agents/model/gateway.py`, `.env.example`, `model/policy.yaml`
- Documentation: `docs/claude-via-litellm-cloudflare.md` (to be deleted), `README.md` (to be updated)
