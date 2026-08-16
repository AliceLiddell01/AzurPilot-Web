# Выводы по безопасности текущего MCP

Источник текущего состояния: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## Как читать severity

Severity оценивает **риск capability/transport при достижимости MCP клиентом**, но не приписывает домашнему ПК недоказанную internet reachability. Router/NAT/firewall/Caddy/OS ACL конкретной машины не входят в статический аудит.

Официальные требования MCP `2026-07-28` ниже помечены как **OFFICIAL MCP TARGET GUIDANCE** и не выдаются за требования, уже реализованные current legacy server.

## MCP-SEC-001 — MCP route family не имеет текущей authentication/authorization boundary

**Severity:** High при network/client reachability.

**CURRENT FACT**

- `mcp_server_sse.py` не добавляет auth/authz middleware;
- 17 tools не получают actor/permission context;
- PyWebIO password gate находится в `module/webui/app.py::_run_gui()`;
- MCP child app монтируется отдельно через `application.mount("/mcp", mcp_app)`;
- standalone mode использует тот же MCP app без дополнительного auth layer.

**Impact if remotely reachable**

Любой клиент, способный достичь transport и пройти protocol/schema validation, получает один и тот же каталог, включая config/process/device/scheduler/system operations.

**Что доказано:** отсутствие auth/authz в проверенном MCP/outer wiring.

**Что не доказано:** доступность порта/route из Интернета на конкретной машине.

**Affected:** все MCP-T-001…017.

**TARGET prerequisite:** authentication + permission-aware authorization до dispatch в service layer.

**OFFICIAL MCP TARGET GUIDANCE:** актуальная спецификация требует proper access controls для tools; remote HTTP authorization имеет отдельную современную authorization model.

## MCP-SEC-002 — wildcard CORS и выключенная SDK DNS-rebinding validation

**Severity:** High при browser/network reachability.

**CURRENT FACT**

MCP Starlette wrapper использует:

```text
allow_origins=["*"]
allow_methods=["*"]
allow_headers=["*"]
```

В точной зависимости `mcp==1.23.0` `SseServerTransport` действительно содержит `TransportSecurityMiddleware`, но при `security_settings=None` middleware специально создаётся с `enable_dns_rebinding_protection=False` для backwards compatibility. Поэтому `Host` и `Origin` не валидируются. POST `Content-Type` проверяется независимо и должен начинаться с `application/json`.

Standalone дополнительно способен bind на `0.0.0.0:22268`.

**Что доказано:** transport security settings и bind instruction в коде.

**Что не доказано:** internet reachability, браузерная эксплуатация или Caddy route конкретной установки.

**Affected:** transport MCP-TR-001/002 и все tools.

**TARGET prerequisite:** secure public-edge topology, Origin/Host/trusted-proxy policy и authentication; не полагаться на wildcard CORS как security mechanism.

**OFFICIAL MCP TARGET GUIDANCE:** Streamable HTTP guidance требует защищаться от DNS rebinding, в том числе проверять `Origin`; локальные серверы рекомендуется bind'ить на localhost, а remote deployments — защищать auth.

## MCP-SEC-003 — плоская privilege plane: R0/R1 и R3/R4 доступны через один dispatch

**Severity:** High.

**CURRENT FACT**

`TOOL_HANDLERS` — единый dictionary dispatch без permission split. В одном каталоге находятся metadata reads и операции:

- `update_config`;
- `start_instance` / `stop_instance`;
- `trigger_task` / `clear_scheduler_queue`;
- `get_screenshot`;
- `restart_emulator`;
- `restart_adb`.

Risk distribution: R0=2, R1=7, R3=7, R4=1.

**Impact:** compromise/misconfiguration одной MCP connection boundary автоматически даёт весь набор привилегий.

**TARGET prerequisite:** least-privilege scopes, service-level policy и запрет implicit escalation от read к control.

## MCP-SEC-004 — `get_config` и `update_config` обходят безопасную доменную границу конфигурации

**Severity:** High.

**CURRENT FACT**

`get_config` может вернуть весь `AzurLaneConfig.data` или raw task section.

`update_config` собирает path из model-controlled `task/group/arg` и вызывает `AzurLaneConfig.cross_set(path,value)`. `cross_set` не проверяет semantic allowlist: он добавляет path в `modified` и вызывает обычный config `update()`.

