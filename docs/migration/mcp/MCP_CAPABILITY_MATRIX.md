# Матрица capabilities текущего MCP

Источник: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## 1. Проверяемые counts

```text
tools discovered: 17 / 17 classified
resources discovered: 0 / 0 classified
resource templates discovered: 0 / 0 classified
prompts discovered: 0 / 0 classified
completions discovered: 0 / 0 classified
subscriptions discovered: 0 / 0 classified
custom transport path families: 2 / 2 classified
experimental/extensions registrations: 0 / 0 classified
unmapped registrations: 0
```

`mcp_server_sse.py::list_tools()` возвращает 17 `Tool(...)`; `TOOL_HANDLERS` содержит те же 17 имён. Зарегистрированы `list_tools()` и `call_tool()`, но не зарегистрированы resource/prompt/completion/experimental handlers.

## 2. Общая current transport/security/error граница

Эти свойства относятся ко всем 17 tools, если запись конкретного tool не уточняет их:

- **Auth/Authz:** отсутствуют в MCP wrapper и handlers; actor/permission context не передаётся.
- **Locality:** собственного localhost guard нет. В `mcp==1.23.0` `SseServerTransport` получает `security_settings=None`, что отключает Host/Origin DNS-rebinding checks для backwards compatibility; POST `Content-Type` всё же проверяется.
- **CORS:** wrapper разрешает `allow_origins/methods/headers=["*"]`.
- **JSON Schema validation:** SDK `Server.call_tool(validate_input=True)` валидирует arguments против advertised `inputSchema` до AzurPilot handler; schema failure возвращается как `isError=true`.
- **Domain/runtime errors:** AzurPilot `call_tool()` ловит исключения и возвращает обычный `TextContent`; SDK затем формирует обычный результат `isError=false`. Ряд handlers также самостоятельно возвращает `Error: ...` как обычный текст.
- **Output schema:** ни один текущий Tool не объявляет `outputSchema`; почти все результаты — unstructured `TextContent`, кроме screenshot `ImageContent`.
- **Timeout:** MCP-owned per-tool timeout отсутствует.
- **Deduplication:** command ID/idempotency key отсутствуют.
- **Rate limit:** MCP-owned rate limit отсутствует.
- **Audit:** нет actor-aware structured audit записи успешного tool call; runtime-компоненты могут писать свои технические логи.
- **Cancellation:** SDK выполняет requests асинхронно, но handlers AzurPilot преимущественно делают синхронную работу внутри `async def`; blocking `time.sleep`, filesystem, subprocess/device/process calls не получают отдельной cooperative cancellation policy от adapter.

## 3. Risk taxonomy

```text
R0 — metadata / pure local computation
R1 — operational read, без целевой mutation, но с runtime/user data
R2 — ограниченная state mutation внутри AzurPilot
R3 — privileged runtime/device/config/process control
R4 — системная/широкая machine-level операция
```

Итоговое распределение:

```text
R0: 2
R1: 7
R2: 0
R3: 7
R4: 1
Total: 17
```

## 4. Tool-by-tool matrix

