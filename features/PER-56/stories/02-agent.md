# Tech Spec — Story 02-agent

**ticket:** 02-agent  
**title:** Add AGENT_MAX_STEPS environment variable parsing  
**story points:** 3  
**made from:** PER-56 requirements + HLD + breakdown  
**source:** designer (LLD)

---

## Acceptance criteria

From requirements.md and breakdown.md:

1. **Valid AGENT_MAX_STEPS:** When the environment variable is a valid positive integer (e.g., `"3"`), `Orchestrator` is instantiated with `max_steps=3` and no warning is logged.
2. **Invalid AGENT_MAX_STEPS:** When the environment variable is invalid (non-integer, non-positive, empty, or decimal), `Orchestrator` is instantiated with `max_steps=5` and a warning is logged: `"Invalid AGENT_MAX_STEPS, using default 5"`.

---

## File plan

### Files to modify

1. **`app/run_server.py`**
   - **Lines affected:** ~10–15
   - **Changes:**
     - Add `import os` at top if not already present.
     - Add `import logging` at top if not already present; get logger: `logger = logging.getLogger(__name__)`.
     - Before instantiating `Orchestrator`, parse `AGENT_MAX_STEPS` from `os.environ.get("AGENT_MAX_STEPS", "")`.
     - Validate: if non-empty, parse as `int`; if `<= 0`, raise `ValueError`. Catch `ValueError`/`TypeError` and log warning, use default `5`.
     - Pass `max_steps=<parsed_value>` to `Orchestrator(...)` constructor call.

2. **`tests/test_http_server.py`** (or `tests/test_run_server.py` if that file exists)
   - **Lines affected:** ~30–50
   - **Changes:**
     - Add 2 new test cases (AC 1–2 above).
     - Mock `os.environ` to control `AGENT_MAX_STEPS` value.
     - Mock `Orchestrator` constructor to capture `max_steps` argument.
     - Mock `logger.warning` to verify warning is logged for invalid values.

### Files to read (context only, no changes)

- **`app/agent/orchestrator.py`** — existing `__init__` signature with `max_steps` parameter (added in Story 01-agent).

---

## Interfaces

### Environment variable parsing logic (run_server.py)

**Algorithm:**

```python
import os
import logging

logger = logging.getLogger(__name__)

# Parse AGENT_MAX_STEPS (before Orchestrator instantiation)
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

# Instantiate Orchestrator with parsed max_steps
orchestrator = Orchestrator(
    llm_client=llm_client,
    mcp_client=mcp_client,
    tools=tools,
    max_steps=max_steps  # <-- PASS PARSED VALUE
)
```

**Key points:**

- `os.environ.get("AGENT_MAX_STEPS", "")` returns empty string if env var is not set.
- Empty string → use default `5` **without warning** (not treated as invalid).
- Non-empty string → parse as `int()`. If `int()` fails or result `<= 0`, catch exception and use default `5` **with warning**.
- `ValueError` catches both `int()` parse failures and the explicit `raise ValueError` for non-positive values.
- `TypeError` catches edge cases where the env var is somehow not a string (defensive).

---

### Orchestrator constructor call (run_server.py)

**Before (Story 01-agent):**

```python
orchestrator = Orchestrator(
    llm_client=llm_client,
    mcp_client=mcp_client,
    tools=tools
    # max_steps defaults to 5 in __init__
)
```

**After (Story 02-agent):**

```python
orchestrator = Orchestrator(
    llm_client=llm_client,
    mcp_client=mcp_client,
    tools=tools,
    max_steps=max_steps  # <-- PASS PARSED VALUE FROM ENV VAR
)
```

---

## Algorithm notes

### Parsing flow

1. Read `AGENT_MAX_STEPS` from `os.environ` (default to `""` if absent).
2. If empty string → use default `5`, no warning (env var not set is normal).
3. If non-empty:
   - Parse as `int()`.
   - If parse succeeds, check `> 0`. If not, raise `ValueError`.
   - If parse fails or value `<= 0`, catch exception → log warning, use default `5`.
