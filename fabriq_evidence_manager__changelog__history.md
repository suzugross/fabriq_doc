# 変更履歴 / Changelog

> **対象**: fabriq_evidence_manager / 全体
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>` / `git -C E:\fabriq_evidence_manager log -1 --format='%h %cd' --date=short` = `45eae22 2026-05-07`）
> **ドキュメント更新日**: 2026-05-07

`FabriqEvidenceManager.csproj` `<Version>` の SemVer 履歴と、各版で導入された主要機能 / 内部リファクタを `git log` から再構成した注釈付き要約。M-numbered milestones（M1〜M14）は ExcelExportService.cs のコメントや commit subject で参照される機能シリーズの識別子。

ソース側 commit 履歴は `git -C E:\fabriq_evidence_manager log` を真値とする。

---

## バージョン互換マトリクス

本アプリは fabriq kernel の公開契約 `EVIDENCE_MANIFEST.md schemaVersion=1` のみをサポート。producer 側 `evidence_config` モジュールのバージョンに対する互換性：

| evidence_manager 版 | サポートする evidence_config | サポートする kernel | 備考 |
|---|---|---|---|
| **3.8.x** | 1.3.0 〜 1.6.0+ | 2.2.2 〜 3.0.0+ | §11 per-source baseline 統合（3.8.1） |
| 3.7.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | PcMemo 機能追加、Excel TruncateForCell |
| 3.6.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | DomainStatus Comparator 追加 |
| 3.5.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | 設定ダイアログ統合 |
| 3.4.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | Baseline Excel 全カテゴリ対応、参照キー化 |
| 3.3.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | LicenseComparator |
| 3.2.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | Baseline plugin リファクタ、PcDetail Window 独立化 |
| 3.1.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | BitLocker キー表示拡張 |
| 3.0.x | 1.3.0 〜 1.5.0+ | 2.2.2+ | 納品ディレクトリ秒精度、Excel PC 個別分割 |
| 2.x | 1.3.0 〜 1.4.x | 2.2.2+ | manifest required 化、ライセンスシート、SN audit、PC 名併記 |
| 1.x（first push 〜 2026-04-02）| ファイル列挙ベース | 任意 | manifest 非対応世代（旧プロトタイプ） |

---

## 3.8.1（2026-05-07、`45eae22`）— `section_11_installed_software_with_per_source_baseline_split`

直前 commit `60ba2d6` で実装された §11 Installed Software のベースライン突合改修に対するバージョン bump。

- §11 を Desktop / Store の per-source 別管理に分離（従来は統合 1 リスト）
- `InstalledAppsComparator` のキー化を「アプリ名」から「Source × アプリ名」に変更し、同名アプリが Desktop / Store の両方にあるケースで誤マッチしないよう改修

---

## 3.8.0（2026-05-01、`fdba640`）— `bump_version_to_v3_8_0`

evidence_config v1.6.0（kernel 3.0.0）の **§24〜§31 6 セクション一括対応** が完成したマイルストーン。直前の 6 commit が個別の M シリーズ：

| commit | M 番号 | 機能 |
|---|---|---|
| `5b79eec` | **M6** | §24 Group Policy（gpresult 構造化 + HTML パス保持、PcDetailWindow ブラウザ起動）+ Excel PC 個別シート再配置 |
| `333f742` | **M5** | §30 PnP デバイス（200〜400 件規模、Excel AutoFilter 活用） |
| `a41116b` | **M4** | §28 Startup Items（Win32_StartupCommand + ScheduledTask 統合 1 リスト） |
| `6618fe9` | **M3** | §27 Environment Variables（Machine + User 混在 1 リスト） |
| `6a210b2` | **M2** | §29 Memory Slots & Array Summary（DIMM 個別 + アレイサマリ 2 ファイル統合） |
| `79c65d4` | **M1** | §31 Hardware Identifiers（Win32_ComputerSystem + CSP + BaseBoard + SystemEnclosure 4 ブロック）+ v1.6.0 互換 |

加えて修正 `77cad11`：

- PC 名のハイパーリンク不具合（`XLHyperlink(string)` の内部セル参照誤解釈）
- PcDetailWindow のクランプロジック追加（低解像度・高 DPI 環境で `WorkArea` 外配置を防止）
- DataGrid 列リサイズ挙動

---

## 3.7.1（2026-04-26、`4fbb79c`）— `pc_memo_and_excel_cell_truncation_v3_7_1`

- **管理者メモ機能（PcMemo）追加**：PC ルート直下 `manager_memo.json` に JSON CamelCase で永続化、`PcMemoService` 新設
- DataGrid 行末尾に `📝` アイコン（`HasMemo=true` 時）、PcDetailWindow のヘッダ直下に編集 TextBox + 保存ボタン
- **Excel セル 32,767 文字超対策**：`TruncateForCell` ヘルパで `\n... [TRUNCATED]` マーカー付きで切り詰め、長文値（環境変数 Path / コマンド / 未知 section RawContent / メモ等）の安全化

---

## 3.6.0（2026-04-26、`a19a174`）— `domain_status_comparator_v3_6_0`

- **DomainStatusComparator 追加**：ベースライン突合の 6 カテゴリ目（[fabriq_evidence_manager__usage__03_baseline.md](fabriq_evidence_manager__usage__03_baseline.md) §「カテゴリ別の差分セマンティクス」§「ドメイン参加状態」）
- §05 DomainStatus の 7 項目（PartOfDomain / Domain / DomainRole / AzureAdJoined / DomainJoined / DomainName / TenantName）を typed model 直比較で照合
- `CurrentUser` は PC 個別のため比較対象外（キッティング作業者が PC ごとに違うのは正常）

---

## 3.5.0（2026-04-26、`581ad75`）— `settings_window_unified_v3_5_0`

- **設定ダイアログ統合**：以前は MainWindow 内に散在していた CheckOptions と BaselineCategories を 1 つの SettingsWindow に集約
- 取得チェック 9 項目 + 余剰プリンタ検出 + ベースラインカテゴリ 6 件
- `MainWindowViewModel.CreateSettingsViewModel()` ヘルパで DI 経由構築、`onBaselineCategoriesChanged` コールバック経由で全 PC 再評価

---

## 3.4.1（2026-04-26、`7138635`）— `baseline_excel_full_coverage_and_dict_key_fix_v3_4_1`

- **Baseline Excel 全カテゴリ出力**：6 カテゴリすべてが pc_details/ サブブックに `ベースライン_*` シートとして出る
- **`ExcelExportOptions` の Dictionary キーを参照同一性化**：以前は `string PcName` キーだったが、同じ計画 PC 名（`selectedNewPcName`）を持つ複数 PC が衝突して上書きするバグ修正。**キーを `PcEvidence` インスタンス参照** に変更

---

## 3.3.1（2026-04-26、`1ed952e`）— `license_comparator_phase4_v3_3_1`

- **LicenseComparator 追加**（Phase 4）：Windows / Office ライセンスのベースライン突合
- Win 4 項目（ライセンスファミリ / プロダクトキーチャネル / ライセンス状態 / KMS Machine）+ Office 4 項目（インストール状態 / C2R ProductReleaseIds / UpdateChannel / ライセンス状態）= 最大 8 項目
- `PcEvidence.WindowsLicense / OfficeLicense` の typed model を直接参照（再パース不要）

---

## 3.2.2（2026-04-26、`24d89a0`）— `baseline_plugin_refactor_phase1_v3_2_2`

- **Baseline プラグインアーキテクチャ Phase 1**：`BaselineService` を薄いオーケストレータ化、6 件の `IBaselineComparator` 実装に分離
- 新カテゴリ追加が「`IBaselineComparator` 実装を 1 つ追加 + DI 登録」のみで完結
- 6 件の Comparator: ExecutionSummary / SystemInfo / Checklist / InstalledApps / License / DomainStatus（License / DomainStatus は Phase 4 で後追加）

---

## 3.2.1（2026-04-26、`0cd7f27`）— `detail_window_split_with_independent_vm_v3_2_1`

- **PcDetailWindow を独立 ViewModel 化**：`PcDetailViewModel` を `MainWindowViewModel` から分離
- 起動時スナップショット保持型（`Pc / VerificationReport / ChecklistResult / ExportHistory / BaselineReport`）
- 異なる PC を別ウィンドウで並列に並べて比較する用途を許可（modeless `Show()`）

---

## 3.1.0（2026-04-26、`bd0a3da`）— `bitlocker_key_display_v3_1_0`

- BitLocker 回復キー表示の改善
- DataGrid 列で `ItemsControl` を使ってドライブ別 1 行 = `{drive}: {recoveryPassword}` 形式
- PcDetailWindow でも `Identifier` GUID を併記表示

---

## 3.0.2（2026-04-26、`28f47d8`）— `delivery_dir_with_seconds_v3_0_2`

- 納品ディレクトリ命名を `{YYYY_MM_DD_HHmmss}_fabriq_evi` に変更（**秒精度のタイムスタンプ**）
- 同日複数回出力時の衝突回避

---

## 3（2026-04-26、`0330586`）— `excel_split_per_pc_and_checklist_verified_fix_v3`

- **Excel PC 個別ブック分割**：`pc_details/` サブディレクトリに `{PcName}_{Date}.xlsx` を分割出力
- main 台帳の PC 名セルから個別ブックへハイパーリンク
- チェックリスト HTML パースで Verified 列対応（kernel 2.x で新設された 5 列目 `<span>`）

---

## 2.2（2026-04-26、`6261e98`）— `sn_source_audit_v2_2`

- **SN 採用ソース監査**：§10 の 4 ブロックパースを state machine 化
- `SerialNumberDetail` / `SerialSourceEntry` / `SerialSourceValidity` 型新設
- Primary canonical = `Win32_BIOS.SerialNumber` 以外を採用したケースを `HasSerialFallbackWarning=true` でオレンジ強調
- 詳細は [fabriq_evidence_manager__reference__serial_number_logic.md](fabriq_evidence_manager__reference__serial_number_logic.md)

---

## 2.1（2026-04-26、`24a0a2b`）— `pc_name_pair_display_v2_1`

- **計画 PC 名 + 実 PC 名の併記表示**：`PcName`（hostlist 計画）と `ActualComputerName`（manifest.computerName = OS 上の名前）を別々に持つ
- DataGrid に `計画 PC 名` / `実 PC 名` の 2 列、不一致時は赤背景強調
- `HasPcNameMismatch / PcNameDisplay / ActualComputerNameDisplay` 算出プロパティ追加

---

## 2（2026-04-26、`73b468f`）— `manifest_required_and_license_sheets_v2`

**マイルストーン: manifest 必須化**

- producer 契約 `EVIDENCE_MANIFEST.md schemaVersion=1` への対応完成
- `ManifestReaderService` の 5 段検証導入、3 例外型新設（`ManifestNotFoundException` / `UnsupportedManifestSchemaException` / `ManifestParseException`）
- §21 Windows License / §22 Office License セクションのパース対応 + 専用 Excel シート 2 種追加
- v1.4 evidence ベースの旧 OSPP/vNext 個別判定経路（後の v1.5+ INTERPRETATION 経路導入は 3.8.0 系で実装）

詳細は [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)。

---

## 1.x 系（2026-04-02 以前）

`first_push`（2026-02-26、`56b8931`）から `evidence_kousei_fix`（2026-04-02、`ca5eb7a`）までの 1.x プロトタイプ世代。**ファイル列挙ベース**（manifest 非対応）で、log_uploader 形式の検出 / フリート画面 / Excel 出力の基本機能を構築した時期。

主な commit：

| commit | 日付 | 内容 |
|---|---|---|
| `56b8931` | 2026-02-26 | `first_push`（プロジェクト初期化） |
| `32f010d` | 2026-02-26 | `refactor_01_ok`（最初の構造改善） |
| `5e3d7b3` | 2026-02-26 | `CentreCOM_ok`（CentreCOM テーマ風 UI 採用） |
| `a018c7e` | 2026-02-26 | `evidence_check_ok`（エビデンス欠損チェック導入） |
| `1bd9e13` | 2026-02-26 | `review_ok`（レビュー対応） |
| `ed0ac3e` | 2026-02-26 | `ar260s_ok`（Allied Telesis AR260S 風 UI 演出整備） |
| `df2f353` | 2026-03-12 | `Add nested evidence mode (log_uploader format) and UI polish`（**log_uploader 形式 = `_evidence` サフィックス対応**）|
| `d29227d` | 2026-03-12 | `evidence_marge_ok` |
| `ed056c3` | 2026-03-12 | `fabriq_evidence_commit_ok` |
| `f55f582 / 2f21c2b` | 2026-03-12〜13 | `evidence_check_ok` シリーズ |
| `f3d6d84` | 2026-03-13 | `hostlist_check_bugfix_ok` |
| `1b054cb` | 2026-03-13 | `NIC_sort_ok`（NIC 列ソート安定化） |
| `a278384` | 2026-03-13 | `noactual_fix_ok`（VerificationStatus.NoActual 対応） |
| `d4bc7bc` | 2026-03-13 | `kajo_printer_check_ok`（余剰プリンタ検出オプション）|
| `a76a67e / 0e7e084` | 2026-03-18〜19 | `baseline_extended_fix` シリーズ（Baseline 機能の試作） |
| `205e988` | 2026-03-24 | `pc_info_fix` |
| `7f4352f` | 2026-03-28 | `remove_flat_mode_and_legacy_compat`（**flat 形式 / 旧互換削除 = log_uploader 形式必須化** に向けた整理）|
| `ca5eb7a` | 2026-04-02 | `evidence_kousei_fix`（1.x の最後の機能追加） |

1.x 系から 2.x への移行で **manifest 必須化 + producer 契約準拠** の方針転換が起き、本アプリは fabriq の公開 evidence consumer として位置づけられた。

---

## M シリーズ機能 ID 索引

ExcelExportService.cs のコメントや commit subject で参照される M-numbered milestones：

| M 番号 | 内容 | 導入版 |
|---|---|---|
| **M1** | §31 Hardware Identifiers + v1.6.0 互換 | 3.8.0 |
| **M2** | §29 Memory Slots & Array Summary | 3.8.0 |
| **M3** | §27 Environment Variables | 3.8.0 |
| **M4** | §28 Startup Items | 3.8.0 |
| **M5** | §30 PnP Devices + 未知 section raw 対応 | 3.8.0 |
| **M6** | §24 Group Policy + Sheet 1 案件メタデータ 6 行ブロック | 3.8.0 |
| **M7** | 未知 section の前方互換 raw 表示 + ManagerVersion 記録 | 3.8.0（M5 と同期） |
| **M8c** | OSPP Office product の Licensed/非 Licensed 着色 | 3.x 系 |
| M9 | xUnit テスト基盤導入（commit `40242a7`） | 3.5 系 |
| M11 | Office vNext サマリブロック（5 列追加）+ INTERPRETATION 経路導入 | 3.8.0 |
| M12 | SecurityBaseline AuditLevel セル単位着色 | 3.8.0 |
| M13 | §25 Certificates シート + DaysToExpiry 着色 | 3.8.0 |
| M14 | §26 Battery Report + 健全性 % 着色 | 3.8.0 |

M9 は機能ではなく開発基盤（テスト追加）。M10 / M8a / M8b 等は内部の細粒度マイルストーンで commit subject には現れない。

---

## バージョン取得手順

```powershell
# csproj <Version> から取得
Select-String -Path E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj -Pattern '<Version>'

