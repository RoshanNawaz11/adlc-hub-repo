# Tech Spec — Story 02-agent

**ticket:** 02-agent  
**title:** Add AGENT_MAX_STEPS environment variable parsing and update endpoint to /query_given  
**story points:** 3  
**made from:** PER-56 requirements + HLD + breakdown + review_feedback.md  
**source:** designer (LLD)

---

## Overview

Read `AGENT_MAX_STEPS` from the environment in `run_server`, validate it as a positive integer, and pass it to the `Orchestrator` constructor. If the value is invalid (non-integer, non-positive, or empty), log a warning and use the default value `5`. This allows operators to tune the step limit per deployment without code changes. Additionally, update the HTTP route handler from `/query` to `/query_given` per review feedback.

This story implements FR-3 (step limit configuration) and the endpoint path change from `review_feedback.md`. It depends on Story 1 (which added the `max_steps` parameter to `Orchestrator.__init__`). Story 3 will expand test coverage for the full loop behavior, and Story 4 will document the new env var and endpoint in the README.

---

## Stack

- **Language:** Python 3.11+
- **Framework:** None (synchronous request handler)
- **HTTP server:** `run_server` (existing, instantiates `Orchestrator` and registers routes)
- **Test framework:** `pytest`
- **Logging:** `logging` module (existing)

---

## Acceptance criteria

From requirements.md and breakdown.md, this story covers:

- **AC-6:** `AGENT_MAX_STEPS` parsing:
  - Valid positive integer (e.g., `"3"`) → `Orchestrator` instantiated with `max_steps=3`, no warning logged.
  - Invalid values (`"foo"`, `"-1"`, `""`, `"3.5"`) → `Orchestrator` instantiated with `max_steps=5`, warning logged.
- **Endpoint path change (review_feedback.md):** HTTP route handler updated from `/query` to `/query_given`.

These ACs map to 3 unit tests in `tests/test_http_server.py` (or `tests/test_run_server.py`).

---

## Schema / API contracts

### Environment variable (new)

**Name:** `AGENT_MAX_STEPS`  
**Type:** string (parsed as positive integer in `run_server`)  
**Default:** `"5"` (when absent or invalid)  
**Valid values:** `"1"`, `"2"`, ..., `"99"`, etc. (any string that parses as a positive int)  
**Invalid values:** `"0"`, `"-1"`, `"foo"`, `"3.5"` → log warning, use default `5`  
**Absent (`""` or not set):** Use default `5`, no warning (env var not set is normal)

Parsing logic in `run_server`:

```python
import os
import logging

logger = logging.getLogger(__name__)

# Parse AGENT_MAX_STEPS before instantiating Orchestrator
try:
    raw = os.environ.get("AGENT_MAX_STEPS", "")
    if raw:
        max_steps = int(raw)
        if max_steps <= 0:
            raise ValueError("AGENT_MAX_STEPS must be positive")
    else:
        max_steps = 5  # default when env var is absent
except (ValueError, TypeError):
    logger.warning("Invalid AGENT_MAX_STEPS, using default 5")
    max_steps = 5

# Pass max_steps to Orchestrator constructor
orchestrator = Orchestrator(
    llm_client=llm_client,
    mcp_client=mcp_client,
    tools=tools,
    max_steps=max_steps  # NEW ARGUMENT
)
```

**Key points:**

- `os.environ.get("AGENT_MAX_STEPS", "")` returns empty string `""` if env var is not set.
- Empty string (`""`) is treated as "not set" → use default `5` without warning.
- Non-empty invalid values (e.g., `"foo"`, `"-1"`) trigger warning and use default `5`.
- Positive integers (e.g., `"3"`) are parsed and passed to `Orchestrator`.

---

### HTTP endpoint contract (updated)

**Endpoint:** `POST /query_given` (was `/query`)  
**Request body:** `{"query": "<user question>"}`  
**Response (success):** `200 OK`, body: `{"answer": "<text answer>"}`  
**Response (validation error):** `400 Bad Request`, body: `{"error": "<ValueError message>"}`  
**Response (other error):** `500 Internal Server Error`, body: `{"error": "<RuntimeError or other exception message>"}`

**Change from prior version:** Route handler path is `/query_given` (was `/query`). Route registration:

```python
@app.route("/query_given", methods=["POST"])
def query_given():
    # existing handler logic (unchanged)
    ...
```

No `/query` route exists after this story. Requests to `POST /query` will return `404 Not Found`.

