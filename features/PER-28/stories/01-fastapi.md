# Story 01-fastapi LLD: Models & Config

**Ticket:** PER-29  
**Story:** 01-fastapi  
**Repo:** See LLD (stack/paths deferred to implementation based on repo context)  
**Status:** Draft

---

## Overview

Define models for query requests/responses and SSE event structures. Add agent service configuration (base URL, timeouts) to config module. This story establishes the data contracts and configuration infrastructure required by all downstream integration stories.

---

## Acceptance Criteria

### AC1: POST response returns agent's immediate reply (schema defined)
**Given** a POST /query request is received  
**When** the endpoint returns a response  
**Then** the response schema supports both "ok" and "rejected" statuses with a run_id field

### AC2: SSE stream forwards status events (event schema defined)
**Given** an SSE stream is initiated for a query run  
**When** events are forwarded from the agent  
**Then** the event schema supports `in_progress`, `completed`, and `failed` event types

---

## Scope

This story defines schemas and configuration only. Concrete file paths, language-specific implementations (Pydantic vs other validation frameworks), and stack choices are deferred to implementation based on actual repo context, which currently shows "(repo path not mapped; do not invent a stack)" per `repo-tree/fastapi`.

---

## Data Contracts

### Query Request Schema

**Fields:**
- `query` (string, required): The query text to submit to the agent

**Validation Requirements:**
- `query` field must be present and non-empty
- Additional fields may be added when agent contract is finalized

**Example:**
```json
{"query": "test query"}
```

---

### Immediate Response Schema

**Fields:**
- `status` (enum: "ok" | "rejected", required): Agent's immediate reply status
- `run_id` (string, required): Unique identifier for this run (UUID4 format)

**Serialization Example:**
```json
{"status": "ok", "run_id": "123e4567-e89b-12d3-a456-426614174000"}
```

---

### SSE Event Schemas

#### InProgressEvent
**Fields:**
- `run_id` (string, required): Run identifier

**SSE Format:**
```
event: in_progress
data: {"run_id": "<uuid>"}
```

#### CompletedEvent
**Fields:**
- `run_id` (string, required): Run identifier
- `response` (object, required): Agent's final response (shape determined by agent contract)

**SSE Format:**
```
event: completed
data: {"run_id": "<uuid>", "response": {...}}
```

**Note:** `response` schema is placeholder; final shape depends on agent service contract.

#### FailedEvent
**Fields:**
- `run_id` (string, required): Run identifier
- `error` (string, required): Error message

**SSE Format:**
```
event: failed
data: {"run_id": "<uuid>", "error": "<message>"}
```

---

## Configuration Requirements

### Agent Service Configuration

**Required Fields:**
- `AGENT_BASE_URL` (string): Base URL of agent service
  - Default: `http://localhost:8001` (local development)
  - Source: Environment variable override supported
- `AGENT_TIMEOUT` (integer): Request timeout in seconds
  - Default: `3` (aligns with immediate reply requirement)
  - Source: Environment variable override supported

**Rationale:**
- Default URL assumes local development agent service
- 3-second timeout aligns with requirements for immediate reply (agent should respond < 3s)
- Environment variables allow production override without code changes

---

## Implementation Guidance

When implementing this story in the actual codebase:

1. **Choose validation framework** based on existing repo patterns (e.g., Pydantic for Python/FastAPI, Joi for Node.js, built-in validation for other stacks)
2. **Locate or create config module** following project conventions
3. **Define schemas** matching the data contracts above, using appropriate types for the chosen stack
4. **Implement validation** to enforce required fields and types
5. **Use dataclasses or equivalent** for SSE events (lighter weight than full validation frameworks)
6. **Add config loading** from environment variables with specified defaults

**Key Principle:** The contracts above (field names, types, defaults) are fixed. The implementation syntax (Pydantic vs other) adapts to the repo's stack.

---

## Tests

### Test 1: Query Request Validation

**Verification:** requirements.md AC1 (schema validation for incoming requests)

**Acceptance Criteria:** AC1 (POST response schema defined)

**Test Coverage:**
- Valid request with `query` field accepted
- Request missing `query` field rejected with validation error
- Validation error message identifies missing `query` field

---

### Test 2: Immediate Response Serialization

