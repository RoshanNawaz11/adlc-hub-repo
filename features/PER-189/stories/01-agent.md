# Story PER-189 — Flag catalogue for reviewers

**Type:** story  
**Status:** draft  
**Track:** Quick  
**Epic:** (none — standalone backlog story)

---

## Description

Reviewers see flags like `invalidDates` or `netPayMismatch` on the review screen, but nothing lists which flags exist, which rule raises each one, or whether it asks them for missing input, a check, or a decision. Today someone has to search the YAML files under `rules/`.

Flags are defined inside the rule files: in a `cases` or `checks` entry's `then`, as `flag: {kind, type, title, ...}` or a `flags: [...]` list, also inside `lines` blocks (see `rules/README.md`). This story builds a catalogue from those files, so it never drifts from the rules the engine runs.

---

## Acceptance Criteria

### AC-1: Command reads rule files and writes catalogue
**Given** rule YAML files under `rules/` (excluding `rules/tables/` and `rules/facts.yaml`)  
**When** `uv run python -m rules_engine.flag_catalogue` is run  
**Then** it reads every rule file under `rules/` (not `rules/tables/`, not `rules/facts.yaml`) and writes `docs/rules/flags.md`

### AC-2: Catalogue organized by agent with flag details
**Given** flags extracted from rule files  
**When** the catalogue is generated  
**Then** the page has one table per agent (the rule's `agent` field): validation, compliance, payroll-calculation, then any other agent alphabetically  
**And** each row shows the flag `type`, its `kind` (input / check / decision), its `title`, and the rule `id` and `version` that raise it  
**And** a flag type raised by several rules gets one row per rule

### AC-3: Deterministic output order
**Given** flag data extracted from rules  
**When** the catalogue is written  
**Then** rows are sorted by `type`, then rule `id`  
**And** running the command twice gives the same file byte for byte

### AC-4: Flags without kind and unparseable rules
**Given** a flag with no `kind` field  
**When** the catalogue is generated  
**Then** it is shown as `—` in the table  
**And** also listed in a "Flags to fix" section at the end, with its rule id

**Given** a rule file that cannot be parsed  
**When** the command runs  
**Then** it stops with an error naming that file

### AC-5: Check mode for drift detection
**Given** an existing `docs/rules/flags.md`  
**When** `--check` is passed  
**Then** nothing is written  
**And** the command exits 0 when `docs/rules/flags.md` is up to date  
**And** exits 1 with a message naming the agents whose tables differ when it is not

### AC-6: No runtime changes
**Given** the agents and the rules engine  
**When** the catalogue command is added  
**Then** nothing the agents or the rules engine do at run time changes

### AC-7: Test coverage
**Given** the catalogue implementation  
**When** tests are written  
**Then** they cover:
- A flag in `cases`
- A flag in `checks`
- A flag inside `lines`
- A `flags:` list
- Agent order and row order
- The "Flags to fix" section
- A broken rule file
- `--check` passing
- `--check` failing

---

## Low-Level Design

### File Plan

#### `src/rules_engine/flag_catalogue.py` (new)

**Module-level constants:**
- `RULES_DIR: Path` — resolved from `rules_engine.evaluate.RULES_DIR`
- `AGENT_ORDER: list[str]` — `["validation", "compliance", "payroll-calculation"]` (others sorted alphabetically)

**Functions:**

```python
def parse_rule_file(path: Path) -> dict[str, Any]
```
Loads and parses one YAML rule file using `yaml.safe_load`. On parse failure, raises `yaml.YAMLError` with the file name in the message.

```python
def extract_flags(rule: dict[str, Any]) -> list[dict[str, Any]]
```
Walks the rule's structure to find all flags:
- In `cases`: each case's `then` may have `flag: {...}` or `flags: [...]`
- In `checks`: each check's `then` may have `flag: {...}` or `flags: [...]`
- In `lines`: each line block has its own `cases` and `checks` — recurse into them

Returns a list of flag records: `[{"type": str, "kind": str | None, "title": str, "rule_id": str, "version": int}, ...]`. Sets `kind` to `None` when the flag has no `kind` field.

**Algorithm:**
1. Initialize empty list `flags`
2. Walk `rule.get("checks", [])` — for each check's `then`, call `_extract_from_then`
3. Walk `rule.get("cases", [])` — for each case's `then`, call `_extract_from_then`
4. Walk `rule.get("lines", [])` — for each line block:
   - Walk `block.get("checks", [])` — for each check's `then`, call `_extract_from_then`
   - Walk `block.get("cases", [])` — for each case's `then`, call `_extract_from_then`
5. Return `flags`

```python
def _extract_from_then(then: dict[str, Any], rule_id: str, version: int, flags: list[dict[str, Any]]) -> None
```
Helper that extracts flags from a single `then` dict:
- If `then` has `"flag"`, append one flag record
- If `then` has `"flags"`, append one flag record per list item
- Each flag record: `{"type": flag["type"], "kind": flag.get("kind"), "title": flag.get("title", ""), "rule_id": rule_id, "version": version}`

```python
def build_catalogue(rules_dir: Path) -> dict[str, Any]
```
Orchestrates parsing and extraction:
1. Gather all rule files: `sorted(rules_dir.glob("*/*.yaml"))`, excluding `rules_dir / "tables"` and `rules_dir / "facts.yaml"`
2. For each file:
   - Call `parse_rule_file(path)` (let `yaml.YAMLError` propagate with file name)
   - Call `extract_flags(rule)` to get flag records
   - Group by `rule["agent"]`
3. Build `agents` dict: key = agent name, value = list of flag records
4. Sort agents: `AGENT_ORDER` entries first (in that order), then others alphabetically
5. For each agent, sort its flag records: first by `type` (ascending), then by `rule_id` (ascending)
6. Build `flags_to_fix`: all flag records where `kind is None`, sorted by `type` then `rule_id`
7. Return `{"agents": agents, "flags_to_fix": flags_to_fix}`

```python
def render_markdown(catalogue: dict[str, Any]) -> str
```
Renders the full markdown document:
1. Start with `"# Rule Flags Catalogue\n\n"`
2. For each agent in `catalogue["agents"]`:
   - Add `f"## {agent}\n\n"`
   - Add table header: `"| Flag Type | Kind | Title | Rule | Version |\n|-----------|------|-------|------|---------|"`
   - For each flag in agent's list:
     - `kind_str = flag["kind"] if flag["kind"] else "—"`
     - Add row: `f"| {flag['type']} | {kind_str} | {flag['title']} | {flag['rule_id']} | {flag['version']} |"`
   - Add `"\n"`
3. If `catalogue["flags_to_fix"]` is not empty:
   - Add `"\n---\n\n## Flags to fix\n\nThe following flags are missing a `kind` field:\n\n"`
   - Add table header: `"| Flag Type | Rule | Version |\n|-----------|------|---------|"`
   - For each flag in `flags_to_fix`:
     - Add row: `f"| {flag['type']} | {flag['rule_id']} | {flag['version']} |"`
4. Return the concatenated string

```python
def write_catalogue(output_path: Path, content: str) -> None
```
Writes `content` to `output_path`, creating parent directories if needed (`output_path.parent.mkdir(parents=True, exist_ok=True)`). Uses UTF-8 encoding.

```python
def check_up_to_date(output_path: Path, content: str) -> tuple[bool, str | None]
```
Compares `content` with the existing file at `output_path`:
- If file does not exist, return `(False, "all agents (file missing)")`
- Read existing content (UTF-8)
- If identical, return `(True, None)`
- Otherwise, parse both to find which agent sections differ:
  - Split by `"## "` to get sections
  - Compare section by section
  - Collect agent names whose sections differ
  - Return `(False, f"agents: {', '.join(differing_agents)}")`

```python
def main(argv: list[str] | None = None) -> int
```
CLI entry point:
1. Parse `argv` (default `sys.argv[1:]`): look for `--check` flag
2. Resolve paths:
   - `rules_dir = RULES_DIR` (from `rules_engine.evaluate`)
   - `output_path = Path(__file__).resolve().parents[3] / "docs" / "rules" / "flags.md"`
3. Try:
   - `catalogue = build_catalogue(rules_dir)`
   - `content = render_markdown(catalogue)`
   - If `--check`:
     - `up_to_date, message = check_up_to_date(output_path, content)`
     - If `up_to_date`: return `0`
     - Else: print `f"Catalogue is out of date. {message}"` to stderr, return `1`
   - Else:
     - `write_catalogue(output_path, content)`
     - Print `f"Wrote {output_path}"` to stdout
     - Return `0`
4. Except `yaml.YAMLError` as e:
   - Print `f"Error parsing {<file from exception>}: {e}"` to stderr
   - Return `1`
5. Except any other exception as e:
   - Print `f"Error: {e}"` to stderr
   - Return `1`

#### `src/rules_engine/__main__.py` (new)

```python
import sys
from rules_engine.flag_catalogue import main

if __name__ == "__main__":
    sys.exit(main())
```

Allows `python -m rules_engine.flag_catalogue` to invoke `main()`.

#### `tests/test_flag_catalogue.py` (new)

**Test cases (matching AC-7):**

```python
def test_flag_in_cases()
```
Creates a minimal rule with `cases: [{"when": true, "then": {"flag": {"kind": "check", "type": "testFlag", "title": "Test"}}}]`. Calls `extract_flags(rule)`. Asserts the flag is in the returned list with correct `type`, `kind`, `title`, `rule_id`, `version`.

```python
def test_flag_in_checks()
```
Creates a minimal rule with `checks: [{"when": true, "then": {"flag": {"kind": "input", "type": "checkFlag", "title": "Check"}}}]`. Calls `extract_flags(rule)`. Asserts the flag is in the returned list.

```python
def test_flag_inside_lines()
```
Creates a minimal rule with `lines: [{"for_each": "items", "cases": [{"when": true, "then": {"flag": {"kind": "decision", "type": "lineFlag", "title": "Line"}}}]}]`. Calls `extract_flags(rule)`. Asserts the flag is in the returned list.

```python
def test_flags_list()
```
Creates a minimal rule with `cases: [{"when": true, "then": {"flags": [{"kind": "check", "type": "flag1", "title": "One"}, {"kind": "input", "type": "flag2", "title": "Two"}]}}]`. Calls `extract_flags(rule)`. Asserts both flags are in the returned list.

```python
def test_agent_order_and_row_order(tmp_path)
```
Creates three rule files in `tmp_path`: one for `validation`, one for `compliance`, one for `payroll-calculation`, one for `zeta` (alphabetically after). Each rule has flags with different `type` values (unsorted). Calls `build_catalogue(tmp_path)`. Asserts:
- Agents appear in order: `["validation", "compliance", "payroll-calculation", "zeta"]`
- Within each agent, flags are sorted by `type` then `rule_id`

```python
def test_flags_to_fix_section(tmp_path)
```
Creates a rule file with a flag missing `kind` (`{"type": "noKind", "title": "No kind"}`). Calls `build_catalogue(tmp_path)`. Asserts `catalogue["flags_to_fix"]` contains the flag with `kind=None`. Calls `render_markdown(catalogue)`. Asserts the output contains the "Flags to fix" section with the flag's `type`, `rule_id`, `version`.

```python
def test_broken_rule_file(tmp_path)
```
Writes an invalid YAML file to `tmp_path / "agent" / "broken.yaml"` (e.g., `"{invalid"`). Calls `parse_rule_file(path)`. Asserts it raises `yaml.YAMLError` with the file name in the message.

```python
def test_check_mode_passing(tmp_path)
```
Creates a rule file, builds catalogue, renders markdown, writes to `output_path`. Calls `check_up_to_date(output_path, content)`. Asserts it returns `(True, None)`.

```python
def test_check_mode_failing(tmp_path)
```
Creates a rule file, builds catalogue, renders markdown, writes to `output_path`. Modifies one agent's flags in the catalogue, renders new markdown. Calls `check_up_to_date(output_path, new_content)`. Asserts it returns `(False, <message naming the changed agent>)`.

```python
def test_main_writes_catalogue(tmp_path, monkeypatch, capsys)
```
Monkeypatches `RULES_DIR` and output path to `tmp_path`. Creates a minimal rule file. Calls `main([])` (no `--check`). Asserts:
- Return code is `0`
- Output file exists and contains expected markdown
- Stdout contains `"Wrote"`

```python
def test_main_check_up_to_date(tmp_path, monkeypatch, capsys)
```
Same setup, writes catalogue, then calls `main(["--check"])`. Asserts return code is `0`, nothing written to stdout.

```python
def test_main_check_out_of_date(tmp_path, monkeypatch, capsys)
```
Same setup, writes catalogue, modifies the file, calls `main(["--check"])`. Asserts return code is `1`, stderr contains `"out of date"` and agent names.

```python
def test_main_parse_error(tmp_path, monkeypatch, capsys)
```
Writes an invalid YAML file, calls `main([])`. Asserts return code is `1`, stderr contains `"Error parsing"` and the file name.

---

## Interfaces

### CLI

**Command:** `uv run python -m rules_engine.flag_catalogue [--check]`

**Arguments:**
- `--check` (optional) — check mode; compares generated content with existing file

**Exit codes:**
- `0` — success (catalogue written, or up to date in `--check`)
- `1` — failure (parse error, drift in `--check`, or other error)

**Stdout:**
- Normal mode: `"Wrote <path>"`
- Check mode (up to date): nothing

**Stderr:**
- Parse error: `"Error parsing <file>: <error>"`
- Drift in check mode: `"Catalogue is out of date. agents: <names>"` or `"Catalogue is out of date. all agents (file missing)"`
- Other error: `"Error: <message>"`

### Internal APIs

**`parse_rule_file(path: Path) -> dict[str, Any]`**
- Input: path to a `.yaml` rule file
- Output: parsed rule dict
- Raises: `yaml.YAMLError` on parse failure (message includes file name)

**`extract_flags(rule: dict[str, Any]) -> list[dict[str, Any]]`**
- Input: parsed rule dict
- Output: list of flag records `[{"type": str, "kind": str | None, "title": str, "rule_id": str, "version": int}, ...]`
- Guarantees: `kind` is `None` when the flag has no `kind` field; empty list when no flags

**`build_catalogue(rules_dir: Path) -> dict[str, Any]`**
- Input: path to `rules/` directory
- Output: `{"agents": {agent_name: [flag_records]}, "flags_to_fix": [flag_records]}`
- Guarantees:
  - Agents are ordered: `validation`, `compliance`, `payroll-calculation`, then others alphabetically
  - Each agent's flag records are sorted by `type`, then `rule_id`
  - `flags_to_fix` contains all flags where `kind is None`, sorted by `type` then `rule_id`
- Raises: `yaml.YAMLError` on parse failure (propagated from `parse_rule_file`)

**`render_markdown(catalogue: dict[str, Any]) -> str`**
- Input: catalogue dict
- Output: markdown string
- Guarantees: deterministic output (same input → same output, byte for byte)

**`write_catalogue(output_path: Path, content: str) -> None`**
- Input: output path, markdown content
- Side effect: writes file (UTF-8), creating parent directories if needed
- Raises: `OSError` on write failure

**`check_up_to_date(output_path: Path, content: str) -> tuple[bool, str | None]`**
- Input: output path, generated markdown content
- Output: `(True, None)` when existing file matches `content`; `(False, <message>)` when different or missing
- Message format: `"agents: <comma-separated names>"` or `"all agents (file missing)"`

**`main(argv: list[str] | None = None) -> int`**
- Input: CLI arguments (default `sys.argv[1:]`)
- Output: exit code (0 on success, 1 on failure)
- Side effects: writes file (normal mode), prints to stdout/stderr

---

## Algorithm Notes

### Flag Extraction

The `extract_flags` function uses a recursive helper `_extract_from_then` to walk the rule structure. The algorithm:

1. For each `then` dict encountered (in `cases`, `checks`, or inside `lines` blocks):
   - If `"flag"` key exists, append one flag record
   - If `"flags"` key exists, append one flag record per list item
2. Each flag record captures: `type`, `kind` (or `None`), `title`, `rule_id`, `version`
3. Recursion: for each `lines` block, walk its `checks` and `cases` with the same logic

**Handling missing `kind`:**
- If a flag dict has no `"kind"` key, `flag.get("kind")` returns `None`
- This `None` is preserved in the flag record and later identified as a "flag to fix"

### Catalogue Building

The `build_catalogue` function:

1. Scans `rules_dir` for `*.yaml` files, excluding `tables/` and `facts.yaml`
2. For each file:
   - Parses it (letting `yaml.YAMLError` propagate with file name)
   - Extracts flags
   - Groups by `rule["agent"]`
3. Sorts agents: `AGENT_ORDER` entries first (preserving their order), then others alphabetically
4. Within each agent, sorts flag records: primary key `type` (ascending), secondary key `rule_id` (ascending)
5. Identifies flags to fix: all flag records where `kind is None`, sorted the same way

**Deterministic output:**
- Sorting is stable and explicit (primary then secondary key)
- Iteration order over agents is controlled (not dict order)
- Result: same input rules → same output, byte for byte

### Markdown Rendering

The `render_markdown` function builds a string incrementally:

1. Header: `"# Rule Flags Catalogue\n\n"`
2. For each agent (in the sorted order from `catalogue["agents"]`):
   - Agent section header: `f"## {agent}\n\n"`
   - Table header (fixed)
   - One row per flag in the agent's sorted list
   - Blank line after table
3. If `flags_to_fix` is not empty:
   - Section divider: `"\n---\n\n"`
   - "Flags to fix" header and explanation
   - Table header (flag type, rule, version)
   - One row per flag in `flags_to_fix`

**`kind` display:**
- When `kind is None`, the table cell shows `"—"`
- Otherwise, shows the `kind` value as is (input / check / decision)

### Check Mode

The `check_up_to_date` function:

1. Reads the existing file (UTF-8)
2. Compares the full content string with the generated content
3. If identical, returns `(True, None)`
4. Otherwise, parses both to find which agent sections differ:
   - Splits by `"## "` to extract sections
   - Compares section-by-section
   - Collects agent names whose sections differ (or `"all agents"` when file is missing)
   - Returns `(False, f"agents: {', '.join(names)}")`

**Edge cases:**
- Missing file: returns `(False, "all agents (file missing)")`
- Empty diff (identical content): returns `(True, None)`
- Partial diff (one agent changed): names only the changed agents

---

## Tests

Tests in `tests/test_flag_catalogue.py` cover AC-7 (9 test cases mapping 1:1 to the verification approach count in breakdown):

1. **Flag in cases** — `test_flag_in_cases`: rule with `flag: {...}` in a case's `then`; `extract_flags` returns it
2. **Flag in checks** — `test_flag_in_checks`: rule with `flag: {...}` in a check's `then`; `extract_flags` returns it
3. **Flag inside lines** — `test_flag_inside_lines`: rule with a `lines` block whose case has a flag; `extract_flags` returns it
4. **Flags list** — `test_flags_list`: rule with `flags: [...]`; `extract_flags` returns every flag
5. **Agent order and row order** — `test_agent_order_and_row_order`: rules for validation, compliance, payroll-calculation, and `zeta`; `build_catalogue` orders agents correctly and sorts rows by `type` then `rule_id`
6. **Flags to fix section** — `test_flags_to_fix_section`: rule with a flag missing `kind`; catalogue's `flags_to_fix` includes it; markdown contains "Flags to fix" section
7. **Broken rule file** — `test_broken_rule_file`: invalid YAML file; `parse_rule_file` raises `yaml.YAMLError` naming the file
8. **Check mode passing** — `test_check_up_to_date`: existing file matches generated content; `check_up_to_date` returns `(True, None)`; also tests `test_main_check_up_to_date` for integration
9. **Check mode failing** — `test_check_out_of_date`: existing file differs; `check_up_to_date` returns `(False, <message>)`; also tests `test_main_check_out_of_date` for integration with agent names in message

**Additional integration tests:**
- `test_main_writes_catalogue`: `main([])` writes the file and exits 0
- `test_main_parse_error`: `main([])` with invalid YAML exits 1 and prints error

**Test utilities:**
- `tmp_path` fixture (pytest built-in) for isolated rule directories
- `monkeypatch` fixture to override `RULES_DIR` and output path
- `capsys` fixture to capture stdout/stderr
- Helper to create minimal rule YAML files for each test case

---

## Notes

- **No runtime changes:** The catalogue command is standalone; it does not modify `src/rules_engine/evaluate.py` or any agent code. The existing `RULES_DIR` constant is reused (via import).
- **YAML library:** Uses `yaml.safe_load` (same as `evaluate.py`) to parse rule files.
- **Path resolution:** `RULES_DIR` is imported from `rules_engine.evaluate`; output path is `docs/rules/flags.md` (resolved from `__file__` to avoid hard-coding the repo root).
- **Idempotency:** Running the command twice with unchanged rules produces the same output, byte for byte (guaranteed by deterministic sorting and stable string building).
- **Error handling:** Parse errors name the file and stop immediately (no partial catalogue); other errors (e.g., write failures) are caught and reported with exit code 1.
- **Agent order:** `AGENT_ORDER = ["validation", "compliance", "payroll-calculation"]` matches the order given in AC-2; other agents (if any) follow alphabetically.
- **Flag fields:** Only `type`, `kind`, `title`, `rule_id`, `version` are shown; `detail` is not catalogued (per Out of Scope).
