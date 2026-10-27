# Tech Spec — Story 04-agent

**ticket:** 04-agent  
**title:** Document AGENT_MAX_STEPS, multi-step capability, and /query_given endpoint in README  
**story points:** 1  
**made from:** PER-56 requirements + HLD + breakdown + review_feedback.md  
**source:** designer (LLD)

---

## Overview

Add `AGENT_MAX_STEPS` to the environment-variables table in the README with its description and default value. Add one sentence describing the multi-step tool-use capability. Update all API endpoint documentation and examples from `/query` to `/query_given`. This ensures operators and developers know about the new configuration option, feature, and endpoint path.

This story implements AC-8 (README updated) and the endpoint path documentation change from `review_feedback.md`. It depends on Story 2 (which added `AGENT_MAX_STEPS` env var parsing and updated the HTTP route to `/query_given`) and Story 3 (which verified the loop behavior with tests). No code changes are made in this story—only documentation updates.

---

## Stack

- **Language:** Markdown (README.md)
- **No code changes:** This story is documentation-only.

---

## Acceptance criteria

From requirements.md and breakdown.md, this story covers:

- **AC-8:** README updated:
  - Environment-variables table includes `AGENT_MAX_STEPS` row with description and default `5`.
  - Capability description sentence present: "The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit."
  - All API endpoint documentation references `/query_given` (not `/query`), including curl examples and route descriptions.

These ACs map to 3 manual checks during review.

---

## Files to create / modify

### Files to modify

1. **`README.md`**
   - **Change:** Add `AGENT_MAX_STEPS` to environment-variables table (or create table if it does not exist).
   - **Change:** Add one-sentence capability description to "Features" section (or create "Capabilities" section if needed).
   - **Change:** Update all API endpoint documentation from `/query` to `/query_given` (route references, curl examples, any endpoint descriptions).
   - **Lines affected:** ~8–12 lines (table row + sentence + endpoint path updates in multiple places).

### Files to read (context only)

- **`README.md`** (existing content, structure, formatting conventions).
- **`requirements.md`** (AC-8 wording for capability description).
- **`hld.md`** (README updates section for exact table row and sentence wording).

---

## Interfaces

### Environment variables table (new or updated section)

**If table exists:** Add new row:

```markdown
| Variable | Description | Default |
|----------|-------------|---------|
| `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
```

**If table does not exist:** Create new section:

```markdown
## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
```

**Placement:** Near other configuration documentation (e.g., after "Installation" or "Usage" section, or in a dedicated "Configuration" section).

---

### Capability description (new or updated section)

**Sentence to add:**

> The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.

**Placement options:**

1. **Existing "Features" section:** Append as a new bullet or paragraph.
2. **Existing "Usage" section:** Add as a paragraph describing the behavior.
3. **New "Capabilities" section:** Create if no suitable section exists.

**Example (Features section):**

```markdown
## Features

- Single-step tool execution (legacy behavior)
- Multi-step tool use: The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.
- Error handling with appropriate HTTP status codes (400 for validation errors, 500 for other errors)
```

---

### API endpoint documentation updates

**Find and replace all occurrences of `/query` with `/query_given`:**

**Example locations to update:**

1. **Endpoint description:**
   ```markdown
   ### POST /query_given
   
   Query the agent with a natural-language question.
   ```

2. **curl examples:**
   ```bash
   curl -X POST http://localhost:8000/query_given \
     -H "Content-Type: application/json" \
     -d '{"query": "Who is EMP-001'\''s manager?"}'
   ```

3. **Route registration notes (if present):**
   ```markdown
   The server registers a single route: `POST /query_given`.
   ```

4. **Any example responses or usage documentation referencing the endpoint.**

**Search pattern:** Look for `/query` (with surrounding context to distinguish from internal function names like `handle_query`).

**Note:** Do not change internal function/method names (e.g., `handle_query` in code examples stays as-is). Only change HTTP endpoint paths in API documentation.

---

## Algorithm notes

### Documentation structure conventions

**Environment variables table:**

- Use consistent column headers: `| Variable | Description | Default |`
- Align columns with pipes (`|`) for readability (Markdown rendering ignores alignment, but it helps human readers).
- Use backticks for variable names: `` `AGENT_MAX_STEPS` ``
- Use sentence case for descriptions, end with period.

**Capability description:**

- Match the tone and style of existing feature descriptions (if any).
- Keep it concise: one sentence is sufficient per requirements.
- Use present tense ("supports", not "will support").

**Endpoint path updates:**

- Use literal paths in examples: `/query_given`, not `<endpoint>` or `{endpoint}`.
- Update both the endpoint definition (e.g., `### POST /query_given`) and all usage examples (curl commands, client code snippets).

---

## Verification Approach

