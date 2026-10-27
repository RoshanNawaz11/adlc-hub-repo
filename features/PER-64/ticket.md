---
key: PER-64
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Remove LiteLLM and Cloudflare AI Gateway; call Anthropic directly
---

## Context

All model traffic in trinet_agents goes agents → LiteLLM (`LITELLM_BASE_URL`) → Cloudflare AI Gateway

→ Anthropic, wired in `src/agents/model/gateway.py`. Skill uploads bypass LiteLLM and go straight to

Cloudflare (`ANTHROPIC_BASE_URL` + `ANTHROPIC_GATEWAY_TOKEN`, `cf-aig-authorization` header).

LiteLLM is also the only LangWatch tracer while `LANGWATCH_DISABLED=true`.

We want to drop both hops and call the Anthropic API directly with our own key.

## Goal

`ModelGateway` talks to Anthropic directly for both model calls and skill uploads, using one

client configured from `ANTHROPIC_API_KEY` (and optional `ANTHROPIC_BASE_URL`). No LiteLLM or

Cloudflare code, config, headers, or docs remain.

## Requirements

* `gateway.py`
  * Build a single `anthropic.AsyncAnthropic` client from `ANTHROPIC_API_KEY`; pass `base_url`

    only when `ANTHROPIC_BASE_URL` is set (default Anthropic endpoint otherwise).
  * `require_client()` and `require_skills_client()` both return that client. Keep both method

    names so `skills/client.py` and `tools/bindings.py` don't change.
  * Remove `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, the `cf-aig-authorization` header, the

    `x-litellm-session-id` header, and the `_add_session_header` request hook.
  * `MissingAPIKeyError` is raised when `ANTHROPIC_API_KEY` is unset (message updated); callers

    that catch it keep working unchanged.
  * Keep the `session_id` ContextVar (rename its ContextVar label from "litellm_session_id" to

    "run_session_id"), `traceparent()` and `trace_id()`: logging per run (`runtime/runner.py`),
    the worker pool (`queue/pool.py`, `queue/subtasks.py`) and the `trace_id` in run results use them.
  * Update the module docstring to describe the direct route.
* Tracing: with LiteLLM gone, agent-side LangWatch tracing (`observability/`) is the tracer.

  `.env.example` sets `LANGWATCH_DISABLED=false` with `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL`

  documented. Update the comment in `runtime/runner.py` `_config()` that says LiteLLM does the tracing.
* `.env.example`: remove `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN` and the Cloudflare URL;

  `ANTHROPIC_API_KEY=` is required, `ANTHROPIC_BASE_URL=` optional and commented out.
* Delete `docs/claude-via-litellm-cloudflare.md`; remove or rewrite every LiteLLM/Cloudflare

  section in `README.md` to describe the direct Anthropic setup.
* `model/policy.yaml`: fix the header comment so it matches the new config.
* No changes to node logic, prompts, models in `policy.yaml`, MCP, Redis or backend code.

## Acceptance criteria

* `grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`
  returns nothing.
* `tests/test_tracing.py` rewritten: drop header assertions; keep (a) `traceparent()` is stable per

  session id and differs between ids, and (b) a worker runs inside its run's `session_id`.
* New `tests/test_gateway.py`:
  * `ANTHROPIC_API_KEY` unset → `require_client()` and `require_skills_client()` raise `MissingAPIKeyError`.
  * Key set, no `ANTHROPIC_BASE_URL` → client uses the default Anthropic base URL.
  * `ANTHROPIC_BASE_URL` set → client uses it; no custom auth headers are added.
  * Both `require_*` methods return the same client instance.
* `uv run pytest -q` passes.
* Manual check (documented in the PR): with a real `ANTHROPIC_API_KEY` and no LiteLLM running,

  `run_so.py` completes one service order, and a skill upload via `skills/client.py` succeeds.

## Out of scope

* Changing models, max_tokens or prompts.
* MCP server, GraphQL, backend or frontend repos.
* Replacing LangWatch with another tracer.
