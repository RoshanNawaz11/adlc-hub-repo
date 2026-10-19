# Dry-run artifact

Generated without live LLM.

## Input summary

Task: Write requirements.md with Given/When/Then acceptance criteria derived ONLY from the ticket. Do not invent unrelated epic-level ACs. Keep scope minimal and testable. Do NOT choose frameworks, languages, ORMs, or cloud stacks — product outcomes only.

## ticket

Serve MCP GraphQL tools over FastAPI + FastMCP HTTP (replace stdio)

**Job story.**
When an agent or developer calls TriNet MCP tools for sales orders and employees, they want to reach the MCP server over HTTP using FastAPI and FastMCP instead of a local stdio process, so that the same tools stay queryable from multiple clients without spawning a per-client stdio server.

**System information**
**System 1 · MCP server (**`mcp` / adlc-mcp-server)

* Reads: GraphQL via `GRAPHQL_ENDPOINT` (same queries as today)
* Writes: none be

