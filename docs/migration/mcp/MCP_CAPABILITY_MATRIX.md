# Матрица capabilities текущего MCP

Источник: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## 1. Проверяемое покрытие

```text
tools: 17 обнаружено / 17 классифицировано
resources: 0 обнаружено / 0 классифицировано
resource templates: 0 обнаружено / 0 классифицировано
prompts: 0 обнаружено / 0 классифицировано
completions: 0 обнаружено / 0 классифицировано
subscriptions: 0 обнаружено / 0 классифицировано
custom MCP transport path families: 2 обнаружено / 2 классифицировано
experimental/extensions registrations: 0 обнаружено / 0 классифицировано
несопоставленных registrations: 0
```

`mcp_server_sse.py::list_tools()` возвращает 17 `Tool(...)`; `TOOL_HANDLERS` содержит те же 17 имён. Зарегистрированы `list_tools()` и `call_tool()`, но не зарегистрированы resource/prompt/completion/experimental handlers.

## 2. Общая current transport/security/error граница

Эти свойства относятся ко всем 17 tools, если запись конкретного tool не уточняет их:

- **Аутентификация/авторизация:** отсутствуют в MCP wrapper и handlers; actor/permission context не передаётся.
- **Локальность:** собственного localhost guard нет. В `mcp==1.23.0` `SseServerTransport` получает `security_settings=None`, что отключает `Host`/`Origin` DNS-rebinding checks для backwards compatibility; POST `Content-Type` всё же проверяется.
- **CORS:** wrapper разрешает `allow_origins/methods/headers=["*"]`.
- **Валидация JSON Schema:** SDK `Server.call_tool(validate_input=True)` валидирует arguments против advertised `inputSchema` до AzurPilot handler; schema failure возвращается как `isError=true`.
- **Доменные/runtime ошибки:** AzurPilot `call_tool()` ловит исключения и возвращает обычный `TextContent`; SDK затем формирует обычный результат `isError=false`. Некоторые handlers также возвращают `Error: ...` как обычный текст.
- **Схема выхода:** ни один текущий `Tool` не объявляет `outputSchema`; почти все результаты — неструктурированный `TextContent`, кроме screenshot `ImageContent`.
- **Таймаут:** MCP-owned per-tool timeout отсутствует.
- **Дедупликация:** command ID/idempotency key отсутствуют.
- **Ограничение частоты:** MCP-owned rate limit отсутствует.
- **Аудит:** нет actor-aware structured audit записи успешного tool call; runtime-компоненты могут писать собственные технические логи.
- **Отмена:** SDK выполняет requests асинхронно, но handlers AzurPilot преимущественно делают синхронную работу внутри `async def`; blocking `time.sleep`, filesystem, subprocess/device/process calls не получают отдельной cooperative cancellation policy от adapter.

## 3. Классы риска

```text
R0 — metadata / чистое локальное вычисление
R1 — operational read без целевого изменения, но с runtime/user data
R2 — ограниченное изменение состояния внутри AzurPilot
R3 — privileged runtime/device/config/process control
R4 — системная/широкая machine-level операция
```

Распределение:

```text
R0: 2
R1: 7
R2: 0
R3: 7
R4: 1
Всего: 17
```

## 4. Tool-by-tool матрица

Для current tools нет индивидуальных HTTP URI: публичное имя вызывается через общий MCP method `tools/call` внутри transport family. Effective transport paths описаны в `CURRENT_MCP_ARCHITECTURE.md`.

