# Основные выводы миграционного аудита

Audit source: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## Краткий вывод

Текущий WebUI нельзя рассматривать как тонкий presentation client. В нём одновременно существуют:

- хорошо отделённые statistics/read-model helpers;
- частичные HTTP/SSE interfaces;
- PyWebIO callbacks с прямой записью config;
- callbacks, управляющие `ProcessManager`, scheduler и process-wide `State`;
- локально-машинные filesystem/launcher workflows;
- event-specific automation safety logic внутри UI mixins;
- legacy remote access и MCP adapters с собственными trust assumptions.

Поэтому безопасная React-миграция должна идти не «страница за страницей», а по dependency boundary: сначала formalized read contracts и service/policy extraction там, где UI сегодня владеет orchestration.

## 1. Что уже отделено лучше всего

### Statistics domain

S-12–S-18 уже читают специализированные backend statistics-модули. UI всё ещё выполняет часть aggregation/chart formatting, но источник данных и persistence находятся не в браузерном слое.

Практическое следствие:

```text
statistics storage/helper
    -> formalized read model
    -> versioned backend contract (отдельный будущий design stage)
    -> React presentation
```

Это архитектурно наиболее ранний migration candidate. Existing `/api/cl1_stats` и `/api/ap_timeline` доказывают, что transport seam уже возможен, но **не** доказывают готовность финального public API.

### Developer Settings как пример adapter boundary

`DeveloperSettingsMixin` уже использует `/api/launcher/*` и `/api/deploy/*`, а не вызывает launcher/deploy persistence прямо из browser-side callback. Это полезный текущий seam, но contract остаётся local/legacy и требует security redesign.

## 2. Где самая высокая связность UI/runtime

### Instance lifecycle и Overview

Instance selection/management и Overview напрямую используют `ProcessManager`, config files, start/stop и runtime log. React не должен импортировать семантику этих callbacks или пытаться воспроизводить process state в клиенте.

**Future backend prerequisite:** единый instance status/read model и lifecycle command boundary с валидацией, authorization и audit.

### Generic Task Configuration

`TaskConfigMixin::_save_config()` является не просто сериализацией формы. Он:

- парсит типы;
- проверяет validation expressions;
- подставляет defaults;
- вызывает `config_updater.save_callback`;
- выполняет UI/domain side effects;
- пишет конфигурацию.

**Future backend prerequisite:** validated configuration operation должна принадлежать backend service boundary. Frontend получает schema/read state и отправляет intent, но не копирует Python save logic.

### Event planning и EventShop safety

Event mixins особенно важны: UI не только отображает план, но и меняет event policy. `EventShopSafetyMixin` способен fail-closed отключить Scheduler и синхронизировать automation filter/state.

**Future backend prerequisite:** event planning/shop safety rules должны быть backend-owned domain/policy operations до переноса editing UI.

### Developer utilities / simulator

WebUI restart, manager state override, simulator lifecycle и marker files — прямой privileged runtime control. Их remote migration не является обязательной частью parity и требует отдельного product/security решения.

## 3. Security/trust blockers для remote WebUI

### PyWebIO login не является общим ASGI auth boundary

Password gate вызывается в `app.py::_run_gui()`. `fastapi.py` при этом добавляет project HTTP routes и `/mcp` на уровне Starlette без общего authentication middleware, подтверждённого audited code.

Следствие: нельзя считать `/api/*` или `/mcp` защищёнными только потому, что обычная PyWebIO страница показывает login.

### Current localhost guard несовместим с целевой Caddy topology

`launcher.py::is_local_request()` требует одновременно loopback peer и loopback `Host`. За same-origin Caddy browser будет отправлять пользовательский domain в Host, поэтому current guard не является готовой proxy-aware authorization model.

Нельзя «исправить» это простым правилом `peer == 127.0.0.1`: тогда любой proxied внешний request выглядел бы локальным.

**Future backend prerequisite:** explicit trusted-proxy handling + authenticated authorization. Forwarded client metadata принимается только от доказанно доверенного proxy peer.

### Legacy import требует отдельной защиты

`/api/import_legacy_upload` является state-changing filesystem handler. В его собственном коде не найден auth/locality guard, хотя обычный OOBE вызывается из password-gated PyWebIO flow.

Это не утверждение о фактической интернет-достижимости на машине пользователя. Это доказанный handler-level trust gap, который должен быть закрыт до публикации через общий public edge.

### Custom live WebSocket handlers не production-connected

`/ws/live_screenshot` и `/ws/live_control` существуют в `api.py`, но `fastapi.py` исключает их из production route extension. Это важное уточнение Stage 0 current-state picture.

Если functionality когда-либо возвращается, её current direct-peer locality guard недостаточен за reverse proxy. Device control потребует `REALTIME_CONTRACT + SECURITY_REDESIGN + PRIVILEGED_POLICY`.

### MCP остаётся отдельным legacy adapter

На Stage 1 достаточно подтверждено, что MCP:

- mounted в `/mcp`;
- использует legacy SSE/message transport;
- напрямую достигает config/process/device/emulator/ADB/log runtime;
- имеет wildcard CORS wrapper;
- не проходит через общий Application/Service Layer.

