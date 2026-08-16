# Текущая архитектура MCP AzurPilot

## 1. Зафиксированный источник

**CURRENT FACT**

```text
backend: AliceLiddell01/AzurPilot-private-Ru
ref: personal/stable
SHA: 3be3696975cb91ba0b85dbea98400381c3ced379
```

Все current-state выводы ниже относятся только к этому snapshot.

## 2. MCP-related production files

### MCP-owned

| Файл | Роль | Доказательство |
| --- | --- | --- |
| `mcp_server_sse.py` | `Server`, 17 tools, dispatch, legacy SSE/message transport, ASGI wrapper, standalone entrypoint | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| `module/config/mcp_helper.py` | task metadata/i18n и Dashboard resource projection для MCP | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |

### Wiring/dependency declarations

| Файл | Роль | Доказательство |
| --- | --- | --- |
| `module/webui/app.py` | production mount `application.mount("/mcp", mcp_app)` | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app.py) |
| `pyproject.toml` | `mcp==1.23.0`, `sse-starlette==3.0.3`, Starlette/Uvicorn constraints | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/pyproject.toml) |

### Общие runtime dependencies

MCP adapter напрямую использует, но не владеет:

- `module.webui.process_manager.ProcessManager`;
- `module.config.config.AzurLaneConfig`;
- `module.webui.setting.State`;
- `module.device.device.Device`;
- config/args/i18n files через `McpConfigHelper`;
- локальные журналы `./log/...`;
- `subprocess` для ADB;
- process-global `ALAS_CONFIG_NAME` и fake-PIL compatibility hook.

## 3. Точная версия SDK

**CURRENT FACT**

`pyproject.toml` содержит:

```text
mcp==1.23.0
sse-starlette==3.0.3
```

Официальный release `mcp` v1.23.0 относится к поколению спецификации `2025-11-25`; текущий AzurPilot не использует Python SDK v2 и не реализует protocol core `2026-07-28`.

**OFFICIAL MCP TARGET GUIDANCE**

Актуальная опубликованная ревизия — `2026-07-28`. Она перешла к stateless protocol core и формально считает legacy HTTP+SSE transport deprecated. Текущая стабильная линия официального Python SDK — v2, тогда как v1.x оставлен в maintenance mode.

Это target reference, а не описание current AzurPilot.

## 4. Server construction и capability registration

**CURRENT FACT**

```python
helper = McpConfigHelper()
mcp_server = Server("AzurPilot-MCP")
```

Зарегистрированы только два low-level handler families:

```text
@mcp_server.list_tools()
@mcp_server.call_tool()
```

Итог:

```text
tools: 17
resources: 0
resource templates: 0
prompts: 0
completions: 0
subscriptions: 0
experimental/extensions registrations: 0
```

В `mcp==1.23.0` capabilities объявляются по наличию соответствующих request handlers. Поскольку AzurPilot не регистрирует `list_resources`, `list_resource_templates`, `list_prompts`, `completion` или experimental handlers, эти capability families не публикуются текущим сервером.

Ни один `Tool(...)` не содержит `annotations` или `outputSchema`.

## 5. Embedded production path

**CURRENT FACT**

`module/webui/app.py::app()` выполняет:

```text
from mcp_server_sse import app as mcp_app
...
application.mount("/mcp", mcp_app)
```

Внутри MCP module:

```text
transport = SseServerTransport("/mcp/messages")
app = Starlette(...)
app.mount("/", mcp_asgi_app)
```

Пользовательский request path:

```text
MCP client
    |
    | GET legacy SSE
    v
outer WebUI Starlette
    |
    | mount /mcp
    v
MCP Starlette wrapper
    |
    | mcp_asgi_app: path.endswith("/sse")
    v
SseServerTransport.connect_sse()
    |
    | endpoint event -> /mcp/messages?session_id=<uuid>
    v
client POST /mcp/messages?session_id=<uuid>
    |
    v
mcp_asgi_app -> SseServerTransport.handle_post_message()
    |
    v
Server.run() -> tools/call -> call_tool() -> TOOL_HANDLERS
    |
    v
AzurPilot runtime/core
```

Номинальные embedded URL families:

```text
GET  /mcp/sse
POST /mcp/messages?session_id=<uuid>
```

