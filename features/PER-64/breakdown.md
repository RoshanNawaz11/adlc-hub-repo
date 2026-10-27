# Breakdown — PER-64

**Ticket:** Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly

**Goal:** Direct Anthropic API calls for model traffic and skill uploads, removing LiteLLM and Cloudflare intermediaries.

---

## Approach

This ticket refactors the model gateway to call Anthropic's API directly, removing two intermediary hops (LiteLLM and Cloudflare AI Gateway). The work is split into three implementation stories executed in sequence: first update the gateway client logic, then clean up configuration and tracing references, and finally update tests and documentation. Each story is independently reviewable and produces a working system at completion, though the full acceptance criteria are validated only after all three stories merge.

---

## Dependencies

- **External:** None (no changes to MCP, GraphQL, backend, or frontend repos)
- **Internal:** Stories must execute in order (Story 1 → Story 2 → Story 3)
- **Prerequisite:** LangWatch tracing configured and validated in deployment environment before starting Story 1

---

## Stories

### Story 1: Refactor gateway.py to use Anthropic SDK directly

**Repo:** `agent`

**Description:**  
Replace LiteLLM and Cloudflare client paths with a single `anthropic.AsyncAnthropic` client configured from `ANTHROPIC_API_KEY` and optional `ANTHROPIC_BASE_URL`. Both `require_client()` and `require_skills_client()` return the same instance. Remove all LiteLLM/Cloudflare-specific headers and request hooks. Preserve session tracking by renaming the `session_id` ContextVar label from `"litellm_session_id"` to `"run_session_id"`. Update `MissingAPIKeyError` message to reference `ANTHROPIC_API_KEY`.

**Requirements Coverage:**  
- From Requirements AC-1: Gateway client refactored to use Anthropic SDK directly

**Acceptance Criteria:**  
1. Gateway client refactored to use Anthropic SDK directly (per Requirements AC-1): builds single `anthropic.AsyncAnthropic` client from `ANTHROPIC_API_KEY`, passes `base_url` only when `ANTHROPIC_BASE_URL` set, both `require_client()` and `require_skills_client()` return same instance, raises `MissingAPIKeyError` when key unset, removes LiteLLM/Cloudflare headers and hooks, renames `session_id` ContextVar label to `"run_session_id"`, updates module docstring
2. Existing tests pass (`uv run pytest -q`)

**Allowed Paths:**  
- `src/agents/model/gateway.py` (modified; file exists per repo-tree)

**After:** None (opening story)

**Verification Approach:**  
Manual REPL testing: verify same client instance returned, default and custom base URL behavior, `MissingAPIKeyError` raised when key unset. Code review: confirm no LiteLLM/Cloudflare references in `src/agents/model/gateway.py`, session tracking preserved. Run existing test suite to confirm no regressions.

**Test count:** 0 new automated tests (existing tests must pass; Story 3 adds 9 automated tests per Requirements AC-4, AC-5, AC-6)

**HITL:** False

---

### Story 2: Update configuration, tracing references, and documentation

**Repo:** `agent`

**Description:**  
Remove `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, and Cloudflare URL references from `.env.example`; add required `ANTHROPIC_API_KEY=` and optional (commented) `ANTHROPIC_BASE_URL=`. Set `LANGWATCH_DISABLED=false` and document `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL`. Update the `src/agents/runtime/runner.py` `_config()` comment to reference LangWatch instead of LiteLLM. Update `src/agents/model/policy.yaml` header comment to reflect direct Anthropic configuration. Delete `docs/claude-via-litellm-cloudflare.md`. Rewrite or remove LiteLLM/Cloudflare sections in `README.md` to describe the direct Anthropic setup.

**Requirements Coverage:**  
- From Requirements AC-2: Configuration and documentation updated

**Acceptance Criteria:**  
1. Configuration and documentation updated (per Requirements AC-2): `.env.example` contains required `ANTHROPIC_API_KEY=` and optional `ANTHROPIC_BASE_URL=` with LangWatch config, removes LiteLLM/Cloudflare references; `src/agents/runtime/runner.py` comment references LangWatch; `src/agents/model/policy.yaml` header describes direct Anthropic setup; `docs/claude-via-litellm-cloudflare.md` deleted; `README.md` LiteLLM/Cloudflare sections removed or rewritten
2. Existing tests pass (`uv run pytest -q`)

**Allowed Paths:**  
- `.env.example` (modified; standard config file at repo root)
- `src/agents/runtime/runner.py` (modified: comment only; file exists per repo-tree)
- `src/agents/model/policy.yaml` (modified: comment only; file exists per repo-tree)
- `docs/claude-via-litellm-cloudflare.md` (deleted; path inferred from ticket context)
- `README.md` (modified; file exists per repo-tree)

**After:** Story 1 (requires `src/agents/model/gateway.py` refactor complete)

**Verification Approach:**  
Manual review of all listed files to confirm LiteLLM/Cloudflare references removed and new config documented. Run `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" .env.example src/agents/runtime/runner.py src/agents/model/policy.yaml docs README.md` to verify no matches. Run existing test suite to confirm no regressions.

