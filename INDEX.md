# Fabriq Series — Unified Documentation Index

**Last updated**: 2026-05-07
**Layout**: Flat. すべての md は本リポジトリのトップ直下に配置されている（`<project>__<category>__<name>.md` 形式）。
**LM 投入**: NotebookLM 等にこのフォルダ全体を投入すれば、ファイル名のプレフィックスでプロジェクト判別が完結する。

各プロジェクトのソース本体は読み取り専用で、本リポジトリは **ドキュメントの一元管理** のみを担う。詳細は [CLAUDE.md](CLAUDE.md) を参照。

---

## プロジェクト別ファイル数（現状）

| プロジェクト | 接頭辞 | ファイル数 | 対象バージョン情報源 |
|---|---|---|---|
| fabriq | `fabriq__` | 106 | `E:\fabriq\kernel\KERNEL_VERSION` = 3.2.2 + per-module `VERSION` |
| fabriq_evidence_manager | `fabriq_evidence_manager__` | 22 | `FabriqEvidenceManager.csproj <Version>` = 3.8.1 |
| fabriq_studio | `fabriq_studio__` | 15 | `FabriqStudio.csproj` に `<Version>` 未設定 → git short hash `3897c6e` |
| tonebender | `tonebender__` | 0 (未着手) | git short hash |
| tonebender-controller | `tonebender_controller__` | 0 (未着手) | git short hash |

---

## fabriq (106 files)

### overview (1)

- [fabriq__overview__readme.md](fabriq__overview__readme.md)

### usage (5)

- [fabriq__usage__01_install_and_first_boot.md](fabriq__usage__01_install_and_first_boot.md)
- [fabriq__usage__02_session_setup.md](fabriq__usage__02_session_setup.md)
- [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)
- [fabriq__usage__04_flexprofile_dashboard.md](fabriq__usage__04_flexprofile_dashboard.md)
- [fabriq__usage__05_evidence_and_quick_actions.md](fabriq__usage__05_evidence_and_quick_actions.md)

### apps (6)

- `fabriq__apps__00_apps_overview.md`
- `fabriq__apps__01_fabriq_operator_dashboard.md`
- `fabriq__apps__02_fabriq_ios.md`
- `fabriq__apps__03_other_apps.md`
- `fabriq__apps__04_dev_template_and_tooling.md`
- `fabriq__apps__05_commands.md`

### contracts (6)

- `fabriq__contracts__evidence_manifest_contract.md`
- `fabriq__contracts__host_environment.md`
- `fabriq__contracts__module_result.md`
- `fabriq__contracts__overlay_contract.md`
- `fabriq__contracts__profile_csv_schema.md`
- `fabriq__contracts__special_markers.md`

### kernel (11)

- `fabriq__kernel__01_overview.md`
- `fabriq__kernel__02_public_api.md`
- `fabriq__kernel__03_orchestration.md`
- `fabriq__kernel__04_csv_encryption.md`
- `fabriq__kernel__05_resume_restart.md`
- `fabriq__kernel__06_status_monitor.md`
- `fabriq__kernel__07_evidence_history.md`
- `fabriq__kernel__08_async_execution.md`
- `fabriq__kernel__09_versioning.md`
- `fabriq__kernel__10_function_index.md`
- `fabriq__kernel__11_directory_layout.md`

### modules (76)

