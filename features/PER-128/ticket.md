---
key: PER-128
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Skip the orchestrator's model call on hops with only one possible route,
  and record token usage per run
---

## Context

Every node already runs on `claude-haiku-4-5-20251001` (`src/agents/model/policy.yaml`), and the skill and MCP calls already use prompt caching. A cheaper model won't lower the cost. Fewer calls will.

The orchestrator makes the most calls. In `src/agents/nodes/orchestrator/node.py`, control comes back to it at the start and after the planner, every agent, every fetch and every review. Each visit does the following:

* `_decide` calls `run_skill` (`src/agents/skills/client.py`), with the skill container and the code-execution tool attached.
* The input is `_brief(state, ...)`. It holds the whole service order, every agent's full result, every flag and every request, so it grows with each hop.
* A route that `broken_rules` rejects is sent back, up to `_ATTEMPTS = 3` calls per visit.
* `run_skill` asks again, with the input and the previous reply, when the answer has no JSON.

Many hops have only one route that `broken_rules` (`route_rules.py`) would accept. For example:

* Before the planner has run, only `["planner"]` is valid.
* After the review, only `["__end__"]` is valid.
* Once execution has finished, only `["output-validator"]` is valid.

The model is paid to give an answer the code already knows.

We also can't see what a run costs:

* `run_skill` logs token usage per call.
* The MCP tool loop in `src/agents/tools/bindings.py` and `ModelGateway.complete` log none.
* Nothing adds usage up per run.

So savings can't be measured today.

This adds a routing component and new run-state fields (usage, and which flags the orchestrator has seen), so it needs a design review before coding. The state, orchestrator and usage changes must agree with each other.

## Goal

The orchestrator calls the model only when there is a real choice to make, or work only the model can do. Every run reports how many model calls and tokens it used, per node, so we can measure the saving.

## Requirements

* Token usage per run
  * Every model call records `input_tokens`, `output_tokens`, `cache_creation_input_tokens` and `cache_read_input_tokens`. That covers `run_skill` (including its JSON retries), each round of the MCP tool loop in `bindings.py` (including the final no-tools call) and `ModelGateway.complete`.
  * `State` gets `usage`: `{node: {calls, input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}}`, with a reducer that adds a node's new counts to its existing ones. A run saved before this change, with no `usage`, must still load and resume.
  * Each node's update carries the usage of the calls it made.
  * `run_query` in `src/agents/runtime/runner.py` logs the run's totals when the run ends or pauses. `GET /runs/{thread_id}` returns them in `values`.
* Forced routes (`route_rules.py`)
  * Add `forced_route(state) -> list[str] | None`. It returns the route when exactly one candidate route passes `broken_rules`, and `None` otherwise.
  * Candidates are each node in `NODES` on its own, plus every non-empty set of stage agents.
  * `broken_rules` stays the only source of the routing rules: `forced_route` must not repeat them.
* Orchestrator (`node.py`)
  * When `forced_route` returns a route, skip `_decide` and use that route, unless the model still has work to do:
    * The route pauses for a person (`flag-review` or `review`). The model writes the reviewer's `summary` before a pause.
    * An unresolved flag exists that the orchestrator's model has not seen yet. Merging duplicate flags needs the model.
  * Track which flags the model has seen in a new state field, set from the flag ids in each brief it receives.
  * A forced hop writes the same step as now: title `Next: <route>`, result `"forced: only one valid route"`, evidence `attempts: 0`. It updates `status`, `flagged`, `priority` and `confidence` exactly as a model hop does, and leaves `summary` unchanged.

## Acceptance criteria

* New tests, with a fake gateway and no live services:
  * `forced_route` returns `["planner"]` before the planner has run, `["__end__"]` after a review, and `["output-validator"]` once execution has finished.
  * `forced_route` returns `None` when two candidates are valid, such as two unfinished stage agents with nothing waiting.
  * The orchestrator makes no model call on a forced hop. It does call the model when the forced route is `flag-review` or `review`, and when a flag it hasn't seen is open.
  * A forced hop's step, `status`, `priority` and `confidence` match what a model hop would write for the same route.
  * `usage` adds counts per node across calls, including JSON retries and every MCP tool round, and a run saved without `usage` still resumes.
* Existing tests pass unchanged, and `uv run pytest -q` passes.
* PR description: model calls and total tokens for one service order (`run_so.py`, e.g. SO-1003), before and after the change.

## Out of scope

* Changing models, `policy.yaml` limits, skill or prompt content.
* Shrinking `_brief`, or removing the skill container and code-execution tool from orchestrator calls.
* The MCP server, the GraphQL service and fastapi-backend.
* Graph routing rules themselves: `broken_rules` behaves as today.
