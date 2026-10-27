# HLD — PER-64

**Ticket:** Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly

**Goal:** Direct Anthropic API calls for model traffic and skill uploads, removing LiteLLM and Cloudflare intermediaries.

---

## Components

### 1. ModelGateway (`src/agents/model/gateway.py`)

**Current state:**
- Maintains separate client paths: model calls → LiteLLM (`LITELLM_BASE_URL`), skill uploads → Cloudflare AI Gateway (`ANTHROPIC_BASE_URL` + `ANTHROPIC_GATEWAY_TOKEN`)
- Injects session tracking via `x-litellm-session-id` and `cf-aig-authorization` headers
- Wraps `anthropic.AsyncAnthropic` with custom request hooks

**Target state:**
- Single `anthropic.AsyncAnthropic` client for both model calls and skill uploads
- Configured from `ANTHROPIC_API_KEY` (required) and optional `ANTHROPIC_BASE_URL`
- No LiteLLM/Cloudflare-specific headers or hooks
- Session tracking preserved via `session_id` ContextVar (renamed label from "litellm_session_id" to "run_session_id")
- Client instance is singleton (shared across all calls to both `require_client()` and `require_skills_client()`)

**Responsibilities:**
- Validate `ANTHROPIC_API_KEY` presence; raise `MissingAPIKeyError` when unset
- Build client configured from environment variables
- Provide `require_client()` and `require_skills_client()` (both return same instance)
- Maintain `session_id`, `traceparent()`, and `trace_id()` for downstream tracing

---

### 2. Configuration (`.env.example`, `model/policy.yaml`)

**Current state:**
- `.env.example` defines `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, `ANTHROPIC_BASE_URL` (Cloudflare gateway URL)
- `model/policy.yaml` header comment references LiteLLM routing

**Target state:**
- `.env.example`: `ANTHROPIC_API_KEY=` (required), optional `ANTHROPIC_BASE_URL=` (commented out)
- `LANGWATCH_DISABLED=false` with `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL` documented
- `model/policy.yaml` header updated to reflect direct Anthropic setup

---

### 3. Tracing (`observability/`, `runtime/runner.py`)

**Current state:**
- LiteLLM acts as LangWatch tracer while `LANGWATCH_DISABLED=true`
- Comment in `runtime/runner.py` `_config()` attributes tracing to LiteLLM

**Target state:**
- Agent-side LangWatch tracing (`observability/`) becomes the active tracer
- `runtime/runner.py` comment updated to reflect LangWatch as the tracer
- Session-based tracing (`session_id`, `traceparent()`, `trace_id()`) preserved for run logging, worker pool, and run results

---

### 4. Documentation (`docs/`, `README.md`)

**Current state:**
- `docs/claude-via-litellm-cloudflare.md` documents LiteLLM → Cloudflare routing
- `README.md` sections reference LiteLLM/Cloudflare setup

**Target state:**
- `docs/claude-via-litellm-cloudflare.md` deleted
- `README.md` sections rewritten to describe direct Anthropic configuration

---

### 5. Tests (`tests/test_gateway.py`, `tests/test_tracing.py`)

**Current state:**
- `tests/test_tracing.py` asserts LiteLLM-specific headers (`x-litellm-session-id`)
- No dedicated gateway client validation

**Target state:**
- New `tests/test_gateway.py` validates client behavior: 6 test cases covering error handling, base URL configuration, header validation, and singleton pattern
- Rewritten `tests/test_tracing.py` validates session tracking behavior: 3 test cases covering `traceparent()` stability, variance, and worker context propagation

---

## Contracts

### ModelGateway Client API

**Input:**
- Environment: `ANTHROPIC_API_KEY` (required), `ANTHROPIC_BASE_URL` (optional)
- Session context: `session_id` ContextVar

**Output:**
- `require_client() -> anthropic.AsyncAnthropic`: Returns configured singleton client instance
- `require_skills_client() -> anthropic.AsyncAnthropic`: Returns same singleton client instance as `require_client()`
- `traceparent() -> str`: Returns W3C traceparent header for current session
- `trace_id() -> str`: Returns extracted trace ID from traceparent

**Error conditions:**
- `MissingAPIKeyError`: Raised when `ANTHROPIC_API_KEY` unset (message: "ANTHROPIC_API_KEY environment variable required")

**Behavior:**
- Client instance is singleton (same object returned by both `require_*` methods)
- When `ANTHROPIC_BASE_URL` is set, client uses that endpoint; otherwise uses Anthropic SDK default
- No custom HTTP headers added beyond Anthropic SDK defaults

---

### Session Tracking

**ContextVar interface:**
- `session_id: ContextVar[str]` (label renamed to "run_session_id")

**Consumers:**
- `runtime/runner.py`: Logs per-run session ID
- `queue/pool.py`, `queue/subtasks.py`: Propagates session ID to worker threads
- Run results: Includes `trace_id` for correlation

**Traceparent format:**
- W3C Trace Context standard: `00-<trace-id>-<parent-id>-<flags>`
- Stable per session ID; deterministic generation

---

### Configuration Schema

**`.env.example` required fields:**
```
ANTHROPIC_API_KEY=
```

**`.env.example` optional fields:**
```
# ANTHROPIC_BASE_URL=https://api.anthropic.com  # uncomment to override default endpoint
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
- Migration documented in PR (manual check includes validating direct Anthropic integration)
- `.env.example` clearly specifies new required/optional variables
- Deployment runbook updated to reference direct Anthropic setup