Manual review of `README.md` diff. No automated tests (this is a documentation-only story).

### Check 1: Environment-variables table includes AGENT_MAX_STEPS row

**Action:**

- Open `README.md` and locate the environment-variables table (or verify it was created if it did not exist).
- Verify the table includes a row for `AGENT_MAX_STEPS`.

**Expected content:**

```markdown
| `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
```

**Pass criteria:**

- Row exists in table.
- Variable name is `` `AGENT_MAX_STEPS` `` (backticks, correct spelling).
- Description matches expected wording (exact match not required, but meaning must be clear).
- Default value is `5`.

---

### Check 2: Capability description sentence present

**Action:**

- Locate the "Features", "Usage", or "Capabilities" section in `README.md`.
- Verify the multi-step tool-use capability is described.

**Expected content (approximate):**

> The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.

**Pass criteria:**

- Sentence exists in a visible section (not hidden in implementation details or comments).
- Meaning matches expected description (exact wording can vary slightly if it's clearer).
- Placed in a logical section (Features, Usage, or new Capabilities section).

---

### Check 3: All endpoint references use /query_given

**Action:**

- Search `README.md` for all occurrences of `/query` (case-insensitive).
- Verify each occurrence is either:
  1. Updated to `/query_given` (for HTTP endpoint references).
  2. Left as-is (for internal function names like `handle_query`).

**Expected updates:**

- Endpoint definitions: `### POST /query_given`
- curl examples: `http://localhost:8000/query_given`
- Route descriptions: "registers a single route: `POST /query_given`"

**Pass criteria:**

- No HTTP endpoint references to `/query` remain.
- Internal function/method names (e.g., `handle_query`, `orchestrator.handle_query()`) are unchanged.
- At least one curl example references `/query_given`.

---

## Edge cases / failure modes

### Edge case 1: Environment-variables table does not exist

**Scenario:** `README.md` has no existing environment-variables section or table.

**Behavior:** Create a new section with the table headers and the `AGENT_MAX_STEPS` row.

**Verification:** Check 1 verifies the table exists and includes the row.

---

### Edge case 2: Multiple sections could host the capability description

**Scenario:** `README.md` has both "Features" and "Usage" sections, and it's unclear where to place the sentence.

**Behavior:** Choose the section that best fits the content. "Features" is preferred if it lists product capabilities. "Usage" is acceptable if it describes how to use the service.

**Verification:** Check 2 verifies the sentence exists in *some* visible section; exact placement is flexible.

---

### Edge case 3: Existing /query references are in code comments or filenames

**Scenario:** `README.md` references `/query` in a code comment (e.g., `# Old endpoint: /query`) or in a changelog entry.

**Behavior:** Update code comments if they describe the current endpoint. Leave changelog entries unchanged (historical references are intentionally preserved).

**Verification:** Check 3 verifies no *HTTP endpoint references* to `/query` remain; historical/comment references can stay if clearly marked as legacy.

---

## Risks

### RISK-1: Incomplete endpoint path updates

**Likelihood:** Medium (README may have multiple endpoint references)  
**Impact:** High (developers may use wrong endpoint path)  
**Mitigation (implemented):** Check 3 explicitly verifies all `/query` references are updated or justified.  
**Residual risk:** Obscure references (e.g., in image URLs or external links) may be missed. Acceptance: manual review catches most cases; users report any remaining issues.

---

### RISK-2: Inconsistent formatting breaks table rendering

**Likelihood:** Low (Markdown is forgiving)  
**Impact:** Low (table may render incorrectly but content is still readable)  
**Mitigation (implemented):** Use standard Markdown table syntax (pipes and dashes). Test rendering locally if possible.  
**Residual risk:** Some Markdown renderers have quirks. Acceptance: standard syntax works in most tools; can be fixed in a follow-up if an issue is reported.

---

### RISK-3: Capability description is too verbose or unclear

**Likelihood:** Low (sentence is short and clear)  
**Impact:** Low (users may need to re-read it)  
**Mitigation (implemented):** Use exact wording from requirements.md AC-8, which was reviewed by the architect.  
**Residual risk:** Some users may want more detail. Acceptance: one sentence is sufficient per requirements; more detail can be added in a follow-up story if needed.

---

## Dependencies

- **Story 1:** Loop logic implemented; README documents the behavior that now exists.
- **Story 2:** `AGENT_MAX_STEPS` env var and `/query_given` endpoint implemented; README documents how to configure and use them.
- **Story 3:** Tests verify the loop behavior; README documents the tested capability.
- **Existing:** `README.md` (structure, formatting conventions).

No new external dependencies.

---

## Implementation notes

