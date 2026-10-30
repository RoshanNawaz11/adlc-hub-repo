Fix these forge PR review comments (auto-route):

### agent comment 4422946932
**AC9 violation: missing test for `traceparent()` stability with explicit session id** — AC9 requires 'a new test verifies `traceparent()` is stable for one session id and differs between different session ids'. The new test `test_traceparent_stable_for_session` only tests the context-var path (`session_id.set()` then `traceparent()` with no arg). The refactored `traceparent(sid: str | None = None)` also accepts an explicit session id, and `trace_id(sid: str)` still calls `traceparent(sid)` with one. Add a case proving `traceparent('session-1') == traceparent('session-1')` and `traceparent('session-1') != traceparent('session-2')` to cover the explicit-arg contract.
