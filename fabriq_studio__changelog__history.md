# fabriq_studio 変更履歴（git log 注釈付き要約）

> **対象**: fabriq_studio / 全体
> **対象バージョン**: commit `3897c6e` (2026-05-06)（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、最新コミット日付）
> **ドキュメント更新日**: 2026-05-07

fabriq_studio は **`<Version>` プロパティを csproj に持たない** ため公式 SemVer はなく、git log の commit hash + コミット日付が事実上の版識別子となる。本書は `git log` から **機能進化の波**をテーマ別に整理した注釈付き要約。

> 注: 公式 `CHANGELOG.md` ファイルは fabriq_studio 側にも存在しない。**本書がドキュメント上の唯一の改訂履歴**。コミットログそのものは `E:\fabriq_studio` 配下で `git log --oneline` を参照。

---

## 全体規模

| 観点 | 数値 |
|---|---|
| 総コミット数 | **109** |
| 開発期間 | **2026-02-22 〜 2026-05-06**（約 2.5 ヶ月） |
| 開発フェーズ数 | **8 つ**（後述） |
| アーキテクチャ | C# / WPF (.NET 8) + MVVM + CommunityToolkit.Mvvm |
| 状態 | アクティブ開発中。現在 Pianist Profile Editor Phase D の追従と README 整理が進行 |

---

## 開発フェーズ全体図

```
Phase 1: 初期 GUI 立ち上げ  ─ 2026-02-22 〜 02-23 (8 コミット)
Phase 2: 編集機能の確立      ─ 2026-02-23 〜 02-25 (15 コミット)
Phase 3: レジストリ辞書       ─ 2026-02-23 〜 03-26 (16 コミット)
Phase 4: ENC: 暗号化 + 各種    ─ 2026-02-27 〜 03-26 (15 コミット)
Phase 5: AutoPilot/Profile 詳細 ─ 2026-04-07 〜 04-21 (5 コミット)
Phase 6: 補助ツール群         ─ 2026-04-20 〜 04-24 (10 コミット)
Phase 7: Pianist Profile Editor ─ 2026-05-03 〜 05-04 (12 コミット)
Phase 8: 仕上げ              ─ 2026-05-04 〜 05-06 (4 コミット)
```

各 Phase の詳細は以下の通り。

---

## Phase 1: 初期 GUI 立ち上げ（2026-02-22 〜 02-23）

最初の数日で **WPF + MVVM の足場と CSV エディタの基本機能** を作り込んだ集中フェーズ。

| コミット | 日付 | 概要 |
|---|---|---|
| `223c998` | 2026-02-22 | 初回コミット |
| `3e1efa6` | 2026-02-22 | phase1_OK — 基本構造 |
| `c476d57`/`e36f0ae` | 2026-02-22 | gitignore 整備 |
| `0568db3` | 2026-02-22 | default_withBOM_save_OK — **UTF-8 BOM 書き込み**規約確立（PowerShell 5.1 互換） |
| `c116cf6` | 2026-02-22 | Profile_View_OK |
| `7a3aa7f` | 2026-02-22 | モジュール一覧表示OK |
| `1e658e1` | 2026-02-22 | PC、モジュール詳細表示OK |
| `b00c15c` | 2026-02-22 | 既存情報の編集保存OK |

**この期に確立した規約**:

- BOM 付き UTF-8 + CsvHelper 33.0.1
- WPF UserControl + MVVM ナビゲーション
- 親ペイン / 詳細ペインの 2 ペイン構成

---

## Phase 2: 編集機能の確立（2026-02-23 〜 02-25）

Profile / Module / App の各 CSV 編集と Reload / Refresh の整備。コミットメッセージに `_ok` `_OK` が頻出する **動作確認単位での頻繁なコミット**スタイル。

