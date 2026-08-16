# Текущие backend interfaces WebUI

Audit source: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## 1. Production assembly

Подтверждённая сборка приложения:

```text
module/webui/app.py::app
  -> asgi_app(applications=[index, manage], static_mounts=...)
       -> PyWebIO webio_routes(...)
       -> /robots.txt
       -> project api_routes, кроме DISABLED_API_ROUTE_PATHS
       -> /static/assets
       -> /static/doc
       -> /pywebio_static
  -> application.mount('/mcp', mcp_app)
```

Evidence:

- [`module/webui/app.py::app`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app.py)
- [`module/webui/fastapi.py::asgi_app`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/fastapi.py)
- [`module/webui/api.py::api_routes`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/api.py)

`fastapi.py` добавляет только `GZipMiddleware` и `HeaderMiddleware`. Общего authentication middleware в этой сборке **не найдено**. Password gate текущего WebUI находится внутри `app.py::_run_gui()` и применяется к PyWebIO session flow, а не автоматически ко всем дополнительным ASGI handlers.

Это утверждение относится к audited code wiring. Внешний firewall/reverse proxy/OS ACL статическим аудитом этого репозитория не устанавливаются.

## 2. Подключённые project HTTP registrations

`api_routes` содержит 14 `Route` registrations; все 14 подключаются production `asgi_app`. Дополнительно `fastapi.py` регистрирует `/robots.txt`, итого **15 explicit project HTTP registrations**.

Если `methods=` в `Route` не указан, таблица не приписывает handler'у несуществующее явное ограничение: указывается намерение handler/docstring/current consumer и факт отсутствия project-level method list.

| Path | Method/transport | Handler | Consumer | R/W | Auth в handler | Locality | Input / output | Runtime dependency | Stability / migration note |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `/robots.txt` | `GET`, `HEAD` | `fastapi.py::robots_txt` | crawler | read | нет | нет | без input → text | нет runtime | internal; оставить инфраструктурным |
| `/api/cl1_stats` | HTTP; `methods` не задан, intended GET | `api.py::api_cl1_stats` | OBS overlay / unknown | read | нет | нет | `instance` query → JSON summary | `module.statistics.opsi_month` | current-partial; `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN` |
| `/api/ap_timeline` | HTTP; `methods` не задан, intended GET | `api.py::api_ap_timeline` | OBS overlay / unknown | read | нет | нет | `instance` query → JSON timeline | `module.statistics.opsi_month` | current-partial; formalize read model |
| `/api/notify` | `POST` | `api.py::api_notify` | notification producer, точный caller не установлен | write to in-memory queue | нет | нет | JSON → `{success}` | process-local `asyncio.Queue` | current-partial; `REALTIME_CONTRACT`, `SECURITY_REDESIGN` |
| `/api/notify_stream` | HTTP; `methods` не задан, intended GET/SSE | `api.py::api_notify_stream` | launcher/notification client, exact caller не установлен | read stream | нет | нет | no body → SSE + keepalive | notification queue | current-partial; `REALTIME_CONTRACT`, `SECURITY_REDESIGN` |
| `/api/launcher/status` | HTTP; `methods` не задан, intended GET | `api.py::api_launcher_status` | Developer Settings | read | нет | вычисляет `request_local`, но не отклоняет non-local | no body → JSON status | `launcher_control` | legacy/current-partial; `LEGACY_COMPATIBILITY` |
| `/api/launcher/startup` | `POST` | `api.py::api_launcher_startup` | Developer Settings | write | нет общего auth | **local-only** через `is_local_request()` | `{enabled: bool}` → JSON | launcher IPC queue | legacy; `SECURITY_REDESIGN`, `PRIVILEGED_POLICY` |
| `/api/launcher/stream` | HTTP; intended GET/SSE | `api.py::api_launcher_stream` | external launcher | mixed command stream | нет общего auth | **local-only** | no body → SSE commands | launcher IPC | legacy; `REALTIME_CONTRACT`, `LEGACY_COMPATIBILITY` |
| `/api/launcher/report` | `POST` | `api.py::api_launcher_report` | external launcher | write | нет общего auth | **local-only** | launcher result JSON → JSON | launcher IPC pending futures | legacy; compatibility only |
| `/api/deploy/settings` | HTTP; first registration intended GET | `api.py::api_deploy_settings` | Developer Settings | read | нет общего auth | **local-only** | no body → localized schema/values | `State.deploy_config` | current-partial admin interface; formalize/redesign |
| `/api/deploy/settings` | `POST` | `api.py::api_deploy_settings_save` | Developer Settings | write | нет общего auth | **local-only** | values JSON → updated keys | deploy config persistence | privileged admin; `PRIVILEGED_POLICY`, `SECURITY_REDESIGN` |
| `/api/deploy/startup-run` | HTTP; intended GET | `api.py::api_deploy_startup_run` | TaskConfig / startup-run | read | нет общего auth | **local-only** | `instance` query → run state | deploy config | legacy/current-partial; formalize if retained |
| `/api/deploy/startup-run` | `POST` | `api.py::api_deploy_startup_run_save` | TaskConfig / startup-run | write | нет общего auth | **local-only** | `{instance, enabled}` → run state | deploy config persistence | privileged admin; redesign |
| `/api/import_legacy_upload` | `POST` | `api.py::api_import_legacy_upload` | OOBE / Manage | **filesystem write** | нет | **не проверяется handler'ом** | multipart files → import counters | project `config/`, `log/cl1/`, AzurStats CSV | legacy; **обязательный `SECURITY_REDESIGN` + `PRIVILEGED_POLICY`** |
| `/obs` | HTTP; intended GET | `api.py::serve_obs_overlay` | OBS/browser source | read | нет | нет | no body → HTML | local `obs_overlay.html` | `LEGACY_COMPATIBILITY`; exposure decision deferred |

