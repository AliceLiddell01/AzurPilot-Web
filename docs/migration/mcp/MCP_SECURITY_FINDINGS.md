# Выводы по безопасности текущего MCP

Источник текущего состояния: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## Правила интерпретации

Severity оценивает риск capability/transport **при достижимости MCP клиентом**. Статический аудит не устанавливает router/NAT/firewall/OS ACL/Caddy и не утверждает, что конкретный домашний ПК доступен из Интернета.

Современные требования MCP помечаются как **OFFICIAL MCP TARGET GUIDANCE** и не выдаются за уже реализованные свойства legacy server.

## MCP-SEC-001 — отсутствует общая authentication/authorization boundary MCP

**Severity:** High при сетевой достижимости.

**CURRENT FACT — доказательство:** `mcp_server_sse.py` не добавляет auth/authz middleware; все 17 tools идут через единый dispatch без actor/permission context. PyWebIO password gate находится в `module/webui/app.py::_run_gui()`, тогда как MCP подключается отдельным child ASGI app через `application.mount("/mcp", mcp_app)`. Standalone использует тот же MCP app без отдельного auth layer.

**Влияние при сетевой достижимости:** клиент, достигший transport и прошедший protocol/schema validation, получает общий каталог от metadata reads до config/process/device/scheduler/system operations.

**Доказано:** отсутствие auth/authz в проверенном MCP wrapper, handlers и embedded wiring.

**Не доказано:** внешняя/LAN/internet reachability конкретной установки.

**Затронуто:** MCP-T-001…MCP-T-017, MCP-TR-001/002.

**Обоснование severity:** одна неразделённая boundary защищает одновременно R0/R1 и R3/R4 operations; при достижимости ущерб не ограничен read-only данными.

**Целевое условие:** authentication + permission-aware authorization до dispatch в Application/Service Layer.

## MCP-SEC-002 — wildcard CORS и выключенная SDK DNS-rebinding validation

**Severity:** High при browser/network reachability.

**CURRENT FACT — доказательство:** MCP Starlette wrapper использует `allow_origins=["*"]`, `allow_methods=["*"]`, `allow_headers=["*"]`. В точной зависимости `mcp==1.23.0` `SseServerTransport` создаёт transport security middleware, но при `security_settings=None` DNS-rebinding protection выключена для backwards compatibility, поэтому `Host` и `Origin` не валидируются. POST `Content-Type` проверяется отдельно. Standalone способен bind на `0.0.0.0:22268`.

**Влияние при сетевой достижимости:** browser/network trust surface не имеет текущей Origin/Host boundary на уровне MCP adapter.

**Доказано:** настройки CORS, SDK security settings и standalone bind instruction.

**Не доказано:** browser exploitability, public routing или Caddy exposure конкретной машины.

**Затронуто:** MCP-TR-001/002 и все tools.

**Обоснование severity:** отсутствие Origin/Host validation рядом с wildcard CORS существенно при reachable HTTP surface, особенно для локального сервера.

**Целевое условие:** secure public-edge topology, Origin/Host/trusted-proxy policy и authentication. Wildcard CORS не считать security mechanism.

**OFFICIAL MCP TARGET GUIDANCE:** современная HTTP transport guidance требует Origin validation против DNS rebinding; локальным серверам рекомендуется loopback bind, remote deployments требуют proper authentication.

## MCP-SEC-003 — плоская privilege plane

**Severity:** High.

**CURRENT FACT — доказательство:** `TOOL_HANDLERS` — единый dictionary dispatch без permission split. В одном каталоге находятся metadata/operational reads и `update_config`, start/stop, scheduler mutations, screenshot, emulator restart и machine-wide ADB restart. Распределение risk: R0=2, R1=7, R2=0, R3=7, R4=1.

**Влияние:** compromise или misconfiguration одной MCP connection boundary автоматически расширяет доступ от чтения к privileged control.

**Доказано:** единый dispatch и отсутствие per-capability permission gate.

