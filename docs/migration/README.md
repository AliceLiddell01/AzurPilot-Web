# Инвентаризация миграционной поверхности AzurPilot

Статус: **Этап 1 — WebUI проинвентаризирован; Этап 2 — текущий MCP проаудирован на том же pinned backend snapshot**.

Источник инвентаризации текущего WebUI и MCP:

```text
AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379
```

Базовая архитектурная граница находится в [архитектурной основе Этапа 0](../architecture/ARCHITECTURE.md).

## Назначение

Эти документы описывают фактический PyWebIO/ASGI WebUI и текущий MCP AzurPilot на одном зафиксированном backend snapshot. Они нужны, чтобы будущие React/HTTP/MCP adapters не копировали прямые обращения к `ProcessManager`, config-файлам, `Device`, ADB, локальной файловой системе или другой runtime-логике.

Это **не** проект финального `/api/v1`, не OpenAPI-спецификация, не реализация Streamable HTTP и не обещание переносить каждую найденную capability.

## Документы Этапа 1 — WebUI

- [Текущие пользовательские поверхности](CURRENT_WEBUI_SURFACES.md) — карта навигации и рабочих сценариев, чтения/записи, связность со средой выполнения и доказательства.
- [Текущие интерфейсы бэкенда](CURRENT_BACKEND_INTERFACES.md) — рабочая сборка HTTP/SSE/PyWebIO/static/MCP и отдельно объявленные, но отключённые custom WebSocket routes.
- [Матрица зависимостей и готовности](DEPENDENCY_AND_READINESS_MATRIX.md) — поверхность → зависимость → теги миграции → требование к бэкенду.
- [Основные выводы](MIGRATION_FINDINGS.md) — security blockers, повторяющиеся обязательные условия и архитектурная готовность.

## Документы Этапа 2 — MCP

- [Индекс аудита MCP](mcp/README.md) — scope, правила доказательности, counts и MCP-owned files.
- [Текущая архитектура MCP](mcp/CURRENT_MCP_ARCHITECTURE.md) — dependency/version, embedded/standalone wiring, legacy session transport, auth/security/error flow.
- [Матрица MCP capabilities](mcp/MCP_CAPABILITY_MATRIX.md) — 17/17 tools с публичным именем/URI semantics, inputs/outputs, side effects, dependencies, risk и evidence.
- [Выводы по безопасности MCP](mcp/MCP_SECURITY_FINDINGS.md) — доказанные transport/capability findings с единым шаблоном evidence/impact/proven/unproven/severity/prerequisite.
- [Целевая граница и решения](mcp/MCP_TARGET_BOUNDARY_AND_DECISIONS.md) — service owners, permissions, Tool Annotations, RETAIN/REDESIGN/DEFER, read-only MVP и human confirmation policy.
- [Сверка приёмки Этапа 2](mcp/STAGE_2_ACCEPTANCE_RECONCILIATION.md) — repository-wide discovery proof, обязательные поля prompt, cross-document consistency и Definition of Done перед Ready.

## Правило доказательности

Факт текущей реализации считается подтверждённым только по коду pinned SHA. Основной формат ссылки:

```text
AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379:<path>::<symbol>
```

Где возможно, рядом используется permanent GitHub permalink на тот же SHA. Плавающая `personal/stable` не используется как доказательство snapshot.

Термины:

- **CURRENT FACT / Подтверждено кодом** — выполнение/подключение можно проследить по pinned snapshot и, если вывод зависит от dependency behavior, по exact dependency version;
- **Не найдено статическим аудитом** — поиск и аудит подключения не обнаружили связь; это не утверждение о мёртвом коде;
- **OFFICIAL MCP TARGET GUIDANCE** — требования/рекомендации актуальной официальной MCP specification/SDK, не current AzurPilot behavior;
- **TARGET DECISION / Целевое решение** — архитектурное решение будущего AzurPilot;
- **UNVERIFIED** — статический аудит не позволяет доказать поведение достаточно надёжно;
- **DEFERRED / Отложенное решение** — продуктовый/архитектурный выбор передан следующему этапу.

## Снимок покрытия WebUI

На pinned backend snapshot подтверждено:

- production `AlasGUI` mixins: **22 обнаружено / 22 сопоставлено / 0 несопоставленных**;
- production `module/webui/app_*.py`: **28 проверено / 28 классифицировано / 0 неклассифицированных**;
- task destinations из `module/config/argument/menu.json`: **90**;
- явные project HTTP registrations, подключённые рабочим ASGI: **15** (14 из `api_routes` + `/robots.txt`);
- SSE surfaces среди них: **2**;
- custom live WebSocket routes: **2 объявлено / 0 подключено через production `api_routes`**;
- production static mounts: **3**;
- PyWebIO application entry points: **2**;
- MCP mount: **1** (`/mcp`).

## Снимок покрытия MCP

```text
tools: 17 / 17 классифицировано
resources: 0
resource templates: 0
prompts: 0
completions: 0
subscriptions: 0
custom MCP transport path families: 2 / 2
experimental/extensions registrations: 0
несопоставленных registrations: 0
```

Распределение риска:

```text
R0=2, R1=7, R2=0, R3=7, R4=1
```

Миграционные решения:

```text
RETAIN=7, REDESIGN=9, REMOVE=0, DEFER=1
```

Кандидат read-only MCP MVP: 7 обязательных read capabilities и условный `get_recent_logs` после bounds/sanitization.

## Важное предупреждение о достижимости

Наличие handler/mount/bind само по себе не доказывает Internet reachability. Например, standalone MCP содержит `uvicorn(... host="0.0.0.0", port=22268)`, но router/NAT/firewall/OS ACL/Caddy конкретной машины статическим аудитом не устанавливаются.

Текущий PyWebIO password gate находится внутри `_run_gui()` и не является общим ASGI auth middleware для MCP child app.

## Область работы

Этапы 1–2 не меняют backend, PyWebIO, MCP transport, auth или Caddy. Этап 2 также не создаёт Service Layer, не мигрирует MCP на Streamable HTTP и не добавляет/удаляет tools. Найденные обязательные условия являются входными данными для следующих backend design stages.
