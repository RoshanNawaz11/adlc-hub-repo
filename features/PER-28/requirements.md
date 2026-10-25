# requirements.md

## Overview
FastAPI backend integration with an agent service that provides immediate synchronous responses and asynchronous status updates via Server-Sent Events (SSE).

## Acceptance Criteria

### AC1: Immediate POST response
**Given** a POST request to `/query`  
**When** the request is received  
**Then** FastAPI returns the agent's immediate reply (Ok or rejected) without delay

### AC2: SSE stream initiation
**Given** a query has been submitted  
**When** the agent run starts  
**Then** the SSE stream sends an `in_progress` event as the first event

### AC3: Fast completion (under 5 seconds)
**Given** the agent completes its response in under 5 seconds  
**When** 5 seconds have elapsed since the first `in_progress` event  
**Then** the stream sends a `completed` event with the agent's response and ends

### AC4: Slow completion (5 seconds or more)
**Given** the agent completes its response in 5 seconds or longer  
**When** the response becomes ready  
**Then** the stream immediately sends a `completed` event with the agent's response and ends

### AC5: Failure handling
**Given** the agent run fails  
**When** the failure is detected  
**Then** the stream sends a `failed` event, does not send `completed`, and ends

### AC6: No polling behavior
**Given** the integration is running  
**When** waiting for agent responses  
**Then** FastAPI does not poll the agent for status or responses

### AC7: No MCP tool calls
**Given** the integration is running  
**When** processing requests  
**Then** FastAPI does not call MCP tools

## Notes
- The 5-second delay applies only to the `completed` event in fast-completion scenarios
- The immediate Ok/rejected reply is never delayed
- The agent call itself is never held or blocked by the delay
- SSE stream lifecycle: exactly one status event (`in_progress`), followed by exactly one terminal event (`completed` or `failed`)