| コミット | 日付 | 概要 |
|---|---|---|
| `dee329a` | 2026-02-22 | リファクタリングOK (#1) |
| `d70aee4` | 2026-02-22 | CSV行作成削除OK |
| `e7f3d4a` | 2026-02-22 | Profile_edit_ok |
| `192c203` | 2026-02-22 | profile_create_ok |
| `a95742c` | 2026-02-22 | autokey_app_ok（後の Pianist 系の前身） |
| `64a6995` | 2026-02-23 | Fabriq read op01 (#2) |
| `aa86aa3` | 2026-02-23 | portable — **アプリの portable 化**（ファイルパスを exe 相対に） |
| `28368c3` | 2026-02-23 | reg_colleciton_ok |
| `718bb80` | 2026-02-23 | profile_edit_gui_ok |
| `d03545d` | 2026-02-23 | profile_module_edit_ok |
| `de18185` | 2026-02-23 | profile_refresh_Ok |
| `0951820` | 2026-02-23 | app_config_editor_ok |
| `e931894` | 2026-02-23 | autokey_recipe_testrun_ok |
| `38fde3a` | 2026-02-23 | gyotaq_editor_ok（業託エディタ、後に廃止） |
| `bac6f15` | 2026-02-23 | module_csv_edit_ok |
| `419ec5a` | 2026-02-23 | csv_import_ok |
| `3458c1a` | 2026-02-24 | template_move_ok |
| `c5d94de` | 2026-02-24 | script_looper_app_OK — **Script Looper 編集機能** 初版 |
| `a6731ea` | 2026-02-24 | template_OK |
| `3230fa9` | 2026-02-24 | digital_gyotaq_editor_uri_union_OK |

---

## Phase 3: レジストリ辞書（2026-02-23 〜 03-26）

**Registry Collection（レジストリ辞書 catalog.json + reg_config.csv export）** が約 1 ヶ月に渡って継続発展。各レジストリ調整項目の追加が逐次進む典型的な「辞書育成」期間。

| コミット | 日付 | 概要 |
|---|---|---|
| `c748171` | 2026-02-23 | regstry_collection_OK — **catalog.json + RegistryTemplateEntry 初版** |
| `2206516` | 2026-02-28 | reg_zisho_entry_ok |
| `b1c947b` | 2026-03-01 | reg_entry_ok |
| `d127996` | 2026-03-03 | reg_catalog_add_ok |
| `e9a8404` | 2026-03-03 | reg_catalog_ok |
| `778f23e` | 2026-03-16 | reg_collection_add_shikakukouka_ok（視覚効果追加） |
| `25e5a4d` | 2026-03-16 | reg_collect_taskbar_ok |
| `ff3636e` | 2026-03-16 | strogesenser_reg_collect_add |
| `b76ed1d` | 2026-03-16 | reg_collect_lostmode_add_ok |
| `277c047` | 2026-03-17 | reg_collect_backgroundapp_stop_add |
| `c16c103` | 2026-03-18 | reg_collection_ok |
| `29cd587` | 2026-03-18 | reg_collect_spotlight_enabled_off_add |
| `ad88a3f` | 2026-03-21 | reg_printer_default_ok |
| `4647b84` | 2026-03-22 | reg_add_ok |
| `4954561` | 2026-03-25 | reg_sleephide_ok |
| `c248ab6` | 2026-03-25 | reg_screensaver_add |
| `39adde9` | 2026-03-26 | reg_collect_add |

**詳細**: [fabriq_studio__apps__03_registry_collection.md](fabriq_studio__apps__03_registry_collection.md) / [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)

---

## Phase 4: ENC: 暗号化 + ホストリスト統合（2026-02-25 〜 03-26）

**Phase 3 と並行して** 進んだセキュリティ機能の整備。fabriq kernel 3.0.0 の `ENC:` プレフィクス方式に追随。

| コミット | 日付 | 概要 |
|---|---|---|
| `97d8966` | 2026-02-25 | crypto_ok — 初期暗号化 |
| `b9d809f` | 2026-02-25 | build_shusei_ok |
| `54f2579` | 2026-02-25 | hostlist_crypto_ok — hostlist の ENC: 化 |
| `94135f4` | 2026-02-25 | hostlist_pin_edit_ok |
| `7a983ea` | 2026-02-25 | Segment_profile_ok |
| `b0deaf3` | 2026-02-26 | token_ok — **passphrase_verify.txt** 連携 |
| `65c5d46` | 2026-02-27 | senyou_view_segment_edit_ok |
| `7bd37ed` | 2026-02-27 | ikkatu_encrypto_ok — 一括暗号化機能 |
| `4a6c8f0` | 2026-03-13 | printer_edit_ok |
| `ab50b86` | 2026-03-13 | hostlist_enc_ikkatu_ok — **ENC: 一括化最終版** |
| `b5bf13f` | 2026-03-05 | publish_fix_done |
| `88ba927` | 2026-03-05 | evidence_shuturyoku_ok — **エビデンス出力連携** |
| `3012adb` | 2026-03-05 | module_csv_fhukusuu_ok（複数化） |
| `11d4af2` | 2026-03-04 | Profile_memo_ok — Profile メモ機能 |
| `6d7f428` | 2026-03-04 | profile_mix_ok |
| `9b461d8` | 2026-03-21 | app_confg_editor_fix |
| `2bd7239` | 2026-03-30 | app_edit_fix |

**詳細**: [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md) / [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md)

---

## Phase 5: AutoPilot / Profile 詳細編集（2026-04-07 〜 04-21）

fabriq kernel の AutoPilot / ErrorMode 仕様に追随した GUI 機能群。

| コミット | 日付 | 概要 |
|---|---|---|
| `6454e19` | 2026-04-07 | autopilotonerror_add — **`ErrorMode` 列の編集 UI** |
| `6fcc015` | 2026-04-08 | 最大化（ウィンドウ最大化対応） |
| `d68528f` | 2026-04-16 | module_edit_search_add — モジュール一覧の検索機能 |
| `d63d927` | 2026-04-20 | datagrid_selection_highlight_fix |
| `90d1fcd` | 2026-04-20 | profile_row_drag_drop_add — **profile 行の並び替え D&D** |
| `de32098` | 2026-04-20 | module_detail_open_folder_add |
| `e3eca7e` | 2026-04-21 | module_detail_preset_dropdown_add — **preset.csv ベースの値入力支援** |
| `7d65edb` | 2026-04-21 | module_detail_disabled_row_grayout_add |
| `0fea3f8` | 2026-04-21 | module_detail_enabled_checkbox_add |

**運用への影響**: profile CSV の編集効率が大幅向上。preset.csv の参照を始めるのもここから。

---

## Phase 6: 補助ツール群（2026-04-20 〜 04-24）

**Printer Driver Detector / Host List Export / FabriqBackup / FabriqUpdate** の 4 補助ツールが連続で追加された期。Studio が「編集機」から「**統合ワークベンチ**」へ進化。

| コミット | 日付 | 概要 |
|---|---|---|
| `adc71e9` | 2026-04-20 | printer_driver_detector_add — **INF パーサ + ScanAsync** |
| `bb6ff16` | 2026-04-20 | printer_driver_detector_extract_export_add — **7-Zip 展開 + workspace export** |
| `af0b3df` | 2026-04-23 | profile_detail_async_autopilot_marker_add — `__ASYNC__` / `__AUTOPILOT_*__` マーカー対応 |
| `71594b1` | 2026-04-23 | module_detail_preset_csv_exclude_add |
| `c6717a4` | 2026-04-23 | host_list_export_add — **タイムスタンプフォルダ + Decrypt オプション** |
| `742f103` | 2026-04-23 | profile_detail_module_double_click_add |
| `7430f85` | 2026-04-23 | profile_detail_special_marker_searchable_add |
| `4da7be0` | 2026-04-23 | fabriq_backup_add — **設定値ミラーバックアップ** |
| `775a4b3` | 2026-04-24 | fabriq_update_from_template_add — **オーバーレイ更新（4 段階フロー）初版** |
| `c22b4d8` | 2026-04-24 | fabriq_update_from_template_fix |
| `1cd6fe9` | 2026-04-24 | fabriq_update_from_template_default_path_add |

**詳細**:

- [fabriq_studio__apps__05_printer_driver_detector.md](fabriq_studio__apps__05_printer_driver_detector.md)
- [fabriq_studio__apps__06_fabriq_backup.md](fabriq_studio__apps__06_fabriq_backup.md)
- [fabriq_studio__apps__07_fabriq_update.md](fabriq_studio__apps__07_fabriq_update.md)

---

## Phase 7: Pianist Profile Editor（2026-05-03 〜 05-04）

fabriq の `modules/extended/pianist` v1.4 / v1.5 に対応する **Pianist Profile Editor を 1 日で集中投入** した期間。これにより Studio の最重要機能が完成形に近づく。

| コミット | 日付 | 概要 |
|---|---|---|
| `4cf6c0f` | 2026-05-03 | pianist_profile_editor_add — **初期投入**（pianist.json + procedure.csv 編集） |
| `ef1d025` | 2026-05-03 | pianist_window_picker_add — Window Spy 機能（`WaitWin` 用ウィンドウタイトル取得） |
| `acb9c6a` | 2026-05-03 | pianist_key_presets_extend |
| `ac00979` | 2026-05-03 | pianist_step_drag_drop_reorder_add — Step の並び替え |
| `062fb6f` | 2026-05-03 | pianist_key_presets_repeat_examples_add |
| `23d53ba` | 2026-05-03 | pianist_list_csv_editor_add — **values.csv の wide format エディタ** |
| `249922c` | 2026-05-03 | delete_prompt — **Pianist 削除確認の堅牢化** |
| `dbabedb` | 2026-05-03 | unsaved_changes_guard_add — **`IDirtyAwareViewModel` パターン**確立 |
| `611be56` | 2026-05-03 | unused_editors_remove_and_dialog_guards_add |
| `a0f2e87` | 2026-05-03 | 小さなメッセージの削除 |
| `42be722` | 2026-05-03 | pianist_test_run_add — **PowerShell 子プロセスでの dry-run** |
| `d08a403` | 2026-05-04 | pianist_v1.5_phaseD_follow_up — Phase D（v1.5 Samples タブ）への追従 |
| `fd4f9f4` | 2026-05-04 | pianist_profile_delete_add |

**詳細**: [fabriq_studio__apps__02_pianist_profile_editor.md](fabriq_studio__apps__02_pianist_profile_editor.md) / [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md)

---

## Phase 8: 仕上げ（2026-05-04 〜 05-06、最新）

| コミット | 日付 | 概要 |
|---|---|---|
| `00d2eb2` | 2026-05-04 | readme_sync_with_current_implementation |
| `b0c67f1` | 2026-05-04 | readme_rpa_to_simple_rpa — README 表現を「RPA」→「簡易 RPA」に修正 |
| `3897c6e` | 2026-05-06 | **readme_publish_and_dirty_guard_add（最新コミット）** |

---

## 命名規約の進化

コミットメッセージから読み取れる **コミットスタイル**の変化:

| 期間 | スタイル | 例 |
|---|---|---|
| 2026-02 前半 | 機能 + `_ok` / `_OK` | `Profile_View_OK`, `crypto_ok` |
| 2026-02 後半〜03 | ローマ字 + `_ok` | `hostlist_pin_edit_ok`, `evidence_shuturyoku_ok` |
| 2026-04 〜 | snake_case + `_add` / `_fix` | `printer_driver_detector_add`, `module_edit_search_add` |
| 2026-05 〜 | 同上、より英語寄り | `pianist_profile_editor_add`, `unsaved_changes_guard_add` |

---

## 廃止された機能

開発過程で導入されたが後に削除されたもの:

- **gyotaq_editor**（業託エディタ、Phase 2 で実装、後に整理）
- **digital_gyotaq_editor**（同上）
- **autokey_recipe testrun**（Phase 2 で実装、Pianist 系統に吸収統合）
- **複数の旧エディタ画面**（commit `611be56` で `unused_editors_remove`）

これらの削除は **Phase 7 の Pianist Profile Editor 投入 + Dirty Guard パターン確立** と同時期に行われ、Studio の機能を **整理して再フォーカス**するクリーンアップ。

---

## ライセンス・配布

- LICENSE は MIT 系（リポジトリルート [LICENSE](file:///E:/fabriq_studio/LICENSE)）
- 配布形式: **portable**（commit `aa86aa3` で確立）。インストーラなし、exe + template フォルダで動作
- 同梱物: `template/template_fabriq/` 配下に **fabriq テンプレート + looper_template** を内包（`Update Dialog` で使用）
- バージョン埋め込み: csproj `<Version>` プロパティが**未設定**。ビルド成果物には反映されない（git short hash で同定）

---

## 関連ドキュメント

| ドキュメント | 関係 |
|---|---|
| [fabriq__changelog__history.md](fabriq__changelog__history.md) | fabriq 本体側の改訂履歴（kernel 3.x の暗号化変更や FlexProfile などが Studio 側機能の前提） |
| [fabriq_studio__overview__readme.md](fabriq_studio__overview__readme.md) | Studio 全体像 |
| [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md) | レイヤアーキテクチャ |

---

## 変更履歴

- 2026-05-07 初版作成（commit `3897c6e` 時点の git log 109 コミットを 8 フェーズに分けて注釈）
