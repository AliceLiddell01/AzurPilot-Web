# Текущие backend interfaces WebUI

Источник аудита: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## 1. Рабочая сборка приложения

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

Доказательства:

- [`module/webui/app.py::app`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app.py)
- [`module/webui/fastapi.py::asgi_app`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/fastapi.py)
- [`module/webui/api.py::api_routes`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/api.py)

`fastapi.py` добавляет только `GZipMiddleware` и `HeaderMiddleware`. Общего middleware аутентификации в этой сборке **не найдено**. Password gate текущего WebUI находится внутри `app.py::_run_gui()` и применяется к PyWebIO session flow, а не автоматически ко всем дополнительным ASGI handlers.

Это утверждение относится к проверенному подключению кода. Внешний firewall/reverse proxy/OS ACL статическим аудитом этого репозитория не устанавливаются.

## 2. Подключённые HTTP registrations проекта

`api_routes` содержит 14 `Route` registrations; все 14 подключаются рабочим `asgi_app`. Дополнительно `fastapi.py` регистрирует `/robots.txt`, итого **15 явных HTTP registrations проекта**.

Если `methods=` в `Route` не указан, таблица не приписывает handler'у несуществующее явное ограничение: указывается намерение handler/docstring/current consumer и факт отсутствия project-level method list.

| Путь | Метод/транспорт | Обработчик | Потребитель | Чтение/изменение | Аутентификация в обработчике | Локальность | Вход / выход | Зависимость среды выполнения | Статус / миграция |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `/robots.txt` | `GET`, `HEAD` | `fastapi.py::robots_txt` | crawler | чтение | нет | нет | без input → text | нет runtime | внутренний; оставить инфраструктурным |
| `/api/cl1_stats` | HTTP; `methods` не задан, intended GET | `api.py::api_cl1_stats` | OBS overlay / неизвестно | чтение | нет | нет | `instance` query → JSON summary | `module.statistics.opsi_month` | current-partial; `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN` |
| `/api/ap_timeline` | HTTP; `methods` не задан, intended GET | `api.py::api_ap_timeline` | OBS overlay / неизвестно | чтение | нет | нет | `instance` query → JSON timeline | `module.statistics.opsi_month` | current-partial; формализовать модель чтения |
| `/api/notify` | `POST` | `api.py::api_notify` | producer уведомлений, точный caller не установлен | запись в in-memory queue | нет | нет | JSON → `{success}` | process-local `asyncio.Queue` | current-partial; `REALTIME_CONTRACT`, `SECURITY_REDESIGN` |
| `/api/notify_stream` | HTTP; `methods` не задан, intended GET/SSE | `api.py::api_notify_stream` | launcher/notification client, exact caller не установлен | поток чтения | нет | нет | no body → SSE + keepalive | notification queue | current-partial; `REALTIME_CONTRACT`, `SECURITY_REDESIGN` |
| `/api/launcher/status` | HTTP; `methods` не задан, intended GET | `api.py::api_launcher_status` | Developer Settings | чтение | нет | вычисляет `request_local`, но не отклоняет non-local | no body → JSON status | `launcher_control` | legacy/current-partial; `LEGACY_COMPATIBILITY` |
| `/api/launcher/startup` | `POST` | `api.py::api_launcher_startup` | Developer Settings | изменение | нет общего auth | **только локально** через `is_local_request()` | `{enabled: bool}` → JSON | launcher IPC queue | legacy; `SECURITY_REDESIGN`, `PRIVILEGED_POLICY` |
| `/api/launcher/stream` | HTTP; intended GET/SSE | `api.py::api_launcher_stream` | внешний launcher | смешанный command stream | нет общего auth | **только локально** | no body → SSE commands | launcher IPC | legacy; `REALTIME_CONTRACT`, `LEGACY_COMPATIBILITY` |
| `/api/launcher/report` | `POST` | `api.py::api_launcher_report` | внешний launcher | изменение | нет общего auth | **только локально** | launcher result JSON → JSON | launcher IPC pending futures | legacy; только совместимость |
| `/api/deploy/settings` | HTTP; первая registration intended GET | `api.py::api_deploy_settings` | Developer Settings | чтение | нет общего auth | **только локально** | no body → localized schema/values | `State.deploy_config` | current-partial admin interface; formalize/redesign |
| `/api/deploy/settings` | `POST` | `api.py::api_deploy_settings_save` | Developer Settings | изменение | нет общего auth | **только локально** | values JSON → updated keys | deploy config persistence | privileged admin; `PRIVILEGED_POLICY`, `SECURITY_REDESIGN` |
| `/api/deploy/startup-run` | HTTP; intended GET | `api.py::api_deploy_startup_run` | TaskConfig / startup-run | чтение | нет общего auth | **только локально** | `instance` query → run state | deploy config | legacy/current-partial; formalize if retained |
| `/api/deploy/startup-run` | `POST` | `api.py::api_deploy_startup_run_save` | TaskConfig / startup-run | изменение | нет общего auth | **только локально** | `{instance, enabled}` → run state | deploy config persistence | privileged admin; redesign |
| `/api/import_legacy_upload` | `POST` | `api.py::api_import_legacy_upload` | OOBE / Manage | **запись в файловую систему** | нет | **не проверяется обработчиком** | multipart files → import counters | project `config/`, `log/cl1/`, AzurStats CSV | legacy; **обязательный `SECURITY_REDESIGN` + `PRIVILEGED_POLICY`** |
| `/obs` | HTTP; intended GET | `api.py::serve_obs_overlay` | OBS/browser source | чтение | нет | нет | no body → HTML | local `obs_overlay.html` | `LEGACY_COMPATIBILITY`; решение о доступности отложено |

### Замечание о семантике методов

