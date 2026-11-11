# Story 03-agent: CLI command for withdrawal — LLD

**Repo:** `agent`  
**Status:** Draft  
**After:** Story 1

---

## Acceptance Criteria (from Requirements & Breakdown)

### AC 1: Withdrawing a case — CLI interface

**Given** a case is paused waiting for answers to its flags (`status == "needs_input"`)  
**When** a reviewer withdraws it via the CLI with a valid reason  
**Then** the CLI prints "Case <thread_id> withdrawn: <reason>" and the timestamp, and the case transitions to `status == "withdrawn"`

---

**Given** a case is paused waiting for the final review (`status == "needs_review"`)  
**When** a reviewer withdraws it via the CLI with a valid reason  
**Then** the CLI prints "Case <thread_id> withdrawn: <reason>" and the timestamp, and the case transitions to `status == "withdrawn"`

---

**Given** a reviewer is withdrawing a case via the CLI  
**When** the reason is empty, or contains only spaces, tabs or line breaks  
**Then** the CLI prints "Error: Reason is required and cannot be empty" and the case remains paused

---

**Given** a reviewer is withdrawing a case via the CLI  
**When** the reason exceeds 500 characters (after trimming)  
**Then** the CLI prints "Error: Reason exceeds 500 characters" and the case remains paused

### AC 2: After withdrawing — CLI output

**Given** a case has been withdrawn via the CLI  
**When** the command completes  
**Then** the output contains "Case <thread_id> withdrawn: <trimmed reason>" and "Withdrawn at: <ISO 8601 timestamp>"

### AC 3: Cases that can't be withdrawn — CLI error messages

**Given** a case is still running (`status == "running"`)  
**When** a reviewer attempts to withdraw it via the CLI  
**Then** the CLI prints "Error: Case is running, wait until it pauses"

---

**Given** a case is done, rejected, or failed (`status in ("done", "rejected", "failed")`)  
**When** a reviewer attempts to withdraw it via the CLI  
**Then** the CLI prints "Error: Case is <status>, cannot withdraw"

---

**Given** a case does not exist (no checkpoint for the `thread_id`)  
**When** a reviewer attempts to withdraw it via the CLI  
**Then** the CLI prints "Error: Case not found"

---

**Given** a case has already been withdrawn  
**When** a reviewer attempts to withdraw it a second time via the CLI  
**Then** the CLI prints the existing withdrawal details ("Case was withdrawn on <timestamp>: <reason>")

### AC 4: Everywhere a case is handled — CLI

**Given** a paused case exists  
**When** a reviewer withdraws it from the CLI  
**Then** the withdrawal proceeds with the same state transition and validation rules as the API

---

## File Plan

### New Files

#### `src/agents/cli.py`

**Purpose:** CLI module providing the `withdraw-case` command. Implements argument parsing, local reason validation, calls the shared `withdraw_case` function from Story 1, and prints success or error messages to stdout/stderr.

**Imports:**

```python
from __future__ import annotations

import argparse
import asyncio
import logging
import os
import sys
from pathlib import Path

# Ensure src/ is on the path (matches the pattern in runner.py and server.py)
_SRC_ROOT = str(Path(__file__).resolve().parent.parent)
if _SRC_ROOT not in sys.path:
    sys.path.insert(0, _SRC_ROOT)

from agents.runtime.withdrawal import CaseNotFoundError, InvalidStateError, withdraw_case
from agents.runtime.run_dir import run_dir

logger = logging.getLogger(__name__)
```

**Command implementation:**

