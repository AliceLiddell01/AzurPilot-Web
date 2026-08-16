# Текущие пользовательские поверхности WebUI

Источник аудита: `AliceLiddell01/AzurPilot-private-Ru@3be3696975cb91ba0b85dbea98400381c3ced379`.

## Как читать карту

Связность со средой выполнения:

- **low** — преимущественно представление и состояние клиентской сессии;
- **medium** — UI напрямую собирает модель чтения или использует backend helper, но не управляет привилегированной средой выполнения;
- **high** — callback/handler сам пишет конфигурацию/файлы, управляет процессом, scheduler, device/emulator либо содержит предметную orchestration policy.

Теги соответствуют определениям Этапа 1. Стабильный `/api/v1` здесь **не подразумевается**.

## S-01 — PyWebIO session shell и аутентификация

- **Назначение:** открыть `index`/`manage`, применить тему, выполнить password gate, создать `AlasGUI` session.
- **Текущая точка входа:** `app()::_run_gui`, `index`, `manage`.
- **Рабочий исходный код:** [`module/webui/app.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app.py) — символы `app`, `_run_gui`, `index`, `manage`; [`module/webui/base.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/base.py) — `Frame`.
- **Состав:** 22 `AlasGUI` mixins + `Frame`.
- **Читает:** deploy theme/password/CDN/Run, browser local storage, user agent.
- **Записи/действия:** session state; при public bind helper может сгенерировать WebUI password и записать его через deploy state.
- **Прямые привилегированные зависимости:** `State`, process startup hook `ProcessManager.restart_processes`.
- **Существующий транспорт:** PyWebIO session через маршруты, генерируемые framework.
- **Доверие/локальность:** password проверяется в `_run_gui`; это не общий Starlette auth middleware.
- **Связность со средой выполнения:** **high** на bootstrap, **medium** на обычную session оболочку.
- **Теги миграции:** `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`.
- **Требование к бэкенду:** отделить browser authentication/session contract от PyWebIO session и не переносить process bootstrap во frontend.
- **Неопределённость:** внутренние protocol routes `webio_routes()` не дублируются как project-owned API.

## S-02 — Первоначальная настройка OOBE

- **Назначение:** первый запуск без пользовательской конфигурации: legacy import, сервер, emulator/ADB, review и создание config.
- **Текущая точка входа:** `HomeMixin.run()` → `OOBEWizard(self).start()` при `is_oobe_needed()`.
- **Рабочий исходный код:** [`module/webui/app_home.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_home.py)::`HomeMixin.run`; [`module/webui/oobe.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/oobe.py)::`OOBEWizard`; [`module/webui/oobe_base.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/oobe_base.py)::`_detect_adb_devices`, `_open_legacy_import_picker`, `_create_config_and_finish`.
- **Читает:** template config, deploy ADB executable, ADB devices, server/package metadata.
- **Записи/действия:** POST legacy files; создаёт новый config через `State.config_updater.write_file`.
- **Прямые привилегированные зависимости:** subprocess `adb devices`, filesystem/config updater.
- **Существующий транспорт:** PyWebIO session + `/api/import_legacy_upload`.
- **Доверие/локальность:** wizard наследует PyWebIO login, но upload endpoint имеет отдельную ASGI reachability и собственного locality/auth guard в handler не содержит.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `PRIVILEGED_POLICY`, `LEGACY_COMPATIBILITY`.
- **Требование к бэкенду:** формализовать bootstrap/config creation и безопасный import boundary до remote React OOBE.

## S-03 — Home

- **Назначение:** домашняя страница, переход к экземплярам/управлению, выбор темы.
- **Текущая точка входа:** `HomeMixin.show_home`, `HomeMixin.run`.
- **Рабочий исходный код:** [`module/webui/app_home.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_home.py)::`HomeMixin`.
- **Читает:** theme, local storage, выбранный instance, `self.alas.state`.
- **Записи/действия:** `set_theme`, browser navigation; запускает session background config-save thread.
- **Прямые привилегированные зависимости:** косвенно `State.deploy_config` через shell theme; текущий manager state.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Требование к бэкенду:** небольшой instance/session read model; theme policy решить отдельно как frontend preference либо backend deploy setting.

## S-04 — Общий shell и переключатель экземпляров