Для routes без `methods=[...]` код проекта не задаёт отдельный список разрешённых методов. Документация поэтому не превращает intended GET из docstring/consumer в доказанный route-level method restriction. Перед формализацией API это поведение должно быть зафиксировано тестами/контрактом конкретной версии Starlette, а не угадано здесь.

## 3. SSE / streaming surfaces

Подтверждено **2** project SSE handlers:

1. `/api/notify_stream` → `api_notify_stream` — process-local notification queue, keepalive каждые 30 секунд.
2. `/api/launcher/stream` → `api_launcher_stream` — local-only launcher command channel, lifecycle `mark_connected/mark_disconnected`.

Обе поверхности являются текущими adapter contracts, не финальными realtime API. Launcher stream дополнительно зависит от direct-local semantics.

## 4. Custom live WebSocket routes: объявлены, но отключены в рабочей сборке

`module/webui/api.py::api_routes` объявляет:

| Путь | Обработчик | Назначение | Проверка локальности | Рабочее подключение |
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

Итог учёта:

```text
custom WebSocket routes declared: 2
custom WebSocket routes attached by production api_routes extension: 0
```

Это **не** утверждение, что файлы/handlers являются мёртвым кодом. Статически доказано только отсутствие подключения через проверенную цепочку `app.py -> fastapi.py::asgi_app`.

Если эти routes когда-либо будут возвращены, current guard нельзя переносить как remote authorization: `_is_local_live_websocket()` смотрит на direct WebSocket peer. За loopback reverse proxy внешнее соединение могло бы выглядеть локальным для этого guard. Требуются `SECURITY_REDESIGN`, `REALTIME_CONTRACT`, `PRIVILEGED_POLICY`.

## 5. Точки входа PyWebIO

Рабочее приложение передаёт в `webio_routes()` две точки входа приложения:

| Точка входа | Символ | Начальная страница | Текущая аутентификация | Примечание |
| --- | --- | --- | --- | --- |
| основная PyWebIO | `app.py::index` | `home` | `_run_gui()` password gate | основной session WebUI |
| управление PyWebIO | `app.py::manage` | `manage` | `_run_gui()` password gate | compatibility entry к instance management |

Framework-generated protocol routes намеренно не перечисляются как project-owned HTTP API. Их точная форма зависит от pinned PyWebIO dependency и не нужна для определения business boundary миграции.

## 6. Статические mount-точки

Рабочее приложение подтверждает **3** static mounts:

| Mount | Источник | Назначение | Миграционное замечание |
| --- | --- | --- | --- |
| `/static/assets` | project `assets/` | GUI/images/CSS/assets | будущий frontend artifact не должен зависеть от Python path semantics |
| `/static/doc` | project `doc/` | документационные/static resources | compatibility/defer |
| `/pywebio_static` | PyWebIO `STATIC_PATH` | runtime assets PyWebIO | `LEGACY_COMPATIBILITY`; исчезает вместе с PyWebIO client |

`fastapi.py` также умеет условно монтировать `/static` через `static_dir`, но `app.py::app` на проверенном snapshot этот параметр не передаёт. Поэтому `/static` не учитывается как фактический рабочий mount Этапа 1.

## 7. MCP mount

Рабочий WebUI делает:

```text
application.mount('/mcp', mcp_app)
```

Текущий MCP adapter находится в backend repo `mcp_server_sse.py` и использует legacy `SseServerTransport('/mcp/messages')`. Внутри mounted adapter path matching обслуживает SSE `/sse` и POST message path `/messages`/`/messages/`.

На архитектурном уровне подтверждено:

- MCP принадлежит backend repo;
- adapter напрямую использует `ProcessManager`, `AzurLaneConfig`, filesystem logs, `Device`, emulator/ADB/subprocess operations;
- Starlette wrapper MCP настроен с wildcard CORS;
- standalone `__main__` режим способен bind'иться на `0.0.0.0:22268`;
- общего shared Application/Service Layer между WebUI и MCP сейчас нет.

Этап 1 **не** выполняет tool-by-tool risk audit. Для миграционной карты достаточно тегов `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`; transport/tool redesign передаётся Этапу 2.

## 8. Текущая семантика локальности

`module/webui/launcher.py::is_local_request()` возвращает true только когда одновременно:

```text
direct request.client.host ∈ {127.0.0.1, ::1, localhost}
AND
normalized Host header ∈ {127.0.0.1, ::1, localhost}
```

Это нельзя считать готовой моделью для будущей same-origin Caddy topology:

- reverse proxy peer для backend будет loopback;
- browser `Host` будет пользовательским доменом;
- current launcher/deploy guard поэтому либо отклонит легитимный proxied request, либо при попытке упрощения до «peer loopback = local» создаст путаницу доверия.

**Требование к бэкенду:** explicit trusted-proxy boundary + authenticated authorization. Forwarded metadata нельзя принимать от произвольного direct client.

## 9. Инвентаризация аутентификации и доступности

Подтверждено кодом:

- PyWebIO password gate: `_run_gui()`;
- общего auth middleware для project HTTP/MCP routes в `fastapi.py` не найдено;
- launcher/deploy mutating routes используют locality guards;
- CL1/AP/notify/notify-stream/import/OBS handlers не содержат собственного auth/locality guard;
- `/api/launcher/status` сообщает `request_local`, но не отклоняет non-local request;
- custom live WS имеют peer-local guard, но отключены в рабочей сборке;
- MCP wrapper не наследует `_run_gui()` password gate через собственный handler path.

Это инвентаризация, не penetration test. Реальная сетeвая достижимость снаружи зависит также от bind/firewall/router/reverse proxy, которые не являются частью доказательств на уровне handler.

## 10. Учёт покрытия

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