---

### R-2: Observability gap during transition
**Impact:** Medium  
**Probability:** Low  
**Mitigation:**
- LangWatch tracing validated before LiteLLM removal
- Session-based tracing (`session_id`, `traceparent()`) preserved; no gap in run correlation
- Rewritten `tests/test_tracing.py` validates tracing behavior pre-merge

---

### R-3: Skill upload regression
**Impact:** High  
**Probability:** Low  
**Mitigation:**
- Manual check validates skill upload via `skills/client.py` before merge (see requirements AC-7)
- `require_skills_client()` returns same client as `require_client()`; no divergent code paths
- Anthropic SDK handles file uploads natively; no custom logic required

---

### R-4: Incomplete cleanup of LiteLLM/Cloudflare references
**Impact:** Medium  
**Probability:** Low  
**Mitigation:**
- Requirements AC-3 enforces `grep` check across all source/config/docs
- Code review checklist includes manual scan for orphaned config/comments
- CI can add `grep` step as guardrail post-merge

---

## Architecture Decisions

### AD-1: Single Anthropic client for all model traffic
**Binds:** `require_client()` and `require_skills_client()` return the same `AsyncAnthropic` instance  
**Prevents:** Client configuration drift between model calls and skill uploads  
**Rule:** No separate client initialization paths; all Anthropic traffic uses one configured client  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-2: Optional base URL override
**Binds:** `ANTHROPIC_BASE_URL` is optional; when unset, client uses Anthropic SDK default endpoint  
**Prevents:** Hard-coded endpoint assumptions; supports testing/proxy scenarios  
**Rule:** Client configured to use custom endpoint when `ANTHROPIC_BASE_URL` is set  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-3: Session tracking persists post-LiteLLM
**Binds:** `session_id` ContextVar and `traceparent()` remain unchanged in behavior  
**Prevents:** Breaking run correlation in logging, worker pool, and observability  
**Rule:** Rename ContextVar label to "run_session_id"; keep ContextVar name `session_id`; `traceparent()`/`trace_id()` logic unchanged  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-4: LangWatch as primary tracer
**Binds:** Agent-side LangWatch tracing enabled by default (`LANGWATCH_DISABLED=false`)  
**Prevents:** Observability regression after LiteLLM removal  
**Rule:** `.env.example` sets `LANGWATCH_DISABLED=false`; `runtime/runner.py` comment updated  
**Status:** [ADOPTED] (ticket requirement)

---

### AD-5: No auth header customization
**Binds:** Anthropic client uses SDK default authentication (API key in `x-api-key` header)  
**Prevents:** Leftover Cloudflare/LiteLLM header logic causing auth failures  
**Rule:** Remove `cf-aig-authorization`, `x-litellm-session-id`, and `_add_session_header` hook; no custom headers added  
**Status:** [ADOPTED] (ticket requirement)

---

## Deferred

- Model selection, `max_tokens`, or prompt changes (out of scope per ticket)
- MCP server, GraphQL, backend, or frontend modifications (out of scope per ticket)
- LangWatch tracer replacement (out of scope per ticket)
- Retry/timeout configuration for Anthropic client (not mentioned in requirements; inherits SDK defaults)
- Rate limiting (not mentioned in requirements; assumes Anthropic SDK handles throttling)

---

## Notes

- **File paths verified against ticket context:** `src/agents/model/gateway.py`, `runtime/runner.py`, `model/policy.yaml`, `tests/test_gateway.py`, `tests/test_tracing.py`, `.env.example`, `README.md`, `docs/claude-via-litellm-cloudflare.md` (all referenced in ticket and breakdown; repo-tree shows empty existing files list indicating tree is incomplete but paths are established by ticket)
- **AC alignment:** 7 requirements ACs map to breakdown stories: AC-1 (gateway refactored) → Story 1, AC-2 (config/docs updated) → Story 2, AC-3 (no LiteLLM/Cloudflare references) → Story 3 final validation, AC-4 (gateway tests) → Story 3, AC-5 (tracing tests) → Story 3, AC-6 (all tests pass) → Story 3 regression check, AC-7 (manual validation) → Story 3 manual check
- **Test count alignment:** 9 automated tests total (6 gateway + 3 tracing) per requirements AC-4, AC-5, AC-6 and breakdown Story 3 verification approach; each test validates specific client behavior or session tracking as specified in breakdown
- **Behavioral validation:** Requirements AC-7 manual check documents live validation with real `ANTHROPIC_API_KEY`; captures success/failure in PR description
- **Migration path:** Deployments must set `ANTHROPIC_API_KEY` and remove `LITELLM_BASE_URL`/`ANTHROPIC_GATEWAY_TOKEN` before deploying this change
