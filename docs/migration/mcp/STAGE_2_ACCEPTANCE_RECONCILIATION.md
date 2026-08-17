# Сверка приёмки Этапа 2

Этот документ закрывает формальные критерии приёмки Этапа 2, которые физически распределены между несколькими файлами аудита. Он **не заменяет** подробную матрицу capabilities и security findings, а связывает их с исходным prompt и фиксирует доказательство полноты discovery.

## 1. Зафиксированные источники

**CURRENT FACT**

```text
AzurPilot-Web base:
1047f3bd9fc193ba5d69ae87db4d8342710ee3a5

AzurPilot-private-Ru audit source:
3be3696975cb91ba0b85dbea98400381c3ced379
```

Backend в Этапе 2 используется строго только для чтения.

## 2. Доказательство полноты discovery

**CURRENT FACT**

Discovery выполнялся не от заранее заданного списка tools, а от production wiring и import graph на одном pinned backend snapshot.

Проверенная цепочка:

```text
repository root
    |
    +-- mcp_server_sse.py
    |      |
    |      +-- mcp.server.Server
    |      +-- mcp.server.sse.SseServerTransport
    |      +-- @mcp_server.list_tools()
    |      +-- @mcp_server.call_tool()
    |      +-- TOOL_HANDLERS
    |      +-- Starlette / CORS / standalone uvicorn
    |
    +-- module/config/mcp_helper.py
    |      `-- MCP-specific task/dashboard metadata helper
    |
    +-- module/webui/app.py
    |      `-- from mcp_server_sse import app as mcp_app
    |          application.mount("/mcp", mcp_app)
    |
    `-- pyproject.toml
           `-- mcp==1.23.0 / sse-starlette==3.0.3
```

Использованы:

1. recursive tree pinned commit для поиска MCP-related paths;
2. проверка корня репозитория;
3. проверка `module/config/`;
4. проверка `module/webui/`;
5. точечное чтение фактического server construction, registrations, dispatch и embedded mount;
6. трассировка импортов из `mcp_server_sse.py` до runtime/core dependencies.

Результат классификации production MCP path:

| Путь | Роль | В production MCP path |
| --- | --- | --- |
| `mcp_server_sse.py` | владелец current MCP adapter, registrations, dispatch, transport wrapper и standalone entrypoint | да |
| `module/config/mcp_helper.py` | MCP-specific helper метаданных | да |
| `module/webui/app.py` | embedded ASGI wiring/mount | да, как wiring |
| `pyproject.toml` | точные dependency/version constraints | да, как декларация runtime dependencies |
| `ProcessManager`, `AzurLaneConfig`, `Device`, `State`, config/i18n, FS/subprocess helpers | общие backend/runtime dependencies | используются, но MCP ими не владеет |

Отдельного второго production MCP server construction, второго registration catalog или альтернативного production mount в проверенном import/wiring graph **не обнаружено**.

Это утверждение означает «не обнаружено статическим repository-wide аудитом pinned snapshot», а не «в репозитории никогда не может появиться иной MCP path».

## 3. Полнота registrations

**CURRENT FACT**

```text
tools: 17 / 17 классифицировано
resources: 0 / 0
resource templates: 0 / 0
prompts: 0 / 0
completions: 0 / 0
subscriptions: 0 / 0
experimental/extensions registrations: 0 / 0
custom MCP transport path families: 2 / 2
unmapped registrations: 0
```

`list_tools()` и `TOOL_HANDLERS` совпадают по набору всех 17 имён.

## 4. Явное сопоставление имени/URI для каждой capability

У current tools нет индивидуальных HTTP URI. Вызов каждого имени идёт через общий MCP method `tools/call` внутри соответствующего transport path. Поэтому поле `Exposed name/URI` для tools состоит из публичного MCP имени и общей transport-семантики, а не из выдуманного REST endpoint.