**Verification:** requirements.md AC1 (response schema supports ok/rejected with run_id)

**Acceptance Criteria:** AC1 (POST response schema defined)

**Test Coverage:**
- Response with `status: "ok"` and `run_id` serializes correctly
- Response with `status: "rejected"` and `run_id` serializes correctly
- Serialized JSON matches expected format: `{"status": "<ok|rejected>", "run_id": "<uuid>"}`

---

### Test 3: SSE Event Schema Instantiation

**Verification:** requirements.md AC2 (event schemas support in_progress/completed/failed)

**Acceptance Criteria:** AC2 (SSE event schemas defined)

**Test Coverage:**
- InProgressEvent with `run_id` instantiates correctly
- CompletedEvent with `run_id` and `response` instantiates correctly
- FailedEvent with `run_id` and `error` instantiates correctly
- Each event converts to expected JSON structure

---

### Test 4: Config Loading

**Verification:** requirements.md AC1, AC2 (config provides agent service parameters for downstream stories)

**Acceptance Criteria:** AC1, AC2 (configuration defined)

**Test Coverage:**
- `AGENT_BASE_URL` defaults to `http://localhost:8001` when environment variable absent
- `AGENT_BASE_URL` loads from environment variable when present
- `AGENT_TIMEOUT` defaults to `3` when environment variable absent
- `AGENT_TIMEOUT` loads from environment variable when present (as integer)

---

## Dependencies

**Runtime:**
- Validation framework (determined by repo stack)
- Configuration loading mechanism (determined by repo conventions)
- Standard library equivalents of dataclasses/structs

**Test:**
- Project's test framework
- Assertion library

**Notes:**
- Dependencies listed abstractly since stack not yet mapped
- Implementation adds concrete dependencies based on chosen stack

---

## Open Questions

1. **Agent response schema:** `CompletedEvent.response` shape is unknown until agent contract finalized. Current definition accepts any object; refine when schema known.

2. **Config location and pattern:** Where does configuration live in this project? (e.g., dedicated config file, environment-based settings class, dotenv integration)

3. **UUID validation strictness:** Should response schema validate UUID4 format, or defer to caller (Story 4 generates UUID4)?

4. **Stack confirmation:** Current repo-tree shows "(repo path not mapped; do not invent a stack)". Before implementing, confirm:
   - Language and framework
   - Existing validation patterns
   - Config management approach
   - File structure conventions

---

## Risks

### Risk 1: Validation framework API changes
**Impact:** Implementation syntax differs from documentation  
**Likelihood:** Low (frameworks typically stable)  
**Mitigation:** Check framework version during implementation; adapt syntax as needed

### Risk 2: Config module pattern mismatch
**Impact:** Config loading differs from project conventions  
**Likelihood:** Medium (project patterns unknown until stack mapped)  
**Mitigation:** Survey existing config patterns before implementing; follow established conventions

### Risk 3: Serialization format assumptions
**Impact:** JSON serialization differs from expected format  
**Likelihood:** Low (JSON format is standard)  
**Mitigation:** Tests verify exact JSON output format

---

## Notes

- **No business logic:** This story is pure schema/config definition. Zero runtime behavior beyond validation and configuration loading.
- **Stack-agnostic design:** Schemas defined as data contracts, not language-specific code. Implementation adapts to repo's actual stack.
- **UUID generation deferred:** Story 4 (POST /query) generates `run_id`. This story only defines the field in response schema.
- **No timer logic:** Per requirements (change_requests.md), FastAPI does not create or delay status events. All timing logic remains on the agent side. This story only defines the schemas for forwarding agent-generated events.
- **Pure pass-through architecture:** The backend is a pure pass-through: agent → backend → SSE stream. This story provides the data contracts for that pass-through, with no timing or status authority.
- **Implementation deferred:** Concrete file paths, imports, and language-specific syntax removed per ADLC rule: "File paths and stack MUST match repo context / existing tree. Never invent another language or framework." Implementation awaits repo-tree mapping.

---

**Document Status:** Draft  
**Story:** 01-fastapi (PER-29)  
**Epic:** FastAPI SSE Integration with Agent Service (requirements.md AC1, AC2)  
**Altitude:** Story-level LLD