- **Назначение:** aside/menu/header, список доступных экземпляров и индикаторы их runtime state.
- **Текущая точка входа:** `AppShellMixin.alas_set_aside`, `refresh_aside_instances`.
- **Рабочий исходный код:** [`module/webui/app_shell.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_shell.py)::`AppShellMixin`; [`module/webui/base.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/base.py)::`Frame`.
- **Читает:** `alas_instance()`, `ProcessManager.get_manager(name).state`, menu/args files.
- **Записи/действия:** выбирает instance/session state; тема меняет `State.deploy_config`.
- **Прямые привилегированные зависимости:** `ProcessManager`, `AzurLaneConfig("template")`, config/menu filesystem.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **medium/high**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Требование к бэкенду:** instance list/status read model, не основанный на прямом `ProcessManager` из UI.

## S-05 — Выбор экземпляра

- **Назначение:** открыть существующий профиль и его task menu.
- **Текущая точка входа:** `InstanceMixin.ui_alas`.
- **Рабочий исходный код:** [`module/webui/app_instances.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_instances.py)::`InstanceMixin.ui_alas`.
- **Читает:** `ProcessManager.get_manager`, instance config/module.
- **Записи/действия:** меняет текущий session instance; вызывает `load_config()`.
- **Прямые привилегированные зависимости:** `ProcessManager`, config/submodule loader.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **medium/high**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Требование к бэкенду:** capability «select/read instance context» должна опираться на backend instance model.

## S-06 — Управление экземплярами

- **Назначение:** список, создание, legacy import, export и удаление профилей.
- **Текущая точка входа:** `InstanceMixin.ui_manage` → `app_manage(self)`.
- **Рабочий исходный код:** [`module/webui/app_manage.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_manage.py)::`app_manage`; [`module/webui/app_instances.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_instances.py)::`ui_add_alas`, `ui_import_legacy`.
- **Читает:** config/args JSON, template configs, running state.
- **Записи/действия:** создаёт config, импортирует/экспортирует файлы, удаляет config через `os.remove`, удаляет manager.
- **Прямые привилегированные зависимости:** filesystem, `State.config_updater`, `ProcessManager`.
- **Существующий транспорт:** PyWebIO session; legacy import использует `/api/import_legacy_upload`.
- **Доверие/локальность:** PyWebIO page защищена session gate; upload handler отдельно locality/auth не проверяет.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `SECURITY_REDESIGN`, `PRIVILEGED_POLICY`, `LEGACY_COMPATIBILITY`.
- **Требование к бэкенду:** безопасная instance lifecycle/config-file boundary с явной политикой create/delete/import/export.

## S-07 — Overview: состояние, запуск/остановка и журнал

- **Назначение:** основной контроль экземпляра, start/stop, runtime log, статистический переход, task queue/ресурсы.
- **Текущая точка входа:** `OverviewMixin.alas_overview`; tool-задачи используют `alas_daemon_overview`.
- **Рабочий исходный код:** [`module/webui/app_overview.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_overview.py)::`OverviewMixin`.
- **Читает:** `self.alas`/manager state, config/scheduler data, resource log, локальный git version, device id.
- **Записи/действия:** `self.alas.start(...)`, `self.alas.stop()`, переключатели session UI.
- **Прямые привилегированные зависимости:** `ProcessManager` session manager, config, device identifier, subprocess `git`.
- **Существующий транспорт:** PyWebIO session; runtime log обновляется server-side.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `REALTIME_CONTRACT`, `SECURITY_REDESIGN`.
- **Требование к бэкенду:** instance lifecycle command boundary + runtime status/log streaming contract с authz/audit.

## S-08 — Dashboard

