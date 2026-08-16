# Матрица зависимостей и миграционной готовности

Audit source: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

Полные evidence-записи находятся в [CURRENT_WEBUI_SURFACES.md](CURRENT_WEBUI_SURFACES.md). Матрица нужна как decision surface для следующих backend/frontend stages и не проектирует финальные API endpoints.

## Матрица

| ID | Surface | Data/action dependency | Current wiring | Coupling | Migration tags | Future backend prerequisite | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
| S-01 | PyWebIO shell/auth | deploy state, password, process bootstrap | `_run_gui` + ASGI startup hooks | high | `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY` | browser auth/session contract отдельно от PyWebIO/process bootstrap | подтверждено кодом |
| S-02 | OOBE | ADB discovery, template/config creation, legacy files | PyWebIO + subprocess + import HTTP | high | `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `PRIVILEGED_POLICY`, `LEGACY_COMPATIBILITY` | bootstrap/config/import boundary | подтверждено кодом |
| S-03 | Home | theme, instance/session status | PyWebIO session + manager read | medium | `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION` | instance/session read model | подтверждено кодом |
| S-04 | Shell/instances | instance list + process states + menu files | direct `ProcessManager`/config/menu reads | medium/high | `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION` | instance list/status read boundary | подтверждено кодом |
| S-05 | Instance selection | manager/config/module | direct session wiring | medium/high | `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION` | backend instance context/model | подтверждено кодом |
| S-06 | Instance management | config files, process managers | direct filesystem + `State.config_updater` + `ProcessManager` | high | `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `PRIVILEGED_POLICY`, `LEGACY_COMPATIBILITY` | lifecycle/create/delete/import/export policy boundary | подтверждено кодом |
| S-07 | Overview/start-stop/log | process lifecycle, scheduler/config, runtime log | direct manager + server-side log rendering | high | `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `REALTIME_CONTRACT`, `SECURITY_REDESIGN` | lifecycle commands + status/log stream + authz/audit | подтверждено кодом |
| S-08 | Dashboard | task/scheduler/resource state | direct config reload/read helpers | medium | `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION` | dashboard read model | подтверждено кодом |
| S-09 | Task configuration | argument definitions + config persistence + save callbacks | PyWebIO pin queue → `_save_config` | high | `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `CONTRACT_FORMALIZATION` | shared validated config read/write boundary | подтверждено кодом |
| S-10 | Tool/daemon family | task process/runtime | daemon overview → manager start/stop | high | `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `INTERNAL_DEVELOPER`, `DEFER_OR_REMOVE_CANDIDATE` | product decision + shared task command policy | подтверждено кодом |
| S-11 | Startup-run | deploy `Run` | current GET/POST deploy endpoint | medium/high | `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY` | authenticated administrative policy behind proxy | подтверждено кодом |
| S-12 | Statistics hub | statistics read views | PyWebIO composition | low/medium | `CONTRACT_FORMALIZATION` | stable statistics read models | подтверждено кодом |
| S-13 | AP statistics | AP/resources timeline + aggregation | direct statistics module, partial current HTTP for AP | medium | `CONTRACT_FORMALIZATION` | one authoritative read model/contract | подтверждено кодом |
| S-14 | Resource statistics | resource timeline | direct statistics module | medium | `CONTRACT_FORMALIZATION` | resource-history read model | подтверждено кодом |
| S-15 | OpSi statistics | CL1/Meow/ship-exp stores/calculations | direct statistics modules + partial CL1 HTTP | medium | `CONTRACT_FORMALIZATION` | coherent OpSi statistics read model | подтверждено кодом |
| S-16 | OpSi export/refresh | stats + local Desktop filesystem | server-side refresh/write | high | `LEGACY_COMPATIBILITY`, `SERVICE_EXTRACTION`, `DEFER_OR_REMOVE_CANDIDATE` | decide browser download vs backend export | подтверждено кодом |
| S-17 | Ship EXP statistics | ship-exp + OpSi stats | direct statistics modules | medium | `CONTRACT_FORMALIZATION` | ship-exp read model | подтверждено кодом |
| S-18 | Commission income | statistics persistence | direct statistics modules | medium | `CONTRACT_FORMALIZATION` | commission read model | подтверждено кодом |
| S-19 | Event profiles | event profile/config + scheduler safety | PyWebIO callbacks → config writes | high | `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY` | event profile command boundary | подтверждено кодом |
| S-20 | Event datamine/source | artifacts + user policy | backend helpers + local persistence | medium/high | `SERVICE_EXTRACTION`, `CONTRACT_FORMALIZATION` | diagnostics/read model + mutation boundary | подтверждено кодом |
| S-21 | Event plan/layout | event plan/config/observations | planner mixin + direct config writes | high | `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `CONTRACT_FORMALIZATION` | event planning domain boundary | подтверждено кодом |
| S-22 | EventShop safety | scheduler enable/filter/safety policy | UI callback mutates automation config | high | `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY` | fail-closed shop policy in backend service layer | подтверждено кодом |
| S-23 | OpSi simulator | simulator process/log/figure | direct simulator runtime + filesystem | high | `INTERNAL_DEVELOPER`, `SERVICE_EXTRACTION`, `DEFER_OR_REMOVE_CANDIDATE` | product decision; service/stream only if retained | подтверждено кодом |
| S-24 | Remote Access status | tunnel/provider state | direct `RemoteAccess` runtime | medium/high | `INTERNAL_DEVELOPER`, `SECURITY_REDESIGN`, `DEFER_OR_REMOVE_CANDIDATE` | decide relation to future Caddy/public edge | подтверждено кодом |
| S-25 | Deploy/Launcher settings | launcher IPC + deploy config | current local-only HTTP handlers | medium/high | `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`, `INTERNAL_DEVELOPER` | authenticated admin contract, trusted proxy semantics | подтверждено кодом |
| S-26 | Developer utilities | manager state override, restart event, marker files, notification | direct privileged runtime callbacks | high | `INTERNAL_DEVELOPER`, `PRIVILEGED_POLICY`, `SECURITY_REDESIGN`, `DEFER_OR_REMOVE_CANDIDATE` | explicit admin product/policy decision | подтверждено кодом |
| S-27 | OBS overlay | CL1/AP read endpoints | standalone HTML + polling | low/medium | `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY` | decide compatibility client + authenticated read exposure | подтверждено кодом |

