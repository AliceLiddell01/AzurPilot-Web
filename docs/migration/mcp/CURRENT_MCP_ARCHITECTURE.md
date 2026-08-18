# Текущая архитектура MCP AzurPilot

## 1. Зафиксированный источник

**CURRENT FACT**

```text
backend: AliceLiddell01/AzurPilot-private-Ru
ref: personal/stable
SHA: 3be3696975cb91ba0b85dbea98400381c3ced379
```

Все утверждения о текущем состоянии ниже относятся только к этому snapshot.

## 2. Файлы production MCP

### Код, принадлежащий MCP

| Файл | Роль | Доказательство |
| --- | --- | --- |
| `mcp_server_sse.py` | `Server`, 17 tools, dispatch, legacy SSE/message transport, ASGI wrapper и standalone entry point | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/mcp_server_sse.py) |
| `module/config/mcp_helper.py` | MCP-specific helper метаданных задач/i18n и проекции Dashboard resources | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/mcp_helper.py) |

### Подключение и зависимости

| Файл | Роль | Доказательство |
| --- | --- | --- |
| `module/webui/app.py` | production mount `application.mount("/mcp", mcp_app)` | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app.py) |
| `pyproject.toml` | `mcp==1.23.0`, `sse-starlette==3.0.3`, `starlette==0.49.1`, Uvicorn | [pinned source](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/pyproject.toml) |

Repository-wide discovery и import/wiring graph не обнаружили второго production MCP server construction, второго registration catalog или альтернативного production mount на этом snapshot. Методика и границы доказательства описаны в [сверке приёмки](STAGE_2_ACCEPTANCE_RECONCILIATION.md).

### Общие зависимости среды выполнения

MCP adapter напрямую использует, но не владеет:

- `module.webui.process_manager.ProcessManager`;
- `module.config.config.AzurLaneConfig`;
- `module.webui.setting.State`;
- `module.device.device.Device`;
- config/args/i18n через `McpConfigHelper`;
- локальные журналы `./log/...`;
- `subprocess` для ADB;
- process-global `ALAS_CONFIG_NAME` и fake-PIL compatibility hook.

## 3. Версии SDK и transport-зависимостей

**CURRENT FACT**

`pyproject.toml` фиксирует:

```text
mcp==1.23.0
sse-starlette==3.0.3
starlette==0.49.1
```

Текущий AzurPilot использует low-level Python SDK v1.x и `SseServerTransport`.

**OFFICIAL MCP TARGET GUIDANCE**

Актуальная опубликованная ревизия MCP — `2026-07-28`; официальный Python SDK v2 является текущей stable line, v1.x — maintenance. Legacy HTTP+SSE transport deprecated. Это целевой ориентир, а не описание текущего AzurPilot.

## 4. Создание сервера и регистрации

**CURRENT FACT**

```python
helper = McpConfigHelper()
mcp_server = Server("AzurPilot-MCP")
```

Зарегистрированы два low-level handler family:

```text
@mcp_server.list_tools()
@mcp_server.call_tool()
```

Покрытие:

```text
tools: 17
resources: 0
resource templates: 0
prompts: 0
completions: 0
subscriptions: 0
experimental/extensions registrations: 0
unmapped registrations: 0
```

Ни один текущий `Tool(...)` не объявляет `annotations` или `outputSchema`.

## 5. Embedded production mode

**CURRENT FACT**

`module/webui/app.py::app()` импортирует MCP child app и после сборки основного ASGI-приложения выполняет:

```text
from mcp_server_sse import app as mcp_app
...
application.mount("/mcp", mcp_app)
```

В `mcp_server_sse.py`:

```text
transport = SseServerTransport("/mcp/messages")
app = Starlette(...)
app.mount("/", mcp_asgi_app)
```

### Эффективные пути

Сочетание outer `Mount("/mcp")`, Starlette `root_path` и уже префиксованного endpoint `"/mcp/messages"` даёт:

```text
GET  /mcp/sse
POST /mcp/mcp/messages?session_id=<uuid>
```

Причина двойного `/mcp`:

1. outer mount добавляет `/mcp` в `scope.root_path` child app;
2. внутренний `Mount("/")` не добавляет непустой префикс;
3. `SseServerTransport.connect_sse()` строит endpoint event как `root_path + self._endpoint`;
4. `self._endpoint` уже равен `/mcp/messages`.