- **Search and replace:** Use a text editor's "find all" to locate `/query` references. Verify each match before updating (don't blindly replace).
- **Table formatting:** Markdown tables use pipes (`|`) to separate columns and dashes (`---`) for the header separator. Ensure at least three dashes per column.
- **Sentence placement:** If uncertain where to place the capability description, create a new "Capabilities" section at the top level (e.g., after "Installation" and before "Usage").
- **Consistency:** Match the existing README's capitalization (e.g., "Features" vs "FEATURES"), punctuation, and markdown style (e.g., ATX-style headers `##` vs Setext-style).
- **No code changes:** This story modifies only `README.md`. If you find yourself editing Python files, stop and re-read the story description.

---

## Out of scope for this story

- **Code changes:** Stories 1, 2, and 3 implemented the code; this story is documentation-only.
- **Test changes:** Story 3 added tests; this story does not modify test files.
- **Release notes:** Out of scope; README is user-facing docs, not a changelog.
- **Example projects or tutorials:** Out of scope; README provides reference documentation, not a step-by-step guide.

---

## Verification checklist

- [ ] Environment-variables table exists in `README.md` (Check 1).
- [ ] Table includes `AGENT_MAX_STEPS` row with description and default `5` (Check 1).
- [ ] Capability description sentence present in "Features", "Usage", or "Capabilities" section (Check 2).
- [ ] Sentence matches expected meaning: multi-step tool use with step limit (Check 2).
- [ ] All HTTP endpoint references updated from `/query` to `/query_given` (Check 3).
- [ ] At least one curl example uses `/query_given` (Check 3).
- [ ] Internal function names (e.g., `handle_query`) unchanged (Check 3).
- [ ] Manual review: table formatting renders correctly (if local preview available).
- [ ] Manual review: no broken links or missing sections after updates.

---

## Tests

Per breakdown.md verification approach and AC-8, this story has **3 manual verification checks** and **0 automated tests**.

### Manual Check 1: Environment-variables table includes AGENT_MAX_STEPS row (AC-8.1)

**Action:**

- Open `README.md` and locate the environment-variables table (or verify it was created if it did not exist).
- Verify the table includes a row for `AGENT_MAX_STEPS`.

**Expected content:**

```markdown
| `AGENT_MAX_STEPS` | Maximum tool-use steps before forcing a final answer | `5` |
```

**Pass criteria:**

- Row exists in table.
- Variable name is `` `AGENT_MAX_STEPS` `` (backticks, correct spelling).
- Description matches expected wording (exact match not required, but meaning must be clear).
- Default value is `5`.

**Maps to:** AC-8 (README environment-variables table)

---

### Manual Check 2: Capability description sentence present (AC-8.2)

**Action:**

- Locate the "Features", "Usage", or "Capabilities" section in `README.md`.
- Verify the multi-step tool-use capability is described.

**Expected content (approximate):**

> The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit.

**Pass criteria:**

- Sentence exists in a visible section (not hidden in implementation details or comments).
- Meaning matches expected description (exact wording can vary slightly if it's clearer).
- Placed in a logical section (Features, Usage, or new Capabilities section).

**Maps to:** AC-8 (README capability description)

---

### Manual Check 3: All endpoint references use /query_given (AC-8.3)

**Action:**

- Search `README.md` for all occurrences of `/query` (case-insensitive).
- Verify each occurrence is either:
  1. Updated to `/query_given` (for HTTP endpoint references).
  2. Left as-is (for internal function names like `handle_query`).

**Expected updates:**

- Endpoint definitions: `### POST /query_given`
- curl examples: `http://localhost:8000/query_given`
- Route descriptions: "registers a single route: `POST /query_given`"

**Pass criteria:**

- No HTTP endpoint references to `/query` remain.
- Internal function/method names (e.g., `handle_query`, `orchestrator.handle_query()`) are unchanged.
- At least one curl example references `/query_given`.

**Maps to:** AC-8 (README endpoint documentation), review_feedback.md (endpoint path change)

---

## Acceptance criteria (mapped to verification checks)

- **AC-8:** README updated:
  - Environment-variables table includes `AGENT_MAX_STEPS` row with description and default `5`. → **Manual Check 1**
  - Capability description sentence present: "The orchestrator supports multi-step tool use, allowing the LLM to call tools, see results, and call additional tools until it produces an answer or reaches the step limit." → **Manual Check 2**
  - All API endpoint documentation references `/query_given` (not `/query`), including curl examples and route descriptions. → **Manual Check 3**

**Verification count:** 3 manual checks (matches breakdown.md count for Story 4)

---

## Notes

- **No automated tests:** This is a documentation-only story; all verification is manual review of `README.md` changes.
- **Maps to breakdown.md:** Story 4 has 0 automated tests and 3 manual checks per breakdown.md verification summary.
- **Consistency with other stories:** Stories 1–3 have automated unit tests; Story 4 is the only documentation-only story in the breakdown.