| Audit ID | Публичное имя / URI | Тип | Registration | Решение | Доказательство |
| --- | --- | --- | --- | --- | --- |
| MCP-T-001 | `list_instances`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_list_instances` | RETAIN | pinned `mcp_server_sse.py` |
| MCP-T-002 | `get_status`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_status` | RETAIN | pinned `mcp_server_sse.py` |
| MCP-T-003 | `list_tasks`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_list_tasks` | RETAIN | pinned `mcp_server_sse.py` |
| MCP-T-004 | `get_task_help`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_task_help` | RETAIN | pinned `mcp_server_sse.py` + `module/config/mcp_helper.py` |
| MCP-T-005 | `get_resources`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_resources` | RETAIN | pinned `mcp_server_sse.py` + helper |
| MCP-T-006 | `get_config`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_config` | REDESIGN | pinned `mcp_server_sse.py` |
| MCP-T-007 | `update_config`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_update_config` | REDESIGN | pinned `mcp_server_sse.py` + `module/config/config.py` |
| MCP-T-008 | `get_recent_logs`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_recent_logs` | REDESIGN | pinned `mcp_server_sse.py` |
| MCP-T-009 | `start_instance`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_start_instance` | REDESIGN | pinned `mcp_server_sse.py` + `process_manager.py` |
| MCP-T-010 | `stop_instance`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_stop_instance` | REDESIGN | pinned `mcp_server_sse.py` + `process_manager.py` |
| MCP-T-011 | `get_screenshot`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_screenshot` | REDESIGN | pinned `mcp_server_sse.py` + `Device` |
| MCP-T-012 | `get_current_running_task`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_current_running_task` | RETAIN | pinned `mcp_server_sse.py` |
| MCP-T-013 | `get_scheduler_queue`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_get_scheduler_queue` | RETAIN | pinned `mcp_server_sse.py` |
| MCP-T-014 | `trigger_task`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_trigger_task` | REDESIGN | pinned `mcp_server_sse.py` + config persistence |
| MCP-T-015 | `clear_scheduler_queue`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_clear_scheduler_queue` | REDESIGN | pinned `mcp_server_sse.py` + config persistence |
| MCP-T-016 | `restart_emulator`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_restart_emulator` | REDESIGN | pinned `mcp_server_sse.py` + `Device` |
| MCP-T-017 | `restart_adb`; общий `tools/call` | tool | `list_tools` → `TOOL_HANDLERS` → `_tool_restart_adb` | DEFER | pinned `mcp_server_sse.py` + `State`/subprocess |
| MCP-TR-001 | embedded `GET /mcp/sse`; standalone suffix `*/sse` | transport | `mcp_asgi_app` → `SseServerTransport.connect_sse` | REDESIGN | pinned `mcp_server_sse.py` + `module/webui/app.py` |
| MCP-TR-002 | embedded endpoint event `/mcp/mcp/messages?...`; standalone `/mcp/messages?...` | transport | `mcp_asgi_app` → `handle_post_message` | REDESIGN | pinned wiring + exact dependency semantics |

Подробные inputs/outputs, read/write, side effects, dependencies, validation, sensitive data, error/timeout/idempotency/destructiveness/open-world/risk/evidence/confidence находятся в [матрице capabilities](MCP_CAPABILITY_MATRIX.md). Future annotations, permissions, service owners, MVP и confirmation policy находятся в [целевой границе и решениях](MCP_TARGET_BOUNDARY_AND_DECISIONS.md).

## 5. Обязательные измерения capability matrix

Сверка исходного prompt с документацией:

| Требование | Где закрыто |
| --- | --- |
| Audit ID | `MCP_CAPABILITY_MATRIX.md` |
| MCP type | `MCP_CAPABILITY_MATRIX.md` |
| Exposed name / URI | раздел 4 этого документа |
| Registration | matrix + раздел 4 |
| Inputs/schema/defaults | matrix |
| Output | matrix |
| Read/Write | matrix |
| Side effects | matrix |
| Dependencies | matrix |
| Privilege domain | matrix |
| Validation | matrix |
| Auth/Authz | matrix + architecture/security |
| Locality | matrix + architecture/security |
| Sensitive data | matrix |
| Error model | matrix + security |
| Timeout/cancel | matrix + architecture/security |
| Idempotency/retry | matrix + target decisions |
| Destructiveness | matrix + target decisions |
| Open-world reach | matrix + target decisions |
| Risk | matrix |
| Future permission | matrix + target decisions |
| Future service owner | matrix + target decisions |
| Decision | matrix + target decisions |
| Evidence | matrix, exact pinned SHA |
| Confidence | matrix |
| Tool Annotations | `MCP_TARGET_BOUNDARY_AND_DECISIONS.md` |
| Human confirmation | `MCP_TARGET_BOUNDARY_AND_DECISIONS.md` |

Ни одно обязательное измерение исходного prompt после этой сверки не остаётся без места в документации.

## 6. Унифицированная сверка security findings

Полный разбор находится в [MCP_SECURITY_FINDINGS.md](MCP_SECURITY_FINDINGS.md). Таблица ниже обеспечивает обязательные поля исходного prompt для каждого значимого finding.