`mcp_asgi_app` использует suffix matching, а не declarative Starlette `Route` methods. Поэтому внутри достигнутого mount он проверяет окончание пути `.../sse`, `.../messages` или `.../messages/`.

## 6. Standalone mode

**CURRENT FACT**

При прямом запуске `mcp_server_sse.py` выполняется:

```python
uvicorn.run(app, host="0.0.0.0", port=22268)
```

То есть процесс **может слушать все интерфейсы** на порту `22268`.

Это не доказывает internet reachability. Router/NAT/firewall/OS ACL/Caddy конкретной машины Этап 2 не проверяет.

Standalone использует ту же Starlette wrapper и тот же `SseServerTransport`; отдельной auth/security policy для standalone нет.

## 7. Legacy SSE session model

**CURRENT FACT**, включая поведение точной зависимости `mcp==1.23.0`.

`SseServerTransport`:

1. создаёт UUID `session_id` при SSE connect;
2. хранит process-local mapping `session_id -> MemoryObjectSendStream`;
3. отправляет клиенту endpoint event с `session_id`;
4. POST message ищет writer по этому UUID;
5. JSON-RPC body валидируется Pydantic-моделью SDK;
6. accepted POST получает HTTP `202`, после чего message передаётся в process-local stream текущей SSE session.

После restart процесса эта in-memory session registry теряется. Horizontal/shared-state semantics отсутствуют.

`mcp_server.run(...)` вызывается с default `stateless=False`, поэтому current lifecycle соответствует handshake/session-era SDK, а не `2026-07-28` stateless core.

**OFFICIAL MCP TARGET GUIDANCE**

Ревизия `2026-07-28` убрала protocol-level handshake/session core и перенесла идентичность/version/capabilities запроса в request metadata. Следующий transport design не должен копировать hidden SSE session state только ради parity.

## 8. Transport security: что реально включено

### CORS

**CURRENT FACT**

MCP Starlette wrapper задаёт:

```python
CORSMiddleware(
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"]
)
```

### SDK-level DNS-rebinding protection

**CURRENT FACT**

В `mcp==1.23.0` `SseServerTransport` создаёт `TransportSecurityMiddleware(security_settings)`.

Но при `security_settings=None`, как в AzurPilot, constructor middleware намеренно подставляет:

```text
enable_dns_rebinding_protection = false
```

для backwards compatibility. Поэтому `Host` и `Origin` **не проверяются** SDK transport middleware в текущем AzurPilot.

При этом POST всегда проходит отдельную проверку `Content-Type`, даже когда DNS-rebinding protection выключена; ожидается `application/json...`.

### Authentication / authorization

**CURRENT FACT**

- в `mcp_server_sse.py` нет auth/authz middleware или locality guard;
- `module/webui/app.py::_run_gui()` выполняет PyWebIO password gate только внутри PyWebIO entry points;
- MCP монтируется отдельным ASGI child app после сборки outer application;
- tool handlers не получают actor/identity/permission context.

Следовательно, PyWebIO password gate не является текущей защитой MCP route family.

Фактическая внешняя сетeвая достижимость не устанавливается этим выводом.

## 9. Input validation

**CURRENT FACT**

Слой валидации состоит из двух частей.

### SDK JSON Schema validation

`mcp==1.23.0::Server.call_tool(validate_input=True)` по умолчанию получает cached `Tool.inputSchema` и выполняет `jsonschema.validate()` до вызова AzurPilot handler. Ошибка схемы становится `CallToolResult(isError=True)`.

Поэтому утверждение «у MCP вообще нет input validation» было бы неверным.

### Domain validation AzurPilot

После schema validation текущие handlers часто имеют минимальные semantic checks:

- `instance` обычно только `string`, без schema enum существующих экземпляров;
- `get_task_help` неизвестной задачи возвращает `{}`;
- `update_config` принимает произвольные `task/group/arg` strings и напрямую вызывает `config.cross_set(path, value)`;
- `trigger_task` не проверяет task против отдельного allowlist перед `cross_set`;
- `get_recent_logs.lines` имеет тип integer, но без min/max;
- `restart_adb.instance` необязателен, но выбранный instance не ограничивает фактическую system-wide ADB операцию.

