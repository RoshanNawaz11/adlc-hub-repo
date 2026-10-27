# Tech Spec — Story 04-agent

**ticket:** 04-agent  
**title:** Document AGENT_MAX_STEPS and multi-step capability in README  
**story points:** 2  
**made from:** PER-56 requirements + HLD + breakdown  
**source:** designer (LLD)

---

## Acceptance criteria

From requirements.md and breakdown.md:

1. **Environment-variables table includes AGENT_MAX_STEPS:** README contains a row for `AGENT_MAX_STEPS` with description "Maximum tool-use steps before forcing a final answer" and default value `5`.
2. **Capability description present:** README includes one sentence describing the multi-step tool-use capability: "The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit."

---

## File plan

### Files to modify

1. **`README.md`**
   - **Lines affected:** ~5–10
   - **Changes:**
     - Add `AGENT_MAX_STEPS` row to environment-variables table (create table if it does not exist).
     - Add one sentence describing multi-step tool-use capability to existing "Features" section, or create a "Capabilities" section if no suitable section exists.
     - Match existing README style (capitalization, punctuation, markdown formatting).

### Files to read (context only, no changes)

- **`README.md`** — scan for existing environment-variables table, "Features" section, or similar documentation structure to determine where to add new content.

---

## Interfaces

### Environment-variables table format

**If table exists:**

Append new row to existing table:

```markdown
| Variable | Description | Default |
|----------|-------------|---------|
| (existing rows...) |  |  |
| `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
```

**If table does not exist:**

Create new table with header and `AGENT_MAX_STEPS` row:

```markdown
## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
```

**Key points:**

- Use backticks around variable name: `` `AGENT_MAX_STEPS` ``.
- Description is a complete sentence (no trailing period, to match common table style).
- Default value is in backticks: `` `5` ``.
- Table uses standard markdown pipe format with header separator (`|---|---|---|`).

---

### Capability description format

**If "Features" section exists:**

Append sentence to existing list or paragraph:

```markdown
## Features

- (existing features...)
- The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.
```

**If no suitable section exists:**

Create new "Capabilities" section:

```markdown
## Capabilities

The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.
```

**Key points:**

- Sentence is exactly as specified in requirements (do not paraphrase).
- Capitalization: "The orchestrator" (lowercase "o" if mid-sentence), "LLM" (uppercase).
- Period at end of sentence.
- If section uses bulleted list, add as bullet; if prose, append as paragraph.

---

## Algorithm notes

### Locating insertion points

**Environment-variables table:**

1. Search README for existing table with headers like "Environment Variables", "Configuration", or "Env Vars".
2. If found, locate last row and append new row after it.
3. If not found, create new "## Environment Variables" section (suggest placement: after "## Installation" or "## Usage", before "## Development").

**Capability description:**

1. Search README for sections like "## Features", "## Capabilities", "## Overview", "## Usage".
2. If "Features" section exists (most likely), append sentence as bullet or paragraph.
3. If no suitable section exists, create new "## Capabilities" section (suggest placement: after "## Overview" or before "## Installation").

### Style matching

1. Read existing README headers to determine heading level (usually `##` for main sections).
2. Read existing lists to determine bullet style (`-` vs `*`).
3. Read existing variable names in text to confirm backtick usage (e.g., `` `PORT` `` vs `PORT`).
4. Match capitalization of existing headings ("Environment Variables" vs "Environment variables").

---

## Tests

### Test 1: Environment-variables table includes AGENT_MAX_STEPS (AC-1)

**Manual verification:**

**Action:**

1. Open `README.md` in text editor.
2. Locate environment-variables table (or new "Environment Variables" section if created).
3. Verify row exists with exact content:
   ```
   | `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
   ```

**Pass criteria:**

- Row is present in table.
- Variable name is `` `AGENT_MAX_STEPS` `` (with backticks).
- Description is exactly "Maximum tool-use steps before forcing a final answer" (no typos, no paraphrasing).
- Default value is `` `5` `` (with backticks).
- Table is well-formed (pipes align, header separator present).

---

### Test 2: Capability description present (AC-2)

**Manual verification:**

**Action:**

1. Open `README.md` in text editor.
2. Locate "Features" section (or new "Capabilities" section if created).
3. Verify sentence is present:
   > The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.

**Pass criteria:**

- Sentence is present in README.
- Text is **exactly** as specified (word-for-word, no paraphrasing).
- Capitalization is correct ("The orchestrator", "LLM").
- Sentence ends with period.
- Formatting matches surrounding text (bullet vs paragraph, heading level).

---

## Implementation checklist

- [ ] Read existing `README.md` to locate environment-variables table (or determine where to create it).
- [ ] Add `AGENT_MAX_STEPS` row to environment-variables table (or create table if it does not exist).
- [ ] Read existing `README.md` to locate "Features" section (or determine where to create "Capabilities" section).
- [ ] Add capability description sentence to "Features" section (or create "Capabilities" section if needed).
- [ ] Match existing README style (heading levels, bullet style, capitalization).
- [ ] Perform Test 1 (Manual Verification AC-1: environment-variables table).
- [ ] Perform Test 2 (Manual Verification AC-2: capability description sentence).

---

## Notes

### Dependency on Story 01-agent, Story 02-agent, Story 03-agent

This story documents the feature implemented in Stories 01–03. All three prior stories must be complete (loop logic, env var parsing, tests) before this documentation is accurate. If Stories 01–03 are not complete, the README will describe functionality that does not yet exist.

### No automated tests

This story has no automated tests. Verification is manual (Test 1 and Test 2 above). The implementation is trivial (add 5 lines to README), so manual verification is sufficient and appropriate.

### Style consistency

The README may have existing conventions (e.g., all env vars in a single table, features as bulleted list, specific heading capitalization). Follow existing conventions rather than inventing new ones. If the README is empty or minimal, use the formats specified in "Interfaces" above.

### Placement suggestions

**Environment-variables table:**

- **Best:** After "## Installation" or "## Usage" section (operators need to know config before running).
- **Acceptable:** Before "## Development" section (config is operational, not dev-only).
- **Avoid:** At end of README (operators may not scroll that far).

**Capability description:**

- **Best:** In existing "## Features" section (most natural fit).
- **Acceptable:** In "## Overview" or "## Usage" section (describes what the service does).
- **Avoid:** In "## Development" section (capability is user-facing, not dev-only).

---

## Verification checklist

- [ ] `README.md` contains environment-variables table (existing or new).
- [ ] Table includes `AGENT_MAX_STEPS` row with correct description and default value.
- [ ] `README.md` contains "Features" or "Capabilities" section (existing or new).
- [ ] Section includes multi-step capability description sentence (exact wording).
- [ ] New content matches existing README style (heading levels, bullet style, capitalization).
- [ ] Test 1 passed (AC-1: env var table).
- [ ] Test 2 passed (AC-2: capability description).
