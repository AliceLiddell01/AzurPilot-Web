# Целевая граница MCP и миграционные решения

Источник current state: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

Этот документ не является реализацией будущего API/MCP. Он связывает обнаруженные capabilities с минимальной целевой backend boundary.

## 1. Принцип целевой архитектуры

**TARGET DECISION**

```text
НЕПРАВИЛЬНО
MCP -> ProcessManager / AzurLaneConfig / Device / filesystem / subprocess

ПРАВИЛЬНО
MCP transport
  -> authentication
  -> authorization / permission
  -> validation / policy / audit
  -> Application / Service Layer
  -> AzurPilot Core

HTTP adapter
  -> тот же Application / Service Layer
```

React и MCP не должны владеть независимыми копиями бизнес-правил.

## 2. Distribution решений

```text
RETAIN:   7
REDESIGN: 9
REMOVE:   0
DEFER:    1
Total:   17
```

`RETAIN` означает сохранить продуктовую capability; direct-call implementation всё равно должен исчезнуть за service boundary.

## 3. Future service boundaries

| Service boundary | Capabilities | Ответственность |
| --- | --- | --- |
| `InstanceService` | MCP-T-001, T-002, T-009, T-010 | list/status/start/stop instance с target resolution и lifecycle policy |
| `TaskService` | MCP-T-003, T-004, T-012 | task metadata/help/current execution read model |
| `StatisticsService` | MCP-T-005 | безопасная operational resource projection |
| `ConfigService` | MCP-T-006, T-007 | allowlisted read/write, type/domain validation, callbacks/policy |
| `LogService` | MCP-T-008 | bounded sanitized tail/search/read model |
| `DeviceService` | MCP-T-011, T-016 | device view и emulator lifecycle policy |
| `SchedulerService` | MCP-T-013, T-014, T-015 | queue read, trigger, clear/cancel policy |
| `SystemService` | MCP-T-017, только если product decision положительное | machine-wide ADB/system operation |

Имена являются архитектурным направлением, а не обязательными Python class/package names.

## 4. Permission mapping, Tool Annotations и confirmation policy

Tool Annotations ниже — **TARGET DECISION**, не current metadata. Current tools annotations не имеют.

`not applicable` используется там, где MCP spec считает hint незначимым для `readOnlyHint=true`.

| ID | Tool | Permission | Owner | readOnlyHint | destructiveHint | idempotentHint | openWorldHint | Decision | Read-only MVP | Human confirmation |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MCP-T-001 | `list_instances` | `instances:read` | InstanceService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-002 | `get_status` | `status:read` | InstanceService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-003 | `list_tasks` | `tasks:read` | TaskService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-004 | `get_task_help` | `tasks:read` | TaskService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-005 | `get_resources` | `resources:read` | StatisticsService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-006 | `get_config` | `config:read` + field policy | ConfigService | true | not applicable | not applicable | false | REDESIGN | нет: raw config слишком широк | confirmation not required после permission/redaction |
| MCP-T-007 | `update_config` | `config:write` + field/domain policy | ConfigService | false | conditional | conditional | conditional | REDESIGN | нет | **confirmation required** |
| MCP-T-008 | `get_recent_logs` | `logs:read` | LogService | true | not applicable | not applicable | false | REDESIGN | **да, после bounds + sanitization** | confirmation not required |
| MCP-T-009 | `start_instance` | `instances:control` | InstanceService | false | false | conditional | true | REDESIGN | нет | **confirmation required** |
| MCP-T-010 | `stop_instance` | `instances:control` | InstanceService | false | true | conditional | false | REDESIGN | нет | **confirmation required** |
| MCP-T-011 | `get_screenshot` | `device:view` | DeviceService | true | not applicable | not applicable | false | REDESIGN | нет | **confirmation recommended** |
| MCP-T-012 | `get_current_running_task` | `tasks:read` | TaskService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-013 | `get_scheduler_queue` | `scheduler:read` | SchedulerService | true | not applicable | not applicable | false | RETAIN | **да** | confirmation not required |
| MCP-T-014 | `trigger_task` | `scheduler:control` | SchedulerService | false | conditional | false | true | REDESIGN | нет | **confirmation required** |
| MCP-T-015 | `clear_scheduler_queue` | `scheduler:control` | SchedulerService | false | true | true | false | REDESIGN | нет | **confirmation required** |
| MCP-T-016 | `restart_emulator` | `device:control` | DeviceService | false | true | false | false | REDESIGN | нет | **confirmation required** |
| MCP-T-017 | `restart_adb` | `system:adb-control` | SystemService при сохранении | false | true | false | false | **DEFER** | нет | **not remotely exposable without separate policy** |