**Не доказано:** факт компрометации конкретного клиента/хоста.

**Затронуто:** все 17 tools.

**Обоснование severity:** максимальная привилегия connection plane определяется R4 capability, а не наиболее безопасным tool.

**Целевое условие:** least-privilege scopes, service-level policy и отсутствие implicit escalation read → control.

## MCP-SEC-004 — raw config read/write обходит безопасную доменную границу

**Severity:** High.

**CURRENT FACT — доказательство:** `get_config` может вернуть весь `AzurLaneConfig.data` или raw task section. `update_config` собирает model-controlled `task/group/arg` path и вызывает `AzurLaneConfig.cross_set(path,value)`. Это не воспроизводит WebUI parsing/validation/`ConfigUpdater.save_callback` policy из Stage 1.

**Влияние:** чрезмерно широкая confidentiality surface для чтения и возможность mutation в обход UI-owned domain validation semantics.

**Доказано:** current raw read/write path и отличие от WebUI validation path.

**Не доказано:** что любой произвольный path эксплуатируем; что каждый config содержит секреты.

**Затронуто:** MCP-T-006, MCP-T-007; косвенно scheduler config mutations.

**Обоснование severity:** capability затрагивает persistent automation configuration и не ограничена field-level authorization unit.

**Целевое условие:** `ConfigService` с allowlist, field/domain validation, redaction и server-owned save policy.

## MCP-SEC-005 — runtime/domain ошибки часто маркируются как успешный MCP result

**Severity:** Medium-High.

**CURRENT FACT — доказательство:** SDK schema failure способен дать `CallToolResult(isError=True)`, но AzurPilot `call_tool()` ловит handler exceptions и возвращает `TextContent("Ошибка: ...")`; SDK затем формирует обычный result с `isError=false`. Ряд handlers также возвращает `Error: ...` как обычный текст.

**Влияние:** модель/клиент не может надёжно отличить validation failure, conflict/precondition, retryable runtime failure и обычный success.

**Доказано:** два разных error channels в текущем adapter/SDK path.

**Не доказано:** как конкретный внешний MCP client интерпретирует текстовую ошибку.

**Затронуто:** все tools; особенно privileged commands.

**Обоснование severity:** ошибочная классификация результата может провоцировать неправильный follow-up или повтор side effect.

**Целевое условие:** структурированная error taxonomy и корректный MCP error/result contract.

## MCP-SEC-006 — ложные или неоднозначные success responses control tools

**Severity:** Medium-High.

**CURRENT FACT — доказательство:** `start_instance` не получает подтверждающий result от `ProcessManager.start()`; `stop_instance` игнорирует bool `ProcessManager.stop()`; `restart_adb` использует `subprocess.run(..., check=False)` и не проверяет return code; `trigger_task` подтверждает запись scheduling intent, а не выполнение task.

**Влияние:** LLM может считать операцию завершённой, хотя она была отклонена, частично выполнена или только поставлена в план.

**Доказано:** current handler/result semantics.

**Не доказано:** что ложный success происходит в каждом вызове.

**Затронуто:** MCP-T-009, T-010, T-014, T-017.

**Обоснование severity:** неоднозначный postcondition опаснее для control plane, где client retry способен повторить disruptive action.

**Целевое условие:** postcondition-aware command result с explicit accepted/running/completed/failed или эквивалентной моделью.

## MCP-SEC-007 — blocking/long-running operations без MCP-level timeout и dedup

**Severity:** High для R3/R4 retry; Medium для availability.

**CURRENT FACT — доказательство:** `restart_emulator` выполняет `time.sleep(60)` внутри `async def`; `restart_adb` делает blocking `subprocess.run` без timeout; screenshot/config/log/process operations вызываются синхронно; command ID/idempotency key/dedup store и per-tool timeout отсутствуют.

**Влияние:** event loop может блокироваться; client disconnect/timeout не создаёт safe retry contract; повтор mutation/restart после неопределённого результата может повторить side effect.