### MCP-T-001 — `list_instances`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `list_instances`; общий `tools/call` |
| Регистрация | `mcp_server_sse.py::list_tools` → `TOOL_HANDLERS["list_instances"]` → `_tool_list_instances` |
| Вход | object, `properties={}`; `additionalProperties:false` не задан |
| Выход | `TextContent` с JSON-массивом имён экземпляров |
| Чтение/запись | чтение |
| Побочные эффекты | целевое изменение не выполняется |
| Зависимости | `module.config.utils.alas_instance()` |
| Область привилегий | обнаружение instance/config |
| Валидация | только SDK JSON Schema object; лишние свойства schema не запрещает |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | имена пользовательских экземпляров могут быть идентифицирующими |
| Модель ошибок | исключение → AzurPilot error text, MCP `isError=false` |
| Таймаут/отмена | timeout нет; синхронный discovery |
| Идемпотентность | да для product read semantics |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `instances:read` |
| Будущий владелец сервиса | `InstanceService` |
| Решение | **RETAIN** |
| Доказательство | [`mcp_server_sse.py::_tool_list_instances`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-002 — `get_status`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_status`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_status` |
| Вход | пустой object; tool возвращает все экземпляры |
| Выход | JSON-массив `{instance, running, state}` в `TextContent` |
| Чтение/запись | operational read; `ProcessManager.alive/state` может выполнять housekeeping устаревшей process registry |
| Побочные эффекты | целевого пользовательского изменения нет; возможна внутренняя очистка stale worker bookkeeping |
| Зависимости | `alas_instance`, `ProcessManager.get_manager`, `alive`, `state` |
| Область привилегий | process/runtime status |
| Валидация | schema object; instance selector отсутствует |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | имена экземпляров + runtime state |
| Модель ошибок | exception text как обычный результат |
| Таймаут/отмена | MCP timeout нет; runtime status path синхронный |
| Идемпотентность | да по product read semantics |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `status:read` |
| Будущий владелец сервиса | `InstanceService` |
| Решение | **RETAIN** |
| Доказательство | [`mcp_server_sse.py::_tool_get_status`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`ProcessManager.alive/state`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/process_manager.py) |
| Уверенность | подтверждено |

### MCP-T-003 — `list_tasks`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `list_tasks`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_list_tasks` |
| Вход | пустой object |
| Выход | JSON-массив имён задач |
| Чтение/запись | чтение метаданных |
| Побочные эффекты | нет после инициализации module/helper |
| Зависимости | `McpConfigHelper.get_tasks()`; cached `args.json` metadata |
| Область привилегий | task metadata |
| Валидация | object schema; лишние свойства не запрещены |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | значимых runtime/user secrets не ожидается; это definition metadata |
| Модель ошибок | exception text как обычный result |
| Таймаут/отмена | timeout нет |
| Идемпотентность | да |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R0** |
| Будущее разрешение | `tasks:read` |
| Будущий владелец сервиса | `TaskService` |
| Решение | **RETAIN** |
| Доказательство | [`mcp_server_sse.py::_tool_list_tasks`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`McpConfigHelper.get_tasks`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |
| Уверенность | подтверждено |

### MCP-T-004 — `get_task_help`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_task_help`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_task_help` |
| Вход | required `task_name:string` |
| Выход | JSON metadata: display name/help/groups/arguments/default/options |
| Чтение/запись | чтение метаданных |
| Побочные эффекты | нет |
| Зависимости | `McpConfigHelper.get_task_details`, args/i18n files cached при создании helper |
| Область привилегий | task/config-schema metadata |
| Валидация | string type; существование task schema не проверяется; unknown task → `{}` |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | definition defaults/options, не текущий user config |
| Модель ошибок | неизвестная задача выглядит как успешный `{}`; exception → обычный error text |
| Таймаут/отмена | timeout нет |
| Идемпотентность | да |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R0** |
| Будущее разрешение | `tasks:read` |
| Будущий владелец сервиса | `TaskService` |
| Решение | **RETAIN** |
| Доказательство | [`mcp_server_sse.py::_tool_get_task_help`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`McpConfigHelper.get_task_details`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |
| Уверенность | подтверждено |

### MCP-T-005 — `get_resources`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_resources`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_resources` |
| Вход | required `instance:string` |
| Выход | JSON Dashboard resource projection: label/value и optional limit/total/last_update |
| Чтение/запись | operational read |
| Побочные эффекты | user-intended mutation нет; создание `AzurLaneConfig` проходит обычный config load/init path |
| Зависимости | `AzurLaneConfig`, `McpConfigHelper.get_dashboard_resources` |
| Область привилегий | per-instance operational resources |
| Валидация | SDK проверяет string, но существование/permission экземпляра отдельно не проверяются MCP layer |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | внутриигровые ресурсы/времена обновления |
| Модель ошибок | config/runtime exception → обычный error text |
| Таймаут/отмена | MCP timeout нет; sync config read |
| Идемпотентность | да по read semantics |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `resources:read` |
| Будущий владелец сервиса | `StatisticsService` |
| Решение | **RETAIN** |
| Доказательство | [`mcp_server_sse.py::_tool_get_resources`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`McpConfigHelper.get_dashboard_resources`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |
| Уверенность | подтверждено |

### MCP-T-006 — `get_config`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_config`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_config` |
| Вход | required `instance:string`; optional `task:string` |
| Выход | весь `config.data` либо raw section указанной task |
| Чтение/запись | широкое чтение operational/config данных |
| Побочные эффекты | целевого изменения нет; constructor config использует обычный load/init/migration path |
| Зависимости | `AzurLaneConfig.data` |
| Область привилегий | configuration |
| Валидация | типы проверяются SDK; field-level allowlist/redaction отсутствуют; unknown task → `{}` |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | широкий raw config может содержать device/runtime/integration identifiers или иные чувствительные значения; конкретное наличие секрета без field audit не предполагается |
| Модель ошибок | exception → обычный error text |
| Таймаут/отмена | MCP timeout нет; sync config load/JSON serialization |
| Идемпотентность | да по read semantics |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `config:read` + field policy |
| Будущий владелец сервиса | `ConfigService` |
| Решение | **REDESIGN** — raw config dump не является target contract |
| Доказательство | [`mcp_server_sse.py::_tool_get_config`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-007 — `update_config`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `update_config`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_update_config` |
| Вход | required `instance`, `task`, `group`, `arg`, `value`; `value` допускает string/number/bool/object/array/null |
| Выход | success text с собранным `task.group.arg` и значением |
| Чтение/запись | изменение |
| Побочные эффекты | `AzurLaneConfig.cross_set(path,value)` → `update()`/save; изменяет persistent config |
| Зависимости | `AzurLaneConfig.cross_set`, `save`, config filesystem |
| Область привилегий | privileged config control |
| Валидация | SDK schema проверяет типы; handler не проверяет существование/allowlist path и не применяет WebUI `parse_pin_value`/validation/`ConfigUpdater.save_callback` policy |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | способен менять широкие runtime/automation параметры; success text повторяет submitted value |
| Модель ошибок | schema errors → `isError=true`; domain/runtime exception → обычный error text `isError=false` |
| Таймаут/отмена | MCP timeout/command ID нет; sync config I/O |
| Идемпотентность | условная: одинаковое присваивание часто имеет тот же state, но callbacks/downstream semantics не гарантируют отсутствия повторного эффекта |
| Разрушительность | условная: значение может отключить/перенастроить automation |
| Доступ во внешний мир | условный через изменённую automation configuration |
| Риск | **R3** |
| Будущее разрешение | `config:write` с field/domain policy |
| Будущий владелец сервиса | `ConfigService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_update_config`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`AzurLaneConfig.cross_set`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/config.py), WebUI validation evidence из Этапа 1 |
| Уверенность | подтверждено |

### MCP-T-008 — `get_recent_logs`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_recent_logs`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_recent_logs` |
| Вход | `instance:string` required; `lines:integer` optional default 50, без min/max |
| Выход | raw joined log lines либо обычный error/not-found text |
| Чтение/запись | filesystem read |
| Побочные эффекты | целевого изменения нет; весь дневной файл загружается `readlines()` до slicing |
| Зависимости | `os.path`, `./log/<date>_<instance>.txt`, fallback `./log/<date>_alas.txt` |
| Область привилегий | logs/filesystem read |
| Валидация | integer type есть; размер/знак `lines` не ограничен; instance не разрешается через `alas_instance()` до filename generation |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | raw logs могут содержать runtime identifiers, paths, errors и operational context |
| Модель ошибок | file exception и not-found возвращаются обычным text result, не `isError=true` |
| Таймаут/отмена | timeout нет; sync full-file `readlines()`; large-output/DoS риск |
| Идемпотентность | да как read operation, но содержимое лога меняется во времени |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `logs:read` |
| Будущий владелец сервиса | `LogService` |
| Решение | **REDESIGN** — bounded/sanitized log read model |
| Доказательство | [`mcp_server_sse.py::_tool_get_recent_logs`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-009 — `start_instance`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `start_instance`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_start_instance` |
| Вход | required `instance:string` |
| Выход | error text если уже alive; иначе success text с config mod |
| Чтение/запись | privileged mutation |
| Побочные эффекты | создаёт/запускает AzurPilot worker process через `ProcessManager.start` |
| Зависимости | `ProcessManager`, `get_config_mod`, worker registry/process lifecycle |
| Область привилегий | process/runtime control |
| Валидация | SDK string; отдельной instance allowlist/authz в MCP нет |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | instance/config mod в output |
| Модель ошибок | already-running — обычный `Error:` text; `ProcessManager.start()` может безопасно отклонить старт, но tool после возврата всё равно формирует `Success:` |
| Таймаут/отмена | MCP timeout/idempotency key нет; process start синхронный; underlying manager имеет lifecycle locks |
| Идемпотентность | условная: повтор при alive обычно не создаёт новый worker, но race/transient rejection меняют retry effect |
| Разрушительность | прямого удаления нет, но операция запускает automation |
| Доступ во внешний мир | да/условно по выполняемым задачам: запущенная automation взаимодействует с игровым окружением |
| Риск | **R3** |
| Будущее разрешение | `instances:control` |
| Будущий владелец сервиса | `InstanceService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_start_instance`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`ProcessManager.start`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/process_manager.py) |
| Уверенность | подтверждено |

### MCP-T-010 — `stop_instance`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `stop_instance`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_stop_instance` |
| Вход | required `instance:string` |
| Выход | error text если уже stopped; иначе `Success: Stopped ...` |
| Чтение/запись | privileged mutation |
| Побочные эффекты | `ProcessManager.stop()` останавливает worker, при необходимости terminate/kill process tree |
| Зависимости | `ProcessManager`, OS process operations |
| Область привилегий | process/runtime control |
| Валидация | SDK string; MCP-level instance permission нет |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | instance name |
| Модель ошибок | `manager.stop()` возвращает bool, но tool игнорирует его и сообщает success после вызова; возможен ложный success при неполной остановке |
| Таймаут/отмена | MCP timeout нет; underlying stop содержит process wait/kill timeouts, не равные request deadline |
| Идемпотентность | условная; повтор обычно не даёт дополнительного эффекта, но race с restart способен изменить цель |
| Разрушительность | да в операционном смысле: принудительно прерывает worker/process tree |
| Доступ во внешний мир | нет прямого внешнего reach |
| Риск | **R3** |
| Будущее разрешение | `instances:control` |
| Будущий владелец сервиса | `InstanceService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_stop_instance`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`ProcessManager.stop`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/process_manager.py) |
| Уверенность | подтверждено |

### MCP-T-011 — `get_screenshot`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_screenshot`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_screenshot` |
| Вход | required `instance:string` |
| Выход | `ImageContent` JPEG Base64; при error — text с exception + traceback |
| Чтение/запись | privileged device read |
| Побочные эффекты | создаёт `Device`, выполняет `device.screenshot()`, временно меняет process-global compatibility context (`ALAS_CONFIG_NAME` только если отсутствует, fake PIL hook) |
| Зависимости | `AzurLaneConfig`, `Device`, PIL, process environment |
| Область привилегий | device/view/privacy |
| Валидация | SDK string; отдельной device/instance authorization policy нет |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | изображение экрана может содержать приватный/игровой контекст; traceback может раскрыть paths/device details |
| Модель ошибок | handler ловит exception и возвращает полный traceback как обычный text `isError=false` |
| Таймаут/отмена | MCP timeout нет; screenshot/device operation синхронна |
| Идемпотентность | read; кадр меняется во времени |
| Разрушительность | нет |
| Доступ во внешний мир | нет: доступ ограничен выбранным локальным device runtime |
| Риск | **R3** |
| Будущее разрешение | `device:view` |
| Будущий владелец сервиса | `DeviceService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_get_screenshot`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-012 — `get_current_running_task`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_current_running_task`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_current_running_task` |
| Вход | required `instance:string` |
| Выход | task name text, `Unknown`, либо обычный `Error:` text |
| Чтение/запись | operational read |
| Побочные эффекты | manager status read + полный synchronous read дневного log для reverse regex scan |
| Зависимости | `ProcessManager`, filesystem log, regex |
| Область привилегий | task/runtime/log read |
| Валидация | SDK string; отдельная task semantics не требуется |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | task name/runtime status; log читается server-side, наружу отдаётся только найденное имя |
| Модель ошибок | not-running → обычный Error text; log parse exceptions подавляются bare `except`, результат `Unknown` |
| Таймаут/отмена | MCP timeout нет; полный log `readlines()` синхронный |
| Идемпотентность | read, но task меняется во времени |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `tasks:read` |
| Будущий владелец сервиса | `TaskService` |
| Решение | **RETAIN** capability; target implementation через runtime read model, не log parsing |
| Доказательство | [`mcp_server_sse.py::_tool_get_current_running_task`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-013 — `get_scheduler_queue`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `get_scheduler_queue`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_get_scheduler_queue` |
| Вход | required `instance:string` |
| Выход | JSON-массив `{task,next_run}` для enabled scheduler tasks |
| Чтение/запись | operational read |
| Побочные эффекты | целевого изменения нет; обычный `AzurLaneConfig` load/init path может выполнять предусмотренные config normalization/migration действия |
| Зависимости | `AzurLaneConfig.data` / Scheduler sections |
| Область привилегий | scheduler read |
| Валидация | SDK string; MCP layer не разрешает отдельный instance permission |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | task names и schedule timing |
| Модель ошибок | runtime/config exception → обычный error text |
| Таймаут/отмена | MCP timeout нет; sync config read/sort |
| Идемпотентность | read, очередь меняется во времени |
| Разрушительность | нет |
| Доступ во внешний мир | нет |
| Риск | **R1** |
| Будущее разрешение | `scheduler:read` |
| Будущий владелец сервиса | `SchedulerService` |
| Решение | **RETAIN** |
| Доказательство | [`mcp_server_sse.py::_tool_get_scheduler_queue`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-014 — `trigger_task`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `trigger_task`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_trigger_task` |
| Вход | required `instance:string`, `task:string` |
| Выход | success text |
| Чтение/запись | privileged scheduler mutation |
| Побочные эффекты | выставляет `<task>.Scheduler.Enable=True` и `NextRun=now`, сохраняя config; активный worker может затем выполнить automation |
| Зависимости | `AzurLaneConfig.cross_set/update/save`, `current_time`, scheduler runtime |
| Область привилегий | scheduler/automation control |
| Валидация | SDK string types; существование/allowlist task перед `cross_set` не проверяется handler |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | task/instance в output |
| Модель ошибок | mutation exception → обычный error text; success не подтверждает последующее выполнение task |
| Таймаут/отмена | MCP timeout/command ID/dedup нет; повтор ставит новое `NextRun=now` |
| Идемпотентность | нет — повтор меняет `NextRun` и может повторно инициировать scheduling intent |
| Разрушительность | условная: downstream task может менять игровое/runtime состояние |
| Доступ во внешний мир | да/условно через запускаемую game automation |
| Риск | **R3** |
| Будущее разрешение | `scheduler:control` |
| Будущий владелец сервиса | `SchedulerService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_trigger_task`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`AzurLaneConfig.cross_set`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/config.py) |
| Уверенность | подтверждено |

### MCP-T-015 — `clear_scheduler_queue`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `clear_scheduler_queue`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_clear_scheduler_queue` |
| Вход | required `instance:string` |
| Выход | success text со списком disabled tasks |
| Чтение/запись | privileged scheduler mutation |
| Побочные эффекты | для каждого enabled Scheduler выставляет `Enable=False`; может отменить все текущие планы экземпляра |
| Зависимости | `AzurLaneConfig`, `cross_set`, config persistence |
| Область привилегий | scheduler control |
| Валидация | SDK instance string; отдельная scope/confirmation policy отсутствует |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | task names в output |
| Модель ошибок | exception → обычный error text; partial mutation до exception отдельным transaction/result не представлена |
| Таймаут/отмена | MCP timeout/transaction/command ID нет; multiple sync config updates возможны |
| Идемпотентность | да относительно уже очищенной очереди, если нет concurrent re-enable |
| Разрушительность | да — массово удаляет enabled scheduling intent |
| Доступ во внешний мир | нет прямого external reach |
| Риск | **R3** |
| Будущее разрешение | `scheduler:control` |
| Будущий владелец сервиса | `SchedulerService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_clear_scheduler_queue`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-016 — `restart_emulator`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `restart_emulator`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_restart_emulator` |
| Вход | required `instance:string` |
| Выход | success text; при exception — exception + full traceback text |
| Чтение/запись | privileged device/runtime mutation |
| Побочные эффекты | `Device.emulator_stop()` → blocking `time.sleep(60)` → `Device.emulator_start()` |
| Зависимости | `AzurLaneConfig`, `Device`, process env/fake-PIL compatibility; `ProcessManager.get_manager(inst)` создаётся, но полученное значение далее не используется |
| Область привилегий | device/emulator lifecycle |
| Валидация | SDK string; MCP policy/precondition проверки активности task/device отсутствуют |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | exception traceback может раскрыть paths/device context |
| Модель ошибок | exception → traceback как обычный `isError=false` text; нет structured phase result stop/wait/start |
| Таймаут/отмена | MCP timeout нет; `time.sleep(60)` блокирует event loop; cancellation/disconnect не прерывает этот sleep cooperatively |
| Идемпотентность | нет — каждый повтор снова останавливает/запускает emulator |
| Разрушительность | да — disruptive restart локального device runtime |
| Доступ во внешний мир | нет прямого network reach; операция ограничена локальным emulator runtime |
| Риск | **R3** |
| Будущее разрешение | `device:control` |
| Будущий владелец сервиса | `DeviceService` |
| Решение | **REDESIGN** |
| Доказательство | [`mcp_server_sse.py::_tool_restart_emulator`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

### MCP-T-017 — `restart_adb`

| Поле | Значение |
| --- | --- |
| Тип MCP | `tool` |
| Публичное имя / URI | `restart_adb`; общий `tools/call` |
| Регистрация | `list_tools` → `_tool_restart_adb` |
| Вход | optional `instance:string`; default `DEFAULT_CONFIG_NAME` |
| Выход | success text с выбранным ADB executable path либо обычный error text |
| Чтение/запись | machine-level mutation |
| Побочные эффекты | определяет ADB executable; запускает `adb kill-server`, затем `adb start-server` |
| Зависимости | `State.deploy_config.AdbExecutable`, local executable search, `subprocess.run` |
| Область привилегий | system-wide ADB lifecycle |
| Валидация | SDK string; `instance` присваивается local variable, но не ограничивает фактическую ADB operation |
| Аутентификация/авторизация | отсутствует |
| Локальность | отдельный guard отсутствует |
| Чувствительные данные | success output может раскрыть абсолютный путь к ADB executable |
| Модель ошибок | `subprocess.run(..., check=False)` return codes игнорируются; nonzero exit способен завершиться ложным `Success:`; exception → обычный error text |
| Таймаут/отмена | subprocess timeout не задан; sync blocking; command ID/dedup нет |
| Идемпотентность | нет в операционном смысле: каждый повтор делает новый kill/start cycle |
| Разрушительность | да — прерывает общий ADB server и может затронуть другие устройства/процессы |
| Доступ во внешний мир | нет прямого open-world network вызова, но scope шире одного AzurPilot instance |
| Риск | **R4** |
| Будущее разрешение | `system:adb-control` |
| Будущий владелец сервиса | `SystemService` при решении сохранить capability |
| Решение | **DEFER** — remote exposure зависит от отдельной policy о machine-wide ADB control |
| Доказательство | [`mcp_server_sse.py::_tool_restart_adb`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Уверенность | подтверждено |

## 5. Сверка registrations

Порядок имён `list_tools()` и keys `TOOL_HANDLERS` совпадает по составу:

```text
list_instances
get_status
list_tasks
get_task_help
get_resources
get_config
update_config
get_recent_logs
start_instance
stop_instance
get_screenshot
get_current_running_task
get_scheduler_queue
trigger_task
clear_scheduler_queue
restart_emulator
restart_adb
```

Несопоставленных tool registrations: **0**.

## 6. Не-tool capabilities и transport registrations

| Audit ID | Тип | Текущее состояние | Доказательство | Решение |
| --- | --- | --- | --- | --- |
| MCP-X-001 | `resources` | не зарегистрированы | отсутствует `mcp_server.list_resources()`/`read_resource()` registration в production MCP module | DEFERRED: не добавлять ради parity |
| MCP-X-002 | `resource templates` | не зарегистрированы | отсутствует `list_resource_templates()` | DEFERRED |
| MCP-X-003 | `prompts` | не зарегистрированы | отсутствуют `list_prompts()`/`get_prompt()` | DEFERRED |
| MCP-X-004 | `completions` | не зарегистрированы | отсутствует `completion()` | DEFERRED |
| MCP-X-005 | `subscriptions` | не зарегистрированы | resource subscription/modern `subscriptions/listen` current server не использует | DEFERRED |
| MCP-X-006 | `experimental/extensions` | не зарегистрированы | `mcp_server.experimental` и experimental capabilities в AzurPilot code не используются | DEFERRED |
| MCP-TR-001 | legacy SSE connect | suffix `*/sse` → `_run_sse` → `SseServerTransport.connect_sse` | pinned `mcp_server_sse.py` | REDESIGN transport |
| MCP-TR-002 | legacy message POST | suffix `*/messages[/]` → `_handle_mcp_post` → `handle_post_message` | pinned `mcp_server_sse.py` | REDESIGN transport |

Отсутствующий capability type не считается предложением добавить его в будущем.

## 7. Целевые Tool Annotations

Для всех 17 tools оценки `readOnlyHint`, `destructiveHint`, `idempotentHint` и `openWorldHint`, а также future permission, service owner, read-only MVP и human confirmation policy сведены в [целевой границе и решениях](MCP_TARGET_BOUNDARY_AND_DECISIONS.md). Это **TARGET DECISION**, а не current metadata: текущие `Tool(...)` annotations не объявляют.

Формальная сверка обязательных полей и discovery completeness находится в [сверке приёмки Этапа 2](STAGE_2_ACCEPTANCE_RECONCILIATION.md).
