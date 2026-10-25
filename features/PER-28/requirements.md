# requirements.md

## Overview
Backend integration with an agent service that provides immediate synchronous responses and asynchronous status updates via Server-Sent Events (SSE).

## Acceptance Criteria

### AC1: Immediate POST response
**Given** a POST request to `/query`  
**When** the request is received  
**Then** the backend returns the agent's immediate reply (Ok or rejected) without delay

### AC2: SSE stream forwarding
**Given** a query has been submitted  
**When** the SSE stream is opened for that run  
**Then** the stream forwards the status and response events the agent already sent

### AC3: No status creation
**Given** the integration is running  
**When** processing requests  
**Then** the backend does not create, decide, or delay run status (no timer logic, no in_progress/completed/failed generation)

### AC4: No polling behavior
**Given** the integration is running  
**When** waiting for agent responses  
**Then** the backend does not poll the agent for status or responses

### AC5: No MCP tool calls
**Given** the integration is running  
**When** processing requests  
**Then** the backend does not call MCP tools

## Notes
- The immediate Ok/rejected reply is never delayed
- The agent call itself is never held or blocked
- All status and timing logic remains on the agent side (addressed in a later ticket)
- The backend is a pure pass-through: agent → backend → SSE stream
- Stack and file paths deferred to breakdown/stories after repo context is mapped