## 10. Error flow

**CURRENT FACT**

SDK v1.23.0 умеет возвращать `CallToolResult(isError=True)` для schema validation и для исключений, которые доходят до wrapper handler.

AzurPilot `call_tool()` при этом сам ловит почти все handler exceptions:

```text
except Exception as e:
    logger.exception(...)
    return TextContent("Ошибка: ...")
```

После такого возврата SDK нормализует список content blocks как обычный `CallToolResult(isError=False)`.

Аналогично несколько individual handlers возвращают строки `Error: ...` вместо structured/error result. Поэтому у current server существуют два разных error channels:

```text
schema error before AzurPilot handler -> MCP isError=true
runtime/domain error, преобразованный AzurPilot в TextContent -> MCP isError=false + error text
```

`get_screenshot` и `restart_emulator` дополнительно включают полный Python traceback в текст ошибки.

## 11. Long-running / cancellation / retry / concurrency

**CURRENT FACT**

В MCP-owned коде не обнаружены:

- per-tool timeout;
- idempotency/deduplication key;
- command ID;
- MCP-level rate limit;
- actor-aware audit event;
- application-level cancellation protocol;
- per-capability permission gate.

Отдельные runtime dependencies могут иметь собственные locks/timeouts. Например, `ProcessManager.start/stop` использует lifecycle locks и stop timeouts; это не является MCP request timeout или deduplication.

Особые случаи:

- `restart_emulator` вызывает `time.sleep(60)` **внутри `async def`**, блокируя event-loop thread этого процесса на минуту до `emulator_start()`;
- `restart_adb` вызывает два `subprocess.run(..., check=False)` без заданного timeout и не проверяет return code;
- `get_recent_logs` загружает весь дневной log через `readlines()` до tail slicing;
- `get_current_running_task` тоже читает дневной log целиком и подавляет ошибки bare `except`;
- повтор `trigger_task`, `clear_scheduler_queue`, start/stop/restart после client timeout не имеет MCP-level deduplication semantics.

## 12. Прямые privilege domains

| Область | Current path |
| --- | --- |
| metadata/config definitions | `McpConfigHelper` -> args/i18n files |
| instance/runtime state | `ProcessManager.get_manager()` |
| process control | `ProcessManager.start/stop()` |
| config read/write | `AzurLaneConfig.data`, `cross_set`, `save` |
| scheduler mutation | config `Scheduler.Enable` / `Scheduler.NextRun` |
| filesystem logs | `./log/<date>_<instance>.txt` |
| device privacy/read | `Device.screenshot()` |
| emulator lifecycle | `Device.emulator_stop/start()` |
| system ADB | executable resolution + `adb kill-server/start-server` |

## 13. Current architecture diagram

```text
                          embedded mode
PyWebIO WebUI app
    |
    +-- _run_gui() password gate ----- PyWebIO sessions
    |
    `-- mount /mcp -------------------------------+
                                                   |
                                                   v
                                         MCP Starlette wrapper
                                         CORS: wildcard
                                         auth: none here
                                                   |
                      +----------------------------+--------------------+
                      |                                                 |
                  */sse GET                                    */messages POST
                      |                                                 |
                      +------ SseServerTransport(session UUID) ---------+
                                                   |
                                                   v
                                           Server("AzurPilot-MCP")
                                                   |
                                    list_tools = 17 / call_tool
                                                   |
                       +---------------------------+--------------------+
                       |             |             |                    |
                ProcessManager  AzurLaneConfig   Device         FS / subprocess
                       |             |             |                    |
                       +-------------+-------------+--------------------+
                                                   |
                                                   v
                                             AzurPilot Core

standalone mode:
uvicorn 0.0.0.0:22268 -> тот же MCP Starlette wrapper
```

## 14. Целевая интерпретация

**TARGET DECISION**

Ни CORS, ни Tool Annotations, ни transport migration сами по себе не являются authorization boundary. Будущий MCP должен зависеть от общего backend policy/service layer:

```text
MCP transport
  -> authentication
  -> authorization / permission
  -> validation / policy / audit
  -> Application / Service Layer
  -> runtime/core
```

Точная реализация Streamable HTTP, OAuth и service packages не проектируется в Этапе 2.
