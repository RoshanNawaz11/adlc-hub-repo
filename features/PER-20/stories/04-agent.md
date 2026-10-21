# LLD: Story 04-agent — Update README to document FastAPI primary path and remove Flask/stdio

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 04-agent  
**After:** Story 03-agent (FastAPI app, LLM client, and FastMCP HTTP client must be complete)

---

## Acceptance Criteria

### AC-5: Setup documentation

**Given** the `adlc-agents` README  
**When** a developer follows setup  
**Then** they run the agent with FastAPI, set `MCP_URL` to the FastMCP SSE URL, set the LLM env, and `POST` a question  
**And** HTTP MCP is the documented primary path  
**And** Flask and stdio are not documented

---

## File Plan

### Modified Files

- `README.md` — Document FastAPI setup, `MCP_URL`, LLM credentials, run instructions, query example; remove all Flask and stdio MCP references

### No New or Removed Files

This story only updates documentation.

---

## README Structure

The updated `README.md` must contain the following sections in order:

1. **Overview** — Brief description of the agent
2. **Prerequisites** — Python version, required environment variables
3. **Installation** — Dependency installation
4. **Configuration** — Environment variable setup (`MCP_URL`, LLM credentials)
5. **Running the Agent** — `uvicorn` command
6. **Usage** — Example query with `curl` or equivalent
7. **Health Check** — Example health endpoint call
8. **Development** — Testing instructions (optional, deferred if not yet implemented)

---

## README Content

### Overview

```markdown
# TriNet Agent

AI agent that answers questions about TriNet employee and sales-order data. The agent uses an LLM (Anthropic Claude or compatible gateway) to decide whether to answer directly or call MCP tools over HTTP, then synthesizes the final answer.

**Architecture:**
- **FastAPI** web server serving `/query` and `/health` endpoints
- **LLM client** for direct answers and synthesis
- **FastMCP HTTP client** to call MCP tools at a shared MCP server
```

### Prerequisites

