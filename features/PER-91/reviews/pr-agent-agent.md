```json
[
  {
    "path": "src/agents/model/gateway.py",
    "line": 50,
    "body": "**AC4 violation: `traceparent()` signature breaking change** — The function now requires a session id argument or raises `ValueError` when none is set, but AC4 says `runtime/runner.py`, `queue/pool.py`, and `queue/subtasks.py` continue using these functions unchanged. The old signature `traceparent(sid: str)` was mandatory-arg; the new `traceparent(sid: str | None = None)` is optional but raises when called with no arg and no context set. Check all call sites: if any pass no argument and expect the context default, they will break when context is unset (e.g. during initialization or in test fixtures that don't set `session_id`)."
  },
  {
    "path": "tests/test_tracing.py",
    "line": 22,
    "body": "**AC9 violation: missing test for `traceparent()` stability with explicit session id** — AC9 requires 'a new test verifies `traceparent()` is stable for one session id and differs between different session ids'. The new test `test_traceparent_stable_for_session` only tests the context-var path (`session_id.set()` then `traceparent()` with no arg). The refactored `traceparent(sid: str | None = None)` also accepts an explicit session id, and `trace_id(sid: str)` still calls `traceparent(sid)` with one. Add a case proving `traceparent('session-1') == traceparent('session-1')` and `traceparent('session-1') != traceparent('session-2')` to cover the explicit-arg contract."
  },
  {
    "path": "src/agents/model/gateway.py",
    "line": 65,
    "body": "**Potential bug: `trace_id(sid: str)` signature is mandatory but `traceparent(sid: str | None = None)` now requires explicit check** — `trace_id` still takes a mandatory `sid` and calls `traceparent(sid)`, which is valid. However, if a caller mistakenly passes `None` to `trace_id(None)` (type system allows `str | None` in some contexts), `traceparent(None)` will hit the `if not sid: raise ValueError` branch instead of deriving a trace id. This is a runtime trap. Either make `trace_id` require `str` (already does, so callers are type-checked) or add an explicit `if sid is None` guard in `trace_id` before calling `traceparent`, depending on whether `None` is a valid input (AC4 says 'run results continue returning `trace_id`', implying it always has a session id). If `None` is never valid, the current code is safe; if it might be, add a guard."
  },
  {
    "path": "tests/test_gateway.py",
    "line": 12,
    "body": "**AC8 violation: test calls `gateway.require_client()` after construction, but `MissingAPIKeyError` is raised in `__init__`** — The test `test_missing_api_key_raises_error` constructs `ModelGateway()` then calls `require_client()`, but the refactored gateway raises `MissingAPIKeyError` in `__init__` when `ANTHROPIC_API_KEY` is unset (line 66 of `gateway.py`). The exception is raised before `require_client()` is ever invoked. The test will fail because `ModelGateway()` itself raises, not `require_client()`. Move the `with pytest.raises(...)` block to wrap `ModelGateway()` construction, or re-test `require_client()` on an already-constructed gateway (which is a no-op if construction succeeded). AC8 says 'API key unset → `require_client()` and `require_skills_client()` raise `MissingAPIKeyError`', but the implementation raises earlier. Either the test or the contract is wrong."
  },
  {
    "path": "tests/test_gateway.py",
    "line": 30,
    "body": "**AC8 gap: missing test for 'no `cf-aig-authorization` header is sent'** — AC8 requires verifying that when `ANTHROPIC_BASE_URL` is set, 'no `cf-aig-authorization` header is sent'. `test_custom_base_url_when_set` asserts `not any(name.lower().startswith('cf-') for name in client.default_headers)`, which checks `default_headers` on the client object. However, the old implementation set `cf-aig-authorization` in `default_headers={'cf-aig-authorization': ...}` (line 86 of the diff's removed code), so the check is valid. Confirm that `client.default_headers` is the correct attribute to inspect (it is per the Anthropic SDK), and that the test covers the AC. If `default_headers` only includes explicitly-set headers (not SDK defaults), the test is sufficient. Otherwise, verify no CF header exists in the actual request headers (may require mocking the HTTP client or inspecting an actual request, which is beyond unit-test scope). The test as written is reasonable for unit scope."
  },
  {
    "path": "README.md",
    "line": 51,
    "body": "**AC7 clarification: 'Model calls go directly to the Anthropic API' — does this cover MCP tool calls?** — The updated README says 'Chat, the MCP tool loop, running Skills, and code execution all use one client'. AC7 requires 'README.md model-route and observability sections are rewritten for direct Anthropic calls'. The original README said 'MCP tool calls are visible as `tool_use` / `tool_result` blocks inside the LLM messages. They are not separate spans, because the agents call the MCP server directly, not through LiteLLM.' The new text implies MCP tools still go through the same client (correct: agents call MCP locally, then send results to Claude via the client). Confirm the updated text preserves the 'MCP is local, not a separate span' semantics. The new text is less explicit but not wrong."
  },
  {
    "path": ".env.example",
    "line": 13,
    "body": "**AC6 violation: `LANGWATCH_DISABLED=false` should be the default, but documentation inconsistency** — AC6 says 'LANGWATCH_DISABLED=false is set', and the diff sets it to `false`. However, the README line 62 says 'LangWatch tracing (enabled by default)', implying the default is enabled (DISABLED=false). But `.env.example` explicitly sets `LANGWATCH_DISABLED=false`, so when the var is unset, what is the default? The code in `runtime/runner.py` line 131 checks `os.getenv('LANGWATCH_DISABLED', 'false').lower() != 'true'`, so the default when unset is `'false'` (enabled). The `.env.example` redundantly sets it to the default. This is not a bug, but AC5 says 'LANGWATCH_DISABLED=false is set' (explicit), and the code default matches. The README is correct: 'enabled by default' means DISABLED defaults to false. No issue, just confirming alignment."
  },
  {
    "path": ".gitignore",
    "line": 23,
    "body": "**Out-of-scope change: ADLC/Cursor tooling entries added to `.gitignore`** — AC1 requires removing LiteLLM and Cloudflare references, and AC6 updates environment configuration. The diff adds 15 lines to `.gitignore` for ADLC, Cursor, Python artifacts, and virtual environments. These are unrelated to PER-91's scope (model gateway refactor) and were not mentioned in any AC. While harmless, they dilute the PR's focus. If this is intentional cleanup, document it in the PR description (AC11 requires a manual check narrative, which could note this). If accidental, remove them or split into a separate commit."
  },
  {
    "path": "src/agents/model/gateway.py",
    "line": 82,
    "body": "**AC2 edge case: `require_client()` docstring says 'same missing-config guard' but no longer applicable** — The old docstring said '`require_client()` has the same missing-config guard as `complete()`, so callers get one consistent failure mode either way.' The refactored gateway raises `MissingAPIKeyError` in `__init__`, so both `require_client()` and `complete()` fail at construction time if the key is missing. The docstring now says 'Raw client access for callers that need more than `complete()`', which is accurate but no longer mentions the failure mode. This is fine (the behavior is correct), but the old docstring's rationale ('one consistent failure mode') is lost. Not a bug, just a documentation evolution."
  },
  {
    "path": "tests/test_gateway.py",
    "line": 1,
    "body": "**AC8 missing: no test for 'client uses the SDK default base URL when `ANTHROPIC_BASE_URL` is not set'** — `test_default_base_url_when_not_set` asserts `client.base_url == 'https://api.anthropic.com'`, which hardcodes the SDK's default. If the Anthropic SDK changes its default URL in a future version, the test will break even though the gateway correctly uses 'the SDK default'. A more robust test would check that `base_url` was not explicitly passed to the client (i.e. the gateway's `kwargs` dict had no `'base_url'` key), or mock the SDK to verify it was called without that argument. However, the test as written satisfies AC8's literal requirement ('client uses the SDK default base URL'), and refactoring it to be SDK-version-proof is beyond the ticket's scope. Acceptable as-is, but a future improvement."
  }
]
```
