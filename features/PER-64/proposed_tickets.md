# Proposed tickets

## 01
- title: Refactor gateway.py to use Anthropic SDK directly
- repo: agent
- ac: Requirements AC-1 (Gateway client refactored to use Anthropic SDK directly: builds single AsyncAnthropic client from ANTHROPIC_API_KEY, passes base_url only when ANTHROPIC_BASE_URL set, require_client and require_skills_client return same instance, raises MissingAPIKeyError when key unset, removes LITELLM_BASE_URL/ANTHROPIC_GATEWAY_TOKEN/cf-aig-authorization/x-litellm-session-id/request hooks, renames session_id ContextVar label to run_session_id, updates module docstring), Requirements AC-6 (partial: existing tests pass after gateway refactor)

## 02
- title: Update configuration, tracing references, and documentation
- repo: agent
- ac: Requirements AC-2 (Configuration and documentation updated: .env.example contains required ANTHROPIC_API_KEY and optional ANTHROPIC_BASE_URL with LANGWATCH_DISABLED=false, removes LITELLM_BASE_URL/ANTHROPIC_GATEWAY_TOKEN/Cloudflare references, src/agents/runtime/runner.py _config comment references LangWatch not LiteLLM, src/agents/model/policy.yaml header describes direct Anthropic config, docs/claude-via-litellm-cloudflare.md deleted, README.md LiteLLM/Cloudflare sections removed or rewritten), Requirements AC-6 (partial: existing tests pass after config/docs updates)

## 03
- title: Add gateway tests and rewrite tracing tests
- repo: agent
- ac: Requirements AC-3 (No LiteLLM or Cloudflare references remain: grep returns zero matches across src tests docs README.md .env.example), Requirements AC-4 (Gateway client behavior validated by 6 automated tests: require_client raises MissingAPIKeyError when ANTHROPIC_API_KEY unset, require_skills_client raises MissingAPIKeyError when ANTHROPIC_API_KEY unset, client uses default Anthropic base URL when ANTHROPIC_BASE_URL unset, client uses ANTHROPIC_BASE_URL when set, no custom auth headers cf-aig-authorization or x-litellm-session-id added, require_client and require_skills_client return same instance), Requirements AC-5 (Tracing tests rewritten to verify session-based behavior with 3 tests: traceparent stable per session id, traceparent differs between session ids, worker runs inside run session_id; no x-litellm-session-id or cf-aig-authorization header assertions remain), Requirements AC-6 (All existing and new automated tests pass: 9 new tests total with 6 in tests/test_gateway.py and 3 in rewritten tests/test_tracing.py, uv run pytest -q exits 0), Requirements AC-7 (Manual validation of direct Anthropic integration: run_so.py completes one service order with real ANTHROPIC_API_KEY and no LiteLLM running, skill upload via src/agents/skills/client.py succeeds, results documented in PR description)