```python
async def _withdraw_case(
    thread_id: str,
    reason: str,
    reviewer_id: str,
    db_path: Path | None = None,
) -> int:
    """
    Withdraw a paused case with a reason.
    
    Returns:
        0 on success, 1 on error.
    """
    # Resolve checkpointer path (matches the API endpoint pattern)
    if db_path is None:
        db_path = run_dir(thread_id) / "checkpoint.db"
    checkpointer_kind = os.getenv("CHECKPOINTER", "sqlite")
    
    try:
        result = await withdraw_case(
            thread_id=thread_id,
            reason=reason,
            reviewer_id=reviewer_id,
            checkpointer_kind=checkpointer_kind,
            db_path=db_path,
        )
        # Success output (AC 2)
        print(f"Case {thread_id} withdrawn: {result['reason']}")
        print(f"Withdrawn at: {result['withdrawn_at']}")
        return 0
    except ValueError as exc:
        # Reason validation failures (AC 1)
        print(f"Error: {exc}", file=sys.stderr)
        return 1
    except CaseNotFoundError as exc:
        # Case not found (AC 9)
        print(f"Error: {exc}", file=sys.stderr)
        return 1
    except InvalidStateError as exc:
        # Invalid state: running, done/rejected/failed, already withdrawn (AC 3, 6, 8, 9)
        print(f"Error: {exc}", file=sys.stderr)
        return 1
    except Exception as exc:
        # Unexpected errors
        logger.exception("Withdrawal failed for %s", thread_id)
        print(f"Error: {exc}", file=sys.stderr)
        return 1


async def _cli(argv: list[str] | None = None) -> int:
    """
    CLI entrypoint for the withdraw-case command.
    
    Usage:
        python -m agents.cli withdraw-case <thread_id> --reason "<reason>" [--reviewer-id <id>] [--db-path <path>]
    """
    parser = argparse.ArgumentParser(
        prog="python -m agents.cli withdraw-case",
        description="Withdraw a paused case with a reason.",
    )
    parser.add_argument(
        "thread_id",
        help="The run's thread_id (e.g. a UUID or SO-1003)",
    )
    parser.add_argument(
        "--reason",
        required=True,
        help="The withdrawal reason (required, non-empty, ≤500 characters)",
    )
    parser.add_argument(
        "--reviewer-id",
        default=None,
        help="The reviewer's identifier (default: REVIEWER_ID env var or 'cli-user')",
    )
    parser.add_argument(
        "--db-path",
        type=Path,
        default=None,
        help="Path to the checkpoint.db file (default: run_dir(thread_id)/checkpoint.db)",
    )
    
    args = parser.parse_args(argv)
    
    # Resolve reviewer_id (matches the open question from Story 1/2)
    # Priority: CLI argument > environment variable > default
    reviewer_id = (
        args.reviewer_id
        or os.getenv("REVIEWER_ID")
        or "cli-user"
    )
    
    return await _withdraw_case(args.thread_id, args.reason, reviewer_id, args.db_path)


def main() -> None:
    """Entry point for `python -m agents.cli withdraw-case`."""
    raise SystemExit(asyncio.run(_cli()))


if __name__ == "__main__":
    main()
```

**Implementation notes:**

- **Path resolution:** Uses `run_dir(thread_id) / "checkpoint.db"` to resolve the db_path by default, matching the API endpoint pattern from Story 2. Accepts `--db-path` for testing and operational flexibility.
- **Checkpointer kind:** Reads `CHECKPOINTER` env var (default `"sqlite"`), matching `runner.py` and `server.py`.
- **Reviewer authentication:** Accepts `--reviewer-id` argument (optional), falls back to `REVIEWER_ID` env var, then to `"cli-user"`. This resolves the open question from Story 1 for the CLI context — a future auth enhancement can replace this.
- **Error handling:** Maps each exception type to an appropriate error message and exit code (0 success, 1 error). Prints errors to stderr per CLI convention.
- **Output format:** Success message matches the CLI contract from the HLD: "Case <thread_id> withdrawn: <reason>" and "Withdrawn at: <timestamp>".
- **Module structure:** Follows the pattern in `runner.py` (async `_cli` function called by synchronous `main`, `if __name__ == "__main__"` guard).

---

### Modified Files

None. This story only creates `src/agents/cli.py`; it imports and calls `withdraw_case` from `src/agents/runtime/withdrawal.py` (created in Story 1), but does not modify it.

---

## Interfaces

### `withdraw-case` CLI command

**Invocation:**

```bash
python -m agents.cli withdraw-case <thread_id> --reason "<reason>" [--reviewer-id <id>] [--db-path <path>]
```

**Arguments:**