- **Назначение:** read-only сводка задач и ресурсов.
- **Текущая точка входа:** `DashboardMixin._dashboard`.
- **Рабочий исходный код:** [`module/webui/app_dashboard.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_dashboard.py)::`DashboardMixin`.
- **Читает:** `self.alas_config.load()`, scheduler/task state, `self.alas.alive`, `LogRes`.
- **Записи/действия:** пользовательских state-changing действий не подтверждено.
- **Прямые привилегированные зависимости:** config/runtime manager read path.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SERVICE_EXTRACTION`.
- **Требование к бэкенду:** формализовать dashboard/task/resource read model, не отдавать raw config как UI contract.

## S-09 — Generic Task Configuration

- **Назначение:** конфигурационные формы для task destinations из `ALAS_MENU`.
- **Текущая точка входа:** `TaskConfigMixin.alas_set_menu` → `alas_set_group`.
- **Рабочий исходный код:** [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`TaskConfigMixin`; [`module/config/argument/menu.json`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/argument/menu.json).
- **Состав:** pinned menu содержит **90 task destinations** в 9 группах; `page=setting` использует общий form engine.
- **Читает:** `ALAS_ARGS`, instance config, server/package-specific options.
- **Записи/действия:** pin watcher → queue → `_save_config`; UI валидирует значения, вызывает `config_updater.save_callback` и пишет config.
- **Прямые привилегированные зависимости:** `State.config_updater`, config filesystem; `get_device_id` для DropRecord.
- **Существующий транспорт:** PyWebIO session; не существует единого current HTTP contract для generic save.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** shared validated configuration command/read boundary. Browser не должен копировать `parse_pin_value`, `save_callback` и предметные side effects.

## S-10 — Tool/daemon task execution family

- **Назначение:** 10 `page=tool` destinations (`Daemon`, `OpsiDaemon`, `EventStory`, `BoxDisassemble`, `AutoEquip`, `Benchmark`, `OcrBenchmark`, `ScreenshotIntervalBenchmark`, `GameManager`, `EmulatorManager`).
- **Текущая точка входа:** `TaskConfigMixin.alas_set_menu` направляет `page=tool` в `OverviewMixin.alas_daemon_overview`.
- **Рабочий исходный код:** [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`alas_set_menu`; [`module/webui/app_overview.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_overview.py)::`alas_daemon_overview`; [`module/config/argument/menu.json`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/config/argument/menu.json).
- **Читает:** runtime manager/log/config для выбранной tool task.
- **Записи/действия:** start/stop конкретной task через manager.
- **Прямые привилегированные зависимости:** process/task runtime.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `INTERNAL_DEVELOPER`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Требование к бэкенду:** отдельное product решение о remote exposure; если переносится — использовать общий task command policy.

## S-11 — Startup-run для экземпляра

- **Назначение:** включить/выключить автоматический запуск instance вместе с WebUI.
- **Текущая точка входа:** `TaskConfigMixin._render_startup_run_setting` внутри task `Alas`.
- **Рабочий исходный код:** [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`_render_startup_run_setting`; [`module/webui/api.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/api.py)::`api_deploy_startup_run*`; [`module/webui/deploy_settings.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/deploy_settings.py)::`get_startup_run`, `set_startup_run`.
- **Читает:** deploy `Run`.
- **Записи/действия:** меняет deploy `Run`.
- **Прямые привилегированные зависимости:** deploy config persistence.
- **Существующий транспорт:** GET/POST `/api/deploy/startup-run`.
- **Доверие/локальность:** оба handler требуют current `is_local_request()`.
- **Связность со средой выполнения:** **medium/high**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`.
- **Требование к бэкенду:** определить authenticated remote policy; current localhost contract за reverse proxy непригоден.

## S-12 — Statistics hub

- **Назначение:** единая страница, собирающая пять статистических read views и периодические refresh tasks.
- **Текущая точка входа:** `StatisticsPageMixin.alas_set_stat`.
- **Рабочий исходный код:** [`module/webui/app_statistics_page.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_statistics_page.py)::`StatisticsPageMixin`.
- **Читает:** через пять специализированных statistics mixins.
- **Записи/действия:** только client/session view state; refresh.
- **Прямые привилегированные зависимости:** прямых process/device writes не подтверждено.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO password gate.
- **Связность со средой выполнения:** **low/medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** набор стабильных statistics read models.

## S-13 — Action Point statistics

- **Назначение:** AP/coins/assets timeline и line/candlestick/detail views.
- **Текущая точка входа:** `ActionPointStatisticsMixin._render_ap_chart`.
- **Рабочий исходный код:** [`module/webui/app_stat_action_point.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_action_point.py)::`_load_ap_chart_timelines`, `_build_ap_chart_*`; toolbar — [`app_stat_action_point_toolbar.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_action_point_toolbar.py).
- **Читает:** `module.statistics.opsi_month` timelines.
- **Записи/действия:** только session view selector.
- **Прямые привилегированные зависимости:** statistics storage через backend module.
- **Существующий транспорт:** current PyWebIO читает module напрямую; отдельно существует `/api/ap_timeline`, используемый OBS overlay.
- **Доверие/локальность:** PyWebIO gate; `/api/ap_timeline` собственного auth/locality guard не содержит.
- **Связность со средой выполнения:** **medium** из-за server-side aggregation.
- **Теги миграции:** `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** единый statistics read model, чтобы не дублировать Python aggregation в React.

## S-14 — Resource statistics

- **Назначение:** timeline нефти, монет, кубов, Pt, AP и других ресурсов.
- **Текущая точка входа:** `ResourceStatisticsMixin._render_resource_chart`.
- **Рабочий исходный код:** [`module/webui/app_stat_resource.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_resource.py)::`_load_resource_timeline`, `_build_resource_series`.
- **Читает:** `get_resource_timeline(instance, limit=500)`.
- **Записи/действия:** нет подтверждённых state changes.
- **Прямые привилегированные зависимости:** statistics storage через backend module.
- **Существующий транспорт:** PyWebIO session; отдельного current route для полной resource timeline не найдено в `api_routes`.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** resource history read model/contract.

## S-15 — Operation Siren statistics

- **Назначение:** monthly CL1/Meow/ship-exp эффективность и связанные статистики.
- **Текущая точка входа:** `OpsiStatisticsMixin._render_opsi_stats`.
- **Рабочий исходный код:** [`module/webui/app_stat_opsi.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_opsi.py)::`OpsiStatisticsMixin`.
- **Читает:** `get_opsi_stats`, `cl1_database`, `get_ship_exp_stats`, monthly calculations.
- **Записи/действия:** refresh; экспорт вынесен в S-16.
- **Прямые привилегированные зависимости:** statistics persistence/helpers.
- **Существующий транспорт:** PyWebIO; `/api/cl1_stats` содержит частичный CL1 read interface, но не покрывает весь view.
- **Доверие/локальность:** PyWebIO gate; `/api/cl1_stats` собственного auth/locality guard не содержит.
- **Связность со средой выполнения:** **medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** формализовать read model, не считать `/api/cl1_stats` готовым финальным контрактом.

## S-16 — Operation Siren export и Meow refresh

- **Назначение:** обновить Meow farming данные и экспортировать monthly CSV на Desktop.
- **Текущая точка входа:** `OpsiExportMixin._refresh_meowofficer_farming`, `_export_opsi_csv`.
- **Рабочий исходный код:** [`module/webui/app_stat_opsi_export.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_opsi_export.py)::`OpsiExportMixin`.
- **Читает:** `AzurStats`, Operation Siren statistics.
- **Записи/действия:** refresh backend statistics helper; пишет CSV в `Path.home()/Desktop`.
- **Прямые привилегированные зависимости:** локальная filesystem пользователя.
- **Существующий транспорт:** PyWebIO session, server-side filesystem write.
- **Доверие/локальность:** функционально привязано к локальной машине.
- **Связность со средой выполнения:** **high** для export.
- **Теги миграции:** `LEGACY_COMPATIBILITY`, `SERVICE_EXTRACTION`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Требование к бэкенду:** решить продуктовую семантику export: browser download или backend-controlled export; не писать Desktop из React semantics.

## S-17 — Ship experience statistics

- **Назначение:** прогресс кораблей, время/EXP efficiency, daily statistics.
- **Текущая точка входа:** `ShipExperienceStatisticsMixin._render_ship_exp`.
- **Рабочий исходный код:** [`module/webui/app_stat_ship.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_ship.py)::`ShipExperienceStatisticsMixin`.
- **Читает:** `get_ship_exp_stats`, `get_opsi_stats`.
- **Записи/действия:** refresh only.
- **Прямые привилегированные зависимости:** statistics storage/helpers.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** ship-exp read model.

## S-18 — Commission income statistics

- **Назначение:** day/week/month summaries и recent commission rewards.
- **Текущая точка входа:** `CommissionIncomeStatisticsMixin._render_commission_income`.
- **Рабочий исходный код:** [`module/webui/app_stat_commission.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_stat_commission.py)::`CommissionIncomeStatisticsMixin`.
- **Читает:** `get_commission_income_summary`, `get_recent_commission_entries`.
- **Записи/действия:** только session period selector.
- **Прямые привилегированные зависимости:** statistics persistence/helpers.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** commission income read model.

## S-19 — Event profiles и event-aware menu

- **Назначение:** выбор/создание/переименование/удаление профилей Event и перестройка Event menu.
- **Текущая точка входа:** `EventProfilesMixin._rewrite_event_menu_for_profiles` и profile callbacks.
- **Рабочий исходный код:** [`module/webui/app_event_profiles.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_profiles.py)::`EventProfilesMixin`.
- **Читает:** event config/profile registry.
- **Записи/действия:** мутирует event config; удаление профиля может менять scheduler state.
- **Прямые привилегированные зависимости:** config persistence.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`.
- **Требование к бэкенду:** event profile command service с валидацией и scheduler-safe semantics.

## S-20 — Event datamine/source policy

- **Назначение:** выбрать/активировать источник Event data и показать diagnostics.
- **Текущая точка входа:** `EventDatamineMixin._activate_generated_event_source` и related render methods.
- **Рабочий исходный код:** [`module/webui/app_event_datamine.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_datamine.py)::`EventDatamineMixin`.
- **Читает:** event source/datamine artifacts, config Pt, user policy state.
- **Записи/действия:** сохраняет event user state/policy.
- **Прямые привилегированные зависимости:** local persistence/artifact registry.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **medium/high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** source/diagnostic read model + explicit mutation boundary for user source policy.

## S-21 — Event plan/layout

- **Назначение:** визуальный event plan, настройки Pt/time/shop и planner interactions.
- **Текущая точка входа:** `EventLayoutMixin` (inherits `EventPlannerMixin`).
- **Рабочий исходный код:** [`module/webui/app_event_layout.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_layout.py)::`EventLayoutMixin`; [`module/webui/app_event_planner.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_planner.py)::`EventPlannerMixin`.
- **Читает:** event config, observations, datamine/source data, plan persistence.
- **Записи/действия:** изменяет event plan/user policy и config, включая Pt target.
- **Прямые привилегированные зависимости:** config/persistent event state.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`, `CONTRACT_FORMALIZATION`.
- **Требование к бэкенду:** event planning domain boundary до React editing.

## S-22 — EventShop safety bridge

- **Назначение:** fail-closed синхронизация визуального shop plan с EventShop automation.
- **Текущая точка входа:** `EventShopSafetyMixin._event_plan_write`, `_sync_shop_plan_fail_closed`.
- **Рабочий исходный код:** [`module/webui/app_event_shop_safety.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_shop_safety.py)::`EventShopSafetyMixin`.
- **Читает:** shop plan, EventShop Scheduler.Enable, PtLimit.
- **Записи/действия:** может выключить Scheduler, меняет preset/custom filter и safety flags.
- **Прямые привилегированные зависимости:** automation config/scheduler policy.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `SERVICE_EXTRACTION`, `PRIVILEGED_POLICY`.
- **Требование к бэкенду:** fail-closed shop automation policy должна принадлежать backend service/policy layer, не browser callback.

## S-23 — Operation Siren simulator

- **Назначение:** автономный simulator UI, start/interrupt, log и generated figure.
- **Текущая точка входа:** task `OpsiSimulator` → `TaskConfigMixin.alas_set_group` → `EventToolsMixin._os_simulator`.
- **Рабочий исходный код:** [`module/webui/app_event_tools.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_event_tools.py)::`EventToolsMixin._os_simulator`; [`module/webui/app_task_config.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_task_config.py)::`_simulator_start`.
- **Читает:** simulator config/log/figure path.
- **Записи/действия:** start/interrupt simulator; читает локальный image-файл.
- **Прямые привилегированные зависимости:** simulator runtime, filesystem.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate; demo intentionally suppresses start.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `INTERNAL_DEVELOPER`, `SERVICE_EXTRACTION`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Требование к бэкенду:** отдельное product решение; если нужен remote UI — simulator service/read stream boundary.

## S-24 — Developer: Remote Access status

- **Назначение:** показать адрес/состояние/error существующего remote access provider.
- **Текущая точка входа:** `DeveloperToolsMixin.dev_remote`.
- **Рабочий исходный код:** [`module/webui/app_developer_tools.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_developer_tools.py)::`dev_remote`; [`module/webui/remote_access.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/remote_access.py).
- **Читает:** `RemoteAccess.get_state/get_entry_point/get_connection_state/get_error`, deploy flags.
- **Записи/действия:** в этом view прямые state changes не подтверждены.
- **Прямые привилегированные зависимости:** network/tunnel runtime.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** view скрывает remote entry point без password/demo; remote service itself может tunnel'ить WebUI.
- **Связность со средой выполнения:** **medium/high**.
- **Теги миграции:** `INTERNAL_DEVELOPER`, `SECURITY_REDESIGN`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Требование к бэкенду:** определить product/security роль remote access после Caddy; не переносить старый tunnel control автоматически.

## S-25 — Developer: Deploy/Launcher settings

- **Назначение:** launcher status/autostart и редактирование `deploy.yaml`.
- **Текущая точка входа:** `DeveloperSettingsMixin.dev_setting`.
- **Рабочий исходный код:** [`module/webui/app_developer_settings.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_developer_settings.py)::`DeveloperSettingsMixin`; [`module/webui/deploy_settings.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/deploy_settings.py).
- **Читает:** `/api/launcher/status`, `/api/deploy/settings`.
- **Записи/действия:** POST launcher autostart; POST deploy settings.
- **Прямые привилегированные зависимости:** через current HTTP handlers — launcher IPC и deploy config persistence.
- **Существующий транспорт:** current partial HTTP endpoints.
- **Доверие/локальность:** mutating launcher/deploy handlers требуют `is_local_request`; deploy read тоже local-only.
- **Связность со средой выполнения:** **medium/high**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`, `INTERNAL_DEVELOPER`.
- **Требование к бэкенду:** отдельный authenticated administrative contract; localhost semantics не масштабировать на remote UI.

## S-26 — Developer utilities

- **Назначение:** debug state overrides, restart WebUI, test notification.
- **Текущая точка входа:** `DeveloperToolsMixin.dev_utils`.
- **Рабочий исходный код:** [`module/webui/app_developer_tools.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/app_developer_tools.py)::`dev_utils`, `request_webui_restart`, `prepare_webui_restart`.
- **Читает:** running instances, current manager states.
- **Записи/действия:** `ProcessManager.set_state_override`, restart event/cleanup, reload marker, notification.
- **Прямые привилегированные зависимости:** process lifecycle, filesystem marker, process-wide State, notification config.
- **Существующий транспорт:** PyWebIO session.
- **Доверие/локальность:** PyWebIO gate; developer visibility не является authorization model для будущего remote exposure.
- **Связность со средой выполнения:** **high**.
- **Теги миграции:** `INTERNAL_DEVELOPER`, `PRIVILEGED_POLICY`, `SECURITY_REDESIGN`, `DEFER_OR_REMOVE_CANDIDATE`.
- **Требование к бэкенду:** не публиковать без отдельного admin policy/audit product decision.

## S-27 — OBS overlay

- **Назначение:** standalone browser/OBS read-only CL1/AP overlay.
- **Текущая точка входа:** GET `/obs`.
- **Рабочий исходный код:** [`module/webui/api.py`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/api.py)::`serve_obs_overlay`; [`module/webui/obs_overlay.html`](https://github.com/AliceLiddell01/AzurPilot-private-Ru/blob/3be3696975cb91ba0b85dbea98400381c3ced379/module/webui/obs_overlay.html)::`fetchData`.
- **Читает:** `/api/cl1_stats`, `/api/ap_timeline` по `instance` query parameter.
- **Записи/действия:** нет state-changing действий.
- **Прямые привилегированные зависимости:** statistics modules через два API handler.
- **Существующий транспорт:** HTTP polling каждые 10 секунд.
- **Доверие/локальность:** три handler (`/obs`, CL1, AP) собственного auth/locality guard не содержат.
- **Связность со средой выполнения:** **low/medium**.
- **Теги миграции:** `CONTRACT_FORMALIZATION`, `SECURITY_REDESIGN`, `LEGACY_COMPATIBILITY`.
- **Требование к бэкенду:** решить, остаётся ли OBS отдельным compatibility client; read endpoints должны получить явный auth/exposure contract.

## Замечание о числе поверхностей

Карта содержит **27** записей. Число не равно 90 task destinations: task definitions используют общий form/execution wiring и разбиваются только там, где код создаёт отдельный workflow или особую dependency boundary. Декоративные widgets отдельными surfaces не считаются.
