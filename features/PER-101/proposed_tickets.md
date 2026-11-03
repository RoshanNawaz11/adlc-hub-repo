# Proposed tickets

## PER-111
- title: Implement Redis-backed idempotency store with in-memory fallback
- repo: trinet-adlc-agents
- ac: AC1 (in-memory replay skips), AC2 (concurrent Redis claims), AC3 (completed run skips), AC4 (released claim re-claimable), AC5 (TTL expires stale claim), AC6 (Redis unavailable raises error), AC7 (Redis failure logs but preserves result), AC9 (completed entry TTL), AC10 (claim atomicity and TTL)
- tests: 11 unit tests in `tests/test_idempotency.py` covering memory store state transitions, Redis key/value/TTL contract, and error handling
- files: `src/agents/runtime/idempotency.py` (major rewrite), `pyproject.toml` (add redis dependency), `tests/test_idempotency.py` (new)

## PER-112
- title: Integrate idempotency store into runner and worker
- repo: trinet-adlc-agents
- ac: AC8 (worker skips status write for already running)
- tests: 8 integration tests in `tests/test_runner.py` covering claim outcomes, pause/exception/cancellation release, and resume flow; 1 test in `tests/test_orders.py` for status write guard
- files: `src/agents/runtime/runner.py` (modify run_query), `src/agents/queue/orders.py` (modify _run), `tests/test_runner.py` (extend), `tests/test_orders.py` (extend)

## PER-113
- title: Document idempotency configuration and update module docstring
- repo: trinet-adlc-agents
- ac: AC11 (test suite passes - all 20 tests from PER-111 and PER-112 green)
- tests: 0 automated tests (manual verification only)
- files: `.env.example` (add three variables), `src/agents/runtime/idempotency.py` (validate docstring from PER-111)
