# Текущие пользовательские поверхности WebUI

Audit source: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## Как читать карту

`Runtime coupling`:

- **low** — преимущественно presentation/client-session state;
- **medium** — UI напрямую собирает read model или использует backend helper, но не управляет привилегированным runtime;
- **high** — callback/handler сам пишет конфигурацию/файлы, управляет процессом, scheduler, device/emulator либо содержит предметную orchestration policy.

Теги соответствуют определениям Этапа 1. Стабильный `/api/v1` здесь **не подразумевается**.

## S-01 — PyWebIO session shell и аутентификация

- **Назначение:** открыть `index`/`manage`, применить тему, выполнить password gate, создать `AlasGUI` session.
- **Current entry point:** `app()::_run_gui`, `index`, `manage`.
- **Production source:** [`module/webui/app.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app.py) — символы `app`, `_run_gui`, `index`, `manage`; [`module/webui/base.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/base.py) — `Frame`.
- **Composition:** 22 `AlasGUI` mixins + `Frame`.
- **Reads:** deploy theme/password/CDN/Run, browser local storage, user agent.
- **Writes/actions:** session state; при public bind helper может сгенерировать WebUI password и записать его через deploy state.
- **Direct privileged dependencies:** `State`, process startup hook `ProcessManager.restart_processes`.
- **Existing transport:** PyWebIO session via framework-generated HTTP/WebSocket routes.
- **Trust/locality:** password проверяется в `_run_gui`; это не общий Starlette auth middleware.
- **Runtime coupling:** **high** на bootstrap, **medium** на обычную session оболочку.
- **Migration tags:** `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`.
- **Future backend prerequisite:** отделить browser authentication/session contract от PyWebIO session и не переносить process bootstrap в frontend.
- **Неопределённость:** внутренние protocol routes `webio_routes()` не дублируются как project-owned API.

## S-02 — Первоначальная настройка OOBE

- **Назначение:** первый запуск без пользовательской конфигурации: legacy import, сервер, emulator/ADB, review и создание config.
- **Current entry point:** `HomeMixin.run()` → `OOBEWizard(self).start()` при `is_oobe_needed()`.
- **Production source:** [`module/webui/app_home.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_home.py)::`HomeMixin.run`; [`module/webui/oobe.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/oobe.py)::`OOBEWizard`; [`module/webui/oobe_base.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/oobe_base.py)::`_detect_adb_devices`, `_open_legacy_import_picker`, `_create_config_and_finish`.
- **Reads:** template config, deploy ADB executable, ADB devices, server/package metadata.
- **Writes/actions:** POST legacy files; создаёт новый config через `State.config_updater.write_file`.
- **Direct privileged dependencies:** subprocess `adb devices`, filesystem/config updater.
- **Existing transport:** PyWebIO session + `/api/import_legacy_upload`.
- **Trust/locality:** наследует PyWebIO login для wizard, но upload endpoint имеет отдельную ASGI reachability и собственного locality/auth guard в handler не содержит.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `PRIVILEGED_POLICY`, `LEGACY_COMPATIBILITY`.
- **Future backend prerequisite:** формализовать bootstrap/config creation и безопасный import boundary до remote React OOBE.

## S-03 — Home

- **Назначение:** домашняя страница, переход к экземплярам/управлению, выбор темы.
- **Current entry point:** `HomeMixin.show_home`, `HomeMixin.run`.
- **Production source:** [`module/webui/app_home.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_home.py)::`HomeMixin`.
- **Reads:** theme, local storage, выбранный instance, `self.alas.state`.
- **Writes/actions:** `set_theme`, browser navigation; запускает session background config-save thread.
- **Direct privileged dependencies:** косвенно `State.deploy_config` через shell theme; текущий manager state.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Future backend prerequisite:** небольшой instance/session read model; theme policy решить отдельно как frontend preference либо backend deploy setting.

## S-04 — Общий shell и переключатель экземпляров

