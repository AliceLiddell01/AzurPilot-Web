# Инвентаризация миграционной поверхности WebUI

Статус: **Этап 1 — текущий snapshot проинвентаризирован статически**.

Current WebUI inventory source:

```text
AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379
```

Базовая архитектурная граница находится в [архитектурной основе Этапа 0](../architecture/ARCHITECTURE.md).

## Назначение

Эти документы описывают фактический WebUI AzurPilot на одном зафиксированном backend snapshot. Они нужны, чтобы будущий React-клиент не копировал в браузер PyWebIO-логику, прямые обращения к `ProcessManager`, config-файлам, устройству, ADB, локальной файловой системе или другим runtime-внутренностям.

Это **не** проект финального `/api/v1`, не OpenAPI-спецификация и не обещание переносить каждую найденную поверхность.

## Документы

- [Текущие пользовательские поверхности](CURRENT_WEBUI_SURFACES.md) — navigation/workflow map, reads/writes, runtime coupling и evidence.
- [Текущие backend interfaces](CURRENT_BACKEND_INTERFACES.md) — production HTTP/SSE/PyWebIO/static/MCP wiring и отдельно объявленные, но отключённые custom WebSocket routes.
- [Матрица зависимостей и готовности](DEPENDENCY_AND_READINESS_MATRIX.md) — surface → dependency → migration tags → backend prerequisite.
- [Основные выводы](MIGRATION_FINDINGS.md) — answer-first итог, security blockers, повторяющиеся prerequisite families и очередность по архитектурной готовности.

## Правило доказательности

Факт текущей реализации считается подтверждённым только по коду pinned SHA. Основной формат ссылки:

```text
AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379:<path>::<symbol>
```

Где возможно, рядом используется permalink на тот же SHA. Плавающая `personal/stable` не используется как доказательство snapshot.

Термины в документах:

- **Подтверждено кодом** — execution/wiring можно проследить по pinned snapshot.
- **Не найдено статическим аудитом** — поиск и wiring audit не обнаружили связь; это не утверждение о dead code.
- **UNVERIFIED** — статический аудит не позволяет доказать поведение достаточно надёжно.
- **Target inference** — следствие архитектурной основы, а не факт текущей реализации.
- **Deferred decision** — продуктовый/архитектурный выбор намеренно оставлен следующему этапу.

## Coverage snapshot

На pinned backend snapshot подтверждено:

- production `AlasGUI` mixins: **22 обнаружено / 22 mapped / 0 unmapped**;
- production `module/webui/app_*.py`: **28 inspected / 28 classified / 0 unclassified**;
- task destinations из `module/config/argument/menu.json`: **90**, объединены в migration families по фактическому общему wiring;
- explicit project HTTP registrations, подключённых production ASGI: **15** (14 из `api_routes` + `/robots.txt`);
- SSE surfaces среди них: **2**;
- custom live WebSocket routes: **2 объявлено / 0 подключено production `api_routes`**, потому что `fastapi.py` их фильтрует;
- production static mounts: **3** (`/static/assets`, `/static/doc`, `/pywebio_static`);
- PyWebIO application entry points: **2** (`index`, `manage`);
- MCP mount: **1** (`/mcp`), legacy SSE/message transport внутри mount.

Точное количество внутренних Starlette routes, генерируемых библиотечной функцией PyWebIO `webio_routes()`, не объявляется как собственный project route count: проект подтверждает два application entry point, а детализация framework-generated protocol routes принадлежит зависимости.

## Важное предупреждение о reachability

Наличие handler или записи в `api_routes` само по себе не доказывает production reachability. Например, `/ws/live_screenshot` и `/ws/live_control` определены в `module/webui/api.py`, но исключаются через `DISABLED_API_ROUTE_PATHS` при сборке production ASGI.

Аналогично текущий PyWebIO password gate находится внутри `_run_gui()` и не является общим ASGI middleware для всех дополнительных HTTP/MCP routes. Поэтому security свойства каждого дополнительного interface фиксируются отдельно.

## Scope

Этап 1 не меняет backend, PyWebIO, API, MCP, auth или Caddy. Все найденные prerequisites являются требованиями к будущей миграции, а не реализацией в этом репозитории.
