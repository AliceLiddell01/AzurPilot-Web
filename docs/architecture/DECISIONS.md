# Architecture Decision Records

Accepted Stage 0 decisions:

| ADR | Decision | Status |
| --- | --- | --- |
| [ADR-0001](adr/0001-frontend-repository-boundary.md) | Keep AzurPilot-Web frontend-only | Accepted |
| [ADR-0002](adr/0002-shared-application-service-layer.md) | Converge UI/HTTP/MCP on one backend application/service layer | Accepted |
| [ADR-0003](adr/0003-self-hosted-edge-topology.md) | Use same-origin Caddy edge with loopback backend | Accepted |
| [ADR-0004](adr/0004-mcp-adapter-boundary.md) | Keep MCP backend-side and migrate target transport to Streamable HTTP | Accepted |
| [ADR-0005](adr/0005-frontend-artifact-integration.md) | Ship the frontend as a versioned static build artifact | Accepted |

ADRs record architectural constraints, not proof that target implementation already exists. Implementation status is tracked separately in the relevant Stage/Issue/PR.