---

## Files to create / modify

### Files to modify

1. **`app/run_server.py`**
   - **Change:** Add `import os` and `import logging` at top (if not already present).
   - **Change:** Parse `AGENT_MAX_STEPS` from `os.environ` before instantiating `Orchestrator` (see parsing logic above).
   - **Change:** Pass `max_steps=<parsed_value>` to `Orchestrator(...)` constructor.
   - **Change:** Update HTTP route handler from `@app.route("/query", methods=["POST"])` to `@app.route("/query_given", methods=["POST"])`.
   - **Lines affected:** ~12–15 lines (env var parsing + logging + constructor call update + route path change).

2. **`tests/test_http_server.py`** (or `tests/test_run_server.py` if separate; create if does not exist)
   - **Change:** Add new test cases for `AGENT_MAX_STEPS` parsing (AC-6, see Test Plan below).
   - **Change:** Add test case for HTTP endpoint path (new route `/query_given`, old route `/query` returns 404).
   - **Lines affected:** ~80–100 lines (3 test cases: valid env var, invalid env var, endpoint path).

### Files to read (context only)

- **`app/agent/orchestrator.py`** (Story 1 added `max_steps` parameter; this story passes it from env var).
- **`app/run_server.py`** (existing HTTP server setup, route registration, `Orchestrator` instantiation).

---

## Interfaces

### run_server startup logic (modified)

**Before:**

```python
# Existing instantiation (Story 1)
orchestrator = Orchestrator(
    llm_client=llm_client,
    mcp_client=mcp_client,
    tools=tools,
    max_steps=5  # hardcoded default
)

@app.route("/query", methods=["POST"])
def query():
    # handler logic
    ...
```

**After:**

```python
import os
import logging

logger = logging.getLogger(__name__)

# NEW: Parse AGENT_MAX_STEPS
try:
    raw = os.environ.get("AGENT_MAX_STEPS", "")
    if raw:
        max_steps = int(raw)
        if max_steps <= 0:
            raise ValueError("AGENT_MAX_STEPS must be positive")
    else:
        max_steps = 5  # default when env var is absent
except (ValueError, TypeError):
    logger.warning("Invalid AGENT_MAX_STEPS, using default 5")
    max_steps = 5

# Pass max_steps from env var
orchestrator = Orchestrator(
    llm_client=llm_client,
    mcp_client=mcp_client,
    tools=tools,
    max_steps=max_steps  # from env var parsing
)

@app.route("/query_given", methods=["POST"])  # CHANGED PATH
def query_given():  # CHANGED FUNCTION NAME
    # handler logic (unchanged)
    ...
```

---

## Algorithm notes

### Environment variable parsing logic

**Step 1:** Read env var with `os.environ.get("AGENT_MAX_STEPS", "")` → returns string or `""`.

**Step 2:** Check if string is non-empty:
- Empty (`""`) → use default `5`, no warning (env var not set is normal).
- Non-empty → attempt to parse as `int`.

**Step 3:** Parse as `int` inside `try/except`:
- Success → validate `> 0`:
  - `<= 0` → raise `ValueError` (caught by `except`).
  - `> 0` → use parsed value.
- Failure (`ValueError` or `TypeError`) → log warning, use default `5`.

**Step 4:** Pass parsed or default `max_steps` to `Orchestrator` constructor.

**Edge cases:**

- `AGENT_MAX_STEPS=""` (empty string) → use default `5`, no warning.
- `AGENT_MAX_STEPS="3"` → use `max_steps=3`, no warning.
- `AGENT_MAX_STEPS="0"` → log warning, use default `5`.
- `AGENT_MAX_STEPS="-1"` → log warning, use default `5`.
- `AGENT_MAX_STEPS="foo"` → log warning, use default `5`.
- `AGENT_MAX_STEPS="3.5"` → log warning (float string fails `int()` parse), use default `5`.

### HTTP route handler path change

**Before:** `@app.route("/query", methods=["POST"])`  
**After:** `@app.route("/query_given", methods=["POST"])`

**Function name change (optional but recommended for clarity):**  
`def query():` → `def query_given():`

**Handler logic:** Unchanged. The handler still calls `orchestrator.handle_query(query_text)` and returns the result with appropriate HTTP status codes (200, 400, 500).

**Old route `/query`:** Does not exist after this story. Requests to `POST /query` return `404 Not Found`.

---

## Edge cases / failure modes

### Edge case 1: AGENT_MAX_STEPS not set (env var absent)

