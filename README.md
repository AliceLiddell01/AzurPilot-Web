# AzurPilot-Web

AzurPilot-Web is the future browser frontend for [AzurPilot](https://github.com/AliceLiddell01/AzurPilot-private-Ru). It is intentionally a **frontend-only** repository: UI code may consume a public, versioned backend contract, but privileged AzurPilot runtime logic remains in the backend/core repository.

## Project status

**Stage 0 — architecture baseline.** The production React application does not exist yet. This repository currently defines the system boundary, delivery model, and contribution rules that future frontend work must follow before a React/Vite foundation is added.

The target direction is a self-hosted control panel served from the user's AzurPilot machine:

```text
Internet -> HTTPS -> Caddy -> AzurPilot backend
                            |-> static frontend artifact
                            |-> /api/v1/...
                            |-> realtime endpoints
                            `-> /mcp
```

The backend, not the browser, owns authentication/authorization, process control, configuration/storage access, device/MuMu interaction, scheduler operations, and MCP implementation.

## Architecture

Start with [docs/architecture/ARCHITECTURE.md](docs/architecture/ARCHITECTURE.md). Accepted architectural decisions are indexed in [docs/architecture/DECISIONS.md](docs/architecture/DECISIONS.md).

Key invariant:

```text
AzurPilot-Web = frontend client
AzurPilot-private-Ru = backend/core owner
HTTP API + MCP = adapters
Application/Service Layer = single business boundary
Caddy = public HTTPS edge
AzurPilot internal runtime = never a frontend concern
```

## Development workflow

Development follows Issue -> short-lived topic branch -> Draft PR -> checks/review -> merge. See [CONTRIBUTING.md](CONTRIBUTING.md).

Do not commit real deployment IPs, domains, credentials, cookies, tokens, device identifiers, private keys, or `.env` contents to this public repository. Documentation uses placeholders such as `<user-domain>` and `127.0.0.1:<backend-port>`.