Подробный tool-by-tool risk/transport redesign намеренно **не выполнялся** и передаётся Stage 2.

## 4. Повторяющиеся migration prerequisite families

### A. Instance lifecycle / process status

Нужно отделить status/read и start/stop/restart commands от `ProcessManager`-specific UI wiring.

Затрагивает S-04–S-07, S-10, S-26 и MCP boundary.

### B. Validated configuration / scheduler commands

Нужен backend-owned путь чтения/изменения config с текущими validation/save-callback/domain semantics.

Затрагивает S-02, S-06, S-08, S-09, S-11, S-19–S-22, S-25.

### C. Statistics read models

Нужны проверяемые read schemas/semantics поверх существующих statistics stores/helpers.

Затрагивает S-12–S-18, S-27.

### D. Authentication / trusted proxy / permissions

До remote exposure нужен единый security boundary, не основанный на UI visibility или direct-local assumptions.

Затрагивает все state-changing remote-capable routes, launcher/deploy, import, будущий device control и MCP.

### E. Realtime lifecycle

Runtime logs, notification SSE, launcher SSE и потенциальный device stream требуют явных connection/auth/backpressure/reconnect contracts.

Stage 1 только инвентаризирует эти surfaces; точный protocol design отложен.

### F. File transfer/export semantics

Legacy folder import, config import/export и Desktop CSV нельзя переносить как server filesystem side effects без отдельного пользовательского contract.

### G. Event domain boundary

Event profile, plan и EventShop safety требуют domain/service extraction, потому что current UI содержит automation policy.

### H. Developer/internal product policy

Simulator, process state override, WebUI restart и legacy remote access не должны автоматически попадать в публичный React только ради parity.

## 5. Что разумно мигрировать раньше

После соответствующей backend formalization:

1. read-only statistics views S-12–S-18;
2. Dashboard S-08 через dedicated read model;
3. Home/shell read-only portions S-03/S-04 после instance status contract;
4. OBS S-27 либо оставить отдельным compatibility client, либо переиспользовать тот же authenticated statistics contract.

Этот порядок является **readiness inference**, а не roadmap со сроками.

## 6. Что разумно отложить

До product/security решения:

- S-10 tool/daemon developer-like tasks;
- S-16 server-side Desktop export semantics;
- S-23 Operation Siren simulator;
- S-24 current RemoteAccess UI/provider model;
- S-26 developer restart/state override/test utilities;
- custom live screenshot/control routes, пока не определён realtime/security contract.

## 7. Unverified / deferred items

### Notification producer/consumer ownership

`/api/notify` и `/api/notify_stream` wiring подтверждено. Точный полный список runtime callers/consumers в Stage 1 не установлен repository-wide до уровня бизнес-владельца.

Статус: `UNVERIFIED` только для **consumer ownership**, не для существования transport.

Future check: на отдельном notification/realtime design stage проследить все вызовы producer API/queue и launcher/desktop consumer implementations.

### Framework-generated PyWebIO routes

Два application entry point доказаны, но Stage 1 не фиксирует внутренний route catalog `webio_routes()` как project contract.

Статус: deferred dependency detail; не влияет на business migration boundary.

### Внешняя сетeвая достижимость

Handler-level guards доказаны кодом. Router/firewall/Caddy/OS exposure на конкретной машине не проверялся и не нужен для Stage 1.

Статус: не подтверждено runtime-аудитом; не ломает вывод о необходимости общего security boundary.

### Product parity developer/internal surfaces

Нужно решение владельца, должны ли S-10/S-23/S-24/S-26 существовать в новом пользовательском UI.

Статус: `DEFER_OR_REMOVE_CANDIDATE`, не техническая неопределённость.

## 8. Coverage result

```text
AlasGUI production mixins discovered: 22
Mapped mixins: 22
Unmapped mixins: 0

Production app_*.py files inspected: 28
Classified: 28
Unclassified: 0

Task destinations discovered in ALAS_MENU: 90
Accounted through generic/specialized workflows: 90

User/system migration surfaces: 27

api.py HTTP Route registrations discovered: 14
Production attached from api_routes: 14
Additional explicit /robots.txt: 1
Explicit project HTTP registrations documented: 15
Unresolved explicit HTTP registrations: 0

SSE handlers: 2 discovered / 2 documented
Custom live WebSocket routes: 2 declared / 2 documented / 0 production-attached
PyWebIO app entry points: 2 / 2
Static mounts: 3 / 3
MCP mounts: 1 / 1
```

## 9. Передача следующим этапам

Stage 1 оставляет следующие задачи намеренно нерешёнными:

- Stage 2: детальный MCP boundary/tool/security/transport audit по отдельному scope;
- будущие backend stages: service extraction, security/trusted proxy, read/command contracts;
- отдельный API design stage: versioned `/api/v1`, schema ownership и generated frontend client;
- отдельный frontend foundation stage: React/Vite после стабилизации нужных backend boundaries.

Главный результат Stage 1: существенные WebUI capabilities теперь имеют известную dependency/migration classification; будущий React не должен копировать PyWebIO/runtime logic только потому, что её граница была скрыта.
