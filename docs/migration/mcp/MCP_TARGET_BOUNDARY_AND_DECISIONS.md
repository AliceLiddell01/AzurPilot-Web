# Целевая граница MCP и миграционные решения

Источник текущего состояния: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

Этот документ не является реализацией будущего API/MCP. Он связывает обнаруженные возможности с минимальной целевой границей бэкенда.

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

## 2. Распределение решений

```text
RETAIN:   7
REDESIGN: 9
REMOVE:   0
DEFER:    1
Всего:   17
```

`RETAIN` означает сохранить продуктовую возможность; реализация с прямыми вызовами всё равно должна исчезнуть за границей сервисного слоя.

## 3. Будущие сервисные границы

| Сервисная граница | Возможности | Ответственность |
| --- | --- | --- |
| `InstanceService` | MCP-T-001, T-002, T-009, T-010 | список/состояние/запуск/остановка instance с разрешением цели и политикой жизненного цикла |
| `TaskService` | MCP-T-003, T-004, T-012 | метаданные задач, справка и модель чтения текущего выполнения |
| `StatisticsService` | MCP-T-005 | безопасная проекция operational resources |
| `ConfigService` | MCP-T-006, T-007 | разрешённые чтение/запись, типовая и доменная валидация, callbacks/policy |
| `LogService` | MCP-T-008 | ограниченная и санитизированная модель tail/search/read |
| `DeviceService` | MCP-T-011, T-016 | просмотр устройства и политика жизненного цикла emulator |
| `SchedulerService` | MCP-T-013, T-014, T-015 | чтение очереди, запуск, очистка/отмена и соответствующая политика |
| `SystemService` | MCP-T-017, только при положительном продуктовом решении | machine-wide ADB/system operation |

Имена являются архитектурным направлением, а не обязательными именами Python classes/packages.

## 4. Разрешения, Tool Annotations и политика подтверждения

Tool Annotations ниже — **TARGET DECISION**, а не текущие метаданные. Текущие tools annotations не имеют.

`not applicable` используется там, где MCP specification считает подсказку несущественной при `readOnlyHint=true`.

| ID | Tool | Разрешение | Владелец | readOnlyHint | destructiveHint | idempotentHint | openWorldHint | Решение | Read-only MVP | Подтверждение человеком |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| MCP-T-001 | `list_instances` | `instances:read` | InstanceService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-002 | `get_status` | `status:read` | InstanceService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-003 | `list_tasks` | `tasks:read` | TaskService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-004 | `get_task_help` | `tasks:read` | TaskService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-005 | `get_resources` | `resources:read` | StatisticsService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-006 | `get_config` | `config:read` + field policy | ConfigService | true | not applicable | not applicable | false | REDESIGN | нет: raw config слишком широк | не требуется после permission/redaction |
| MCP-T-007 | `update_config` | `config:write` + field/domain policy | ConfigService | false | conditional | conditional | conditional | REDESIGN | нет | **обязательно** |
| MCP-T-008 | `get_recent_logs` | `logs:read` | LogService | true | not applicable | not applicable | false | REDESIGN | **да, после bounds + sanitization** | не требуется |
| MCP-T-009 | `start_instance` | `instances:control` | InstanceService | false | false | conditional | true | REDESIGN | нет | **обязательно** |
| MCP-T-010 | `stop_instance` | `instances:control` | InstanceService | false | true | conditional | false | REDESIGN | нет | **обязательно** |
| MCP-T-011 | `get_screenshot` | `device:view` | DeviceService | true | not applicable | not applicable | false | REDESIGN | нет | **рекомендуется** |
| MCP-T-012 | `get_current_running_task` | `tasks:read` | TaskService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-013 | `get_scheduler_queue` | `scheduler:read` | SchedulerService | true | not applicable | not applicable | false | RETAIN | **да** | не требуется |
| MCP-T-014 | `trigger_task` | `scheduler:control` | SchedulerService | false | conditional | false | true | REDESIGN | нет | **обязательно** |
| MCP-T-015 | `clear_scheduler_queue` | `scheduler:control` | SchedulerService | false | true | true | false | REDESIGN | нет | **обязательно** |
| MCP-T-016 | `restart_emulator` | `device:control` | DeviceService | false | true | false | false | REDESIGN | нет | **обязательно** |
| MCP-T-017 | `restart_adb` | `system:adb-control` | SystemService при сохранении | false | true | false | false | **DEFER** | нет | **не публиковать удалённо без отдельной политики** |

### Почему annotations не совпадают с классом риска

`readOnlyHint=true` у screenshot не делает его R0/R1: изображение устройства — privacy-sensitive privileged read. Аналогично `destructiveHint=false` у start не означает, что операция безопасна без подтверждения: она запускает automation и имеет R3.

**OFFICIAL MCP TARGET GUIDANCE:** annotations являются hints и не заменяют контроль доступа.

## 5. Кандидат read-only MCP MVP

**TARGET DECISION**

Минимальный безопасный продуктовый набор после появления сервисных границ и моделей чтения:

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

- ограничения `lines`/объёма в байтах;
- server-side redaction/sanitization;
- запрета произвольных путей файловой системы;
- стабильной схемы результата;
- разрешения `logs:read`;
- rate limit.

### Что намеренно не входит в MVP

- `get_config` — текущая raw config surface слишком широка;
- `get_screenshot` — privacy/device privilege;
- все изменения состояния;
- `restart_adb` — machine-wide scope.

## 6. Решения RETAIN

### MCP-T-001 / T-002 — обнаружение instance и состояние

Сохранить продуктовую возможность. Целевая реализация использует модель чтения `InstanceService`, а не прямой вызов `ProcessManager` из adapter.