Это отличается от текущего WebUI `TaskConfigMixin::_save_config`, где есть parsing, validation expressions, defaults, `ConfigUpdater.save_callback` и отдельные domain warnings/side effects.

**Impact:** MCP mutation path способен обойти UI-specific validation/domain semantics; raw config read имеет чрезмерно широкую confidentiality surface.

**Что не утверждается:** что каждый произвольный path обязательно приводит к exploitable состоянию или что в каждом config есть секрет.

**Affected:** MCP-T-006, MCP-T-007; косвенно scheduler tools.

**TARGET prerequisite:** `ConfigService` с server-owned schema/field policy/domain validation; никаких raw JSON dump/write как public capability.

## MCP-SEC-005 — runtime/domain ошибки часто маркируются как успешный MCP tool result

**Severity:** Medium-High.

**CURRENT FACT**

Точная версия SDK умеет вернуть `CallToolResult(isError=True)` для JSON Schema validation failure.

Но AzurPilot `call_tool()` ловит exceptions и возвращает `TextContent("Ошибка: ...")`. После этого SDK видит обычный content list и формирует `isError=False`. Аналогично handlers возвращают обычный текст для:

- already running/stopped;
- not running;
- log read/not-found;
- screenshot/emulator failure;
- restart ADB failure.

**Impact:** модель/клиент не может надёжно отличить validation failure, retryable runtime failure и обычный успешный text result.

**TARGET prerequisite:** структурированная error taxonomy с validation/conflict/precondition/retryable/internal классами и корректным MCP error/result contract.

## MCP-SEC-006 — ложные или неоднозначные success responses у control tools

**Severity:** Medium-High.

**CURRENT FACT**

- `start_instance`: `ProcessManager.start()` может безопасно отклонить start при restart/cleanup/registry condition, но возвращает `None`; tool после возврата всё равно выдаёт `Success`.
- `stop_instance`: `ProcessManager.stop()` возвращает bool, но tool игнорирует его и выдаёт `Success` после вызова, даже если manager сообщил неполную остановку.
- `restart_adb`: оба `subprocess.run(..., check=False)` игнорируют return code, после чего выдаётся `Success`.
- `trigger_task`: success означает запись scheduling intent, а не подтверждение выполнения task.

**Impact:** LLM может принять неисполненную/частично исполненную операцию за завершённую и принять неверное следующее решение или повторить disruptive command.

**TARGET prerequisite:** postcondition-aware command results и explicit status.

## MCP-SEC-007 — long-running/blocking операции не имеют MCP-level timeout/dedup semantics

**Severity:** High для R3/R4 retries; Medium для availability.

**CURRENT FACT**

- `restart_emulator` выполняет `time.sleep(60)` внутри `async def` между stop/start;
- `restart_adb` использует blocking `subprocess.run` без timeout;
- `Device.screenshot`, config I/O, log I/O и ProcessManager operations вызываются синхронно из async handlers;
- command ID/idempotency key/dedup store отсутствуют;
- per-tool timeout отсутствует.

**Impact:** event loop может быть заблокирован; disconnect/timeout клиента не создаёт доказанного safe retry contract. Повтор restart/trigger после неопределённого результата способен повторить side effect.

**TARGET prerequisite:** bounded operations; explicit command lifecycle/postcondition; для реально долгих действий — отдельный long-running contract. Возможность использовать современную Tasks extension можно оценить позже, но Этап 2 её не выбирает и не реализует.

## MCP-SEC-008 — raw logs имеют privacy и large-output surface

**Severity:** Medium.

**CURRENT FACT**

`get_recent_logs`:

- принимает `lines:integer` без min/max;
- читает весь дневной файл `readlines()` до tail slicing;
- возвращает raw log text без MCP-level sanitization/redaction;
- при отсутствии instance-specific файла использует общий `..._alas.txt` fallback.

`get_current_running_task` тоже читает дневной log целиком, хотя наружу возвращает только найденное имя task.

**Impact:** возможен большой memory/output cost и раскрытие operational identifiers/paths/error details из raw logs.