**Scenario:** `AGENT_MAX_STEPS` is not set in the environment (i.e., `os.environ.get("AGENT_MAX_STEPS", "")` returns `""`).

**Behavior:** Use default `5`, no warning logged.

**Test:** Variant of Test 1 where `os.environ` does not contain `AGENT_MAX_STEPS` → assert `max_steps=5`, no warning.

---

### Edge case 2: AGENT_MAX_STEPS set to very large value

**Scenario:** `AGENT_MAX_STEPS="9999"` or `"1000000"`.

**Behavior:** Parsed as `int`, validated as `> 0` → used as-is. No upper bound check (per requirements; deployment can set any positive int).

**Test:** Not added (no upper bound requirement; valid behavior).

---

### Edge case 3: AGENT_MAX_STEPS set to zero

**Scenario:** `AGENT_MAX_STEPS="0"`.

**Behavior:** Parsed as `int(0)`, fails `> 0` check → `ValueError` raised and caught → warning logged, default `5` used.

**Test:** Sub-case of Test 2.

---

### Edge case 4: Multiple instantiations (e.g., test fixtures)

**Scenario:** Test suite instantiates `Orchestrator` multiple times with different env var values.

**Behavior:** Each instantiation parses `AGENT_MAX_STEPS` independently. No global state or caching (env var is read fresh each time).

**Test:** Not added (no cross-instantiation behavior; each parse is independent).

---

## Risks

### RISK-1: Typo in env var name breaks configuration

**Likelihood:** Low (env var name is documented in README in Story 4)  
**Impact:** Medium (deployment uses default `5` instead of intended value)  
**Mitigation (implemented):** Env var name is spelled consistently across code and docs: `AGENT_MAX_STEPS`. Story 4 documents it in README.  
**Residual risk:** Operators must spell env var correctly. Accepted: standard risk for env var configuration.

---

### RISK-2: Warning log message not seen in production

**Likelihood:** Medium (operators may not monitor logs closely)  
**Impact:** Low (default `5` is used, which is reasonable)  
**Mitigation (implemented):** Warning logged at `logger.warning` level (typically visible in production logs). Invalid env var does not crash the server, just uses default.  
**Residual risk:** Operators may not notice invalid config until they observe default behavior. Accepted: logging is the standard notification mechanism.

---

### RISK-3: Endpoint path change breaks existing clients

**Likelihood:** High (if existing clients are hardcoded to `/query`)  
**Impact:** High (all requests fail with 404)  
**Mitigation (implemented):** README updated in Story 4 with new endpoint path `/query_given`. Test case verifies old route `/query` returns 404.  
**Residual risk:** Clients must be updated to use `/query_given`. This is a **breaking change** per review_feedback.md; it is intentional. Clients that do not update will break.  
**Acceptance:** This is a deliberate breaking change requested in review feedback. No backward-compatibility route is added.

---

## Dependencies

- **Story 1:** `Orchestrator.__init__` accepts `max_steps` parameter (added in Story 1). This story passes it from env var parsing.
- **Existing:** `llm_client`, `mcp_client`, `tools` are already instantiated before `Orchestrator` (unchanged).
- **Existing:** Flask (or equivalent HTTP framework) app and route registration (unchanged except route path).
- **Story 3 dependency:** Test expansion for loop behavior (Story 3 will add orchestrator-level tests; this story adds env var and endpoint tests only).
- **Story 4 dependency:** README documentation (Story 4 will document `AGENT_MAX_STEPS` and `/query_given` endpoint).

No new external dependencies.

---

## Implementation notes

- **Env var parsing:** Use `os.environ.get("AGENT_MAX_STEPS", "")` to read env var. Empty string means "not set" → use default `5`, no warning. Non-empty invalid values → log warning, use default `5`.
- **Validation logic:** `try/except (ValueError, TypeError)` catches both `int()` parse failures and comparison failures (`<= 0` raises `ValueError`).
- **Logging:** Use `logger.warning("Invalid AGENT_MAX_STEPS, using default 5")` (no exception details in log message; the message is clear enough).
- **Route path change:** Update `@app.route("/query", ...)` to `@app.route("/query_given", ...)`. Function name change (`def query():` → `def query_given():`) is optional but recommended for clarity. Handler logic (calling `orchestrator.handle_query`, returning JSON response, handling exceptions) is unchanged.
- **Backward compatibility:** No backward-compatibility route for `/query`. The old endpoint no longer exists. This is a **breaking change**.

