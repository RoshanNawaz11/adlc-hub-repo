# Requirements — PER-189

**Story:** Flag catalogue for reviewers

**Source:** Ticket PER-189

---

## Context

Reviewers see flags like `invalidDates` or `netPayMismatch` on the review screen, but nothing lists which flags exist, which rule raises each one, or whether it asks them for missing input, a check, or a decision. Today someone has to search the YAML files under `rules/`.

Flags are defined inside the rule files: in a `cases` or `checks` entry's `then`, as `flag: {kind, type, title, ...}` or a `flags: [...]` list, also inside `lines` blocks (see `rules/README.md`).

This story builds a catalogue from those files, so it never drifts from the rules the engine runs.

---

## Requirements

### REQ-1: Command reads rule files and writes catalogue
**Given** rule YAML files under `rules/` (excluding `rules/tables/` and `rules/facts.yaml`)  
**When** `uv run python -m rules_engine.flag_catalogue` is run  
**Then** it reads every rule file under `rules/` (not `rules/tables/`, not `rules/facts.yaml`) and writes `docs/rules/flags.md`

---

### REQ-2: Catalogue organized by agent with flag details
**Given** flags extracted from rule files  
**When** the catalogue is generated  
**Then** the page has one table per agent (the rule's `agent` field): validation, compliance, payroll-calculation, then any other agent alphabetically  
**And** each row shows the flag `type`, its `kind` (input / check / decision), its `title`, and the rule `id` and `version` that raise it  
**And** a flag type raised by several rules gets one row per rule

---

### REQ-3: Deterministic output order
**Given** flag data extracted from rules  
**When** the catalogue is written  
**Then** rows are sorted by `type`, then rule `id`  
**And** running the command twice gives the same file byte for byte

---

### REQ-4: Flags without kind and unparseable rules
**Given** a flag with no `kind` field  
**When** the catalogue is generated  
**Then** it is shown as `—` in the table  
**And** also listed in a "Flags to fix" section at the end, with its rule id

**Given** a rule file that cannot be parsed  
**When** the command runs  
**Then** it stops with an error naming that file

---

### REQ-5: Check mode for drift detection
**Given** an existing `docs/rules/flags.md`  
**When** `--check` is passed  
**Then** nothing is written  
**And** the command exits 0 when `docs/rules/flags.md` is up to date  
**And** exits 1 with a message naming the agents whose tables differ when it is not

---

### REQ-6: No runtime changes
**Given** the agents and the rules engine  
**When** the catalogue command is added  
**Then** nothing the agents or the rules engine do at run time changes

---

### REQ-7: Test coverage
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

## Out of Scope

- Showing flags in the review UI
- Changing any rule or flag
- Documenting `note`s (notes are not flags)

---

## Open Questions

None.

---

## Notes

- Flag structure per `rules/README.md`: `flag: {kind, type, title, detail}` or `flags: [...]`
- Flags can appear in `cases`, `checks`, or `lines` blocks
- The `kind` field is required for proper categorization: `input`, `check`, or `decision`
- The catalogue is a reference document for reviewers, not a runtime component