- `thread_id` (positional, required): The run's unique identifier (matches `State["run_id"]` and the checkpointer's `thread_id`).
- `--reason` (required): The withdrawal reason. Trimmed and validated by the `withdraw_case` function (non-empty, ≤500 characters).
- `--reviewer-id` (optional): The authenticated reviewer's identifier (e.g. user ID, email). Defaults to `REVIEWER_ID` environment variable, or `"cli-user"` if neither is provided.
- `--db-path` (optional): Path to the checkpoint.db file. Defaults to `run_dir(thread_id)/checkpoint.db`. Added for testing and operational flexibility.

**Success output (stdout):**

```
Case <thread_id> withdrawn: <trimmed reason>
Withdrawn at: <ISO 8601 timestamp in UTC>
```

**Error output (stderr):**

```
Error: Reason is required and cannot be empty
```

or

```
Error: Reason exceeds 500 characters
```

or

```
Error: Case is running, wait until it pauses
```

or

```
Error: Case is <status>, cannot withdraw
```

or

```
Error: Case <thread_id> not found
```

or

```
Error: Case was withdrawn on <timestamp>: <reason>
```

**Exit codes:**

- `0`: Success (case withdrawn)
- `1`: Error (validation failure, invalid state, case not found, or unexpected exception)

---

### Integration with Story 1

The CLI command calls `withdraw_case` from `src/agents/runtime/withdrawal.py` with the same parameters as the API endpoint (Story 2):

```python
result = await withdraw_case(
    thread_id=thread_id,
    reason=reason,
    reviewer_id=reviewer_id,
    checkpointer_kind=checkpointer_kind,
    db_path=db_path,
)
```

The function's return value (`{"status": "withdrawn", "reason": "<trimmed>", "withdrawn_at": "<timestamp>"}`) is used to construct the success output message.

The function's exceptions (`ValueError`, `InvalidStateError`, `CaseNotFoundError`) are caught and mapped to CLI error messages and exit codes.

---

## Algorithm Notes

### Command flow

```
1. Parse CLI arguments (thread_id, --reason, --reviewer-id, --db-path)
2. Resolve reviewer_id: CLI arg > REVIEWER_ID env var > "cli-user"
3. Resolve checkpointer path: --db-path arg or run_dir(thread_id) / "checkpoint.db"
4. Resolve checkpointer kind: CHECKPOINTER env var (default "sqlite")
5. Call withdraw_case(thread_id, reason, reviewer_id, checkpointer_kind, db_path)
6. On success:
   - Print "Case <thread_id> withdrawn: <result['reason']>"
   - Print "Withdrawn at: <result['withdrawn_at']>"
   - Exit 0
7. On ValueError (reason validation):
   - Print "Error: <exception message>" to stderr
   - Exit 1
8. On CaseNotFoundError:
   - Print "Error: <exception message>" to stderr
   - Exit 1
9. On InvalidStateError (state transition failure):
   - Print "Error: <exception message>" to stderr
   - Exit 1
10. On unexpected exception:
    - Log exception at ERROR level
    - Print "Error: <exception message>" to stderr
    - Exit 1
```

### Error message mapping

The error messages printed by the CLI match the exception messages from `withdraw_case` (Story 1):

- `ValueError("Reason is required and cannot be empty")` → `"Error: Reason is required and cannot be empty"`
- `ValueError("Reason exceeds 500 characters")` → `"Error: Reason exceeds 500 characters"`
- `InvalidStateError("Case is running, wait until it pauses")` → `"Error: Case is running, wait until it pauses"`
- `InvalidStateError("Case is <status>, cannot withdraw")` → `"Error: Case is <status>, cannot withdraw"`
- `CaseNotFoundError("Case <thread_id> not found")` → `"Error: Case <thread_id> not found"`
- `InvalidStateError("Case was withdrawn on <timestamp>: <reason>")` → `"Error: Case was withdrawn on <timestamp>: <reason>"`

The CLI does not transform the messages; it prints them verbatim (with the `"Error: "` prefix) to stderr.

---

### Module discovery

The command is invoked as `python -m agents.cli withdraw-case`. Python's `-m` flag:

1. Looks for `agents/cli.py` on the module path.
2. Executes it as `__main__` (the `if __name__ == "__main__"` guard triggers).
3. Calls `main()`, which runs `asyncio.run(_cli())`.

The `_cli` function parses the arguments (everything after `withdraw-case` on the command line) and dispatches to `_withdraw_case`.

**Alternative implementation (subcommand pattern):**