| ID | Краткое содержание | Доказано | Не доказано | Затронуто | Обоснование severity | Целевое условие |
| --- | --- | --- | --- | --- | --- | --- |
| MCP-SEC-001 | нет общей MCP auth/authz boundary | wrapper/handlers не имеют actor/permission gate; PyWebIO login находится в другом entrypoint | internet reachability конкретной машины | все tools/transport | High при reachability: единая граница открывает R0–R4 | authentication + permission-aware authorization до service dispatch |
| MCP-SEC-002 | wildcard CORS + выключенная Host/Origin validation SDK | `allow_*=["*"]`; `security_settings=None`; standalone bind `0.0.0.0:22268` | браузерная эксплуатация и внешняя достижимость | transport + все tools | High при reachability из-за DNS-rebinding/browser trust surface | Origin/Host/trusted-edge policy + auth |
| MCP-SEC-003 | плоская privilege plane | один `TOOL_HANDLERS` содержит R0/R1 и R3/R4 без permission split | факт компрометации клиента | все 17 tools | High: одна boundary даёт metadata и privileged control | least-privilege scopes + service policy |
| MCP-SEC-004 | raw config read/write обходит безопасную доменную границу | `get_config` отдаёт raw config; `update_config` делает model-controlled `cross_set` | что любой произвольный path эксплуатируем; наличие секрета в каждом config | T-006/T-007, частично scheduler mutations | High: широкая confidentiality/integrity surface | allowlisted `ConfigService`, domain validation/redaction |
| MCP-SEC-005 | runtime/domain ошибки часто возвращаются как успешный MCP result | AzurPilot ловит exceptions и возвращает `TextContent`; schema errors отдельно `isError=true` | конкретное поведение внешнего клиента на таком тексте | все tools, особенно control | Medium-High: клиент не различает failure classes/retry | structured error taxonomy |
| MCP-SEC-006 | control tools могут сообщать ложный/неполный success | start/stop return semantics игнорируются; ADB return codes не проверяются; trigger подтверждает intent, не execution | что ложный success возникает в каждом вызове | T-009/T-010/T-014/T-017 | Medium-High: модель может принять неверное следующее действие/retry | postcondition-aware command results |
| MCP-SEC-007 | нет request timeout/dedup; есть blocking operations | `time.sleep(60)`, blocking subprocess/device/FS; command ID/dedup отсутствуют | фактическая длительность каждого вызова | R3/R4 и тяжёлые reads | High для side-effect retry, Medium для availability | bounded/non-blocking commands + replay semantics |
| MCP-SEC-008 | raw logs: privacy/large-output surface | `lines` без bounds; полный `readlines()`; raw text; общий fallback log | наличие конкретного секрета в логе | T-008/T-012 | Medium: утечка operational data и memory/output pressure | bounded sanitized `LogService` |
| MCP-SEC-009 | traceback выходит в model-facing result | screenshot/emulator handlers добавляют traceback | наличие секрета в конкретном traceback | T-011/T-016 | Medium: раскрытие внутренних paths/device details | server-side diagnostics + sanitized stable errors |
| MCP-SEC-010 | screenshot — privileged privacy read | реальный `Device.screenshot()` → `ImageContent` | наличие конкретных приватных данных в каждом кадре | T-011 | Medium-High privacy: read-only ≠ low privilege | `device:view`, target resolution, audit/privacy policy |
| MCP-SEC-011 | `restart_adb` machine-wide, instance argument не ограничивает scope | общий `adb kill-server/start-server`; instance не scoping operation | влияние на конкретные сторонние процессы в каждом случае | T-017 | High: scope шире выбранного instance и disruptive | отдельная product/security policy; remote exposure не по умолчанию |
| MCP-SEC-012 | нет actor-aware audit и rate limit | adapter не формирует actor/permission/correlation audit; limiter отсутствует | отсутствие всех runtime technical logs | все tools | Medium: слабая attributable control/abuse resistance | structured audit + limits actor/capability/target |
| MCP-SEC-013 | JSON Schema не заменяет semantic validation | schemas в основном primitive; нет target/task/path/precondition/permission checks | что все domain paths некорректны | большинство tools, особенно T-006/T-007/T-014 | Medium-High для privileged operations | service-level semantic validation/preconditions |
| MCP-SEC-014 | legacy process-local session transport — migration debt | UUID→memory stream registry, SSE+POST, state теряется при restart | необходимость сохранять legacy совместимость | transport | Medium architecture/operations: hidden process state мешает stateless target | migration после auth/policy/service boundary |

### Impact if remotely reachable

Для findings, зависящих от сетевой достижимости, impact оценивается **условно**: если клиент способен достичь MCP transport. Статический аудит не утверждает, что домашний ПК доступен из Интернета.

## 7. Сверка с официальным MCP target baseline

**OFFICIAL MCP TARGET GUIDANCE**

По официальным источникам на момент финальной сверки Этапа 2:

- опубликованная ревизия MCP `2026-07-28` использует stateless protocol core;
- legacy HTTP+SSE официально deprecated с compatibility off-ramp;
- официальный Python SDK v2 является текущей stable line, v1.x — maintenance line;
- Streamable HTTP является стандартным HTTP transport современной линии SDK/spec;
- для HTTP transport необходима отдельная security policy: Origin/DNS-rebinding protection и authentication;
- Tool Annotations (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`) — hints и не заменяют authorization/policy/validation;
- современная authorization model рассматривается отдельно от текущего legacy AzurPilot adapter.

Источники:

- <https://blog.modelcontextprotocol.io/posts/2026-07-28/>
- <https://modelcontextprotocol.io/specification/2026-07-28>
- <https://github.com/modelcontextprotocol/python-sdk>
- <https://github.com/modelcontextprotocol/python-sdk/blob/main/SECURITY.md>

Эти пункты являются target guidance и не выдаются за уже реализованные свойства AzurPilot.

## 8. Формальная сверка Definition of Done

| Критерий | Статус | Доказательство / комментарий |
| --- | --- | --- |
| Issue создан и связан | выполнено | Issue #5, PR #6 |
| отдельная ветка от актуального `main` | выполнено | `docs/stage-2-mcp-audit` от `1047f3bd...` |
| Draft PR открыт после содержательного commit | выполнено | PR #6 |
| backend не изменён | выполнено | backend только read-only; финальный drift check выполняется перед Ready |
| exact backend SHA | выполнено | `3be36969...` во всех audit docs |
| все MCP-related production files обнаружены | выполнено | раздел 2 |
| embedded mount/transport доказан | выполнено | `CURRENT_MCP_ARCHITECTURE.md` |
| standalone mode доказан | выполнено | `CURRENT_MCP_ARCHITECTURE.md` |
| SDK/version constraints | выполнено | `mcp==1.23.0`, dependency evidence |
| 100% tools | выполнено | 17/17 |
| resources/templates/prompts | выполнено | 0, отсутствие registrations доказано |
| прочие registrations | выполнено | transport 2/2, experimental 0 |
| unmapped registrations | выполнено | 0 |
| pinned evidence для capability | выполнено | matrix + exact SHA permalinks |
| read/write и side effects | выполнено | 17/17 matrix |
| direct dependencies | выполнено | 17/17 matrix |
| risk | выполнено | R0=2, R1=7, R2=0, R3=7, R4=1 |
| target annotations | выполнено | target decisions 17/17 |
| future permissions | выполнено | 17/17 |
| service owner RETAIN/REDESIGN | выполнено | 16/16; DEFER имеет условный owner |
| migration decision | выполнено | RETAIN=7, REDESIGN=9, REMOVE=0, DEFER=1 |
| read-only MVP | выполнено | 7 обязательных + 1 условный redesigned logs candidate |
| human confirmation policy | выполнено | target decisions |
| security ↔ official MCP baseline | выполнено | security docs + раздел 7 |
| CURRENT/TARGET не смешаны | выполнено | пять явных статусов документации |
| internet reachability не выдумана | выполнено | условная формулировка во всех network findings |
| production/backend/frontend implementation changes отсутствуют | выполняется финальным diff gate | только Markdown ожидается |
| секреты отсутствуют | выполняется финальным secret-pattern/diff gate | реальные credentials/paths/log payloads не документировались |
| `docs/migration/README.md` обновлён | выполнено | индекс Этапа 2 |
| internal links | выполняется финальным gate | проверяются после последней правки |
| counts docs ↔ PR | выполняется финальным gate | сверяется после последней правки |
| backend drift | выполняется непосредственно перед Ready | повторный SHA обязателен |
| PR body отражает реальные checks/ограничения | выполняется после финального gate | старый промежуточный body должен быть заменён |
| Ready только после DoD | ожидает финальный gate | PR остаётся Draft до проверки |
| merge без команды владельца | выполнено | merge не выполнялся |

## 9. Что остаётся UNVERIFIED / DEFERRED

**UNVERIFIED**

- реальная internet/LAN reachability конкретной установки;
- router/NAT/firewall/OS ACL/Caddy wiring конкретной машины;
- фактическое содержимое приватных config/log/screenshot данных — намеренно не читалось и не переносилось в документацию;
- runtime timing конкретных device/process операций, поскольку destructive/runtime smoke не требуется для docs-only Этапа 2.

**DEFERRED**

- должен ли `restart_adb` вообще оставаться remotely exposable product capability;
- точная будущая реализация Streamable HTTP/auth;
- Tasks extension против bounded synchronous/asynchronous command contract;
- окончательные package/class names Service Layer;
- compatibility off-ramp для legacy MCP clients, если он действительно потребуется.

Эти пункты не блокируют статический migration-boundary audit: для них уже определены owner/policy prerequisites и не требуется предполагать runtime behavior.