**TARGET prerequisite:** `LogService` с bounded tail, size limits, sanitization/redaction и predictable schema.

## MCP-SEC-009 — screenshot/emulator ошибки возвращают полный traceback

**Severity:** Medium.

**CURRENT FACT**

`get_screenshot` и `restart_emulator` формируют error text из `str(e)` + `traceback.format_exc()`.

**Impact:** traceback может содержать absolute paths, device/runtime detail и внутреннюю структуру; при этом result маркируется обычным text success на MCP-level.

**TARGET prerequisite:** internal traceback только в server log/audit; наружу — sanitized stable error.

## MCP-SEC-010 — screenshot является high-privilege read, хотя не меняет состояние

**Severity:** Medium-High privacy.

**CURRENT FACT**

`get_screenshot` создаёт `Device(config)` и получает реальное изображение эмулятора. Результат возвращается как JPEG `ImageContent`.

**Impact:** read-only annotation в будущем не должна понижать authorization/confirmation policy: изображение может содержать приватный игровой/пользовательский контекст.

**TARGET prerequisite:** отдельный `device:view` scope, target resolution, audit и privacy policy.

## MCP-SEC-011 — `restart_adb` имеет machine-wide scope, не соответствующий `instance` argument

**Severity:** High.

**CURRENT FACT**

Tool schema допускает optional `instance`, handler присваивает его local variable, но дальше не использует для ограничения операции. Выполняется общий `adb kill-server`, затем `adb start-server`.

Success response включает фактический ADB executable path, который может быть absolute path пользователя.

**Impact:** вызов для одного предполагаемого экземпляра способен затронуть все ADB connections этого machine process context; path может утечь клиенту.

**TARGET prerequisite:** отдельное product решение по machine-wide ADB control; до него capability помечена `DEFER` и не должна быть remote-exposable автоматически.

## MCP-SEC-012 — нет actor-aware audit trail и rate limiting

**Severity:** Medium.

**CURRENT FACT**

MCP adapter логирует transport events и exceptions, но успешный `call_tool` не формирует structured event с actor, permission, capability, target, request/command ID, result class и audit outcome. Rate limiter отсутствует.

Без authentication actor identity сейчас в принципе не определена.

**TARGET prerequisite:** audit event до/после privileged command + rate limit по actor/capability/target с отдельными limits для screenshot/log/control/system operations.

**OFFICIAL MCP TARGET GUIDANCE:** tools security considerations требуют access control, rate limiting, output sanitization; клиентам рекомендуется user confirmation для sensitive operations, timeout и tool-usage logging.

## MCP-SEC-013 — semantic validation не покрывается одной JSON Schema

**Severity:** Medium-High.

**CURRENT FACT**

SDK schema validation действительно есть, но schemas в основном проверяют только primitive types/required fields. Они не выражают:

- существующий instance target;
- task existence/capability;
- field-specific config rules;
- scheduler preconditions;
- running/stopped/device health preconditions;
- machine-vs-instance scope;
- permission scope.

`get_task_help` unknown task возвращает `{}` как success; `trigger_task` принимает произвольный task string до `cross_set`.

**TARGET prerequisite:** service-level semantic validation и safe preconditions поверх protocol schema.

## MCP-SEC-014 — legacy session transport является migration debt, а не target security model

**Severity:** Medium architecture / operational.

**CURRENT FACT**

Current transport держит UUID session → memory-stream mapping внутри одного process и требует long-lived SSE connection + POST message channel.

**OFFICIAL MCP TARGET GUIDANCE**

Спецификация `2026-07-28` перешла к stateless protocol core; legacy HTTP+SSE официально deprecated. Актуальный Python SDK v2 является текущей stable line и поддерживает новую ревизию.

**TARGET prerequisite:** transport migration должна происходить после/вместе с auth/policy/service boundary и не должна переносить process-local legacy session semantics как архитектурный инвариант.

## Tool Annotations не являются исправлением security boundary

**OFFICIAL MCP TARGET GUIDANCE**

`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` — только hints. Спецификация требует считать annotations недоверенными, если сам server не доверен.

**TARGET DECISION**

Annotations будут добавляться только после определения реальной capability semantics, но authorization, validation, permissions, human confirmation и audit остаются отдельными обязательными механизмами.