---

## Verification checklist

- [ ] All unit tests in `tests/test_http_server.py` pass (Tests 1–3).
- [ ] `python -m pytest tests/test_http_server.py -q` exits 0 (all HTTP server tests pass).
- [ ] Valid `AGENT_MAX_STEPS` (e.g., `"3"`) → `Orchestrator` instantiated with `max_steps=3`, no warning.
- [ ] Invalid `AGENT_MAX_STEPS` (e.g., `"foo"`, `"-1"`, `"3.5"`) → `Orchestrator` instantiated with `max_steps=5`, warning logged.
- [ ] Empty `AGENT_MAX_STEPS` (`""`) or absent → `Orchestrator` instantiated with `max_steps=5`, no warning.
- [ ] HTTP `POST /query_given` succeeds (200 OK).
- [ ] HTTP `POST /query` fails (404 Not Found).
- [ ] Manual review: no changes to `orchestrator.py` or loop logic (this story only modifies `run_server.py` and adds tests).

---

## Out of scope for this story

- **Orchestrator loop logic:** Story 1 implemented the loop; this story only passes `max_steps` from env var.
- **Orchestrator unit tests:** Story 3 will expand `test_orchestrator.py` with loop behavior tests; this story adds only env var and endpoint tests in `test_http_server.py`.
- **README documentation:** Story 4 will document `AGENT_MAX_STEPS` and `/query_given` endpoint; this story focuses on code changes only.
- **Conversation memory across requests:** Out of scope for entire feature (per requirements.md).
- **Streaming responses:** Out of scope for entire feature (per requirements.md).

---

## Tests

### Test 1: Valid AGENT_MAX_STEPS (AC-6)

**File:** `tests/test_http_server.py`

**Test method:** `test_valid_agent_max_steps`

**Mock setup:**
- `os.environ = {"AGENT_MAX_STEPS": "3"}`
- `Orchestrator` constructor mocked or spied (capture `max_steps` argument)
- `logger.warning` mocked (verify not called)

**Execution:**
- Call startup logic (or function that parses env var and instantiates `Orchestrator`)

**Assertions:**
- `Orchestrator` called with `max_steps=3`
- `logger.warning` not called

---

### Test 2: Invalid AGENT_MAX_STEPS (AC-6)

**File:** `tests/test_http_server.py`

**Test method:** `test_invalid_agent_max_steps` (parameterized with 4 sub-cases)

**Sub-case 1: Non-integer string**
- Mock `os.environ = {"AGENT_MAX_STEPS": "foo"}`
- Assert: `Orchestrator` called with `max_steps=5`, `logger.warning` called once with message `"Invalid AGENT_MAX_STEPS, using default 5"`

**Sub-case 2: Non-positive integer**
- Mock `os.environ = {"AGENT_MAX_STEPS": "-1"}`
- Assert: `Orchestrator` called with `max_steps=5`, `logger.warning` called once with message `"Invalid AGENT_MAX_STEPS, using default 5"`

**Sub-case 3: Empty string (not set)**
- Mock `os.environ = {"AGENT_MAX_STEPS": ""}` or `os.environ = {}` (no key)
- Assert: `Orchestrator` called with `max_steps=5`, `logger.warning` **not** called

**Sub-case 4: Float string**
- Mock `os.environ = {"AGENT_MAX_STEPS": "3.5"}`
- Assert: `Orchestrator` called with `max_steps=5`, `logger.warning` called once with message `"Invalid AGENT_MAX_STEPS, using default 5"`

---

### Test 3: HTTP endpoint path (new)

**File:** `tests/test_http_server.py`

**Test method:** `test_query_given_endpoint_path`

**Mock setup:**
- Start Flask test client (or equivalent HTTP test server)
- Mock `Orchestrator.handle_query` to return `"Test answer"`

**Execution (positive case):**
- Send `POST /query_given` with JSON body `{"query": "test"}`

**Assertions:**
- Response status code is `200`
- Response JSON is `{"answer": "Test answer"}`

**Execution (negative case):**
- Send `POST /query` with JSON body `{"query": "test"}`

**Assertions:**
- Response status code is `404` (route not found)

---

## Acceptance criteria (mapped to tests)

- **AC-6 (AGENT_MAX_STEPS parsing):** Tests 1–2
- **Endpoint path change (review_feedback.md):** Test 3

Total: 3 unit tests in `tests/test_http_server.py`.