### MCP-T-003 / T-004 / T-012 — метаданные задач и текущая задача

Сохранить. Целевая реализация `get_current_running_task` не должна зависеть от regex-парсинга полного журнала; текущая выполняемая задача должна быть частью runtime read model.

### MCP-T-005 — ресурсы

Сохранить как узкую проекцию чтения. Не отдавать raw config вместо модели ресурсов.

### MCP-T-013 — очередь планировщика

Сохранить как модель чтения `SchedulerService` с предсказуемым типизированным результатом.

## 7. Решения REDESIGN

### MCP-T-006 — `get_config`

Сохранение цели «получить нужные настройки» допустимо, но не raw whole-config dump. Нужны группы полей, redaction и permission policy.

### MCP-T-007 — `update_config`

Должен выражать валидированное намерение изменения конфигурации через `ConfigService`, включая текущую доменную валидацию и `save callback` semantics. Управляемый моделью произвольный `task.group.arg` path не является целевой единицей авторизации.

### MCP-T-008 — журналы

Нужен `LogService` вместо прямого чтения файла. Результат должен быть ограниченным, санитизированным и структурированным.

### MCP-T-009 / T-010 — запуск/остановка

Операции остаются продуктово полезными, но требуют lifecycle preconditions, permission, audit, postcondition result и replay semantics.

### MCP-T-011 — screenshot

Оставить возможность только за `device:view`, разрешением target/device и privacy policy. Не включать автоматически в минимальный read-only набор.

### MCP-T-014 / T-015 — управление планировщиком

Нужны command semantics через `SchedulerService`, валидация task, подтверждение и audit. `clear` должен иметь понятный transactional/partial-failure contract.

### MCP-T-016 — перезапуск emulator

Нужна команда `DeviceService`, preconditions и неблокирующая модель долгой операции. `time.sleep(60)` в transport handler переносить нельзя.

## 8. Решение DEFER

### MCP-T-017 — `restart_adb`

Причина DEFER конкретна:

- текущая операция machine-wide, а не instance-scoped;
- ADB server может обслуживать несколько devices/consumers;
- текущий аргумент `instance` не ограничивает target;
- операция может нарушить соединения вне выбранного instance;
- нужен отдельный продуктовый/security ответ, должен ли LLM вообще получать machine-wide ADB lifecycle control.

Если возможность будет принята позже, владелец — `SystemService`, разрешение — отдельное `system:adb-control`, а не общий `device:control`.

## 9. Политика подтверждения человеком

**Подтверждение не требуется** после успешной authz/read-policy:

- metadata/status/resources/scheduler reads;
- bounded sanitized logs.

**Подтверждение рекомендуется**:

- screenshot, даже при `device:view`, из-за privacy.

**Подтверждение обязательно**:

- изменение конфигурации;
- запуск/остановка instance;
- trigger/clear scheduler;
- перезапуск emulator.

**Не публиковать удалённо без отдельной политики**:

- machine-wide ADB restart.

Эта политика позже может поддерживаться client confirmation, MRTR/elicitation или продуктовой pre-authorization моделью; Этап 2 не выбирает wire implementation.

## 10. Обязательные условия миграции transport

**OFFICIAL MCP TARGET GUIDANCE**

`2026-07-28` — stateless protocol core; legacy HTTP+SSE deprecated. Актуальная stable line официального Python SDK — v2.

**TARGET DECISION**

До production transport migration должны быть определены:

1. authentication и authorization scopes;
2. политика trusted public-edge / `Origin` / `Host`;
3. service layer и удаление прямых runtime calls из MCP adapter;
4. типизированная семантика успеха/ошибки;
5. модель audit events;
6. ограничения частоты;
7. политика ограниченного объёма output;
8. replay/idempotency semantics для изменений состояния;
9. политика долгих команд;
10. legacy client compatibility/off-ramp, если он реально нужен.

Не следует сначала заменить SSE на Streamable HTTP, оставив прежнюю плоскую privilege plane.

## 11. Долгие операции

### Текущие кандидаты

- перезапуск emulator;
- остановка/запуск процесса при медленном lifecycle;
- screenshot/device operations при проблемном device;
- системный перезапуск ADB.

**TARGET prerequisite:** результат команды должен различать accepted/running/completed/failed либо предоставлять эквивалентную безопасную lifecycle модель, если операция реально асинхронная.

Современная MCP Tasks extension является доступным протокольным механизмом для долгих операций, но выбор Tasks против обычной bounded command **DEFERRED** до service/transport design. Этап 2 не создаёт job queue.

## 12. Целевая модель ошибок

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

Текстовое `Error: ...` внутри успешного tool result не является достаточным контрактом для privileged automation.

Traceback и абсолютные пользовательские пути должны оставаться в server-side diagnostics/audit, а не в model-facing result.

## 13. Базовая политика rate limit и audit

Минимальная будущая политика:

- чтение metadata/status — мягкий rate limit;
- logs/screenshot — отдельный output/call limit;
- config/scheduler/process/device control — строгий actor + target audit;
- system operations — самый строгий gate;
- audit должен фиксировать actor, permission, tool/capability, resolved target, sanitized input summary, result class, correlation/command ID и время.

Точная storage/retention schema откладывается.

## 14. Итог

Этап 2 не предлагает «переписать 17 tools один к одному». Он разделяет текущий каталог на продуктовые возможности и будущие границы бэкенда:

```text
17 current tools
  -> 8 общих service boundaries
  -> 7 RETAIN
  -> 9 REDESIGN
  -> 1 DEFER
  -> 0 REMOVE
  -> 7 обязательных + 1 условный read-only MVP candidate
```

Следующий этап проектирования бэкенда теперь может строить общий слой по возможностям, а не по именам legacy MCP handlers.
