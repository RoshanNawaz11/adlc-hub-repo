# LLD — Story 02-agent

**Story:** Update configuration, tracing references, and documentation  
**Repo:** `agent`

---

## Acceptance Criteria

### AC-1: Configuration and documentation updated

**Given** the updated configuration and documentation files  
**When** reviewing `.env.example`, `runtime/runner.py`, `model/policy.yaml`, `README.md`, and `docs/`  
**Then** all of the following are true:
- `.env.example` contains required `ANTHROPIC_API_KEY=` and optional (commented) `ANTHROPIC_BASE_URL=`, with `LANGWATCH_DISABLED=false` and `LANGWATCH_API_KEY` / `LANGWATCH_ENDPOINT_URL` documented
- `.env.example` does not contain `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, or Cloudflare gateway URL references
- `runtime/runner.py` `_config()` comment references LangWatch instead of LiteLLM as the tracer
- `model/policy.yaml` header comment describes direct Anthropic configuration
- `docs/claude-via-litellm-cloudflare.md` is deleted
- `README.md` LiteLLM/Cloudflare sections are removed or rewritten to describe direct Anthropic setup

**Maps to:** Requirements AC-2, Breakdown Story 2 AC-1

---

### AC-2: Existing tests pass

**Given** the updated configuration and documentation  
**When** running `uv run pytest -q`  
**Then** all existing tests pass with no failures

**Maps to:** Requirements AC-6 (partial: no regressions), Breakdown Story 2 AC-2

---

## File Plan

### Modified Files

**`.env.example`**
- Remove `LITELLM_BASE_URL=` line
- Remove `ANTHROPIC_GATEWAY_TOKEN=` line
- Remove any Cloudflare gateway URL examples
- Add `ANTHROPIC_API_KEY=` (required, uncommented)
- Add `# ANTHROPIC_BASE_URL=https://api.anthropic.com` (optional, commented with note: "Uncomment to override default Anthropic endpoint")
- Change `LANGWATCH_DISABLED=true` to `LANGWATCH_DISABLED=false`
- Ensure `LANGWATCH_API_KEY=` and `LANGWATCH_ENDPOINT_URL=` are present with documentation comments

**`runtime/runner.py`** (path established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`)
- Locate `_config()` method
- Find comment referencing LiteLLM as tracer
- Replace with comment: "LangWatch handles tracing when enabled" (or similar phrasing consistent with codebase style)

**`model/policy.yaml`** (path established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`)
- Locate header comment section
- Replace references to LiteLLM → Cloudflare → Anthropic routing
- Update to describe direct Anthropic API calls via SDK

