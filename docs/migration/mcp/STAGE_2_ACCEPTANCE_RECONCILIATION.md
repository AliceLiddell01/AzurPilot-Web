# Сверка приёмки Этапа 2

Этот документ связывает исходные критерии Этапа 2 с фактическими артефактами аудита и фиксирует полноту discovery. Он не заменяет подробные документы, а служит контрольной картой перед переводом PR в Ready.

## 1. Зафиксированные источники

**CURRENT FACT**

```text
AzurPilot-Web base:
1047f3bd9fc193ba5d69ae87db4d8342710ee3a5

AzurPilot-private-Ru audit source:
3be3696975cb91ba0b85dbea98400381c3ced379
```

Backend в Этапе 2 использовался строго только для чтения.

## 2. Доказательство полноты discovery

**CURRENT FACT**

Аудит выполнялся от production wiring и import graph, а не от заранее придуманного списка tools.

Проверенная цепочка:

```text
repository root
    |
    +-- mcp_server_sse.py
    |      +-- mcp.server.Server
    |      +-- mcp.server.sse.SseServerTransport
    |      +-- @mcp_server.list_tools()
    |      +-- @mcp_server.call_tool()
    |      +-- TOOL_HANDLERS
    |      +-- Starlette / CORS / standalone Uvicorn
    |
    +-- module/config/mcp_helper.py
    |      `-- MCP-specific metadata helper
    |
    +-- module/webui/app.py
    |      `-- import mcp_app -> Mount /mcp
    |
    `-- pyproject.toml
           `-- mcp==1.23.0 / sse-starlette==3.0.3 / starlette==0.49.1
```

Discovery включал recursive tree pinned commit, корень репозитория, `module/config/`, `module/webui/`, точечное чтение server construction/registrations/dispatch/mount и трассировку импортов до runtime/core dependencies.

| Путь | Роль | В production MCP path |
| --- | --- | --- |
| `mcp_server_sse.py` | current MCP adapter, registrations, dispatch, transport wrapper, standalone entry point | да |
| `module/config/mcp_helper.py` | MCP-specific helper метаданных | да |
| `module/webui/app.py` | embedded ASGI wiring/mount | да, как wiring |
| `pyproject.toml` | точные dependency/version constraints | да, как декларация зависимостей |
| `ProcessManager`, `AzurLaneConfig`, `Device`, `State`, config/i18n, FS/subprocess | общие backend/runtime dependencies | используются, но MCP ими не владеет |

Отдельного второго production MCP server construction, второго registration catalog или альтернативного production mount в проверенном graph **не обнаружено**.

