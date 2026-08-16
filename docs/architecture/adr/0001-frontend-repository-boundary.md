# ADR-0001: Frontend repository boundary

- Status: Accepted
- Date: 2026-08-16

## Context

AzurPilot already owns privileged Python runtime behavior: process lifecycle, config/storage access, device and emulator control, scheduler logic, authentication concerns and MCP. A new React repository creates a risk of duplicating these responsibilities merely because browser UI work is now separated physically.

## Decision

`AzurPilot-Web` is a **frontend-only** repository.

It owns browser UI, TypeScript, frontend routing/state, API client, design system, frontend tests/build and the frontend release artifact.

It does not own AzurPilot business logic, `ProcessManager`, config/storage access, ADB/MuMu/OCR/game automation, server-side auth, MCP implementation or Caddy runtime management.

The backend/core repository, `AzurPilot-private-Ru`, remains the owner of privileged runtime integrations and server-side adapters.

## Consequences

- React depends on a versioned public backend contract, never Python implementation classes or local files.
- Cross-repo features may require coordinated PRs, but each repository keeps a clear responsibility boundary.
- Backend prerequisites are implemented in backend Stages rather than smuggled into the frontend repository.
- A monorepo is not required for the current self-hosted product.