### Method semantics note

Для routes без `methods=[...]` код проекта не задаёт отдельный список разрешённых методов. Документация поэтому не превращает intended GET из docstring/consumer в доказанный route-level method restriction. Перед формализацией API это поведение должно быть зафиксировано тестами/контрактом конкретной версии Starlette, а не угадано здесь.

## 3. SSE / streaming surfaces

Подтверждено **2** project SSE handlers:

1. `/api/notify_stream` → `api_notify_stream` — process-local notification queue, keepalive каждые 30 секунд.
2. `/api/launcher/stream` → `api_launcher_stream` — local-only launcher command channel, lifecycle `mark_connected/mark_disconnected`.

Обе поверхности являются current adapter contracts, не финальными realtime API. Launcher stream дополнительно зависит от direct-local semantics.

## 4. Custom live WebSocket routes: объявлены, но production-disabled

`module/webui/api.py::api_routes` объявляет:

| Path | Handler | Intended behavior | Locality guard | Production wiring |
| --- | --- | --- | --- | --- |
| `/ws/live_screenshot` | `_ws_live_screenshot_guarded` → `ws_live_screenshot` | H.264/screenshot live preview, device/scrcpy/ffmpeg | `_reject_nonlocal_live_websocket` | **отфильтрован** |
| `/ws/live_control` | `_ws_live_control_guarded` → `ws_live_control` | tap/drag/key/text/device control | `_reject_nonlocal_live_websocket` | **отфильтрован** |

Причина подтверждена кодом `module/webui/fastapi.py`:

```text
DISABLED_API_ROUTE_PATHS = {
  /ws/live_screenshot,
  /ws/live_control
}
```

`asgi_app()` расширяет routes только теми entries, чьи `path` отсутствуют в этом наборе.

Итог accounting:

```text
custom WebSocket routes declared: 2
custom WebSocket routes attached by production api_routes extension: 0
```

Это **не** утверждение, что файлы/handlers являются dead code. Статически доказано только отсутствие подключения через audited production `app.py -> fastapi.py::asgi_app` wiring.

Если эти routes когда-либо будут возвращены, current guard нельзя переносить как remote authorization: `_is_local_live_websocket()` смотрит на direct WebSocket peer. За loopback reverse proxy внешнее соединение могло бы выглядеть локальным для этого guard. Требуются `SECURITY_REDESIGN`, `REALTIME_CONTRACT`, `PRIVILEGED_POLICY`.

## 5. PyWebIO entry points

Production app передаёт в `webio_routes()` два application entry points:

| Entry | Symbol | Initial page | Current auth | Notes |
| --- | --- | --- | --- | --- |
| PyWebIO main | `app.py::index` | `home` | `_run_gui()` password gate | основной session WebUI |
| PyWebIO manage | `app.py::manage` | `manage` | `_run_gui()` password gate | compatibility entry к instance management |

Framework-generated protocol routes намеренно не перечисляются как project-owned HTTP API. Их точная форма зависит от pinned PyWebIO dependency и не нужна для определения migration business boundary.

## 6. Static mounts

Production app подтверждает **3** static mounts:

| Mount | Source | Purpose | Migration note |
| --- | --- | --- | --- |
| `/static/assets` | project `assets/` | GUI/images/CSS/assets | будущий frontend artifact не должен зависеть от Python path semantics |
| `/static/doc` | project `doc/` | документационные/static resources | compatibility/defer |
| `/pywebio_static` | PyWebIO `STATIC_PATH` | runtime assets PyWebIO | `LEGACY_COMPATIBILITY`; исчезает вместе с PyWebIO client |

`fastapi.py` также умеет условно монтировать `/static` через `static_dir`, но `app.py::app` на audited snapshot этот параметр не передаёт. Поэтому `/static` не учитывается как фактический production mount Stage 1.

## 7. MCP mount

Production WebUI делает:

```text
application.mount('/mcp', mcp_app)
```

Current MCP adapter находится в backend repo `mcp_server_sse.py` и использует legacy `SseServerTransport('/mcp/messages')`. Внутри mounted adapter path matching обслуживает SSE `/sse` и POST message path `/messages`/`/messages/`.

На архитектурном уровне подтверждено:

- MCP принадлежит backend repo;
- adapter напрямую использует `ProcessManager`, `AzurLaneConfig`, filesystem logs, `Device`, emulator/ADB/subprocess operations;
- Starlette wrapper MCP настроен с wildcard CORS;
- standalone `__main__` режим способен bind'иться на `0.0.0.0:22268`;
- общего shared Application/Service Layer между WebUI и MCP сейчас нет.

Stage 1 **не** выполняет tool-by-tool risk audit. Для миграционной карты достаточно тегов `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`; transport/tool redesign передаётся Stage 2.

## 8. Current locality semantics

`module/webui/launcher.py::is_local_request()` возвращает true только когда одновременно:

```text
direct request.client.host ∈ {127.0.0.1, ::1, localhost}
AND
normalized Host header ∈ {127.0.0.1, ::1, localhost}
```

Это безопасно не экстраполировать на будущий same-origin Caddy topology:

- reverse proxy peer для backend будет loopback;
- browser `Host` будет пользовательским доменом;
- current launcher/deploy guard поэтому либо отклонит легитимный proxied request, либо при попытке упрощения до «peer loopback = local» создаст trust confusion.

**Future backend prerequisite:** explicit trusted-proxy boundary + authenticated authorization. Forwarded metadata нельзя принимать от произвольного direct client.

## 9. Authentication/exposure inventory

Подтверждено кодом:

- PyWebIO password gate: `_run_gui()`;
- общего auth middleware для project HTTP/MCP routes в `fastapi.py` не найдено;
- launcher/deploy mutating routes используют locality guards;
- CL1/AP/notify/notify-stream/import/OBS handlers не содержат собственного auth/locality guard;
- `/api/launcher/status` сообщает `request_local`, но не отклоняет non-local request;
- custom live WS имеют peer-local guard, но production-disabled;
- MCP wrapper не наследует `_run_gui()` password gate через собственный handler path.

Это inventory, не penetration test. Реальная сетeвая достижимость снаружи зависит также от bind/firewall/router/reverse proxy, которые не являются частью handler-level evidence.

## 10. Coverage accounting

```text
api.py Route registrations discovered: 14
api.py Route registrations attached by production asgi_app: 14
fastapi.py explicit /robots.txt registration: 1
explicit project HTTP registrations documented: 15
unresolved explicit project HTTP registrations: 0

SSE handlers discovered/documented: 2 / 2
custom WebSocket routes declared/documented: 2 / 2
custom WebSocket routes attached in production: 0

PyWebIO application entry points discovered/documented: 2 / 2
production static mounts discovered/documented: 3 / 3
MCP mounts discovered/documented: 1 / 1
```