**Test count:** 0 new automated tests (existing tests must pass; Story 3 adds 9 automated tests per Requirements AC-4, AC-5, AC-6)

**HITL:** False

---

### Story 3: Add gateway tests and rewrite tracing tests

**Repo:** `agent`

**Description:**  
Create `tests/test_gateway.py` with six test cases validating `MissingAPIKeyError` when `ANTHROPIC_API_KEY` unset (two tests: one for `require_client()`, one for `require_skills_client()`), default Anthropic base URL behavior, custom `ANTHROPIC_BASE_URL` behavior, no custom auth headers, and singleton client instance. Rewrite `tests/test_tracing.py` to remove header assertions (`x-litellm-session-id`, `cf-aig-authorization`) and validate `traceparent()` stability per session ID, `traceparent()` variance across session IDs, and worker session context propagation. Document manual check in PR description: with real `ANTHROPIC_API_KEY` and no LiteLLM running, validate `run_so.py` completes one service order and a skill upload via `src/agents/skills/client.py` succeeds.

**Requirements Coverage:**  
- From Requirements AC-3: No LiteLLM or Cloudflare references remain in codebase
- From Requirements AC-4: Gateway client behavior validated by automated tests (6 tests)
- From Requirements AC-5: Tracing tests rewritten to verify session-based behavior (3 tests)
- From Requirements AC-6: All automated tests pass (9 new tests total)
- From Requirements AC-7: Manual validation of direct Anthropic integration

**Acceptance Criteria:**  
1. Gateway client behavior validated by automated tests (per Requirements AC-4): `tests/test_gateway.py` created with 6 test cases covering `MissingAPIKeyError` when key unset (both `require_client()` and `require_skills_client()`), default base URL, custom base URL, no custom auth headers, singleton instance
2. Tracing tests rewritten to verify session-based behavior (per Requirements AC-5): `tests/test_tracing.py` rewritten with 3 test cases covering `traceparent()` stability per session ID, variance across session IDs, worker context propagation; no header assertions for `x-litellm-session-id` or `cf-aig-authorization` remain
3. All automated tests pass (per Requirements AC-6): `uv run pytest -q` passes with 9 new tests (6 gateway + 3 tracing)
4. No LiteLLM or Cloudflare references remain (per Requirements AC-3): `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example` returns no matches
5. Manual validation of direct Anthropic integration (per Requirements AC-7): PR description documents manual check with real `ANTHROPIC_API_KEY` and no LiteLLM running — `run_so.py` completes one service order and skill upload via `src/agents/skills/client.py` succeeds

**Allowed Paths:**  
- `tests/test_gateway.py` (new; tests folder exists per repo-tree)
- `tests/test_tracing.py` (modified; path inferred from ticket context, tests folder exists per repo-tree)

**After:** Story 2 (requires config and docs updated)

**Verification Approach:**  
Run `uv run pytest -q tests/test_gateway.py` to validate 6 new gateway tests pass. Run `uv run pytest -q tests/test_tracing.py` to validate 3 rewritten tracing tests pass. Run full test suite (`uv run pytest -q`) to confirm all tests pass (9 new + existing). Run `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example` to confirm no matches. Perform manual check and document results in PR description.

