# HLD — PER-64

**Ticket:** Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly

**Goal:** Direct Anthropic API calls for model traffic and skill uploads, removing LiteLLM and Cloudflare intermediaries.

---

## Components

### 1. ModelGateway (`src/agents/model/gateway.py`)

**Current state:**
- Maintains separate client paths: model calls → LiteLLM (`LITELLM_BASE_URL`), skill uploads → Cloudflare AI Gateway (`ANTHROPIC_BASE_URL` + `ANTHROPIC_GATEWAY_TOKEN`)
- Injects session tracking via `x-litellm-session-id` and `cf-aig-authorization` headers through `_add_session_header` request hook
- Wraps `anthropic.AsyncAnthropic` with custom request event handlers

**Target state:**
- Single `anthropic.AsyncAnthropic` client for both model calls and skill uploads
- Configured from `ANTHROPIC_API_KEY` (required) and optional `ANTHROPIC_BASE_URL`
- No LiteLLM/Cloudflare-specific headers or request hooks
- Session tracking preserved via `session_id` ContextVar (label renamed from "litellm_session_id" to "run_session_id")
- Client instance is singleton: module-level `_client` variable cached on first call
- Both `require_client()` and `require_skills_client()` delegate to internal `_get_or_create_client()` helper

**Responsibilities:**
- Validate `ANTHROPIC_API_KEY` presence; raise `MissingAPIKeyError` (message: "ANTHROPIC_API_KEY environment variable required") when unset
- Build client configured from environment variables (pass `base_url` kwarg only when `ANTHROPIC_BASE_URL` is set)
- Provide `require_client()` and `require_skills_client()` (both return same instance)
- Maintain `session_id`, `traceparent()`, and `trace_id()` for downstream tracing (no logic changes to these functions)

---

### 2. Configuration (`.env.example`, `src/agents/model/policy.yaml`)

**Current state:**
- `.env.example` defines `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, `ANTHROPIC_BASE_URL` (pointing to Cloudflare gateway)
- `src/agents/model/policy.yaml` header comment references LiteLLM → Cloudflare → Anthropic routing

**Target state:**
- `.env.example`: `ANTHROPIC_API_KEY=` (required, uncommented), optional `# ANTHROPIC_BASE_URL=` (commented with guidance)
- `LANGWATCH_DISABLED=false` with `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL` documented
- `src/agents/model/policy.yaml` header updated to reflect direct Anthropic setup
- All `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, and Cloudflare URL references removed

---

### 3. Tracing (`src/agents/observability/`, `src/agents/runtime/runner.py`)

**Current state:**
- LiteLLM acts as tracer while `LANGWATCH_DISABLED=true`
- Comment in `src/agents/runtime/runner.py` `_config()` method attributes tracing to LiteLLM

**Target state:**
- Agent-side LangWatch tracing (`src/agents/observability/langwatch_setup.py`) becomes active tracer
- `src/agents/runtime/runner.py` comment updated to reference LangWatch instead of LiteLLM
- Session-based tracing (`session_id`, `traceparent()`, `trace_id()`) preserved for run logging (`src/agents/runtime/runner.py`), worker pool (`src/agents/queue/pool.py`, `src/agents/queue/subtasks.py`), and run results

---

### 4. Documentation (`docs/`, `README.md`)

**Current state:**
- `docs/claude-via-litellm-cloudflare.md` documents LiteLLM → Cloudflare routing
- `README.md` sections reference LiteLLM/Cloudflare setup

**Target state:**
- `docs/claude-via-litellm-cloudflare.md` deleted
- `README.md` sections rewritten to describe direct Anthropic configuration: required `ANTHROPIC_API_KEY`, optional `ANTHROPIC_BASE_URL`, LangWatch tracing enabled by default

---

### 5. Tests (`tests/test_gateway.py`, `tests/test_tracing.py`)

**Current state:**
- `tests/test_tracing.py` asserts LiteLLM-specific headers (`x-litellm-session-id`, `cf-aig-authorization`)
- No dedicated gateway client validation

**Target state:**
- New `tests/test_gateway.py` validates client behavior: 6 test cases covering error handling (2: `require_client()` and `require_skills_client()` both raise when key unset), base URL configuration (2: default and custom), header validation (1: no custom headers added), and singleton pattern (1: both methods return same instance)
- Rewritten `tests/test_tracing.py` validates session tracking behavior: 3 test cases covering `traceparent()` stability per session, variance across sessions, and worker context propagation; all header assertions removed

---

## Contracts

### ModelGateway Client API

**Input:**
- Environment: `ANTHROPIC_API_KEY` (required), `ANTHROPIC_BASE_URL` (optional)
- Session context: `session_id` ContextVar

**Output:**
- `require_client()`: Returns configured singleton client instance
- `require_skills_client()`: Returns same singleton client instance as `require_client()`
- `traceparent()`: Returns W3C traceparent header for current session (format: `00-<trace-id>-<parent-id>-<flags>`)
- `trace_id()`: Returns extracted trace ID from traceparent

**Error conditions:**
- `MissingAPIKeyError`: Raised when `ANTHROPIC_API_KEY` unset (message: "ANTHROPIC_API_KEY environment variable required")

**Behavior:**
- Client instance is singleton: module-level `_client` variable initialized to `None`, created on first call to `require_client()` or `require_skills_client()` via internal `_get_or_create_client()` helper, cached for subsequent calls
- When `ANTHROPIC_BASE_URL` is set, client configured with that base URL; otherwise uses Anthropic SDK default
- No custom HTTP headers added beyond Anthropic SDK defaults (no `cf-aig-authorization`, no `x-litellm-session-id`, no custom HTTP client or event hooks)

---

### Session Tracking

**ContextVar interface:**
- `session_id: ContextVar[str]` (label renamed to "run_session_id"; variable name unchanged)

**Consumers:**
- `src/agents/runtime/runner.py`: Logs per-run session ID in `_config()` method
- `src/agents/queue/pool.py`, `src/agents/queue/subtasks.py`: Propagates session ID to worker threads
- Run results: Includes `trace_id` for correlation

**Traceparent format:**
- W3C Trace Context standard: `00-<trace-id>-<parent-id>-<flags>`
- Stable per session ID; deterministic generation (same session → same traceparent on repeated calls)
- Differs across session IDs (different sessions → different traceparents)

---

### Configuration Schema

**`.env.example` required fields:**
```
ANTHROPIC_API_KEY=
```

**`.env.example` optional fields:**
```
# ANTHROPIC_BASE_URL=https://api.anthropic.com
# Uncomment to override default Anthropic endpoint (e.g., for proxies or testing)