- **Назначение:** aside/menu/header, список доступных экземпляров и индикаторы их runtime state.
- **Current entry point:** `AppShellMixin.alas_set_aside`, `refresh_aside_instances`.
- **Production source:** [`module/webui/app_shell.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_shell.py)::`AppShellMixin`; [`module/webui/base.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/base.py)::`Frame`.
- **Reads:** `alas_instance()`, `ProcessManager.get_manager(name).state`, menu/args files.
- **Writes/actions:** выбирает instance/session state; тема меняет `State.deploy_config`.
- **Direct privileged dependencies:** `ProcessManager`, `AzurLaneConfig("template")`, config/menu filesystem.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **medium/high**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Future backend prerequisite:** instance list/status read model, не основанный на прямом `ProcessManager` из UI.

## S-05 — Выбор экземпляра

- **Назначение:** открыть существующий профиль и его task menu.
- **Current entry point:** `InstanceMixin.ui_alas`.
- **Production source:** [`module/webui/app_instances.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_instances.py)::`InstanceMixin.ui_alas`.
- **Reads:** `ProcessManager.get_manager`, instance config/module.
- **Writes/actions:** меняет текущий session instance; вызывает `load_config()`.
- **Direct privileged dependencies:** `ProcessManager`, config/submodule loader.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **medium/high**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Future backend prerequisite:** capability «select/read instance context» должна опираться на backend instance model.

## S-06 — Управление экземплярами

- **Назначение:** список, создание, legacy import, export и удаление профилей.
- **Current entry point:** `InstanceMixin.ui_manage` → `app_manage(self)`.
- **Production source:** [`module/webui/app_manage.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_manage.py)::`app_manage`; [`module/webui/app_instances.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_instances.py)::`ui_add_alas`, `ui_import_legacy`.
- **Reads:** config/args JSON, template configs, running state.
- **Writes/actions:** создаёт config, импортирует/экспортирует файлы, удаляет config через `os.remove`, удаляет manager.
- **Direct privileged dependencies:** filesystem, `State.config_updater`, `ProcessManager`.
- **Existing transport:** PyWebIO session; legacy import использует `/api/import_legacy_upload`.
- **Trust/locality:** PyWebIO page защищена session gate; upload handler отдельно locality/auth не проверяет.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `PRIVILEGED_POLICY`, `LEGACY_COMPATIBILITY`.
- **Future backend prerequisite:** безопасная instance lifecycle/config-file boundary с явной политикой create/delete/import/export.

## S-07 — Overview: состояние, запуск/остановка и журнал

- **Назначение:** основной контроль экземпляра, start/stop, runtime log, статистический переход, task queue/ресурсы.
- **Current entry point:** `OverviewMixin.alas_overview`; tool-задачи используют `alas_daemon_overview`.
- **Production source:** [`module/webui/app_overview.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_overview.py)::`OverviewMixin`.
- **Reads:** `self.alas`/manager state, config/scheduler data, resource log, локальный git version, device id.
- **Writes/actions:** `self.alas.start(...)`, `self.alas.stop()`, переключатели session UI.
- **Direct privileged dependencies:** `ProcessManager` session manager, config, device identifier, subprocess `git`.
- **Existing transport:** PyWebIO session; runtime log обновляется server-side.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `REALTIME_CONTRACT`, `SECURITY_REDESIGN`.
- **Future backend prerequisite:** instance lifecycle command boundary + runtime status/log streaming contract с authz/audit.

## S-08 — Dashboard