4. Pass parsed `max_steps` to `Orchestrator(...)` constructor.

### Edge cases

- **Env var absent (`""`):** Use default `5`, no warning.
- **Env var = `"0"`:** Parse as `0`, fail `> 0` check → warning, default `5`.
- **Env var = `"-1"`:** Parse as `-1`, fail `> 0` check → warning, default `5`.
- **Env var = `"foo"`:** `int()` raises `ValueError` → warning, default `5`.
- **Env var = `"3.5"`:** `int()` raises `ValueError` (cannot parse float string) → warning, default `5`.
- **Env var = `"3"`:** Parse as `3`, pass `> 0` check → use `3`, no warning.

### Log message

- **Warning message:** `"Invalid AGENT_MAX_STEPS, using default 5"`
- **Log level:** `logger.warning(...)`
- **Triggered when:** Any exception (`ValueError` or `TypeError`) during parsing/validation.

---

## Tests

### Test structure

All tests in `tests/test_http_server.py` (or `tests/test_run_server.py`).  
Mock `os.environ` to control `AGENT_MAX_STEPS` value.  
Mock `Orchestrator` constructor to capture `max_steps` argument.  
Mock `logger.warning` to verify warning is logged (or not logged).

---

### Test 1: Valid AGENT_MAX_STEPS (AC-1)

**Setup:**
- Mock `os.environ.get("AGENT_MAX_STEPS")` to return `"3"`.
- Mock `Orchestrator` constructor to capture arguments (e.g., using `unittest.mock.patch` with side_effect or return_value).
- Mock `logger.warning` to capture calls.