If the CLI module is extended in the future with other commands (e.g. `python -m agents.cli list-cases`, `python -m agents.cli resume-case`), the argument parser should be restructured to use subparsers:

```python
async def _cli(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="python -m agents.cli")
    subparsers = parser.add_subparsers(dest="command", required=True)
    
    # withdraw-case subcommand
    withdraw_parser = subparsers.add_parser("withdraw-case", help="Withdraw a paused case")
    withdraw_parser.add_argument("thread_id")
    withdraw_parser.add_argument("--reason", required=True)
    withdraw_parser.add_argument("--reviewer-id", default=None)
    withdraw_parser.add_argument("--db-path", type=Path, default=None)
    
    # (future: other subcommands here)
    
    args = parser.parse_args(argv)
    
    if args.command == "withdraw-case":
        reviewer_id = args.reviewer_id or os.getenv("REVIEWER_ID") or "cli-user"
        return await _withdraw_case(args.thread_id, args.reason, reviewer_id, args.db_path)
    
    # (future: dispatch to other commands)
    
    return 1
```

For this story, the simpler single-command parser is sufficient (the user must type `python -m agents.cli withdraw-case <args>`). The breakdown and ticket do not specify other CLI commands, so the subcommand pattern is deferred.

---

## Tests Section

**File:** `tests/test_withdrawal_cli.py` (new file)

### End-to-end tests (5 total)

All tests invoke the CLI as a subprocess (using `subprocess.run`) and assert on the exit code, stdout, and stderr. The tests create a case in a known state (via the shared `setup_case` helper), then invoke the CLI and verify the output.

**Setup helper (shared across tests):**

```python
import os
import subprocess
import tempfile
from pathlib import Path
from typing import Any

import pytest

from agents.graph.states import State
from agents.runtime.checkpointer import get_checkpointer

# Helper to create a case in a known state (reusable from Story 1/2 tests)
async def setup_case(thread_id: str, status: str, **extra_state: Any) -> Path:
    """
    Write a case with the given status to the checkpointer.
    
    Returns:
        The path to the run directory (for cleanup).
    """
    # Use a temp directory for the run (avoids polluting logs/ during tests)
    test_dir = Path(tempfile.mkdtemp())
    db_path = test_dir / "checkpoint.db"
    
    checkpointer = get_checkpointer("sqlite", db_path)
    await checkpointer.setup()
    
    state: State = {
        "schema_version": 1,
        "run_id": thread_id,
        "status": status,
        "flagged": False,
        "results": {},
        "steps": [],
        "flags": [],
        "needs": [],
        "runs": {},
        **extra_state,
    }
    
    config = {"configurable": {"thread_id": thread_id}}
    
    # Create a minimal checkpoint (sketch; may need adjustment for real LangGraph schema)
    from langgraph.checkpoint.base import Checkpoint
    import uuid
    checkpoint = Checkpoint(
        v=1,
        id=str(uuid.uuid4()),
        ts="2026-10-15T14:00:00Z",
        channel_values={"state": state},
        channel_versions={},
        versions_seen={},
        pending_sends=[],
    )
    await checkpointer.aput(config, checkpoint, state, {})
    
    conn = getattr(checkpointer, "conn", None)
    if conn is not None:
        await conn.close()
    
    return test_dir


def run_cli(*args: str, env: dict[str, str] | None = None) -> subprocess.CompletedProcess:
    """
    Run the CLI command as a subprocess.
    
    Returns:
        CompletedProcess with returncode, stdout, stderr.
    """
    cmd = ["python", "-m", "agents.cli", "withdraw-case", *args]
    env_merged = {**os.environ, **(env or {})}
    return subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        env=env_merged,
    )
```

---

**Test 1: Successful withdrawal from `needs_input`**