**Доказано:** отсутствие adapter-level timeout/dedup и конкретные blocking paths.

**Не доказано:** фактическая wall-clock длительность каждого device/process вызова на пользовательской машине.

**Затронуто:** прежде всего MCP-T-009/010/011/014/015/016/017, а также тяжёлые log reads.

**Обоснование severity:** сочетание privileged side effects и неизвестного результата при timeout создаёт replay risk; blocking path дополнительно влияет на availability.

**Целевое условие:** bounded/non-blocking command semantics, explicit lifecycle/postcondition и replay/idempotency policy. Выбор Tasks extension против иной command model отложен.

## MCP-SEC-008 — raw logs имеют privacy и large-output surface

**Severity:** Medium.

**CURRENT FACT — доказательство:** `get_recent_logs.lines` не имеет min/max; handler читает весь дневной файл через `readlines()` до slicing, возвращает raw text без MCP-level sanitization и может использовать общий fallback log. `get_current_running_task` тоже читает дневной log целиком, хотя наружу возвращает только task name.

**Влияние:** memory/output pressure и потенциальное раскрытие operational identifiers, paths и error details.

**Доказано:** full-file read, отсутствие bounds/redaction и fallback semantics.

**Не доказано:** наличие конкретных secrets в пользовательском log.

**Затронуто:** MCP-T-008, частично MCP-T-012.

**Обоснование severity:** read-only capability не меняет runtime, но имеет confidentiality/DoS surface.

**Целевое условие:** `LogService` с bounded tail/byte limits, sanitization/redaction и predictable output schema.

## MCP-SEC-009 — screenshot/emulator errors раскрывают traceback

**Severity:** Medium.

**CURRENT FACT — доказательство:** `get_screenshot` и `restart_emulator` включают `traceback.format_exc()` в model-facing text error.

**Влияние:** раскрытие внутренних paths/device/runtime details; ошибка при этом может оставаться обычным text result.

**Доказано:** traceback входит в current error text.

**Не доказано:** наличие credentials/secrets в конкретном traceback.

**Затронуто:** MCP-T-011, MCP-T-016.

**Обоснование severity:** disclosure ограничено failure path, но раскрывает внутреннюю среду и ухудшает стабильность error contract.

**Целевое условие:** full traceback только в server-side diagnostics/audit; наружу — sanitized stable error.

## MCP-SEC-010 — screenshot является high-privilege read

**Severity:** Medium-High по privacy.

**CURRENT FACT — доказательство:** `get_screenshot` создаёт `Device(config)`, получает реальное изображение эмулятора и возвращает JPEG `ImageContent`.

**Влияние:** изображение может содержать приватный игровой/пользовательский контекст.

**Доказано:** capability читает реальное device image.

**Не доказано:** наличие чувствительных данных в каждом конкретном кадре.

**Затронуто:** MCP-T-011.

**Обоснование severity:** `readOnlyHint=true` не означает низкую привилегию для privacy-sensitive device view.

**Целевое условие:** отдельный `device:view`, target resolution, audit и privacy/confirmation policy.

## MCP-SEC-011 — `restart_adb` имеет machine-wide scope

**Severity:** High.

**CURRENT FACT — доказательство:** optional `instance` читается handler, но не ограничивает операцию; выполняется общий `adb kill-server`, затем `adb start-server`. Success output способен раскрыть фактический executable path.

**Влияние:** вызов, выглядящий instance-oriented, может прервать ADB connections шире одного AzurPilot instance.

**Доказано:** machine-wide command construction и отсутствие instance scoping.

**Не доказано:** конкретное воздействие на каждый сторонний процесс/device конкретной машины.

**Затронуто:** MCP-T-017.

**Обоснование severity:** system-wide disruptive scope шире заявленного target и является единственным R4 tool.

**Целевое условие:** отдельное product/security решение; до него capability остаётся DEFER и не считается автоматически remotely exposable.