- **Назначение:** read-only сводка задач и ресурсов.
- **Current entry point:** `DashboardMixin._dashboard`.
- **Production source:** [`module/webui/app_dashboard.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_dashboard.py)::`DashboardMixin`.
- **Reads:** `self.alas_config.load()`, scheduler/task state, `self.alas.alive`, `LogRes`.
- **Writes/actions:** пользовательских state-changing действий не подтверждено.
- **Direct privileged dependencies:** config/runtime manager read path.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Future backend prerequisite:** формализовать dashboard/task/resource read model, не отдавать raw config как UI contract.

## S-09 — Generic Task Configuration

- **Назначение:** конфигурационные формы для task destinations из `ALAS_MENU`.
- **Current entry point:** `TaskConfigMixin.alas_set_menu` → `alas_set_group`.
- **Production source:** [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`TaskConfigMixin`; [`module/config/argument/menu.json`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/argument/menu.json).
- **Composition:** pinned menu содержит **90 task destinations** в 9 группах; `page=setting` использует общий form engine.
- **Reads:** `ALAS_ARGS`, instance config, server/package-specific options.
- **Writes/actions:** pin watcher → queue → `_save_config`; UI валидирует значения, вызывает `config_updater.save_callback` и пишет config.
- **Direct privileged dependencies:** `State.config_updater`, config filesystem; `get_device_id` для DropRecord.
- **Existing transport:** PyWebIO session; не существует единого current HTTP contract для generic save.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** shared validated configuration command/read boundary. Browser не должен копировать `parse_pin_value`, `save_callback` и предметные side effects.

## S-10 — Tool/daemon task execution family

- **Назначение:** 10 `page=tool` destinations (`Daemon`, `OpsiDaemon`, `EventStory`, `BoxDisassemble`, `AutoEquip`, `Benchmark`, `OcrBenchmark`, `ScreenshotIntervalBenchmark`, `GameManager`, `EmulatorManager`).
- **Current entry point:** `TaskConfigMixin.alas_set_menu` направляет `page=tool` в `OverviewMixin.alas_daemon_overview`.
- **Production source:** [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`alas_set_menu`; [`module/webui/app_overview.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_overview.py)::`alas_daemon_overview`; [`module/config/argument/menu.json`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/argument/menu.json).
- **Reads:** runtime manager/log/config для выбранной tool task.
- **Writes/actions:** start/stop конкретной task через manager.
- **Direct privileged dependencies:** process/task runtime.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `INTERNAL_DEVELOPER`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Future backend prerequisite:** отдельное product решение о remote exposure; если переносится — использовать общий task command policy.

## S-11 — Startup-run для экземпляра

- **Назначение:** включить/выключить автоматический запуск instance вместе с WebUI.
- **Current entry point:** `TaskConfigMixin._render_startup_run_setting` внутри task `Alas`.
- **Production source:** [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`_render_startup_run_setting`; [`module/webui/api.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/api.py)::`api_deploy_startup_run*`; [`module/webui/deploy_settings.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/deploy_settings.py)::`get_startup_run`, `set_startup_run`.
- **Reads:** deploy `Run`.
- **Writes/actions:** меняет deploy `Run`.
- **Direct privileged dependencies:** deploy config persistence.
- **Existing transport:** GET/POST `/api/deploy/startup-run`.
- **Trust/locality:** оба handler требуют current `is_local_request()`.
- **Runtime coupling:** **medium/high**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`.
- **Future backend prerequisite:** определить authenticated remote policy; current localhost contract за reverse proxy непригоден.

## S-12 — Statistics hub

- **Назначение:** единая страница, собирающая пять статистических read views и периодические refresh tasks.
- **Current entry point:** `StatisticsPageMixin.alas_set_stat`.
- **Production source:** [`module/webui/app_statistics_page.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_statistics_page.py)::`StatisticsPageMixin`.
- **Reads:** через пять специализированных statistics mixins.
- **Writes/actions:** только client/session view state; refresh.
- **Direct privileged dependencies:** прямых process/device writes не подтверждено.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO password gate.
- **Runtime coupling:** **low/medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** набор стабильных statistics read models.

## S-13 — Action Point statistics