### MCP-T-001 — `list_instances`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `mcp_server_sse.py::list_tools` → `TOOL_HANDLERS["list_instances"]` → `_tool_list_instances` |
| Inputs | object, `properties={}`; `additionalProperties:false` не задан |
| Output | `TextContent` с JSON-массивом имён экземпляров |
| Read/Write | read |
| Side effects | целевая mutation не выполняется |
| Dependencies | `module.config.utils.alas_instance()` |
| Privilege domain | instance/config discovery |
| Validation | только SDK JSON Schema object; лишние свойства schema не запрещает |
| Auth/Authz | отсутствует |
| Locality | отсутствует отдельный guard |
| Sensitive data | имена пользовательских экземпляров могут быть идентифицирующими |
| Error model | исключение → AzurPilot error text, MCP `isError=false` |
| Timeout/cancel | timeout нет; синхронный discovery |
| Idempotency | yes для product read semantics |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `instances:read` |
| Future service owner | `InstanceService` |
| Decision | **RETAIN** |
| Evidence | [`mcp_server_sse.py::_tool_list_instances`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-002 — `get_status`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_status` |
| Inputs | пустой object; tool возвращает **все** экземпляры |
| Output | JSON-массив `{instance, running, state}` в `TextContent` |
| Read/Write | operational read; `ProcessManager.alive/state` может выполнять housekeeping устаревшей process registry |
| Side effects | целевая пользовательская mutation отсутствует; возможна внутренняя очистка stale worker bookkeeping |
| Dependencies | `alas_instance`, `ProcessManager.get_manager`, `alive`, `state` |
| Privilege domain | process/runtime status |
| Validation | schema object; instance selector отсутствует |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | имена экземпляров + runtime state |
| Error model | exception text как обычный результат |
| Timeout/cancel | MCP timeout нет; runtime status path синхронный |
| Idempotency | yes по product read semantics |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `status:read` |
| Future service owner | `InstanceService` |
| Decision | **RETAIN** |
| Evidence | [`mcp_server_sse.py::_tool_get_status`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`ProcessManager.alive/state`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/process_manager.py) |
| Confidence | confirmed |

### MCP-T-003 — `list_tasks`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_list_tasks` |
| Inputs | пустой object |
| Output | JSON-массив task names |
| Read/Write | metadata read |
| Side effects | нет после module/helper initialization |
| Dependencies | `McpConfigHelper.get_tasks()`; cached `args.json` metadata |
| Privilege domain | task metadata |
| Validation | object schema; лишние свойства не запрещены |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | значимых runtime/user secrets не ожидается; это definition metadata |
| Error model | exception text как обычный result |
| Timeout/cancel | timeout нет |
| Idempotency | yes |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R0** |
| Future permission | `tasks:read` |
| Future service owner | `TaskService` |
| Decision | **RETAIN** |
| Evidence | [`mcp_server_sse.py::_tool_list_tasks`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`McpConfigHelper.get_tasks`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |
| Confidence | confirmed |

### MCP-T-004 — `get_task_help`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_task_help` |
| Inputs | required `task_name:string` |
| Output | JSON metadata: display name/help/groups/arguments/default/options |
| Read/Write | metadata read |
| Side effects | нет |
| Dependencies | `McpConfigHelper.get_task_details`, args/i18n files cached at helper creation |
| Privilege domain | task/config-schema metadata |
| Validation | string type; существование task schema не проверяет; unknown task → `{}` |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | definition defaults/options, не текущий user config |
| Error model | неизвестная задача выглядит как успешный `{}`; exception → ordinary error text |
| Timeout/cancel | timeout нет |
| Idempotency | yes |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R0** |
| Future permission | `tasks:read` |
| Future service owner | `TaskService` |
| Decision | **RETAIN** |
| Evidence | [`mcp_server_sse.py::_tool_get_task_help`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`McpConfigHelper.get_task_details`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |
| Confidence | confirmed |

### MCP-T-005 — `get_resources`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_resources` |
| Inputs | required `instance:string` |
| Output | JSON Dashboard resource projection: label/value и optional limit/total/last_update |
| Read/Write | operational read |
| Side effects | user-intended mutation нет; создание `AzurLaneConfig` проходит обычный config load/init path |
| Dependencies | `AzurLaneConfig`, `McpConfigHelper.get_dashboard_resources` |
| Privilege domain | per-instance operational resources |
| Validation | SDK проверяет string, но существование/permission экземпляра отдельно не проверяются MCP layer |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | внутриигровые ресурсы/времена обновления |
| Error model | config/runtime exception → ordinary error text |
| Timeout/cancel | MCP timeout нет; sync config read |
| Idempotency | yes по read semantics |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `resources:read` |
| Future service owner | `StatisticsService` |
| Decision | **RETAIN** |
| Evidence | [`mcp_server_sse.py::_tool_get_resources`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`McpConfigHelper.get_dashboard_resources`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |
| Confidence | confirmed |

### MCP-T-006 — `get_config`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_config` |
| Inputs | required `instance:string`; optional `task:string` |
| Output | весь `config.data` либо raw section указанной task |
| Read/Write | broad operational/config read |
| Side effects | целевая mutation отсутствует; config constructor использует обычный load/init/migration path |
| Dependencies | `AzurLaneConfig.data` |
| Privilege domain | configuration |
| Validation | типы проверяются SDK; field-level allowlist/redaction отсутствуют; unknown task → `{}` |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | широкий raw config **может** содержать device/runtime/integration identifiers или другие чувствительные значения; конкретное наличие секрета не предполагается без field audit |
| Error model | exception → ordinary error text |
| Timeout/cancel | MCP timeout нет; sync config load/JSON serialization |
| Idempotency | yes по read semantics |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `config:read` + field policy |
| Future service owner | `ConfigService` |
| Decision | **REDESIGN** — raw config dump не является target contract |
| Evidence | [`mcp_server_sse.py::_tool_get_config`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-007 — `update_config`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_update_config` |
| Inputs | required `instance`, `task`, `group`, `arg`, `value`; `value` допускает string/number/bool/object/array/null |
| Output | success text с собранным `task.group.arg` и значением |
| Read/Write | mutation |
| Side effects | `AzurLaneConfig.cross_set(path,value)` → `update()`/save; изменяет persistent config |
| Dependencies | `AzurLaneConfig.cross_set`, `save`, config filesystem |
| Privilege domain | privileged config control |
| Validation | SDK schema проверяет типы; MCP handler не проверяет существование/allowlist path и не применяет WebUI `parse_pin_value`/validation/`ConfigUpdater.save_callback` policy |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | способен менять широкие runtime/automation параметры; success text повторяет submitted value |
| Error model | schema errors → `isError=true`; domain/runtime exception → ordinary error text `isError=false` |
| Timeout/cancel | MCP timeout/command id нет; sync config I/O |
| Idempotency | conditional: одинаковое присваивание часто имеет тот же state, но config update/migration callbacks и downstream runtime meaning не гарантируют отсутствие повторного эффекта |
| Destructiveness | conditional: значение может отключить/перенастроить automation |
| Open-world reach | conditional через изменённую automation configuration |
| Risk | **R3** |
| Future permission | `config:write` с field/domain policy |
| Future service owner | `ConfigService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_update_config`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`AzurLaneConfig.cross_set`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/config.py), WebUI validation evidence из Этапа 1 |
| Confidence | confirmed |

### MCP-T-008 — `get_recent_logs`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_recent_logs` |
| Inputs | `instance:string` required; `lines:integer` optional default 50, без min/max |
| Output | raw joined log lines либо обычный error/not-found text |
| Read/Write | filesystem read |
| Side effects | нет целевой mutation; весь дневной файл загружается `readlines()` до slicing |
| Dependencies | `os.path`, local `./log/<date>_<instance>.txt`, fallback `./log/<date>_alas.txt` |
| Privilege domain | logs/filesystem read |
| Validation | integer type есть; размер/знак `lines` не ограничен; instance не разрешается через `alas_instance()` до filename generation |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | raw logs могут содержать runtime identifiers, paths, errors и operational context |
| Error model | file exception и not-found возвращаются обычным text result, не `isError=true` |
| Timeout/cancel | timeout нет; sync full-file `readlines()`; large-output/DoS риск |
| Idempotency | yes как read operation, но содержимое лога меняется во времени |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `logs:read` |
| Future service owner | `LogService` |
| Decision | **REDESIGN** — bounded/sanitized log read model |
| Evidence | [`mcp_server_sse.py::_tool_get_recent_logs`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-009 — `start_instance`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_start_instance` |
| Inputs | required `instance:string` |
| Output | error text если уже alive; иначе success text с config mod |
| Read/Write | privileged mutation |
| Side effects | создаёт/запускает AzurPilot worker process через `ProcessManager.start` |
| Dependencies | `ProcessManager`, `get_config_mod`, worker registry/process lifecycle |
| Privilege domain | process/runtime control |
| Validation | SDK string; отдельной instance allowlist/authz в MCP нет; `get_config_mod`/manager дают дальнейшую runtime semantics |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | instance/config mod в output |
| Error model | already-running — ordinary `Error:` text; `ProcessManager.start()` возвращает `None` и может отклонить старт из-за lifecycle state, но tool после возврата всё равно формирует `Success:` |
| Timeout/cancel | MCP timeout/idempotency key нет; process start sync; underlying manager имеет lifecycle locks |
| Idempotency | conditional: повтор при alive не создаёт новый worker, но race/transient rejection меняют эффект retry |
| Destructiveness | нет прямого удаления, но операция запускает automation |
| Open-world reach | **да/conditional по выполняемым задачам**: запущенная automation взаимодействует с игровым окружением |
| Risk | **R3** |
| Future permission | `instances:control` |
| Future service owner | `InstanceService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_start_instance`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`ProcessManager.start`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/process_manager.py) |
| Confidence | confirmed |

### MCP-T-010 — `stop_instance`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_stop_instance` |
| Inputs | required `instance:string` |
| Output | error text если already stopped; иначе `Success: Stopped ...` |
| Read/Write | privileged mutation |
| Side effects | `ProcessManager.stop()` останавливает worker, при необходимости terminate/kill process tree |
| Dependencies | `ProcessManager`, OS process operations |
| Privilege domain | process/runtime control |
| Validation | SDK string; MCP-level instance permission нет |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | instance name |
| Error model | `manager.stop()` возвращает bool, но tool игнорирует его и сообщает success после вызова; возможен ложный success при неполной остановке |
| Timeout/cancel | MCP timeout нет; underlying stop содержит собственные process wait/kill timeouts, не равные request deadline |
| Idempotency | conditional; повтор обычно не даёт дополнительного эффекта, но race с restart способен изменить цель |
| Destructiveness | **да** в операционном смысле: принудительно прерывает worker/process tree |
| Open-world reach | нет прямого внешнего reach |
| Risk | **R3** |
| Future permission | `instances:control` |
| Future service owner | `InstanceService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_stop_instance`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`ProcessManager.stop`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/process_manager.py) |
| Confidence | confirmed |

### MCP-T-011 — `get_screenshot`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_screenshot` |
| Inputs | required `instance:string` |
| Output | `ImageContent` JPEG Base64; при error — text с exception + traceback |
| Read/Write | privileged device read |
| Side effects | создаёт `Device`, выполняет `device.screenshot()`, временно меняет process-global compatibility context (`ALAS_CONFIG_NAME` только если отсутствует, fake PIL hook) |
| Dependencies | `AzurLaneConfig`, `Device`, PIL, process environment |
| Privilege domain | device/view/privacy |
| Validation | SDK string; отдельной device/instance authorization policy нет |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | изображение экрана может содержать приватный/игровой контекст; traceback может раскрыть paths/device details |
| Error model | handler ловит exception и возвращает полный traceback как ordinary text `isError=false` |
| Timeout/cancel | MCP timeout нет; screenshot/device operation синхронна |
| Idempotency | read; кадр меняется во времени |
| Destructiveness | нет |
| Open-world reach | нет: доступ ограничен выбранным локальным device runtime |
| Risk | **R3** |
| Future permission | `device:view` |
| Future service owner | `DeviceService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_get_screenshot`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-012 — `get_current_running_task`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_current_running_task` |
| Inputs | required `instance:string` |
| Output | task name text, `Unknown`, либо ordinary `Error:` text |
| Read/Write | operational read |
| Side effects | manager status read + полный synchronous read дневного log для reverse regex scan |
| Dependencies | `ProcessManager`, filesystem log, regex |
| Privilege domain | task/runtime/log read |
| Validation | SDK string; no task semantics needed |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | task name/runtime status; log сам читается server-side, но наружу отдаётся только найденное имя |
| Error model | not-running → ordinary Error text; log parse exceptions полностью подавляются bare `except`, результат `Unknown` |
| Timeout/cancel | MCP timeout нет; full log `readlines()` sync |
| Idempotency | read, но task меняется во времени |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `tasks:read` |
| Future service owner | `TaskService` |
| Decision | **RETAIN** capability; target implementation через runtime read model, не log parsing |
| Evidence | [`mcp_server_sse.py::_tool_get_current_running_task`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-013 — `get_scheduler_queue`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_get_scheduler_queue` |
| Inputs | required `instance:string` |
| Output | JSON-массив `{task,next_run}` для enabled scheduler tasks |
| Read/Write | operational read |
| Side effects | целевая mutation отсутствует; обычный `AzurLaneConfig` load/init path может выполнять предусмотренные config normalization/migration действия |
| Dependencies | `AzurLaneConfig.data` / Scheduler sections |
| Privilege domain | scheduler read |
| Validation | SDK string; MCP layer не разрешает отдельный instance permission |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | task names и schedule timing |
| Error model | runtime/config exception → ordinary error text |
| Timeout/cancel | MCP timeout нет; sync config read/sort |
| Idempotency | read, очередь меняется во времени |
| Destructiveness | нет |
| Open-world reach | нет |
| Risk | **R1** |
| Future permission | `scheduler:read` |
| Future service owner | `SchedulerService` |
| Decision | **RETAIN** |
| Evidence | [`mcp_server_sse.py::_tool_get_scheduler_queue`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-014 — `trigger_task`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_trigger_task` |
| Inputs | required `instance:string`, `task:string` |
| Output | success text |
| Read/Write | privileged scheduler mutation |
| Side effects | выставляет `<task>.Scheduler.Enable=True` и `NextRun=now`, сохраняя config; активный worker может затем выполнить automation |
| Dependencies | `AzurLaneConfig.cross_set/update/save`, `current_time`, scheduler runtime |
| Privilege domain | scheduler/automation control |
| Validation | SDK string types; существование/allowlist task перед `cross_set` не проверяется MCP handler; `cross_set` сам помещает path в `modified` и вызывает config update |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | task/instance in output |
| Error model | mutation exception → ordinary error text; success не подтверждает последующее выполнение task |
| Timeout/cancel | no MCP timeout/command id/dedup; повтор ставит новое `NextRun=now` |
| Idempotency | **no** — повтор меняет NextRun и может повторно инициировать scheduling intent |
| Destructiveness | conditional: downstream task может менять игровое/runtime состояние |
| Open-world reach | **да/conditional** через запускаемую game automation |
| Risk | **R3** |
| Future permission | `scheduler:control` |
| Future service owner | `SchedulerService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_trigger_task`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py), [`AzurLaneConfig.cross_set`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/config.py) |
| Confidence | confirmed |

### MCP-T-015 — `clear_scheduler_queue`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_clear_scheduler_queue` |
| Inputs | required `instance:string` |
| Output | success text со списком disabled tasks |
| Read/Write | privileged scheduler mutation |
| Side effects | для каждого enabled Scheduler выставляет `Enable=False`; может отменить все текущие планы экземпляра |
| Dependencies | `AzurLaneConfig`, `cross_set`, config persistence |
| Privilege domain | scheduler control |
| Validation | SDK instance string; отдельная scope/confirmation policy отсутствует |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | task names в output |
| Error model | exception → ordinary error text; partial mutation до exception отдельным transaction/result не представлена |
| Timeout/cancel | MCP timeout/transaction/command id нет; multiple sync config updates возможны |
| Idempotency | yes относительно уже очищенной очереди, если нет concurrent re-enable |
| Destructiveness | **да** — массово удаляет enabled scheduling intent |
| Open-world reach | нет прямого external reach |
| Risk | **R3** |
| Future permission | `scheduler:control` |
| Future service owner | `SchedulerService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_clear_scheduler_queue`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-016 — `restart_emulator`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_restart_emulator` |
| Inputs | required `instance:string` |
| Output | success text; при exception — exception + full traceback text |
| Read/Write | privileged device/runtime mutation |
| Side effects | `Device.emulator_stop()` → blocking `time.sleep(60)` → `Device.emulator_start()` |
| Dependencies | `AzurLaneConfig`, `Device`, process env/fake-PIL compatibility; `ProcessManager.get_manager(inst)` создаётся, но полученное значение в handler далее не используется |
| Privilege domain | device/emulator lifecycle |
| Validation | SDK string; MCP policy/precondition проверки активности task/device отсутствуют |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | exception traceback может раскрыть paths/device context |
| Error model | exception → traceback как ordinary `isError=false` text; нет structured phase result stop/wait/start |
| Timeout/cancel | **нет MCP timeout**; `time.sleep(60)` блокирует event loop; cancellation/disconnect не прерывает этот sleep cooperatively |
| Idempotency | **no** — каждый повтор снова останавливает/запускает emulator |
| Destructiveness | **да** — disruptive restart локального device runtime |
| Open-world reach | нет прямого network reach; operation ограничена локальным emulator runtime |
| Risk | **R3** |
| Future permission | `device:control` |
| Future service owner | `DeviceService` |
| Decision | **REDESIGN** |
| Evidence | [`mcp_server_sse.py::_tool_restart_emulator`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

### MCP-T-017 — `restart_adb`

| Поле | Значение |
| --- | --- |
| MCP type | tool |
| Registration | `list_tools` → `_tool_restart_adb` |
| Inputs | optional `instance:string`; default `DEFAULT_CONFIG_NAME` |
| Output | success text с выбранным ADB executable path либо ordinary error text |
| Read/Write | machine-level mutation |
| Side effects | определяет ADB executable; запускает `adb kill-server`, затем `adb start-server` |
| Dependencies | `State.deploy_config.AdbExecutable`, local executable search, `subprocess.run` |
| Privilege domain | system-wide ADB lifecycle |
| Validation | SDK string; `instance` присваивается local variable, но **не ограничивает** фактическую ADB operation |
| Auth/Authz | отсутствует |
| Locality | отдельный guard отсутствует |
| Sensitive data | success output может раскрыть абсолютный путь к ADB executable |
| Error model | `subprocess.run(..., check=False)` return codes игнорируются; nonzero exit способен завершиться ложным `Success:`; exception → ordinary error text |
| Timeout/cancel | subprocess timeout не задан; sync blocking; command id/dedup нет |
| Idempotency | **no** в операционном смысле: каждый повтор делает новый kill/start cycle |
| Destructiveness | **да** — прерывает общий ADB server и может затронуть другие устройства/процессы |
| Open-world reach | нет прямого open-world network вызова, но scope шире одного AzurPilot instance |
| Risk | **R4** |
| Future permission | `system:adb-control` |
| Future service owner | `SystemService` при решении сохранить capability |
| Decision | **DEFER** — remote exposure зависит от отдельной policy о machine-wide ADB control |
| Evidence | [`mcp_server_sse.py::_tool_restart_adb`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| Confidence | confirmed |

## 5. Registration reconciliation

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

## 6. Не-tool capabilities

| Audit ID | Type | Current state | Evidence | Decision |
| --- | --- | --- | --- | --- |
| MCP-X-001 | resources | не зарегистрированы | отсутствует `mcp_server.list_resources()`/`read_resource()` registration в production MCP module | DEFERRED: не добавлять ради parity |
| MCP-X-002 | resource templates | не зарегистрированы | отсутствует `list_resource_templates()` | DEFERRED |
| MCP-X-003 | prompts | не зарегистрированы | отсутствуют `list_prompts()`/`get_prompt()` | DEFERRED |
| MCP-X-004 | completions | не зарегистрированы | отсутствует `completion()` | DEFERRED |
| MCP-X-005 | subscriptions | не зарегистрированы | resource subscription/modern `subscriptions/listen` current server не использует | DEFERRED |
| MCP-X-006 | experimental/extensions | не зарегистрированы | `mcp_server.experimental` и experimental capabilities в AzurPilot code не используются | DEFERRED |
| MCP-TR-001 | legacy SSE connect | suffix `*/sse` → `_run_sse` → `SseServerTransport.connect_sse` | pinned `mcp_server_sse.py` | REDESIGN transport |
| MCP-TR-002 | legacy message POST | suffix `*/messages[/]` → `_handle_mcp_post` → `handle_post_message` | pinned `mcp_server_sse.py` | REDESIGN transport |

Отсутствующий capability type не считается предложением добавить его в будущем.