Граница доказательства: это результат статического repository-wide аудита конкретного pinned snapshot, а не утверждение о будущих версиях репозитория.

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
несопоставленных registrations: 0
```

`list_tools()` и `TOOL_HANDLERS` совпадают по набору всех 17 имён.

## 4. Публичное имя и transport-семантика

У current tools нет индивидуальных HTTP URI. Каждый tool вызывается по публичному MCP имени через общий method `tools/call` внутри transport family.

| Audit ID | Публичное имя / URI | Решение |
| --- | --- | --- |
| MCP-T-001 | `list_instances`; общий `tools/call` | RETAIN |
| MCP-T-002 | `get_status`; общий `tools/call` | RETAIN |
| MCP-T-003 | `list_tasks`; общий `tools/call` | RETAIN |
| MCP-T-004 | `get_task_help`; общий `tools/call` | RETAIN |
| MCP-T-005 | `get_resources`; общий `tools/call` | RETAIN |
| MCP-T-006 | `get_config`; общий `tools/call` | REDESIGN |
| MCP-T-007 | `update_config`; общий `tools/call` | REDESIGN |
| MCP-T-008 | `get_recent_logs`; общий `tools/call` | REDESIGN |
| MCP-T-009 | `start_instance`; общий `tools/call` | REDESIGN |
| MCP-T-010 | `stop_instance`; общий `tools/call` | REDESIGN |
| MCP-T-011 | `get_screenshot`; общий `tools/call` | REDESIGN |
| MCP-T-012 | `get_current_running_task`; общий `tools/call` | RETAIN |
| MCP-T-013 | `get_scheduler_queue`; общий `tools/call` | RETAIN |
| MCP-T-014 | `trigger_task`; общий `tools/call` | REDESIGN |
| MCP-T-015 | `clear_scheduler_queue`; общий `tools/call` | REDESIGN |
| MCP-T-016 | `restart_emulator`; общий `tools/call` | REDESIGN |
| MCP-T-017 | `restart_adb`; общий `tools/call` | DEFER |
| MCP-TR-001 | embedded `GET /mcp/sse`; standalone suffix `*/sse` | REDESIGN transport |
| MCP-TR-002 | embedded `/mcp/mcp/messages?...`; standalone `/mcp/messages?...` | REDESIGN transport |

Подробные registrations, inputs/outputs, read/write, side effects, dependencies, validation, sensitive data, errors, timeout/cancel, idempotency, destructiveness, open-world reach, risk, permission, owner, decision, evidence и confidence находятся в [матрице capabilities](MCP_CAPABILITY_MATRIX.md).

## 5. Сверка обязательных измерений capability

| Требование | Артефакт |
| --- | --- |
| Audit ID, тип MCP, публичное имя/URI | `MCP_CAPABILITY_MATRIX.md` |
| Registration, inputs/defaults, output | `MCP_CAPABILITY_MATRIX.md` |
| Read/write, side effects, dependencies | `MCP_CAPABILITY_MATRIX.md` |
| Privilege domain, validation, auth/authz, locality | `MCP_CAPABILITY_MATRIX.md` + architecture/security |
| Sensitive data, error model, timeout/cancel | `MCP_CAPABILITY_MATRIX.md` + security |
| Idempotency, destructiveness, open-world reach | matrix + target decisions |
| Risk | matrix |
| Future permission | matrix + target decisions |
| Future service owner | matrix + target decisions |
| RETAIN/REDESIGN/REMOVE/DEFER | matrix + target decisions |
| Pinned evidence и confidence | matrix |
| `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint` | target decisions |
| Read-only MVP | target decisions |
| Human confirmation policy | target decisions |

Ни одно обязательное измерение исходного задания не остаётся без места в документации.

## 6. Сверка значимых выводов безопасности

Полный разбор находится в [выводах по безопасности](MCP_SECURITY_FINDINGS.md). Каждый MCP-SEC-001…014 теперь явно содержит:

```text
ID;
краткое содержание;
CURRENT FACT и доказательство;
влияние;
что доказано;
что не доказано;
затронутые возможности;
обоснование severity;
целевое условие.
```

Ключевые группы:

- отсутствие общей MCP auth/authz boundary;
- wildcard CORS и выключенная `Host`/`Origin` validation current SDK transport;
- плоская privilege plane R0–R4;
- raw config read/write;
- неоднозначная error/success semantics;
- blocking/long-running paths без request timeout/dedup;
- log/screenshot privacy и output surface;
- machine-wide `restart_adb`;
- отсутствие actor-aware audit/rate limit;
- semantic validation gaps;
- legacy process-local SSE session debt.

Для network-dependent findings влияние сформулировано условно: **при достижимости MCP клиентом**. Internet/LAN reachability конкретной установки не утверждается.

## 7. Официальный MCP target baseline

**OFFICIAL MCP TARGET GUIDANCE**

На момент финальной сверки:

- опубликованная ревизия MCP — `2026-07-28`;
- protocol core является stateless;
- legacy HTTP+SSE deprecated;
- официальный Python SDK v2 — текущая stable line, v1.x — maintenance;
- Streamable HTTP — современный HTTP transport;
- HTTP transport требует отдельной security policy, включая Origin/DNS-rebinding protection и authentication;
- Tool Annotations являются hints и не заменяют authorization/policy/validation.

Официальные источники:

- <https://blog.modelcontextprotocol.io/posts/2026-07-28/>
- <https://modelcontextprotocol.io/specification/2026-07-28>
- <https://github.com/modelcontextprotocol/python-sdk>
- <https://github.com/modelcontextprotocol/python-sdk/blob/main/SECURITY.md>

Эти пункты не выдаются за current behavior AzurPilot.

## 8. Формальная сверка Definition of Done

| Критерий | Статус |
| --- | --- |
| Issue создан и связан | выполнено: #5 |
| отдельная ветка от актуального `main` | выполнено: `docs/stage-2-mcp-audit` от `1047f3bd...` |
| Draft PR открыт после содержательного commit | выполнено: #6 |
| backend не изменён | выполнено |
| exact backend SHA | выполнено: `3be36969...` |
| все MCP-related production files обнаружены | выполнено |
| embedded mount/transport | выполнено |
| standalone mode | выполнено |
| SDK/version constraints | выполнено |
| tools/resources/templates/prompts/прочие registrations | выполнено |
| unmapped registrations | 0 |
| pinned evidence | выполнено |
| read/write, side effects, dependencies | 17/17 |
| risk classification | R0=2, R1=7, R2=0, R3=7, R4=1 |
| target annotations | 17/17 |
| future permissions | 17/17 |
| service owners | все RETAIN/REDESIGN; DEFER имеет условный owner |
| migration decisions | RETAIN=7, REDESIGN=9, REMOVE=0, DEFER=1 |
| read-only MVP | 7 обязательных + 1 условный после redesign |
| human confirmation policy | выполнено |
| security ↔ official MCP baseline | выполнено |
| CURRENT/TARGET terminology | разведено явно |
| недоказанная internet reachability | не приписывается |
| production/backend/frontend implementation changes | отсутствуют; diff docs-only |
| `docs/migration/README.md` | обновлён |
| внутренние ссылки | проверены по существующим файлам ветки |
| backend drift | отсутствует на финальном pre-Ready check |
| merge без команды владельца | не выполнялся |

## 9. Проверки и ограничения среды

Фактически выполнены:

- GitHub base/head comparison;
- changed-files scope review;
- PR diff review;
- Markdown structure review;
- проверка внутренних ссылок по существующим файлам ветки;
- проверка representative pinned permalinks exact backend SHA;
- reconciliation 17 `list_tools()` ↔ 17 `TOOL_HANDLERS`;
- risk/decision/permission/service-owner coverage;
- cross-check security findings;
- CURRENT/TARGET terminology review;
- ручной pattern/diff audit на секреты;
- backend drift check;
- GitHub reviews/threads/comments/checks/workflow state.

Локальные `markdownlint`, Gitleaks и detect-secrets в среде отсутствуют. Локальный checkout недоступен: Git не может разрешить `github.com`, поэтому `git diff --check` не запускался. Успешный результат этих инструментов не заявляется; вместо этого использованы GitHub compare/PR diff и ручные статические проверки.

Status/check runs и PR-triggered workflow runs отсутствуют; это не выдаётся за «зелёный CI».

CodeRabbit не запускался.

Runtime/MuMu smoke не выполнялся: исходное задание для docs-only Этапа 2 его не требует.

## 10. UNVERIFIED и DEFERRED

**UNVERIFIED**

- реальная Internet/LAN reachability конкретной установки;
- router/NAT/firewall/OS ACL/Caddy конкретной машины;
- фактическое содержимое приватных config/log/screenshot данных — намеренно не читалось и не переносилось;
- wall-clock timing device/process operations без runtime smoke.

**DEFERRED**

- должен ли `restart_adb` оставаться remotely exposable product capability;
- точная реализация Streamable HTTP/auth;
- MCP Tasks extension против иной bounded command model;
- окончательные package/class names Service Layer;
- legacy-client compatibility off-ramp, если он реально потребуется.

Эти пункты не блокируют статический migration-boundary audit: для них определены границы и будущие prerequisites без выдумывания runtime behavior.
