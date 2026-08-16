# ADR-0005: Frontend artifact integration

- Status: Accepted
- Date: 2026-08-16

## Context

AzurPilot is a Python application. Requiring a Node.js development server in production would add an unnecessary runtime dependency and make frontend/backend repository coupling harder to control.

Vite's production build produces a bundle suitable for static serving.

## Decision

AzurPilot-Web builds independently and publishes a **versioned static frontend artifact**. The AzurPilot runtime integrates and serves that artifact through its Python/ASGI deployment.

```text
AzurPilot-Web -> build -> versioned dist artifact -> AzurPilot runtime integration -> Python/ASGI serving
```

Node.js/pnpm are build-time tools only. A Git submodule/subtree is not an implicit runtime integration mechanism; adopting one would require a separate ADR and demonstrated need.

## Consequences

- Production users do not need Node.js merely to run the WebUI.
- Artifact versioning/integrity and exact packaging are deferred until the frontend build exists.
- Backend integration can evolve without making the two repositories a monorepo.

## Reference

- <https://vite.dev/guide/build>
