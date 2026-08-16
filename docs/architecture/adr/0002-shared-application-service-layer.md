# ADR-0002: Shared application/service layer

- Status: Accepted
- Date: 2026-08-16

## Context

Current WebUI and legacy MCP code can reach runtime internals directly. If React HTTP endpoints and future MCP tools independently reproduce these actions, AzurPilot would have multiple business implementations with inconsistent validation, authorization and safety behavior.

## Decision

HTTP/UI, MCP and local compatibility adapters converge on one backend **application/service layer**.

```text
React/HTTP ----+
               |
MCP -----------+----> Application/Service Layer ----> AzurPilot Core
               |
Local UI ------+
```

Adapters translate transport/protocol concerns. Services own use-case orchestration and call runtime internals. Destructive actions share a backend policy path for authorization, validation, audit and safety requirements.

## Consequences

- Target MCP tools must not directly call `ProcessManager`, `AzurLaneConfig`, `Device` or storage primitives.
- React must not reimplement scheduler/config/process behavior.
- Backend service boundaries may need to be introduced incrementally before equivalent React/MCP actions are exposed.
- Service package/class naming is deferred to backend implementation work.