# git の最新コミット (短縮ハッシュ + 日付)
git -C E:\fabriq_evidence_manager log -1 --format='%h %cd' --date=short

# 履歴一覧
git -C E:\fabriq_evidence_manager log --pretty=format:'%h %cd %s' --date=short
```

ドキュメント先頭の `> **対象バージョン**:` メタブロック生成に使用。

---

## 関連ドキュメント

- プロジェクト概要: [fabriq_evidence_manager__overview__readme.md](fabriq_evidence_manager__overview__readme.md)
- 階層構造: [fabriq_evidence_manager__architecture__01_layers.md](fabriq_evidence_manager__architecture__01_layers.md)
- consumer 側 manifest 消費契約（v2.x で導入）: [fabriq_evidence_manager__contracts__manifest_schema.md](fabriq_evidence_manager__contracts__manifest_schema.md)
- セクション ID dispatch（M1〜M6 で §24〜§31 追加）: [fabriq_evidence_manager__contracts__section_dispatch.md](fabriq_evidence_manager__contracts__section_dispatch.md)
- Excel 出力（M シリーズの結果が反映されたシート構成）: [fabriq_evidence_manager__reference__excel_layout.md](fabriq_evidence_manager__reference__excel_layout.md)
- Office License 評価（v1.5+ INTERPRETATION 経路導入は 3.8.0 系 M11）: [fabriq_evidence_manager__reference__office_license_evaluation.md](fabriq_evidence_manager__reference__office_license_evaluation.md)
- SerialNumber ロジック（v2.2 で導入）: [fabriq_evidence_manager__reference__serial_number_logic.md](fabriq_evidence_manager__reference__serial_number_logic.md)