### Почему annotations не совпадают с risk class

`readOnlyHint=true` у screenshot не делает его R0/R1: device image — privacy-sensitive privileged read. Аналогично `destructiveHint=false` у start не означает, что операция безопасна без confirmation: она запускает automation и имеет R3.

**OFFICIAL MCP TARGET GUIDANCE:** annotations являются hints и не заменяют access control.

## 5. Read-only MCP MVP candidate

**TARGET DECISION**

Минимальный безопасный product set после появления service/read boundaries:

```text
MCP-T-001 list_instances
MCP-T-002 get_status
MCP-T-003 list_tasks
MCP-T-004 get_task_help
MCP-T-005 get_resources
MCP-T-012 get_current_running_task
MCP-T-013 get_scheduler_queue
```

Условный восьмой кандидат:

```text
MCP-T-008 get_recent_logs
```

только после:

- bounded `lines`/byte limit;
- server-side redaction/sanitization;
- запрета raw arbitrary filesystem paths;
- стабильной result schema;
- `logs:read` permission;
- rate limit.

### Что намеренно не входит в MVP

- `get_config` — current raw config surface слишком широка;
- `get_screenshot` — privacy/device privilege;
- все mutations;
- `restart_adb` — machine-wide scope.

## 6. RETAIN decisions

### MCP-T-001 / T-002 — instance discovery/status

Сохранить product capability. Target implementation использует `InstanceService` read model, а не `ProcessManager` напрямую из adapter.

### MCP-T-003 / T-004 / T-012 — task metadata/current task

Сохранить. `get_current_running_task` target не должен зависеть от full-log regex parsing; текущая выполняемая task должна быть частью runtime read model.

### MCP-T-005 — resources

Сохранить как узкую read projection. Не отдавать raw config вместо resources model.

### MCP-T-013 — scheduler queue

Сохранить как `SchedulerService` read model с predictable typed result.

## 7. REDESIGN decisions

### MCP-T-006 — `get_config`

Сохранение цели «получить нужные настройки» допустимо, но не raw whole-config dump. Нужны field groups, redaction и permission policy.

### MCP-T-007 — `update_config`

Должен выражать validated config intent через `ConfigService`, включая текущие domain validation/save callback semantics. Model-controlled arbitrary `task.group.arg` path не является target authorization unit.

### MCP-T-008 — logs

Нужен `LogService` вместо прямого файла. Результат должен быть bounded, sanitized и структурированным.

### MCP-T-009 / T-010 — start/stop

Операции остаются продуктово полезными, но требуют lifecycle preconditions, permission, audit, postcondition result и replay semantics.

### MCP-T-011 — screenshot

Оставить возможность только за `device:view`, target/device resolution и privacy policy. Не включать в минимальный read-only set автоматически.

### MCP-T-014 / T-015 — scheduler control

Нужны command semantics через `SchedulerService`, task validation, confirmation и audit. `clear` должен иметь понятный transactional/partial-failure contract.

### MCP-T-016 — emulator restart

Нужен `DeviceService` command, preconditions и non-blocking long-running semantics. `time.sleep(60)` в transport handler переносить нельзя.

## 8. DEFER decision