LANGWATCH_DISABLED=false
LANGWATCH_API_KEY=
LANGWATCH_ENDPOINT_URL=
```

**Removed fields:**
- `LITELLM_BASE_URL`
- `ANTHROPIC_GATEWAY_TOKEN`
- Cloudflare gateway URL in `ANTHROPIC_BASE_URL` example

---

## Risks

### R-1: Breaking change for deployments using LiteLLM/Cloudflare
**Impact:** High  
**Probability:** Certain  
**Mitigation:**
- Migration documented in PR (manual check validates direct Anthropic integration per Requirements AC-7)
- `.env.example` clearly specifies new required/optional variables
- Deployment runbook updated to reference direct Anthropic setup
- Story 2 updates all configuration templates and documentation before Story 3 final validation

---

### R-2: Observability gap during transition
**Impact:** Medium  
**Probability:** Low  
**Mitigation:**
- LangWatch tracing validated before LiteLLM removal (prerequisite per breakdown dependencies)
- Session-based tracing (`session_id`, `traceparent()`) preserved; no gap in run correlation
- Rewritten `tests/test_tracing.py` validates tracing behavior pre-merge (Story 3)
- `src/agents/runtime/runner.py` comment updated to reflect LangWatch as tracer (Story 2)

---

### R-3: Skill upload regression
**Impact:** High  
**Probability:** Low  
**Mitigation:**
- Manual check validates skill upload via `src/agents/skills/client.py` before merge (Requirements AC-7, Breakdown Story 3 AC-5)
- `require_skills_client()` returns same client as `require_client()`; no divergent code paths
- Anthropic SDK handles file uploads natively; no custom logic required
- Both methods tested for singleton behavior in `tests/test_gateway.py` (Story 3)

---

### R-4: Incomplete cleanup of LiteLLM/Cloudflare references
**Impact:** Medium  
**Probability:** Low  
**Mitigation:**
- Requirements AC-3 enforces `grep` check across all source/config/docs (Story 3 AC-4)
- Story 2 removes all config/doc references before Story 3 validation
- Story 3 final `grep` validation confirms zero matches before manual check
- Code review checklist includes manual scan for orphaned config/comments

---

### R-5: Module reload issues in tests
**Impact:** Low  
**Probability:** Medium  
**Mitigation:**
- Gateway module initializes singleton at import; tests use fixture-based reset pattern
- `pytest` autouse fixture resets `gateway._client = None` before/after each test
- Prevents state leakage between tests
- Story 3 LLD documents reload strategy and fixture pattern

---

## Architecture Decisions

### AD-1: Single Anthropic client for all model traffic
**Binds:** `require_client()` and `require_skills_client()` return the same client instance  
**Prevents:** Client configuration drift between model calls and skill uploads  
**Rule:** No separate client initialization paths; all Anthropic traffic uses one configured singleton created by internal `_get_or_create_client()` helper  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-2: Optional base URL override
**Binds:** `ANTHROPIC_BASE_URL` is optional; when unset, client uses Anthropic SDK default endpoint  
**Prevents:** Hard-coded endpoint assumptions; supports testing/proxy scenarios  
**Rule:** Client configured to use custom endpoint when `ANTHROPIC_BASE_URL` is set (passed as `base_url` kwarg to client constructor)  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-3: Session tracking persists post-LiteLLM
**Binds:** `session_id` ContextVar and `traceparent()` remain unchanged in behavior  
**Prevents:** Breaking run correlation in logging, worker pool, and observability  
**Rule:** Rename ContextVar label to "run_session_id"; keep ContextVar variable name `session_id`; `traceparent()` and `trace_id()` logic unchanged  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-4: LangWatch as primary tracer
**Binds:** Agent-side LangWatch tracing enabled by default (`LANGWATCH_DISABLED=false`)  
**Prevents:** Observability regression after LiteLLM removal  
**Rule:** `.env.example` sets `LANGWATCH_DISABLED=false`; `src/agents/runtime/runner.py` comment updated to reference LangWatch instead of LiteLLM  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-5: No auth header customization
**Binds:** Anthropic client uses SDK default authentication (API key in standard header)  
**Prevents:** Leftover Cloudflare/LiteLLM header logic causing auth failures  
**Rule:** Remove `cf-aig-authorization`, `x-litellm-session-id`, and `_add_session_header` request hook; no custom HTTP client or event hooks passed to client constructor  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-6: Singleton client via module-level variable
**Binds:** Client instance cached in module-level `_client` variable (initialized to `None`)  
**Prevents:** Multiple client instantiations causing connection pool bloat or inconsistent configuration  
**Rule:** First call to `require_client()` or `require_skills_client()` triggers `_get_or_create_client()` which creates and caches client; subsequent calls return cached instance  
**Status:** [ADOPTED] (implementation detail)

---

## Deferred

- Model selection, `max_tokens`, or prompt changes (out of scope per ticket)
- MCP server, GraphQL, backend, or frontend modifications (out of scope per ticket)
- LangWatch tracer replacement (out of scope per ticket)
- Retry/timeout configuration for Anthropic client (not mentioned in requirements; inherits SDK defaults)
- Rate limiting (not mentioned in requirements; assumes Anthropic SDK handles throttling)
- Custom HTTP client configuration beyond `base_url` (not required; SDK defaults sufficient)

---

## Notes

- **File paths:** All modified files (`src/agents/model/gateway.py`, `src/agents/runtime/runner.py`, `src/agents/skills/client.py`, `README.md`) confirmed in repo-tree at `/mnt/c/trinet-adlc-agents/trinet-adlc-agents`; `src/agents/model/policy.yaml` confirmed in repo-tree; `.env.example`, `tests/test_tracing.py`, `tests/test_gateway.py` (new), and `docs/claude-via-litellm-cloudflare.md` (deleted) paths established by ticket context
- **AC alignment:** 7 Requirements ACs map to breakdown stories: AC-1 (gateway refactored) → Story 1, AC-2 (config/docs updated) → Story 2, AC-3 (no LiteLLM/Cloudflare references) → Story 3 final validation, AC-4 (6 gateway tests) → Story 3, AC-5 (3 tracing tests) → Story 3, AC-6 (all tests pass: 9 new) → Story 3, AC-7 (manual validation) → Story 3 manual check
- **Test count alignment:** 9 automated tests total (6 gateway + 3 tracing) per Requirements AC-4, AC-5, AC-6 and breakdown Story 3 verification approach; Story 1 adds 0 tests, Story 2 adds 0 tests, Story 3 adds 9 tests (sum: 0+0+9=9)
- **Behavioral outcomes specified:** HLD describes output formats (traceparent W3C format, error message text, client singleton behavior) without prescribing Python syntax
- **Migration path:** Deployments must set `ANTHROPIC_API_KEY` and remove `LITELLM_BASE_URL`/`ANTHROPIC_GATEWAY_TOKEN` before deploying this change; documented in PR per breakdown Story 3 AC-5
- **LangWatch prerequisite:** Breakdown dependencies specify "LangWatch tracing configured and validated in deployment environment before starting Story 1" to avoid observability gap
- **Stack matches repo context:** Python 3.11+ with `anthropic` SDK, `pytest`, `uv` package manager per `pyproject.toml` and ticket context; no TypeScript/JavaScript (repo is Python)