```markdown
## Prerequisites

- **Python 3.11+** (or version specified in `pyproject.toml` / `requirements.txt`)
- **MCP Server** running at a known URL (e.g. `http://127.0.0.1:8000/sse`)
  - See [PER-18](https://linear.app/personalgs/issue/PER-18/serve-mcp-graphql-tools-over-fastapi-fastmcp-http-replace-stdio) for MCP server setup
- **LLM credentials** (Anthropic API key or gateway URL)
```

### Installation

```markdown
## Installation

1. Clone the repository:
   ```bash
   git clone <repo-url>
   cd adlc-agents
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
```

### Configuration

```markdown
## Configuration

Set the following environment variables:

| Variable | Required | Description | Example |
|---|---|---|---|
| `MCP_URL` | Yes | FastMCP SSE endpoint URL | `http://127.0.0.1:8000/sse` |
| `ANTHROPIC_API_KEY` | Yes* | Anthropic API key for Claude LLM | `sk-ant-...` |
| `LLM_GATEWAY_URL` | Yes* | Alternative: LLM gateway URL if not using Anthropic directly | `http://llm-gateway:8080` |

_*Either `ANTHROPIC_API_KEY` or `LLM_GATEWAY_URL` is required, depending on your LLM setup._

**Example `.env` file:**
```bash
MCP_URL=http://127.0.0.1:8000/sse
ANTHROPIC_API_KEY=sk-ant-your-api-key-here
```

Load environment variables:
```bash
export $(cat .env | xargs)
```
```

### Running the Agent

```markdown
## Running the Agent

Start the FastAPI server with `uvicorn`:

```bash
uvicorn agent.main:app --host 0.0.0.0 --port 8001
```

**Options:**
- `--reload` — auto-reload on code changes (development only)
- `--workers N` — run N worker processes (production)

The agent will be available at `http://localhost:8001`.
```

### Usage

```markdown
## Usage

### Query Endpoint

Send questions to the agent via `POST /query`:

```bash
curl -X POST http://localhost:8001/query \
  -H "Content-Type: application/json" \
  -d '{"question": "list sales orders"}'
```

**Response (success):**
```json
{
  "answer": "Here are the sales orders: SO-001 ($100), SO-002 ($250), ..."
}
```

**Response (error):**
```json
{
  "error": "MCP server unreachable"
}
```

**Example questions:**
- `"hello"` — Direct greeting (no MCP call)
- `"list sales orders"` — Calls `list_sales_order` tool
- `"list employees"` — Calls `list_employees` tool
- `"get employee emp-001"` — Calls `get_employee_by_id` tool (ID normalized to `EMP-001`)
- `"get sales order so-123"` — Calls `get_sales_order_by_id` tool (ID normalized to `SO-123`)

### Health Check

Check agent and MCP server status:

```bash
curl http://localhost:8001/health
```

**Response:**
```json
{
  "status": "up",
  "mcp_reachable": true
}
```

- `status: "up"` — Agent process is running
- `mcp_reachable: true` — MCP server at `MCP_URL` is reachable
- `mcp_reachable: false` — MCP server is down or `MCP_URL` is not set
```

### Development (Optional)

```markdown
## Development

### Running Tests

```bash
pytest tests/ -v
```

**Test coverage:**
- Direct LLM answers (no MCP call)
- MCP tool calls over HTTP
- Error handling (MCP unreachable, tool errors, LLM unavailable)
- Health endpoint

### Project Structure

```
adlc-agents/
├── agent/
│   ├── main.py            # FastAPI app entry point
│   ├── orchestrator.py    # Routes questions to LLM/MCP
│   ├── llm_client.py      # LLM client wrapper
│   ├── mcp_client.py      # FastMCP HTTP client
│   └── health.py          # Health check logic
├── tests/                 # Test suite
├── requirements.txt       # Python dependencies
└── README.md              # This file
```
```

---

## Removed Content

The following sections must be **removed** from the existing `README.md` (if present):

- Any references to **Flask** (app setup, routes, `flask run`, etc.)
- Any references to **stdio MCP** (subprocess, `stdio_mcp.py`, local MCP process, etc.)
- Any setup instructions for running a local MCP server via stdio
- Any environment variables for stdio transport (e.g. `MCP_STDIO_PATH`, `MCP_LOCAL`, etc.)

**Search patterns to remove:**
- `flask`
- `stdio`
- `subprocess`
- `local MCP`
- `stdio_mcp`

---

## Algorithm Notes

### README Update Process

1. **Read existing `README.md`** (if it exists)
2. **Remove all Flask and stdio references** (see Removed Content above)
3. **Replace or add sections** per README Structure and Content above
4. **Preserve any existing sections** not covered by this story (e.g. License, Contributing) unless they reference Flask/stdio
5. **Write updated `README.md`**

**Pseudocode:**
```python
# Read existing README
try:
    with open("README.md", "r") as f:
        existing_content = f.read()
except FileNotFoundError:
    existing_content = ""

# Remove Flask/stdio references
patterns_to_remove = ["flask", "stdio", "subprocess", "local MCP", "stdio_mcp"]
lines = existing_content.split("\n")
filtered_lines = [
    line for line in lines
    if not any(pattern.lower() in line.lower() for pattern in patterns_to_remove)
]

# Build new README content (per structure above)
new_content = """
# TriNet Agent

AI agent that answers questions about TriNet employee and sales-order data...

[... full content per sections above ...]
"""

# Write updated README
with open("README.md", "w") as f:
    f.write(new_content)
```

**Notes:**
- Simple line-based filtering suffices if Flask/stdio content is minimal.
- If existing README has complex structure, use section-based replacement (parse headers, replace matching sections).
- Preserve any **License**, **Contributing**, **Changelog** sections unless they reference Flask/stdio.

---

## Tests

### Test 1: End-to-end README setup verification

**Coverage:** AC-5 (README setup → successful query)

**Given:** A clean environment (no prior agent installation)  
**When:** A developer follows the README setup instructions step-by-step  
**Then:** They successfully run the agent and query it

**Manual Steps (CI-automatable):**

1. **Clone repo** (or use CI checkout):
   ```bash
   git clone <repo-url>
   cd adlc-agents
   ```

2. **Install dependencies** per README:
   ```bash
   pip install -r requirements.txt
   ```

3. **Set environment variables** per README:
   ```bash
   export MCP_URL=http://127.0.0.1:8000/sse
   export ANTHROPIC_API_KEY=sk-ant-test-key
   ```
   _(In CI: use mock MCP server at `http://127.0.0.1:8000/sse` and mock LLM client)_

4. **Run agent** per README:
   ```bash
   uvicorn agent.main:app --host 0.0.0.0 --port 8001 &
   sleep 2  # Allow startup
   ```

5. **Query agent** per README usage example:
   ```bash
   curl -X POST http://localhost:8001/query \
     -H "Content-Type: application/json" \
     -d '{"question": "hello"}'
   ```
   
   **Expected response:**
   ```json
   {
     "answer": "Hello! How can I help you?"
   }
   ```

6. **Verify no Flask or stdio references** in README:
   ```bash
   grep -i "flask" README.md && exit 1  # Should not find Flask
   grep -i "stdio" README.md && exit 1  # Should not find stdio
   echo "README clean"
   ```

**Automated Test (CI):**

**File:** `tests/test_readme_setup.py`

```python
import subprocess
import time
import requests
import pytest

def test_readme_setup_end_to_end(tmp_path, monkeypatch):
    """
    Follow README setup from clean environment and verify successful query.
    
    This test runs in CI with a mock MCP server and mock LLM client.
    """
    # Setup: Start mock MCP server (or use existing test fixture)
    # (Deferred to CI infra; assumes mock MCP at http://127.0.0.1:8000/sse)
    
    # Set env vars per README
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    monkeypatch.setenv("ANTHROPIC_API_KEY", "sk-ant-test-key")
    
    # Start agent (assumes agent/ code is importable in CI)
    proc = subprocess.Popen(
        ["uvicorn", "agent.main:app", "--host", "0.0.0.0", "--port", "8001"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    # Wait for startup
    time.sleep(2)
    
    try:
        # Query agent per README usage example
        response = requests.post(
            "http://localhost:8001/query",
            json={"question": "hello"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "answer" in data
        assert len(data["answer"]) > 0  # Non-empty answer
    
    finally:
        # Cleanup
        proc.terminate()
        proc.wait(timeout=5)

def test_readme_no_flask_or_stdio():
    """
    Verify README does not contain Flask or stdio references.
    """
    with open("README.md", "r") as f:
        content = f.read().lower()
    
    assert "flask" not in content, "README still references Flask"
    assert "stdio" not in content, "README still references stdio"
```

**Notes:**
- Manual steps provided for human verification; automated test for CI.
- Mock MCP server and mock LLM client required for CI (or skip integration test and rely on unit tests from Stories 1-3).
- `test_readme_no_flask_or_stdio` is lightweight and always runs; `test_readme_setup_end_to_end` may be CI-only.

---

## Verification Summary

**Total test count:** 1 end-to-end verification (per breakdown: 1 test for Story 4)

| Test | File | Coverage |
|---|---|---|
| Follow README setup from clean environment | `tests/test_readme_setup.py` | AC-5 (README primary path, no Flask/stdio) |

**Run tests:**
```bash
pytest tests/test_readme_setup.py -v
```

---

## Risks

_None beyond README accuracy, mitigated by end-to-end verification test._

---

## Notes

- **Example `MCP_URL` value:** `http://127.0.0.1:8000/sse` included in Configuration section.
- **LLM setup reference:** README documents both `ANTHROPIC_API_KEY` and `LLM_GATEWAY_URL` as alternatives; developer chooses based on their setup. If neither is set, agent returns `{"error": "LLM unavailable"}` (covered by Story 2 tests).
- **Flask and stdio removal:** All references removed in Removed Content section; verified by `test_readme_no_flask_or_stdio`.
- **Closing story:** This story closes the epic when AC-5 is verified (README setup works end-to-end, no Flask/stdio).
- **Development section:** Optional; included here for completeness. If tests are not yet implemented, omit or mark "Coming soon."
- **Existing README sections:** Preserve **License**, **Contributing**, **Changelog** sections if they exist and do not reference Flask/stdio.
- **README template:** The content above is the full README; replace existing `README.md` entirely unless preserving specific sections (see Algorithm Notes).

---

## Open Questions

_None. README structure, content, removed references, and verification test are fully specified._
