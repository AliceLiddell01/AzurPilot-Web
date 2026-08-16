# ADR-0004: MCP adapter boundary

- Status: Accepted
- Date: 2026-08-16

## Context

`AzurPilot-private-Ru` already contains `mcp_server_sse.py`. The current implementation uses the older SSE transport and directly reaches privileged runtime internals for process, config, device, scheduler, logs, ADB and emulator actions.

The MCP specification replaced HTTP+SSE with Streamable HTTP, and the official Python SDK recommends Streamable HTTP for deployed servers.

## Decision

MCP remains in `AzurPilot-private-Ru`; it is not moved into `AzurPilot-Web`, and the frontend has no MCP SDK dependency.

The current MCP server is classified as a **legacy current-state adapter**. The target architecture is:

```text
MCP client -> Caddy -> /mcp -> MCP adapter -> Application/Service Layer -> AzurPilot Core
```

Target production transport is **Streamable HTTP**. Legacy SSE may remain only as a deliberate temporary compatibility mechanism if required by real clients.

Resources, Prompts and Tools are designed according to MCP semantics rather than modelling every capability as a Tool. Write/destructive tools require server-side permission checks, validation, audit and human-in-the-loop policy where necessary.

Web authentication and MCP authentication remain distinct security contracts even if they later share identity/authorization infrastructure.

## Consequences

- No new frontend dependency on the MCP Python SDK.
- MCP business logic must migrate toward shared services before privileged capability expansion.
- `/mcp` is published through the same protected public edge, not as an unauthenticated standalone Internet port.
- Exact MCP auth mechanism is deferred.

## References

- <https://modelcontextprotocol.io/specification/2025-03-26/basic/transports>
- <https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/index.md>
- <https://github.com/modelcontextprotocol/python-sdk/blob/main/docs/run/asgi.md>
