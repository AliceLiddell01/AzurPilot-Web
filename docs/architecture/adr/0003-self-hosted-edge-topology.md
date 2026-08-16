# ADR-0003: Self-hosted public edge topology

- Status: Accepted
- Date: 2026-08-16

## Context

The intended product is a personal self-hosted control panel on the user's PC with public Internet access. Publishing AzurPilot's internal application ports directly would enlarge the trust boundary and complicate browser security.

## Decision

Caddy is the target **single public HTTP(S) edge**.

```text
Internet -> HTTPS -> Caddy -> 127.0.0.1:<backend-port>
```

Production is same-origin:

```text
https://<user-domain>/
https://<user-domain>/api/v1/...
wss://<user-domain>/...
https://<user-domain>/mcp
```

The backend should bind to loopback/local interface in the target deployment. Wildcard CORS is not the production default.

A reverse-proxy TCP peer on loopback does not make the originating request a trusted local-user request. Forwarded metadata is trusted only from the configured proxy boundary, and authorization remains a backend responsibility.

## Consequences

- Caddy owns public TLS termination/reverse proxying.
- Direct external exposure of internal AzurPilot ports is not the target deployment.
- Backend proxy-awareness and remote authentication are prerequisites for public deployment.
- Caddy installation, DNS and router forwarding remain deployment work outside Stage 0.

## References

- <https://caddyserver.com/docs/automatic-https>
- <https://caddyserver.com/docs/caddyfile/directives/reverse_proxy>