POST остаётся достижимым внутри текущего wrapper, потому что `mcp_asgi_app` использует suffix matching `path.endswith("/messages")`.

Двойной префикс является **CURRENT FACT о compatibility quirk**, а не целевым URL-контрактом.

### Полный путь вызова

```text
MCP client
    |
    | GET /mcp/sse
    v
WebUI Starlette
    |
    | Mount /mcp
    v
MCP Starlette wrapper
    |
    v
SseServerTransport.connect_sse()
    |
    | endpoint event: /mcp/mcp/messages?session_id=<uuid>
    v
client POST /mcp/mcp/messages?... 
    |
    v
SseServerTransport.handle_post_message()
    |
    v
Server.run()
    |
    v
tools/call -> call_tool() -> TOOL_HANDLERS
    |
    v
ProcessManager / AzurLaneConfig / Device / FS / subprocess
    |
    v
AzurPilot runtime/core
```

## 6. Standalone mode

**CURRENT FACT**

При прямом запуске `mcp_server_sse.py`:

```python
uvicorn.run(app, host="0.0.0.0", port=22268)
```

То есть процесс способен слушать все интерфейсы на порту `22268`.

Это **не доказывает** Internet/LAN reachability конкретного компьютера: router, NAT, firewall, OS ACL и Caddy не входят в статический аудит.

Standalone использует тот же MCP Starlette wrapper и тот же `SseServerTransport`, но top-level `root_path` пуст. Поэтому endpoint event содержит:

```text
/mcp/messages?session_id=<uuid>
```

SSE connect принимается current suffix-router по пути, заканчивающемуся на `/sse`.

Таким образом, embedded и standalone режимы имеют разные фактические endpoint-event paths.

## 7. Жизненный цикл и состояние legacy SSE

**CURRENT FACT**

`SseServerTransport` точной версии `mcp==1.23.0`:

1. создаёт UUID `session_id` при SSE connect;
2. хранит process-local mapping `session_id -> MemoryObjectSendStream`;
3. отправляет endpoint event клиенту;
4. POST ищет writer по UUID;
5. JSON-RPC body проходит SDK parsing/validation;
6. принятый POST получает HTTP `202`, после чего сообщение попадает в stream текущей SSE session.

Состояние сессий находится в памяти процесса. После restart процесса registry теряется; общей межпроцессной/horizontal state модели нет.

`mcp_server.run(...)` использует session-era semantics SDK v1.x. Это не целевая stateless модель MCP `2026-07-28`.

## 8. CORS, Host, Origin и локальность

### CORS

**CURRENT FACT**

Wrapper задаёт:

```python
CORSMiddleware(
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"]
)
```

### SDK DNS-rebinding protection

**CURRENT FACT**

`SseServerTransport` создаёт `TransportSecurityMiddleware(security_settings)`. В AzurPilot `security_settings=None`; compatibility-конструктор SDK v1.23.0 при этом отключает `enable_dns_rebinding_protection`.

Следствие для current adapter:

```text
Host validation: выключена
Origin validation: выключена
POST Content-Type validation: выполняется отдельно
```

### Аутентификация и авторизация

**CURRENT FACT**

- MCP wrapper не добавляет auth/authz middleware или locality guard;
- tool handlers не получают actor/identity/permission context;
- PyWebIO login выполняется внутри `_run_gui()`;
- MCP монтируется отдельным ASGI child app.

Поэтому PyWebIO password gate не является общей защитой MCP route family.

Фактическая внешняя достижимость этим выводом не устанавливается.

## 9. Валидация входа

**CURRENT FACT**

### SDK JSON Schema

`Server.call_tool(validate_input=True)` валидирует arguments против advertised `Tool.inputSchema`. Ошибка схемы возвращается SDK как `CallToolResult(isError=True)`.

### Доменная валидация AzurPilot

После protocol/schema validation текущие handlers часто ограничиваются минимальными semantic checks:

- `instance` обычно только `string`, без отдельной policy разрешённых экземпляров;
- `get_task_help` возвращает `{}` для неизвестной задачи;
- `update_config` принимает model-controlled `task/group/arg` и вызывает `cross_set`;
- `trigger_task` не валидирует task по отдельному allowlist перед config mutation;
- `get_recent_logs.lines` не имеет min/max;
- `restart_adb.instance` не ограничивает machine-wide ADB operation.