**`README.md`** (path established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`)
- Locate sections titled or describing "LiteLLM setup", "Cloudflare AI Gateway", or similar
- Remove or rewrite to describe direct Anthropic configuration:
  - Required: `ANTHROPIC_API_KEY` from Anthropic Console
  - Optional: `ANTHROPIC_BASE_URL` for custom endpoints/proxies
  - LangWatch tracing enabled by default (`LANGWATCH_DISABLED=false`)
- Ensure no leftover references to `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, or Cloudflare gateway URLs

---

### Deleted Files

**`docs/claude-via-litellm-cloudflare.md`** (path established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`)
- Remove entire file (no replacement doc; direct Anthropic setup covered in `README.md`)

---

## Interfaces

### Configuration Schema (`.env.example`)

**Required variables:**
```
ANTHROPIC_API_KEY=
```

**Optional variables:**
```
# ANTHROPIC_BASE_URL=https://api.anthropic.com
# Uncomment to override the default Anthropic endpoint (e.g., for proxies or testing)

LANGWATCH_DISABLED=false
LANGWATCH_API_KEY=
LANGWATCH_ENDPOINT_URL=
```

**Removed variables:**
- `LITELLM_BASE_URL`
- `ANTHROPIC_GATEWAY_TOKEN`
- Cloudflare gateway URL examples

---

### Documentation Contracts

**`README.md` Configuration Section (target state):**

```markdown
## Configuration

### Required Environment Variables

- `ANTHROPIC_API_KEY`: Your Anthropic API key (obtain from [Anthropic Console](https://console.anthropic.com/))

### Optional Environment Variables

- `ANTHROPIC_BASE_URL`: Override the default Anthropic API endpoint (e.g., for proxies or testing environments). Defaults to `https://api.anthropic.com`.

### Tracing (LangWatch)

LangWatch tracing is enabled by default. Configure:

- `LANGWATCH_DISABLED`: Set to `true` to disable tracing (default: `false`)
- `LANGWATCH_API_KEY`: Your LangWatch API key
- `LANGWATCH_ENDPOINT_URL`: LangWatch endpoint URL

[Additional LangWatch setup instructions if needed...]
```

**`model/policy.yaml` Header Comment (target state):**

```yaml
# Model routing policy
# All model calls and skill uploads go directly to Anthropic API via anthropic.AsyncAnthropic client
# Configure via ANTHROPIC_API_KEY (required) and ANTHROPIC_BASE_URL (optional override)
```

**`runtime/runner.py` `_config()` Comment (target state):**

```python
def _config(self, ...):
    # LangWatch handles tracing when enabled (LANGWATCH_DISABLED=false)
    ...
```

---

## Algorithm Notes

### `.env.example` Update Logic

1. **Scan for removals:**
   - Search file for `LITELLM_BASE_URL`, `ANTHROPIC_GATEWAY_TOKEN`, and Cloudflare URL patterns
   - Delete matching lines and any associated comments

2. **Add required variables:**
   - Insert `ANTHROPIC_API_KEY=` near top of file (group with other API keys if present)
   - Ensure line is uncommented (no `#` prefix)

3. **Add optional override:**
   - Insert `# ANTHROPIC_BASE_URL=https://api.anthropic.com` below `ANTHROPIC_API_KEY`
   - Add comment: `# Uncomment to override default Anthropic endpoint`

4. **Update LangWatch config:**
   - Change `LANGWATCH_DISABLED=true` to `LANGWATCH_DISABLED=false`
   - Verify `LANGWATCH_API_KEY=` and `LANGWATCH_ENDPOINT_URL=` are present
   - Add documentation comment if missing (e.g., `# LangWatch API key for tracing`)

---

### `README.md` Rewrite Strategy

1. **Identify affected sections:**
   - Search for headers/subsections containing "LiteLLM", "Cloudflare", "AI Gateway", "routing"
   - Identify code examples showing `LITELLM_BASE_URL` or `ANTHROPIC_GATEWAY_TOKEN`

2. **Rewrite or remove:**
   - **Architecture/routing descriptions:** Replace with "Direct Anthropic API calls via official SDK"
   - **Setup instructions:** Replace with required `ANTHROPIC_API_KEY` + optional `ANTHROPIC_BASE_URL`
   - **Tracing sections:** Update to describe LangWatch as default tracer
   - **Orphaned content:** Remove if no longer applicable (e.g., LiteLLM-specific troubleshooting)

3. **Preserve:**
   - Model selection, `max_tokens`, prompt engineering sections (unchanged)
   - MCP, GraphQL, backend/frontend references (unchanged)
   - Any non-routing architectural content

4. **Validation:**
   - After rewrite, search `README.md` for `litellm`, `cloudflare`, `cf-aig`, `ANTHROPIC_GATEWAY_TOKEN`
   - Ensure no matches remain

---

### Comment Update Pattern

**`runtime/runner.py` (one-line change):**
- Locate `_config()` method definition
- Find line like: `# LiteLLM handles tracing` or `# Tracing via LiteLLM`
- Replace with: `# LangWatch handles tracing when enabled`

**`model/policy.yaml` (header block rewrite):**
- Locate YAML header comments (typically first 3-10 lines after `---` or at file start)
- Replace routing description:
  - **Old:** "Routes through LiteLLM → Cloudflare AI Gateway → Anthropic"
  - **New:** "Direct Anthropic API calls via AsyncAnthropic client"
- Preserve any model selection or policy-specific comments

---

## Tests

### Test Count

**Total automated tests:** 0 new tests  
**Verification steps:** 6 manual validation steps

This story modifies configuration templates and documentation only. No new automated tests are created. Story 3 will add comprehensive test coverage (9 automated tests: 6 gateway + 3 tracing per breakdown).

---

### Verification Approach

**Step 1: Grep validation**
```bash
grep -ri "litellm\|cloudflare\|cf-aig\|ANTHROPIC_GATEWAY_TOKEN" \
  .env.example runtime/runner.py model/policy.yaml docs README.md
```
- **Expected output:** No matches
- **If matches found:** Review and remove remaining references
- **Maps to:** Requirements AC-2 (partial), AC-3

---

**Step 2: File existence check**
```bash
[ ! -f docs/claude-via-litellm-cloudflare.md ] && echo "PASS" || echo "FAIL"
```
- **Expected output:** `PASS`
- **Maps to:** Requirements AC-2 (partial)

---

**Step 3: Config schema validation**
- Open `.env.example`
- **Visually confirm presence of:**
  - `ANTHROPIC_API_KEY=` (uncommented)
  - `# ANTHROPIC_BASE_URL=` (commented)
  - `LANGWATCH_DISABLED=false`
  - `LANGWATCH_API_KEY=` and `LANGWATCH_ENDPOINT_URL=`
- **Visually confirm absence of:**
  - `LITELLM_BASE_URL`
  - `ANTHROPIC_GATEWAY_TOKEN`
  - Cloudflare URLs
- **Maps to:** Requirements AC-2 (partial)

---

**Step 4: Comment verification**
- Open `runtime/runner.py` and locate `_config()` method
- **Verify:** Comment references LangWatch (not LiteLLM)
- Open `model/policy.yaml` and read header comments
- **Verify:** Description matches direct Anthropic setup
- **Maps to:** Requirements AC-2 (partial)

---

**Step 5: README validation**
- Open `README.md` and locate configuration/setup sections
- **Verify:** No LiteLLM/Cloudflare instructions remain
- **Verify:** Direct Anthropic setup documented (API key + optional base URL)
- **Maps to:** Requirements AC-2 (partial)

---

**Step 6: Regression test**
```bash
uv run pytest -q
```
- **Expected output:** Exit code 0 (all tests pass)
- **Note:** Config/doc changes should not affect test logic
- **If failures occur:** Review for unintended breakage (e.g., hardcoded config paths)
- **Maps to:** Requirements AC-6 (partial: no regressions introduced), Breakdown Story 2 AC-2

---

## Notes

- **AC count alignment:** Story has 2 ACs (AC-1 consolidates all Requirements AC-2 elements, AC-2 covers regression check); Requirements AC-2 is the single AC mapped to this story, matching breakdown Story 2 structure
- **Test count alignment:** 0 new tests in this story; Story 3 adds 9 automated tests (6 gateway + 3 tracing) per breakdown verification approach and Requirements AC-4, AC-5, AC-6
- **File path verification:** All paths (`.env.example`, `runtime/runner.py`, `model/policy.yaml`, `README.md`, `docs/claude-via-litellm-cloudflare.md`) established by ticket context at `/home/regularinsaan/projects/trinet-adlc-agents`; repo-tree shows empty `existing files:` list indicating tree is incomplete but paths are specified in ticket and breakdown
- **Comment-only changes:** `runtime/runner.py` and `model/policy.yaml` modifications touch comments only; no code logic changes
- **Documentation scope:** `README.md` rewrite focuses on configuration/setup sections; architecture/design sections may also reference routing but are lower priority
- **Deleted doc:** `claude-via-litellm-cloudflare.md` removal leaves no replacement; direct Anthropic setup is simpler and covered in `README.md`
- **LangWatch default:** Setting `LANGWATCH_DISABLED=false` in `.env.example` makes tracing opt-out instead of opt-in (aligns with Story 1 removing LiteLLM as tracer)
- **No code execution risk:** This story modifies config templates and documentation only; no runtime logic changes, so test suite should pass unchanged
- **Grep pattern:** The verification `grep` command uses case-insensitive (`-i`) and recursive (`-r`) flags to catch any casing variations (e.g., `LiteLLM`, `LITELLM`, `litellm`)