## MCP-SEC-012 — отсутствуют actor-aware audit trail и rate limiting

**Severity:** Medium.

**CURRENT FACT — доказательство:** MCP adapter не формирует structured successful-call audit event с actor, permission, capability, resolved target, command/correlation ID и result class. MCP-level rate limiter отсутствует.

**Влияние:** слабая attributable control и защита от повторных/частых calls, особенно для screenshot/log/control/system capabilities.

**Доказано:** отсутствие actor-aware audit/rate-limit mechanism в MCP adapter.

**Не доказано:** отсутствие всех технических логов внутри runtime dependencies.

**Затронуто:** все tools.

**Обоснование severity:** finding сам по себе не создаёт mutation, но существенно ухудшает расследование и abuse/replay resistance privileged plane.

**Целевое условие:** audit до/после privileged command + limits по actor/capability/target.

## MCP-SEC-013 — protocol JSON Schema не покрывает semantic validation

**Severity:** Medium-High.

**CURRENT FACT — доказательство:** `mcp==1.23.0` валидирует arguments против advertised `inputSchema`, но schemas в основном ограничиваются primitive types/required fields. Они не выражают существующий instance, task existence, field-specific config rules, scheduler/process/device preconditions, machine-vs-instance scope или permission scope.

**Влияние:** protocol-valid input всё ещё может быть domain-invalid или иметь чрезмерный privilege target.

**Доказано:** schema validation существует, но перечисленные semantic checks не представлены текущими schemas/handlers как policy boundary.

**Не доказано:** что каждый protocol-valid/domain-invalid input обязательно приводит к вредному состоянию.

**Затронуто:** большинство tools; наиболее критично MCP-T-006/007/014/015/017.

**Обоснование severity:** для privileged mutations type correctness существенно слабее authorization/precondition/domain correctness.

**Целевое условие:** service-level semantic validation и safe preconditions поверх protocol schema.

## MCP-SEC-014 — legacy process-local session transport является migration debt

**Severity:** Medium по архитектуре/эксплуатации.

**CURRENT FACT — доказательство:** current `SseServerTransport` держит UUID session → memory stream mapping в одном процессе, использует long-lived SSE + message POST и теряет process-local registry при restart.

**Влияние:** hidden transport state затрудняет restart/horizontal semantics и не соответствует целевому stateless protocol core.

**Доказано:** current process-local session lifecycle точной зависимости `mcp==1.23.0`.

**Не доказано:** необходимость сохранять compatibility с конкретными legacy clients после migration.

**Затронуто:** MCP-TR-001/002 и lifecycle всех calls.

**Обоснование severity:** это не самостоятельная privilege escalation, но operational debt влияет на reliability/migration design.

**Целевое условие:** transport migration после/вместе с auth/policy/service boundary; не переносить hidden legacy session semantics как target invariant.

**OFFICIAL MCP TARGET GUIDANCE:** опубликованная ревизия `2026-07-28` использует stateless protocol core; legacy HTTP+SSE официально deprecated. Официальный Python SDK v2 — текущая stable line, v1.x — maintenance.

## Tool Annotations не являются security boundary

**OFFICIAL MCP TARGET GUIDANCE**

`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` — hints о семантике. Они не заменяют authentication, authorization, validation, permission policy, audit и human confirmation.

**TARGET DECISION**

Будущие annotations перечислены в [целевой границе и решениях](MCP_TARGET_BOUNDARY_AND_DECISIONS.md), но security boundary остаётся server-owned.

## Сверка с приёмочными полями

Каждый finding выше теперь явно содержит:

```text
стабильный ID;
summary;
current evidence;
impact;
что доказано;
что не доказано;
affected capabilities;
severity/risk rationale;
target prerequisite.
```

Repository-wide discovery, `Exposed name/URI` и формальная DoD-сверка вынесены в [сверку приёмки Этапа 2](STAGE_2_ACCEPTANCE_RECONCILIATION.md).