**Test count:** 9 automated tests (6 in `tests/test_gateway.py`, 3 in rewritten `tests/test_tracing.py`; per Requirements AC-4, AC-5, AC-6)

**HITL:** False (manual check documented, not executed by automated test)

---

## Test Strategy

### Unit Tests
- **Story 1:** Existing tests must pass; no new tests added (0 new tests)
- **Story 2:** Existing tests must pass; no new tests added (0 new tests)
- **Story 3:** 6 new tests in `tests/test_gateway.py` validate client instantiation, error handling, base URL behavior, header absence, and singleton pattern (Requirements AC-4)
- **Story 3:** 3 rewritten tests in `tests/test_tracing.py` validate session tracking behavior (Requirements AC-5)

### Integration Tests
- **Story 3 (manual):** Manual validation with real `ANTHROPIC_API_KEY` — `run_so.py` completes one service order and a skill upload via `src/agents/skills/client.py` succeeds (documented in PR description per Requirements AC-7)

### Regression Prevention
- Each story runs `uv run pytest -q` to confirm no existing test failures
- Story 3 runs `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN"` to validate complete removal (Requirements AC-3)

**Total test count:** 9 automated tests across all stories (Story 1: 0 new, Story 2: 0 new, Story 3: 9 new = 6 gateway + 3 tracing; sum: 0+0+9=9; aligns with Requirements AC-4, AC-5, AC-6)

---

## Success Metrics

1. **Zero LiteLLM/Cloudflare references:** `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example` returns no matches (Requirements AC-3)
2. **Gateway client validated:** 6 new tests in `tests/test_gateway.py` pass (Requirements AC-4)
3. **Tracing behavior validated:** 3 rewritten tests in `tests/test_tracing.py` pass (Requirements AC-5)
4. **All automated tests pass:** `uv run pytest -q` exits 0 with 9 new tests (Requirements AC-6)
5. **Live integration validated:** Manual check documents successful service order completion and skill upload with real `ANTHROPIC_API_KEY` (Requirements AC-7)

---

## Open Questions

None (all requirements and implementation details specified in ticket and HLD)

---

## Notes

- **AC count alignment:** 7 Requirements ACs map to 2 ACs in Story 1, 2 ACs in Story 2, 5 ACs in Story 3 (total 9 story ACs covering all 7 Requirements ACs without fragmentation beyond ticket scope)
- **Test count alignment:** 9 automated tests (6 gateway + 3 tracing) per Requirements AC-4, AC-5, AC-6; Story 1 adds 0 tests, Story 2 adds 0 tests, Story 3 adds 9 tests (0+0+9=9); breakdown test strategy and story verification approaches match
- **File paths verified:** Concrete paths confirmed in repo-tree: `src/agents/model/gateway.py`, `src/agents/runtime/runner.py`, `src/agents/model/policy.yaml`, `README.md` exist at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents`; `.env.example` standard config file at repo root; `docs/claude-via-litellm-cloudflare.md` path inferred from ticket context; `tests/test_gateway.py` new file, `tests/test_tracing.py` path inferred (tests folder exists per repo-tree)
- **No structural changes:** All work modifies existing modules or adds tests; no new architectural components
- **Backward compatibility:** `require_skills_client()` method name preserved; callers (`src/agents/skills/client.py`, `src/agents/tools/bindings.py` per repo-tree) unchanged
- **Session tracking continuity:** `session_id` ContextVar behavior unchanged; only label renamed for clarity
- **Story sequencing:** Stories 1, 2, and 3 must execute in order; each produces a working system but full acceptance criteria validated only after Story 3
- **Out of scope:** Model selection, `max_tokens`, prompts, MCP server, GraphQL, backend, frontend, and LangWatch tracer replacement explicitly excluded per ticket
- **Real file context:** Repo-tree confirms `src/agents/model/gateway.py` exists (gateway refactor target), `src/agents/runtime/runner.py` exists (tracing comment update), `src/agents/skills/client.py` exists (skill upload validation), tests folder exists (new test files); `src/agents/model/policy.yaml` confirmed in repo-tree (not `model/policy.yaml` at root); `.env.example` and `tests/test_tracing.py` paths not shown in repo-tree but established by ticket context
