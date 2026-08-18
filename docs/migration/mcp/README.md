# Аудит текущего MCP AzurPilot

Статус: **Этап 2 — статический аудит текущего MCP на зафиксированном backend snapshot**.

Источник текущего состояния:

```text
AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379
```

База документации frontend-репозитория:

```text
AliceLiddell01/AzurPilot-Web@1047f3bd9fc193ba5d69ae87db4d8342710ee3a5
```

## Назначение

Эти документы описывают фактически опубликованный MCP AzurPilot до его рефакторинга. Аудит нужен, чтобы следующий backend-этап мог вынести общие возможности из прямых вызовов `ProcessManager`, `AzurLaneConfig`, `Device`, файловой системы и системных операций в общий Application/Service Layer, который позднее смогут использовать и MCP, и HTTP API.

Этап 2 **не** меняет production MCP, backend, transport, auth, React или Caddy.

## Правило терминологии

В документах явно различаются пять статусов:

- **CURRENT FACT** — доказано кодом pinned backend snapshot и, когда вывод зависит от библиотеки, точной версией `mcp==1.23.0`;
- **OFFICIAL MCP TARGET GUIDANCE** — требования и рекомендации актуальной опубликованной спецификации MCP `2026-07-28` и официального Python SDK;
- **TARGET DECISION** — решение для будущей архитектуры AzurPilot, а не описание текущего кода;
- **UNVERIFIED** — факт нельзя доказать достаточно надёжно статическим аудитом;
- **DEFERRED** — решение намеренно передано следующему продуктово-архитектурному этапу.

## Документы

- [Текущая архитектура MCP](CURRENT_MCP_ARCHITECTURE.md) — production wiring, SDK/transport, embedded и standalone режимы, lifecycle, security и прямые runtime dependencies.
- [Матрица capabilities](MCP_CAPABILITY_MATRIX.md) — полный tool-by-tool аудит, risk, validation, error/retry semantics и evidence.
- [Выводы по безопасности](MCP_SECURITY_FINDINGS.md) — доказанные transport/capability findings с обязательными полями доказательства, влияния, границ доказанности, severity и целевых условий.
- [Целевая граница и решения](MCP_TARGET_BOUNDARY_AND_DECISIONS.md) — service owner, permissions, Tool Annotations, RETAIN/REDESIGN/REMOVE/DEFER, read-only MVP и confirmation policy.
- [Сверка приёмки Этапа 2](STAGE_2_ACCEPTANCE_RECONCILIATION.md) — discovery-first completeness, явный `Exposed name/URI`, cross-document coverage и Definition of Done перед Ready.

## Проверяемое покрытие

На pinned snapshot обнаружено:

```text
tools: 17
resources: 0
resource templates: 0
prompts: 0
completions: 0
subscriptions: 0
custom MCP transport path families: 2
experimental/extensions registrations: 0
unmapped registrations: 0
```

Все 17 tools регистрируются одним `list_tools()` и имеют соответствующие entries в `TOOL_HANDLERS`. Единственный зарегистрированный MCP protocol capability family — Tools. Resources, prompts и completions не регистрируются; experimental API у `Server` не используется.

## MCP-owned и wiring-файлы

Фактическая production-цепочка MCP использует:

- `mcp_server_sse.py` — server construction, 17 tool registrations, dispatch, legacy SSE/message transport, ASGI wrapper и standalone entry point;
- `module/config/mcp_helper.py` — MCP-specific metadata helper для task definitions и Dashboard resources;
- `module/webui/app.py` — embedded production mount `/mcp`;
- `pyproject.toml` — точная зависимость `mcp==1.23.0` и связанные HTTP/runtime dependencies.

`ProcessManager`, `AzurLaneConfig`, `Device`, `State`, config/i18n helpers и другие импортируемые модули считаются **общими backend/runtime dependencies**, а не MCP-owned кодом.

Repository-wide discovery и import/wiring graph не обнаружили второго production MCP server construction, второго registration catalog или альтернативного production mount на pinned snapshot. Подробная методика и границы такого вывода записаны в сверке приёмки.

## Источники доказательств

Current-state ссылки используют exact backend SHA, например:

```text
https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py
```

Официальный target baseline:

- спецификация MCP `2026-07-28`: <https://modelcontextprotocol.io/specification/2026-07-28>;
- release/deprecation context: <https://blog.modelcontextprotocol.io/posts/2026-07-28/>;
- официальный Python SDK: <https://github.com/modelcontextprotocol/python-sdk>;
- security/support status SDK: <https://github.com/modelcontextprotocol/python-sdk/blob/main/SECURITY.md>.

На момент финальной сверки Этапа 2 официальный Python SDK v2 является текущей stable line; v1.x находится в maintenance mode. Это **OFFICIAL MCP TARGET GUIDANCE**, а не описание текущего `mcp==1.23.0` в AzurPilot.

## Главная граница

```text
CURRENT
MCP legacy adapter
    -> ProcessManager / AzurLaneConfig / Device / FS / subprocess

TARGET
MCP adapter
    -> authentication / authorization / policy
    -> Application / Service Layer
    -> AzurPilot Core

HTTP adapter
    -> тот же Application / Service Layer
```

`RETAIN` в документах означает сохранение продуктовой capability, а **не** сохранение legacy direct-call implementation.