- **Назначение:** AP/coins/assets timeline и line/candlestick/detail views.
- **Current entry point:** `ActionPointStatisticsMixin._render_ap_chart`.
- **Production source:** [`module/webui/app_stat_action_point.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_action_point.py)::`_load_ap_chart_timelines`, `_build_ap_chart_*`; toolbar — [`app_stat_action_point_toolbar.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_action_point_toolbar.py).
- **Reads:** `module.statistics.opsi_month` timelines.
- **Writes/actions:** только session view selector.
- **Direct privileged dependencies:** statistics storage через backend module.
- **Existing transport:** current PyWebIO читает module напрямую; отдельно существует `/api/ap_timeline`, используемый OBS overlay.
- **Trust/locality:** PyWebIO gate; `/api/ap_timeline` собственного auth/locality guard не содержит.
- **Runtime coupling:** **medium** из-за server-side aggregation.
- **Migration tags:** `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** единый statistics read model, чтобы не дублировать Python aggregation в React.

## S-14 — Resource statistics

- **Назначение:** timeline нефти, монет, кубов, Pt, AP и других ресурсов.
- **Current entry point:** `ResourceStatisticsMixin._render_resource_chart`.
- **Production source:** [`module/webui/app_stat_resource.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_resource.py)::`_load_resource_timeline`, `_build_resource_series`.
- **Reads:** `get_resource_timeline(instance, limit=500)`.
- **Writes/actions:** нет подтверждённых state changes.
- **Direct privileged dependencies:** statistics storage через backend module.
- **Existing transport:** PyWebIO session; отдельного current route для полной resource timeline не найдено в `api_routes`.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** resource history read model/contract.

## S-15 — Operation Siren statistics

- **Назначение:** monthly CL1/Meow/ship-exp эффективность и связанные статистики.
- **Current entry point:** `OpsiStatisticsMixin._render_opsi_stats`.
- **Production source:** [`module/webui/app_stat_opsi.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_opsi.py)::`OpsiStatisticsMixin`.
- **Reads:** `get_opsi_stats`, `cl1_database`, `get_ship_exp_stats`, monthly calculations.
- **Writes/actions:** refresh; экспорт вынесен в S-16.
- **Direct privileged dependencies:** statistics persistence/helpers.
- **Existing transport:** PyWebIO; `/api/cl1_stats` содержит частичный CL1 read interface, но не покрывает весь view.
- **Trust/locality:** PyWebIO gate; `/api/cl1_stats` собственного auth/locality guard не содержит.
- **Runtime coupling:** **medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** формализовать read model, не считать `/api/cl1_stats` готовым финальным контрактом.

## S-16 — Operation Siren export и Meow refresh

- **Назначение:** обновить Meow farming данные и экспортировать monthly CSV на Desktop.
- **Current entry point:** `OpsiExportMixin._refresh_meowofficer_farming`, `_export_opsi_csv`.
- **Production source:** [`module/webui/app_stat_opsi_export.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_opsi_export.py)::`OpsiExportMixin`.
- **Reads:** `AzurStats`, Operation Siren statistics.
- **Writes/actions:** refresh backend statistics helper; пишет CSV в `Path.home()/Desktop`.
- **Direct privileged dependencies:** локальная filesystem пользователя.
- **Existing transport:** PyWebIO session, server-side filesystem write.
- **Trust/locality:** функционально привязано к локальной машине.
- **Runtime coupling:** **high** для export.
- **Migration tags:** `LEGACY_COMPATIBILITY`, `SERVICE_EXTRACTION`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Future backend prerequisite:** решить продуктовую семантику export: browser download или backend-controlled export; не писать Desktop из React semantics.

## S-17 — Ship experience statistics

- **Назначение:** прогресс кораблей, время/EXP efficiency, daily statistics.
- **Current entry point:** `ShipExperienceStatisticsMixin._render_ship_exp`.
- **Production source:** [`module/webui/app_stat_ship.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_ship.py)::`ShipExperienceStatisticsMixin`.
- **Reads:** `get_ship_exp_stats`, `get_opsi_stats`.
- **Writes/actions:** refresh only.
- **Direct privileged dependencies:** statistics storage/helpers.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** ship-exp read model.

## S-18 — Commission income statistics

- **Назначение:** day/week/month summaries и recent commission rewards.
- **Current entry point:** `CommissionIncomeStatisticsMixin._render_commission_income`.
- **Production source:** [`module/webui/app_stat_commission.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_commission.py)::`CommissionIncomeStatisticsMixin`.
- **Reads:** `get_commission_income_summary`, `get_recent_commission_entries`.
- **Writes/actions:** только session period selector.
- **Direct privileged dependencies:** statistics persistence/helpers.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** commission income read model.