### MCP-T-017 — `restart_adb`

Причина DEFER конкретна:

- current operation machine-wide, а не instance-scoped;
- ADB server может обслуживать несколько devices/consumers;
- current `instance` argument не ограничивает target;
- operation может нарушить соединения вне выбранного instance;
- нужен отдельный product/security ответ, должен ли LLM вообще получать machine-wide ADB lifecycle control.

Если capability будет принята позже, owner — `SystemService`, permission — отдельный `system:adb-control`, а не общий `device:control`.

## 9. Human confirmation policy

**confirmation not required** после успешной authz/read-policy:

- metadata/status/resources/scheduler reads;
- bounded sanitized logs.

**confirmation recommended**:

- screenshot, даже при `device:view`, из-за privacy.

**confirmation required**:

- config write;
- start/stop instance;
- trigger/clear scheduler;
- restart emulator.

**not remotely exposable without separate policy**:

- machine-wide ADB restart.

Эта policy может позже поддерживаться client confirmation, MRTR/elicitation или продуктовой pre-authorization моделью; Этап 2 не выбирает wire implementation.

## 10. Transport migration prerequisites

**OFFICIAL MCP TARGET GUIDANCE**

`2026-07-28` — stateless protocol core; legacy HTTP+SSE deprecated. Актуальная stable line официального Python SDK — v2.

**TARGET DECISION**

До production transport migration должны быть определены:

1. authentication и authorization scopes;
2. trusted public-edge / Origin / Host policy;
3. service layer и removal direct runtime calls из MCP adapter;
4. typed success/error semantics;
5. audit event model;
6. rate limits;
7. bounded output policy;
8. replay/idempotency semantics для mutations;
9. long-running command policy;
10. legacy client compatibility/off-ramp, если он реально нужен.

Не следует сначала заменить SSE на Streamable HTTP, оставив прежнюю плоскую privilege plane.

## 11. Long-running operations

### Текущие кандидаты

- emulator restart;
- process stop/start при медленном lifecycle;
- screenshot/device operations при проблемном device;
- system ADB restart.

**TARGET prerequisite:** command result должен различать accepted/running/completed/failed либо эквивалентную безопасную lifecycle модель, если операция реально асинхронная.

Современная MCP Tasks extension является доступным протокольным механизмом для долгих операций, но выбор Tasks vs обычный bounded command **DEFERRED** до service/transport design. Этап 2 не создаёт job queue.

## 12. Error model target

Будущие services/adapters должны различать как минимум:

```text
VALIDATION_ERROR
NOT_FOUND
FORBIDDEN
CONFLICT / PRECONDITION_FAILED
RETRYABLE_RUNTIME_ERROR
TIMEOUT
PARTIAL_FAILURE
INTERNAL_ERROR
```

Текстовое `Error: ...` внутри успешного tool result не является достаточным contract для privileged automation.

Traceback и абсолютные user paths должны оставаться в server-side diagnostics/audit, а не в model-facing result.

## 13. Rate-limit/audit baseline

Минимальная будущая policy:

- read metadata/status — мягкий rate limit;
- logs/screenshot — отдельный output/call limit;
- config/scheduler/process/device control — строгий actor + target audit;
- system operations — самый строгий gate;
- audit должен фиксировать actor, permission, tool/capability, resolved target, sanitized input summary, result class, correlation/command ID и time.

Точная storage/retention schema откладывается.

## 14. Итог

Stage 2 не предлагает «переписать 17 tools один к одному». Он разделяет текущий каталог на product capabilities и будущие backend boundaries:

```text
17 current tools
  -> 8 общих service boundaries
  -> 7 RETAIN
  -> 9 REDESIGN
  -> 1 DEFER
  -> 0 REMOVE
  -> 7 обязательных + 1 условный read-only MVP candidate
```

Следующий backend design stage теперь может проектировать общий layer по возможностям, а не по названиям legacy MCP handlers.
