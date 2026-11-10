## What would go wrong

### C1
- file: stories/01-graphql.md
- section: (title)
- quote: # Story — PER-200
- problem: The story file is named `01-graphql.md` but describes a Python REST endpoint (`GET /ready`) in the agents service. The repo outline shows `app` is a Python repo, and the ticket and requirements reference Python files (`src/agents/runtime/server.py`, `runtime/checkpointer.py`). The "graphql" name implies a different stack or service.
- change: Rename the file to match the work (e.g., `01-readiness-check.md`) or verify the file is in the correct location for the Python agents service.

### C2
- file: stories/01-graphql.md
- section: ## Tests
- quote: See `tests/test_health_checks.py` and `tests/test_ready_endpoint.py`:
- match: requirements.md › AC-7: Test coverage
- problem: AC-7 lists seven test scenarios, but the story's Tests section lists ten test cases. The extra three are `test_checkpointer_fails`, `test_redis_fails`, `test_model_gateway_fails`, and `test_mcp_fails` counted separately instead of as "each individual check failing". The developer would write ten tests when seven scenarios were approved.
- change: Consolidate the four individual check failure tests into one scenario ("each check failing") or list all ten explicitly in AC-7 if each dependency needs its own test.

### C3
- file: stories/01-graphql.md
- section: ## Verification
- quote: 1. **Manual:** `curl http://localhost:8000/ready` returns 200 with all checks `ok: true` when dependencies are healthy; returns 503 with at least one `ok: false` when a dependency is unavailable.
- problem: The verification steps are manual checks that duplicate the automated test coverage in AC-7 and the Tests section. AC-7 already requires automated tests for 200/503 responses, parallel execution, timeouts, and secret sanitization. The developer would not know whether to run these manual steps in addition to the automated suite or instead of it.
- change: Remove the manual verification steps or clarify they are for local development only, since AC-7 mandates automated coverage of the same scenarios.

---

VERDICT: FAIL