- `fabriq__modules__00_modules_overview.md`
- `fabriq__modules__acl_config.md`
- `fabriq__modules__app_config.md`
- `fabriq__modules__autologon_config.md`
- `fabriq__modules__azure_ad_join_check.md`
- `fabriq__modules__bitlocker_config.md`
- `fabriq__modules__bloatware_export.md`
- `fabriq__modules__bloatware_remove.md`
- `fabriq__modules__brightness_config.md`
- `fabriq__modules__browser_addon_config.md`
- `fabriq__modules__builtin_admin_config.md`
- `fabriq__modules__cert_config.md`
- `fabriq__modules__copyfile_config.md`
- `fabriq__modules__default_app_config.md`
- `fabriq__modules__desktop_icon_config.md`
- `fabriq__modules__directory_cleaner.md`
- `fabriq__modules__display_config.md`
- `fabriq__modules__domain_join.md`
- `fabriq__modules__dpi_api_config.md`
- `fabriq__modules__dpi_config.md`
- `fabriq__modules__driver_config.md`
- `fabriq__modules__evidence_config.md`
- `fabriq__modules__fabriq_app_launcher.md`
- `fabriq__modules__file_delete.md`
- `fabriq__modules__firewall_config.md`
- `fabriq__modules__firewall_rule_config.md`
- `fabriq__modules__firewall_rule_make_config.md`
- `fabriq__modules__generic_batch_runner.md`
- `fabriq__modules__generic_process_runner.md`
- `fabriq__modules__group_config.md`
- `fabriq__modules__history_destroyer.md`
- `fabriq__modules__hostname_config.md`
- `fabriq__modules__ipaddress_config.md`
- `fabriq__modules__ipv6_config.md`
- `fabriq__modules__local_user_config.md`
- `fabriq__modules__log_uploader.md`
- `fabriq__modules__manual_kitting_assistant.md`
- `fabriq__modules__network_profile_config.md`
- `fabriq__modules__odt_config.md`
- `fabriq__modules__office_license_config.md`
- `fabriq__modules__office_update.md`
- `fabriq__modules__partition_config.md`
- `fabriq__modules__pianist.md`
- `fabriq__modules__power_config.md`
- `fabriq__modules__ppkg_config.md`
- `fabriq__modules__printer_delete.md`
- `fabriq__modules__printer_driver_config.md`
- `fabriq__modules__process_killer.md`
- `fabriq__modules__profile_delete.md`
- `fabriq__modules__reg_hkcu_config.md`
- `fabriq__modules__reg_hklm_config.md`
- `fabriq__modules__reg_template.md`
- `fabriq__modules__resolution_api_config.md`
- `fabriq__modules__restart_config.md`
- `fabriq__modules__restore_point.md`
- `fabriq__modules__robocopy_config.md`
- `fabriq__modules__scheduled_task_config.md`
- `fabriq__modules__script_looper.md`
- `fabriq__modules__signout_config.md`
- `fabriq__modules__spi_config.md`
- `fabriq__modules__ssid_config.md`
- `fabriq__modules__startlayout_config.md`
- `fabriq__modules__startup_command_config.md`
- `fabriq__modules__storeapp_config.md`
- `fabriq__modules__sysprep_config.md`
- `fabriq__modules__system_finalize.md`
- `fabriq__modules__taskbar_config.md`
- `fabriq__modules__temp_ipaddress_config.md`
- `fabriq__modules__test_error_module.md`
- `fabriq__modules__test_harness_config.md`
- `fabriq__modules__time_sync_config.md`
- `fabriq__modules__volume_config.md`
- `fabriq__modules__wallpaper_config.md`
- `fabriq__modules__windows_license_config.md`
- `fabriq__modules__windows_update.md`
- `fabriq__modules__winget_install.md`

### profiles (1)

- `fabriq__profiles__00_profiles_overview.md`

---

## fabriq_evidence_manager (22 files)

### overview (1)

- [fabriq_evidence_manager__overview__readme.md](fabriq_evidence_manager__overview__readme.md)

### architecture (3)