## S-19 — Event profiles и event-aware menu

- **Назначение:** выбор/создание/переименование/удаление профилей Event и перестройка Event menu.
- **Current entry point:** `EventProfilesMixin._rewrite_event_menu_for_profiles` и profile callbacks.
- **Production source:** [`module/webui/app_event_profiles.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_profiles.py)::`EventProfilesMixin`.
- **Reads:** event config/profile registry.
- **Writes/actions:** мутирует event config; удаление профиля может менять scheduler state.
- **Direct privileged dependencies:** config persistence.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`.
- **Future backend prerequisite:** event profile command service с валидацией и scheduler-safe semantics.

## S-20 — Event datamine/source policy

- **Назначение:** выбрать/активировать источник Event data и показать diagnostics.
- **Current entry point:** `EventDatamineMixin._activate_generated_event_source` и related render methods.
- **Production source:** [`module/webui/app_event_datamine.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_datamine.py)::`EventDatamineMixin`.
- **Reads:** event source/datamine artifacts, config Pt, user policy state.
- **Writes/actions:** сохраняет event user state/policy.
- **Direct privileged dependencies:** local persistence/artifact registry.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **medium/high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** source/diagnostic read model + explicit mutation boundary for user source policy.

## S-21 — Event plan/layout

- **Назначение:** визуальный event plan, настройки Pt/time/shop и planner interactions.
- **Current entry point:** `EventLayoutMixin` (inherits `EventPlannerMixin`).
- **Production source:** [`module/webui/app_event_layout.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_layout.py)::`EventLayoutMixin`; [`module/webui/app_event_planner.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_planner.py)::`EventPlannerMixin`.
- **Reads:** event config, observations, datamine/source data, plan persistence.
- **Writes/actions:** изменяет event plan/user policy и config, включая Pt target.
- **Direct privileged dependencies:** config/persistent event state.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `CONTRACT_FORMALIZATION`.
- **Future backend prerequisite:** event planning domain boundary до React editing.

## S-22 — EventShop safety bridge

- **Назначение:** fail-closed синхронизация визуального shop plan с EventShop automation.
- **Current entry point:** `EventShopSafetyMixin._event_plan_write`, `_sync_shop_plan_fail_closed`.
- **Production source:** [`module/webui/app_event_shop_safety.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_shop_safety.py)::`EventShopSafetyMixin`.
- **Reads:** shop plan, EventShop Scheduler.Enable, PtLimit.
- **Writes/actions:** может выключить Scheduler, меняет preset/custom filter и safety flags.
- **Direct privileged dependencies:** automation config/scheduler policy.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate.
- **Runtime coupling:** **high**.
- **Migration tags:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`.
- **Future backend prerequisite:** fail-closed shop automation policy должна принадлежать backend service/policy layer, не browser callback.

## S-23 — Operation Siren simulator

- **Назначение:** автономный simulator UI, start/interrupt, log и generated figure.
- **Current entry point:** task `OpsiSimulator` → `TaskConfigMixin.alas_set_group` → `EventToolsMixin._os_simulator`.
- **Production source:** [`module/webui/app_event_tools.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_tools.py)::`EventToolsMixin._os_simulator`; [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`_simulator_start`.
- **Reads:** simulator config/log/figure path.
- **Writes/actions:** start/interrupt simulator; читает локальный image-файл.
- **Direct privileged dependencies:** simulator runtime, filesystem.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate; demo intentionally suppresses start.
- **Runtime coupling:** **high**.
- **Migration tags:** `INTERNAL_DEVELOPER`, `SERVICE_EXTRACTION`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Future backend prerequisite:** отдельное product решение; если нужен remote UI — simulator service/read stream boundary.

## S-24 — Developer: Remote Access status

- **Назначение:** показать адрес/состояние/error существующего remote access provider.
- **Current entry point:** `DeveloperToolsMixin.dev_remote`.
- **Production source:** [`module/webui/app_developer_tools.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_developer_tools.py)::`dev_remote`; [`module/webui/remote_access.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/remote_access.py).
- **Reads:** `RemoteAccess.get_state/get_entry_point/get_connection_state/get_error`, deploy flags.
- **Writes/actions:** в этом view прямые state changes не подтверждены.
- **Direct privileged dependencies:** network/tunnel runtime.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** view скрывает remote entry point без password/demo; remote service itself может tunnel'ить WebUI.
- **Runtime coupling:** **medium/high**.
- **Migration tags:** `INTERNAL_DEVELOPER`, `SECURITY_REDESIGN`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Future backend prerequisite:** определить product/security роль remote access после Caddy; не переносить старый tunnel control автоматически.

## S-25 — Developer: Deploy/Launcher settings

- **Назначение:** launcher status/autostart и редактирование `deploy.yaml`.
- **Current entry point:** `DeveloperSettingsMixin.dev_setting`.
- **Production source:** [`module/webui/app_developer_settings.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_developer_settings.py)::`DeveloperSettingsMixin`; [`module/webui/deploy_settings.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/deploy_settings.py).
- **Reads:** `/api/launcher/status`, `/api/deploy/settings`.
- **Writes/actions:** POST launcher autostart; POST deploy settings.
- **Direct privileged dependencies:** через current HTTP handlers — launcher IPC и deploy config persistence.
- **Existing transport:** current partial HTTP endpoints.
- **Trust/locality:** mutating launcher/deploy handlers требуют `is_local_request`; deploy read тоже local-only.
- **Runtime coupling:** **medium/high**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`, `INTERNAL_DEVELOPER`.
- **Future backend prerequisite:** отдельный authenticated administrative contract; localhost semantics не масштабировать на remote UI.

## S-26 — Developer utilities

- **Назначение:** debug state overrides, restart WebUI, test notification.
- **Current entry point:** `DeveloperToolsMixin.dev_utils`.
- **Production source:** [`module/webui/app_developer_tools.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_developer_tools.py)::`dev_utils`, `request_webui_restart`, `prepare_webui_restart`.
- **Reads:** running instances, current manager states.
- **Writes/actions:** `ProcessManager.set_state_override`, restart event/cleanup, reload marker, notification.
- **Direct privileged dependencies:** process lifecycle, filesystem marker, process-wide State, notification config.
- **Existing transport:** PyWebIO session.
- **Trust/locality:** PyWebIO gate; developer visibility не является authorization model для будущего remote exposure.
- **Runtime coupling:** **high**.
- **Migration tags:** `INTERNAL_DEVELOPER`, `PRIVILEGED_POLICY`, `SECURITY_REDESIGN`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Future backend prerequisite:** не публиковать без отдельного admin policy/audit product decision.

## S-27 — OBS overlay

- **Назначение:** standalone browser/OBS read-only CL1/AP overlay.
- **Current entry point:** GET `/obs`.
- **Production source:** [`module/webui/api.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/api.py)::`serve_obs_overlay`; [`module/webui/obs_overlay.html`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/obs_overlay.html)::`fetchData`.
- **Reads:** `/api/cl1_stats`, `/api/ap_timeline` по `instance` query parameter.
- **Writes/actions:** нет state-changing действий.
- **Direct privileged dependencies:** statistics modules через два API handler.
- **Existing transport:** HTTP polling каждые 10 секунд.
- **Trust/locality:** три handler (`/obs`, CL1, AP) собственного auth/locality guard не содержат.
- **Runtime coupling:** **low/medium**.
- **Migration tags:** `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`.
- **Future backend prerequisite:** решить, остаётся ли OBS отдельным compatibility client; read endpoints должны получить явный auth/exposure contract.

## Замечание о числе surfaces

Карта содержит **27** записей. Число не равно 90 task destinations: task definitions используют общий form/execution wiring и разбиваются только там, где код создаёт отдельный workflow или особую dependency boundary. Декоративные widgets отдельными surfaces не считаются.