**Execute:**
- Run the `run_server` startup logic (or call the env-parsing function/module directly if it's factored out).

**Assert:**
- `Orchestrator` constructor was called with `max_steps=3`.
- `logger.warning` was not called.

**Implementation:**

```python
@patch("app.run_server.logger.warning")
@patch("app.run_server.Orchestrator")
@patch.dict("os.environ", {"AGENT_MAX_STEPS": "3"})
def test_valid_agent_max_steps(mock_orchestrator, mock_warning):
    # Setup: mock dependencies
    mock_llm_client = Mock()
    mock_mcp_client = Mock()
    mock_tools = [{"name": "test_tool"}]
    
    # Execute: run env var parsing and Orchestrator instantiation
    # (assumes this logic is in a function like `create_orchestrator()` or in module-level code)
    import app.run_server
    # Trigger module reload or call the relevant function
    
    # Assert
    mock_orchestrator.assert_called_once()
    assert mock_orchestrator.call_args[1]["max_steps"] == 3
    mock_warning.assert_not_called()
```

---

### Test 2: Invalid AGENT_MAX_STEPS (AC-2)

**Setup:**
- Mock `os.environ.get("AGENT_MAX_STEPS")` to return invalid values in separate sub-tests:
  - `"foo"` (non-integer)
  - `"-1"` (non-positive)
  - `""` (empty string — **special case: no warning expected**, see below)
  - `"3.5"` (decimal, not parseable as int)
- Mock `Orchestrator` constructor to capture arguments.
- Mock `logger.warning` to capture calls.

**Execute:**
- Run the `run_server` startup logic for each invalid value.

**Assert for `"foo"`, `"-1"`, `"3.5"`:**
- `Orchestrator` constructor was called with `max_steps=5` (default).
- `logger.warning` was called once with message `"Invalid AGENT_MAX_STEPS, using default 5"`.

**Assert for `""` (empty string, env var absent):**
- `Orchestrator` constructor was called with `max_steps=5` (default).
- `logger.warning` was **not** called (empty string is normal, not invalid).

**Implementation:**

```python
@pytest.mark.parametrize("env_value,should_warn", [
    ("foo", True),
    ("-1", True),
    ("3.5", True),
    ("", False),  # empty string = absent, no warning
])
@patch("app.run_server.logger.warning")
@patch("app.run_server.Orchestrator")
def test_invalid_agent_max_steps(mock_orchestrator, mock_warning, env_value, should_warn):
    # Setup: set env var
    with patch.dict("os.environ", {"AGENT_MAX_STEPS": env_value} if env_value else {}, clear=True):
        mock_llm_client = Mock()
        mock_mcp_client = Mock()
        mock_tools = [{"name": "test_tool"}]
        
        # Execute: run env var parsing and Orchestrator instantiation
        import app.run_server
        # Trigger module reload or call the relevant function
        
        # Assert
        mock_orchestrator.assert_called_once()
        assert mock_orchestrator.call_args[1]["max_steps"] == 5
        
        if should_warn:
            mock_warning.assert_called_once_with("Invalid AGENT_MAX_STEPS, using default 5")
        else:
            mock_warning.assert_not_called()
```

**Corrected assertion summary:**

| Value | `max_steps` | Warning logged? |
|-------|-------------|-----------------|
| `"3"` (valid) | `3` | No |
| `"foo"` | `5` | Yes |
| `"-1"` | `5` | Yes |
| `"3.5"` | `5` | Yes |
| `""` (absent) | `5` | **No** |

**Total sub-cases:** 5 (1 valid + 4 invalid/absent scenarios).

**Test count:** 2 test methods (Test 1 for valid, Test 2 with 4 sub-cases for invalid).

---

## Implementation checklist

- [ ] Add `import os` at top of `app/run_server.py` (if not already present).
- [ ] Add `import logging` and `logger = logging.getLogger(__name__)` at top of `app/run_server.py` (if not already present).
- [ ] Add env var parsing logic before `Orchestrator` instantiation (see algorithm above).
- [ ] Update `Orchestrator(...)` constructor call to pass `max_steps=max_steps`.
- [ ] Add Test 1 (valid `AGENT_MAX_STEPS = "3"`).
- [ ] Add Test 2 with 4 sub-cases (invalid values `"foo"`, `"-1"`, `"3.5"`, and absent `""`).
- [ ] Run `python -m pytest tests/test_http_server.py -q` (or `tests/test_run_server.py`) and confirm all tests pass.

---

## Notes

### Dependency on Story 01-agent

This story depends on Story 01-agent having added the `max_steps: int = 5` parameter to `Orchestrator.__init__`. If Story 01-agent is not complete, this story's constructor call will fail (unexpected keyword argument).

### Empty string behavior

**Clarification:** The breakdown says `""` is invalid and should trigger a warning. However, requirements.md FR-3 and HLD AD-3 state:

> "If the value is invalid (non-integer, non-positive, or **empty**) → log a warning and use default 5."

But also:

> "When env var is **absent** or invalid, use default 5."

And the parsing logic in HLD AD-3 shows:

```python
if raw:
    max_steps = int(raw)
    ...
else:
    max_steps = 5  # default when env var is absent
```

This branch (`else`) is reached when `raw == ""`, and **no exception is raised**, so **no warning is logged**. The breakdown's "Invalid values" list includes `""`, but the HLD's parsing logic treats `""` (absent) as a normal case, not an error.

**Resolution:** Follow the HLD (the design of record). Empty string → default 5, no warning. Only **non-empty invalid** values trigger the warning. Update Test 2 assertion for `""` to expect **no warning**.

### Test framework

- Use `unittest.mock` (Python standard library) or `pytest-mock` (if pytest is the test framework).
- Use `patch.dict("os.environ", ...)` to mock environment variables.
- Use `patch("module.Orchestrator")` to mock the constructor and capture arguments.
- Use `patch("module.logger.warning")` to mock logger calls and assert on warning message.

---

## Verification checklist

- [ ] All unit tests in `tests/test_http_server.py` pass (Tests 1–2).
- [ ] `python -m pytest tests/test_http_server.py -q` exits 0.
- [ ] Manual review: `Orchestrator` constructor in `run_server.py` receives `max_steps` from env var.
- [ ] Manual review: Invalid `AGENT_MAX_STEPS` logs warning with correct message.
- [ ] Manual review: Absent or empty `AGENT_MAX_STEPS` does **not** log warning (uses default 5 silently).