- [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- [fabriq_evidence_manager__architecture__03_warning_caution_model.md](fabriq_evidence_manager__architecture__03_warning_caution_model.md)

### contracts (2)

- [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
- [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)

### apps (3)

- [fabriq_evidence_manager__apps__01_main_window.md](fabriq_evidence_manager__apps__01_main_window.md)
- [fabriq_evidence_manager__apps__02_pc_detail_window.md](fabriq_evidence_manager__apps__02_pc_detail_window.md)
- [fabriq_evidence_manager__apps__03_settings_window.md](fabriq_evidence_manager__apps__03_settings_window.md)

### usage (5)

- [fabriq_evidence_manager__usage__01_import.md](fabriq_evidence_manager__usage__01_import.md)
- [fabriq_evidence_manager__usage__02_hostlist_verification.md](fabriq_evidence_manager__usage__02_hostlist_verification.md)
- [fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md)
- [fabriq_evidence_manager__usage__04_export_delivery.md](fabriq_evidence_manager__usage__04_export_delivery.md)
- [fabriq_evidence_manager__usage__05_pc_memo.md](fabriq_evidence_manager__usage__05_pc_memo.md)

### reference (6)

- [fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md)
- [fabriq_evidence_manager__reference__file_format__pc_information.md](fabriq_evidence_manager__reference__file_format__pc_information.md)
- [fabriq_evidence_manager__reference__hostlist_csv_schema.md](fabriq_evidence_manager__reference__hostlist_csv_schema.md)
- [fabriq_evidence_manager__reference__model_catalog.md](fabriq_evidence_manager__reference__model_catalog.md)
- [fabriq_evidence_manager__reference__office_license_evaluation.md](fabriq_evidence_manager__reference__office_license_evaluation.md)
- [fabriq_evidence_manager__reference__serial_number_logic.md](fabriq_evidence_manager__reference__serial_number_logic.md)

### troubleshooting (1)

- [fabriq_evidence_manager__troubleshooting__manifest_errors.md](fabriq_evidence_manager__troubleshooting__manifest_errors.md)

### changelog (1)

- [fabriq_evidence_manager__changelog__history.md](fabriq_evidence_manager__changelog__history.md)

---

## fabriq_studio (15 files)

### overview (1)

- `fabriq_studio__overview__readme.md`

### architecture (2)

- `fabriq_studio__architecture__01_layers.md`
- `fabriq_studio__architecture__02_workspace.md`

### apps (4)

- `fabriq_studio__apps__01_main_pages.md`
- `fabriq_studio__apps__02_pianist_profile_editor.md`
- `fabriq_studio__apps__03_registry_collection.md`
- `fabriq_studio__apps__04_other_tools.md`

### contracts (1)

- `fabriq_studio__contracts__crypto_interop.md`

### usage (1)

- `fabriq_studio__usage__01_workspace_setup.md`

### reference (6)

- [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md)
- [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md)
- [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)
- [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md)
- [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md)
- [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md)

---

## tonebender (0 files)

未着手。`tonebender__` プレフィックスで以下のような構成を予定。

- `tonebender__overview__readme.md`
- `tonebender__architecture__01_winpe_runtime.md`
- `tonebender__usage__01_image_capture.md`
- `tonebender__usage__02_image_apply.md`
- `tonebender__usage__03_autopilot_recovery.md`
- `tonebender__reference__cli_options.md`

---

## tonebender-controller (0 files)

未着手。**ハイフンはアンダースコアに変換**して `tonebender_controller__` プレフィックスを使用。

- `tonebender_controller__overview__readme.md`
- `tonebender_controller__architecture__01_pipeline.md`
- `tonebender_controller__usage__01_profile_json.md`
- `tonebender_controller__usage__02_usb_partitioning.md`
- `tonebender_controller__usage__03_winpe_build.md`
- `tonebender_controller__usage__04_oem_driver_injection.md`
- `tonebender_controller__reference__profile_schema.md`

---

## カテゴリ語彙（横断）

| カテゴリ | 用途 |
|---|---|
| `overview` | 概要・README |
| `kernel` | コア / フレームワーク本体（fabriq 専用） |
| `modules` | 機能モジュール毎の解説（fabriq 専用） |
| `apps` | サブアプリ・GUI ツール |
| `architecture` | 設計思想・レイヤ構成 |
| `contracts` | 公開 API・スキーマ・契約 |
| `profiles` | プロファイル / 設定ファイル仕様 |
| `usage` | 使い方・操作手順 |
| `startup` | 導入・初期セットアップ |
| `troubleshooting` | トラブル対応・既知の問題 |
| `reference` | 参照表・一覧 |
| `changelog` | 変更履歴ミラー / 注釈付き要約 |

新規カテゴリを追加する場合は本表を更新すること。