JSON Schema здесь не заменяет authorization, target resolution, field policy и runtime preconditions.

## 10. Поток ошибок

**CURRENT FACT**

SDK умеет представить schema validation failure как `isError=true`.

Однако `mcp_server_sse.py::call_tool()` ловит handler exceptions и возвращает обычный `TextContent("Ошибка: ...")`. После этого SDK нормализует content как обычный tool result с `isError=false`.

Имеются два канала ошибок:

```text
schema failure до AzurPilot handler
    -> MCP isError=true

runtime/domain failure, превращённый adapter в TextContent
    -> MCP isError=false + текст ошибки
```

Несколько handlers самостоятельно возвращают `Error: ...` как обычный текст. `get_screenshot` и `restart_emulator` дополнительно способны включать traceback в model-facing result.

## 11. Таймауты, отмена, повтор и конкурентность

**CURRENT FACT**

В MCP-owned коде не обнаружены:

- per-tool timeout;
- command ID;
- idempotency/deduplication key;
- MCP-level rate limit;
- actor-aware audit event;
- отдельный application-level cancellation protocol;
- per-capability permission gate.

Отдельные runtime dependencies могут иметь собственные locks/timeouts. Например, `ProcessManager` имеет lifecycle locks и stop timeouts, но это не MCP request timeout и не replay contract.

Критические примеры:

- `restart_emulator` выполняет `time.sleep(60)` внутри `async def`;
- `restart_adb` выполняет blocking `subprocess.run(..., check=False)` без timeout и не проверяет return code;
- `get_recent_logs` читает весь дневной файл через `readlines()` до tail slicing;
- `get_current_running_task` тоже читает журнал целиком и подавляет ошибки чтения/парсинга;
- повтор control operation после client timeout не имеет MCP-level dedup semantics.

Статический аудит не утверждает точную wall-clock длительность runtime/device операций конкретной установки.

## 12. Прямые области привилегий

| Область | Текущий путь |
| --- | --- |
| metadata/config definitions | `McpConfigHelper` → args/i18n files |
| instance/runtime state | `ProcessManager.get_manager()` |
| process control | `ProcessManager.start/stop()` |
| config read/write | `AzurLaneConfig.data`, `cross_set`, `save` |
| scheduler mutation | config `Scheduler.Enable` / `Scheduler.NextRun` |
| filesystem logs | `./log/<date>_<instance>.txt` |
| device privacy/read | `Device.screenshot()` |
| emulator lifecycle | `Device.emulator_stop/start()` |
| system ADB | executable resolution + `adb kill-server/start-server` |

## 13. Глобальное и разделяемое состояние

**CURRENT FACT**

Текущий MCP path опирается на несколько уровней состояния:

- process-local registry SSE sessions внутри `SseServerTransport`;
- singleton-like manager registry через `ProcessManager`/`State`;
- persistent config-файлы через `AzurLaneConfig`;
- process environment/`ALAS_CONFIG_NAME` и fake-PIL compatibility context для device paths;
- локальную файловую систему журналов;
- системный ADB server за пределами одного instance.

Эти уровни не образуют единую транзакционную или actor-aware модель.

## 14. ASCII-схема текущей архитектуры

```text
                          embedded mode
PyWebIO WebUI app
    |
    +-- _run_gui() password gate ----- PyWebIO sessions
    |
    `-- Mount /mcp -------------------------------+
                                                   |
                                                   v
                                         MCP Starlette wrapper
                                         CORS: wildcard
                                         auth: отсутствует
                                                   |
                      +----------------------------+--------------------+
                      |                                                 |
              /mcp/sse GET                         /mcp/mcp/messages POST
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
endpoint event -> /mcp/messages?session_id=<uuid>
```

## 15. Целевая интерпретация

**TARGET DECISION**

Ни CORS, ни Tool Annotations, ни одна только transport migration не являются authorization boundary. Будущий MCP должен зависеть от общей политики бэкенда и сервисного слоя:

```text
MCP transport
  -> authentication
  -> authorization / permission
  -> validation / policy / audit
  -> Application / Service Layer
  -> AzurPilot Core
```

Точная реализация Streamable HTTP, OAuth и service packages не проектируется в Этапе 2.