## Прямые dependency families

### ProcessManager / worker lifecycle

Доказанные UI execution paths: S-01, S-04, S-05, S-06, S-07, S-10, S-26. `app_lifecycle.py` также управляет общим startup/cleanup процесса WebUI.

**Следствие:** status и lifecycle нельзя моделировать как frontend-local state. Нужны backend read/command boundaries и единая privileged policy.

### Config / scheduler / persistent settings

Доказанные прямые или тонко-обёрнутые пути: S-02, S-04, S-05, S-06, S-08, S-09, S-11, S-19, S-20, S-21, S-22, S-25.

Особенно существенны:

- S-09: UI сам валидирует и выполняет `save_callback` перед записью config;
- S-22: UI реализует fail-closed automation policy и меняет Scheduler;
- S-25: current HTTP handler пишет deploy config, но только через local-only guard.

**Следствие:** future contract должен выражать domain/config commands, а не отдавать браузеру право редактировать raw JSON/YAML.

### Filesystem

Доказанные пользовательские workflow:

- S-02 — legacy import и config creation;
- S-06 — import/export/delete config;
- S-16 — CSV на Desktop;
- S-20 — event artifacts/persistence;
- S-23 — generated simulator figure;
- S-26 — restart/reload marker.

Отдельно `/api/import_legacy_upload` пишет выбранные config/log paths из ASGI handler.

**Следствие:** browser migration требует file-transfer/export semantics, а не копирования server filesystem paths.

### Device / ADB / emulator

В обычных user surfaces прямой device path наиболее очевиден в OOBE ADB discovery и simulator/runtime tooling. Отдельно в `api.py` существуют live screenshot/control handlers с прямым device/scrcpy/ADB поведением, но они production-disabled. Legacy MCP также напрямую создаёт `Device` и выполняет emulator/ADB operations.

**Следствие:** любое последующее remote device control требует отдельного realtime + privileged + security contract.

### Statistics persistence/read models

S-12–S-18 и S-27 читают специализированные statistics-модули. Здесь уже существует полезная backend data ownership, но presentation mixins часто сами делают агрегацию/форматирование.

**Следствие:** это лучший кандидат на раннюю contract formalization: переносить данные/semantic read model, а не Python chart implementation.

### Network / remote access / launcher

S-24 использует current RemoteAccess providers. S-25 и S-11 используют launcher/deploy local-only HTTP. Эти contracts были созданы до целевого Caddy edge и не считаются remote-ready.

## Business logic leakage classification

| Класс | Основные surfaces | Вывод |
| --- | --- | --- |
| Presentation only / mostly presentation | S-12, части S-13–S-18, S-27 HTML | можно мигрировать после read contract formalization |
| Presentation + orchestration | S-03, S-04, S-05, S-08, S-13–S-18, S-20, S-25 | нельзя переносить server-side aggregation/runtime lookup как client business logic |
| UI-owned business/orchestration logic | S-09, S-19, S-21, S-22 | сначала backend service/domain extraction |
| Direct privileged runtime control | S-02, S-06, S-07, S-10, S-23, S-26 | обязательны command policy/authz/audit boundaries |
| Shared backend/runtime helper | statistics modules, `ProcessManager`, `State`, deploy settings, RemoteAccess | владельцем остаётся backend; React зависит только от stable contract |

## Readiness bands

### Относительно ранняя миграционная готовность

После формализации read contract: S-12, S-13, S-14, S-15, S-17, S-18. S-27 может остаться отдельным compatibility client.

### Требуется backend extraction до UI migration

S-04–S-10, S-19–S-22, а также command часть S-02/S-25.

### Требуется security/trust redesign до remote exposure

S-01/S-02/S-06/S-07/S-11/S-24/S-25/S-26/S-27 и current transport families, перечисленные в `CURRENT_BACKEND_INTERFACES.md`.

### Отложить до product parity decision

S-10 developer/tool tasks, S-16 Desktop export semantics, S-23 simulator, S-24 legacy remote access, S-26 developer utilities.