```python
@pytest.mark.asyncio
async def test_withdraw_from_needs_input_success() -> None:
    """AC 1, AC 2: A case waiting for flag answers can be withdrawn via CLI, prints success message."""
    thread_id = "test-cli-1"
    test_dir = await setup_case(thread_id, "needs_input")
    
    result = run_cli(
        thread_id,
        "--reason", "Test withdrawal",
        "--db-path", str(test_dir / "checkpoint.db"),
    )
    
    assert result.returncode == 0
    assert "Case test-cli-1 withdrawn: Test withdrawal" in result.stdout
    assert "Withdrawn at:" in result.stdout
    
    # Cleanup
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 2: Empty reason rejected**

```python
@pytest.mark.asyncio
async def test_withdraw_empty_reason_rejected() -> None:
    """AC 1: An empty reason (after trimming) prints error and exits 1."""
    thread_id = "test-cli-2"
    test_dir = await setup_case(thread_id, "needs_input")
    
    result = run_cli(
        thread_id,
        "--reason", "   ",
        "--db-path", str(test_dir / "checkpoint.db"),
    )
    
    assert result.returncode == 1
    assert "Error: Reason is required and cannot be empty" in result.stderr
    
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 3: Case running rejected**

```python
@pytest.mark.asyncio
async def test_withdraw_running_case_rejected() -> None:
    """AC 8: Withdrawing a running case prints error and exits 1."""
    thread_id = "test-cli-3"
    test_dir = await setup_case(thread_id, "running")
    
    result = run_cli(
        thread_id,
        "--reason", "Test",
        "--db-path", str(test_dir / "checkpoint.db"),
    )
    
    assert result.returncode == 1
    assert "Error: Case is running, wait until it pauses" in result.stderr
    
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 4: Case not found**

```python
@pytest.mark.asyncio
async def test_withdraw_not_found() -> None:
    """AC 9: Withdrawing a non-existent case prints error and exits 1."""
    thread_id = "test-cli-4-nonexistent"
    # No case created; db_path points to an empty directory
    test_dir = Path(tempfile.mkdtemp())
    
    result = run_cli(
        thread_id,
        "--reason", "Test",
        "--db-path", str(test_dir / "checkpoint.db"),
    )
    
    assert result.returncode == 1
    assert "Error: Case" in result.stderr
    assert "not found" in result.stderr
    
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test 5: Already withdrawn case prints existing details**

```python
@pytest.mark.asyncio
async def test_withdraw_already_withdrawn_prints_details() -> None:
    """AC 6: Withdrawing an already-withdrawn case prints existing details and exits 1."""
    thread_id = "test-cli-5"
    test_dir = await setup_case(
        thread_id,
        "withdrawn",
        withdrawn_at="2026-10-14T10:00:00Z",
        withdrawn_by="reviewer-1",
        withdrawal_reason="Original reason",
    )
    
    result = run_cli(
        thread_id,
        "--reason", "Second attempt",
        "--db-path", str(test_dir / "checkpoint.db"),
    )
    
    assert result.returncode == 1
    assert "Error: Case was withdrawn on 2026-10-14T10:00:00Z: Original reason" in result.stderr
    
    import shutil
    shutil.rmtree(test_dir)
```

---

**Test count alignment with breakdown:** 5 tests (1 success case, 4 error cases).

**Coverage:**

- AC 1 (withdrawing a case): Tests 1, 2
- AC 2 (after withdrawing): Test 1
- AC 3 (cases that can't be withdrawn): Tests 3, 4, 5
- AC 4 (everywhere a case is handled): All tests (CLI uses the same `withdraw_case` function as the API)

---

## Open Questions (carried forward from Story 1 / HLD)

1. **Reviewer authentication:** The CLI accepts `--reviewer-id` as an optional argument, falls back to `REVIEWER_ID` env var, then to `"cli-user"`. When authentication is implemented, the CLI may need to accept a token or session file instead of a plain reviewer_id. This is a future enhancement; the placeholder approach is acceptable for this story.

2. **Database path resolution:** The CLI resolves `db_path` as `run_dir(thread_id) / "checkpoint.db"` by default, with an optional `--db-path` argument added for testing and operational flexibility. This extension is not in the original CLI contract from the HLD, but is a small, useful addition.

3. **CLI module location:** The LLD creates `src/agents/cli.py` as a new module, invoked as `python -m agents.cli withdraw-case`. The HLD asked whether the command should live in a new module, be added to `run_so.py`, or exist as a standalone script. The new module approach matches the structure of `runner.py` and `server.py` (both have `main()` entry points), and allows future expansion (other CLI commands can be added to `cli.py` as subcommands).

---

**End of LLD.**
