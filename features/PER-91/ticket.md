---
key: PER-91
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

Every model call in trinet-adlc-agents goes agents → LiteLLM (`LITELLM_BASE_URL`) → Cloudflare AI Gateway → Anthropic.

The route is wired in one file, `src/agents/model/gateway.py`:

* `_build_client()` points the Anthropic SDK at LiteLLM and adds the `_add_session_header` request hook

  (`x-litellm-session-id` and `traceparent` headers).
* `_build_skills_client()` sends skill uploads straight to Cloudflare (`ANTHROPIC_BASE_URL` +

  `ANTHROPIC_GATEWAY_TOKEN`, `cf-aig-authorization` header).
* With `LANGWATCH_DISABLED=true` (the default in `.env.example`), LiteLLM is the only thing that traces runs.

We want to drop both hops and call the Anthropic API directly with our own key.

This is one story in one repo (`agent`). The model ids in `model/policy.yaml` are already real

Anthropic ids (`claude-haiku-4-5-20251001`), so they stay as they are.

## Goal

`ModelGateway` uses one Anthropic client, built from `ANTHROPIC_API_KEY`, for both model calls and

skill uploads. No LiteLLM or Cloudflare code, config, headers or docs remain in the repo.

## Requirements

* `src/agents/model/gateway.py`
  * Build one `anthropic.AsyncAnthropic` client from `ANTHROPIC_API_KEY`. Pass `base_url` only when

    `ANTHROPIC_BASE_URL` is set; otherwise use the SDK default.
  * `require_client()` and `require_skills_client()` both return that same client. Keep both method

    names so `skills/client.py` and `tools/bindings.py` do not change.
  * Remove: `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, the `cf-aig-authorization` header,

    `_add_session_header` and its `event_hooks`, `_build_skills_client()`, and the `"litellm"`
    fallback key in `__init__`.
  * Raise `MissingAPIKeyError` when `ANTHROPIC_API_KEY` is unset, with the message

    "ANTHROPIC_API_KEY is not set". Callers that catch it stay unchanged.
  * Keep `session_id`, `traceparent()` and `trace_id()`: `runtime/runner.py`, `queue/pool.py` and

    `queue/subtasks.py` use them, and run results return `trace_id`. Rename the ContextVar label from
    `"litellm_session_id"` to `"run_session_id"`.
  * Rewrite the module docstring to describe the direct route.
* Tracing: with LiteLLM gone, agent-side LangWatch tracing is the tracer.
  * `.env.example` sets `LANGWATCH_DISABLED=false` and documents `LANGWATCH_API_KEY` and

    `LANGWATCH_ENDPOINT_URL`.
  * `runtime/runner.py`: update the comments on line 131 (`_config`) and line 154 (`session_id.set`)

    that say LiteLLM does the tracing.
* `.env.example`: remove `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN` and the Cloudflare URL.

  `ANTHROPIC_API_KEY=` is required. `ANTHROPIC_BASE_URL=` is optional and commented out.
* Docs: delete `docs/claude-via-litellm-cloudflare.md`. In `README.md`, rewrite the model-route and

  observability sections, the env-var table and the troubleshooting table for the direct Anthropic

  setup, and remove "The LiteLLM + LangWatch stack" and "Connecting LiteLLM to the Cloudflare AI

  Gateway" sections.
* `model/policy.yaml`: make the header comment match the new setup. Do not change models or settings.
* `tests/test_tracing.py` (it exists): remove the two header tests (`_add_session_header`,

  `x-litellm-session-id`). Keep `test_a_worker_runs_inside_its_run`. Add a test that `traceparent()`

  is stable for one session id and differs between ids. Fix the module docstring.

## Acceptance criteria

* `grep -riE "litellm|cloudflare|cf-aig|ANTHROPIC_GATEWAY_TOKEN" src tests docs README.md .env.example`
  returns nothing.
* New `tests/test_gateway.py`, with env vars set through `monkeypatch`:
  * `ANTHROPIC_API_KEY` unset → `require_client()` and `require_skills_client()` raise `MissingAPIKeyError`.
  * Key set, no `ANTHROPIC_BASE_URL` → client uses the SDK default base URL.
  * `ANTHROPIC_BASE_URL` set → client uses it, and no `cf-aig-authorization` header is sent.
  * `require_client() is require_skills_client()`.
* `tests/test_tracing.py` passes with the changes above.
* `uv run pytest -q` passes.
* PR description: manual check with a real `ANTHROPIC_API_KEY` and no LiteLLM running. One service

  order completes via `run_so.py`, and a skill upload via `skills/client.py` succeeds.

## Out of scope

* Changing models, `max_tokens`, prompts or node logic.
* MCP server, GraphQL, backend or frontend repos, Redis or the worker queue.
* Replacing LangWatch with another tracer.
* The separate `llm-gateway-observability-poc` repo (LiteLLM's own config lives there).
