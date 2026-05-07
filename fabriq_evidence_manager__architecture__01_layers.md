# 階層構造と DI

> **対象**: fabriq_evidence_manager / architecture
> **対象バージョン**: 3.8.1（取得元: `E:\fabriq_evidence_manager\FabriqEvidenceManager\FabriqEvidenceManager.csproj` `<Version>`、最新コミット `45eae22` (2026-05-07)）
> **ドキュメント更新日**: 2026-05-07

## 全体俯瞰

fabriq_evidence_manager は典型的な MVVM + Service レイヤ + DI の WPF アーキテクチャを採用する。**ビジネスロジックは ViewModel に書かず、必ず Service に切り出して `Microsoft.Extensions.DependencyInjection` 経由で注入する**（CLAUDE.md `Important Rules` 1.「スモールステップでの実装」と同根の方針）。

```
┌─────────────────────────────────────────────────────────────┐
│ Views (XAML)                                                │
│   App.xaml          ── CentreCOM テーマ + Resources        │
│   MainWindow        ── フリート画面                         │
│   PcDetailWindow    ── PC 個別詳細 (modeless, 並列起動可)  │
│   SettingsWindow    ── 取得チェック + ベースラインカテゴリ  │
└──────────────┬──────────────────────────────────────────────┘
               │ DataContext
┌──────────────┴──────────────────────────────────────────────┐
│ ViewModels (CommunityToolkit.Mvvm)                          │
│   MainWindowViewModel   [Transient]                         │
│   PcDetailViewModel     [手動 new、起動時スナップショット]  │
│   SettingsViewModel     [手動 new、Modeless ダイアログ]    │
└──────────────┬──────────────────────────────────────────────┘
               │ コンストラクタ注入
┌──────────────┴──────────────────────────────────────────────┐
│ Services (DI 経由)                                          │
│   Discovery / Manifest / Parser / Hostlist / Verify /       │
│   Checklist / Baseline (+ 6 Comparators) / PcMemo /         │
│   Collector [Transient] / Excel [Transient]                 │
└──────────────┬──────────────────────────────────────────────┘
               │ 操作
┌──────────────┴──────────────────────────────────────────────┐
│ Models (DTO / ObservableObject)                             │
│   PcEvidence (集約ルート)                                    │
│   EvidenceManifest / ManifestSection (公開契約)              │
│   60+ DTO (各セクションの構造化結果)                         │
└─────────────────────────────────────────────────────────────┘
                                    │
              読込のみ              ▼
                          ┌────────────────────┐
                          │ evidence/          │
                          │   pc_information/  │
                          │   auto_capture/    │
                          │   bitlocker/       │
                          │   checklist/       │
                          │   export_history/  │
                          └────────────────────┘
```

## エントリポイント — App.xaml.cs

`App` クラスのコンストラクタで `ServiceCollection` を組み立て、`OnStartup` で `MainWindow` を DI 経由で解決して `Show()` する。`ShutdownMode="OnMainWindowClose"` のため、Modeless で開いた `PcDetailWindow` / `SettingsWindow` がまだ表示されていても、メインを閉じればプロセス終了する。

DI 登録の構成（`App.xaml.cs` `ConfigureServices`）：

```csharp
// Discovery + Manifest
services.AddSingleton<NestedEvidenceDiscoveryService>();
services.AddSingleton<IManifestReaderService, ManifestReaderService>();

// 11 サブパーサ + マスターパーサ
services.AddSingleton<IWindowsLicenseParserService, WindowsLicenseParserService>();
services.AddSingleton<IOfficeLicenseParserService, OfficeLicenseParserService>();
services.AddSingleton<ISecurityBaselineParserService, SecurityBaselineParserService>();
services.AddSingleton<ICertificatesParserService, CertificatesParserService>();
services.AddSingleton<IBatteryReportParserService, BatteryReportParserService>();
services.AddSingleton<IHardwareIdentifiersParserService, HardwareIdentifiersParserService>();
services.AddSingleton<IMemoryInventoryParserService, MemoryInventoryParserService>();
services.AddSingleton<IEnvironmentVariablesParserService, EnvironmentVariablesParserService>();
services.AddSingleton<IStartupItemsParserService, StartupItemsParserService>();
services.AddSingleton<IPnpDevicesParserService, PnpDevicesParserService>();
services.AddSingleton<IGroupPolicyParserService, GroupPolicyParserService>();
services.AddSingleton<IEvidenceParserService, EvidenceParserService>();

// 突合系
services.AddSingleton<IHostlistService, HostlistService>();
services.AddSingleton<IEvidenceVerificationService, EvidenceVerificationService>();
services.AddSingleton<IChecklistParserService, ChecklistParserService>();

// Baseline plugin chain (v3.8.1 で 7 件、InstalledApps を Desktop / Store に分割)
services.AddSingleton<IBaselineComparator, ExecutionSummaryComparator>();
services.AddSingleton<IBaselineComparator, SystemInfoComparator>();
services.AddSingleton<IBaselineComparator, ChecklistComparator>();
services.AddSingleton<IBaselineComparator, DesktopAppsComparator>();
services.AddSingleton<IBaselineComparator, StoreAppsComparator>();
services.AddSingleton<IBaselineComparator, LicenseComparator>();
services.AddSingleton<IBaselineComparator, DomainStatusComparator>();
services.AddSingleton<IBaselineService, BaselineService>();

// その他
services.AddSingleton<IPcMemoService, PcMemoService>();
services.AddTransient<IEvidenceCollectorService, EvidenceCollectorService>();
services.AddTransient<IExcelExportService, ExcelExportService>();

// VM + View
services.AddTransient<MainWindowViewModel>();
services.AddTransient<MainWindow>();
```

`Singleton` と `Transient` の使い分けは次の方針：

- **`Singleton`**: 状態を持つキャッシュ系（`HostlistService` の読み込み済みエントリ、`BaselineService` の baseline PC 参照、6 Comparator の baseline キャッシュ）と、純粋関数的なパーサ系（state-less だが頻繁に再利用する）。`NestedEvidenceDiscoveryService` も `Singleton` だが、内部状態は持たず注入受けの参照のみ
- **`Transient`**: 1 回の出力ごとに完結する `EvidenceCollectorService` / `ExcelExportService` と、Window / ViewModel ライフタイム

## Views 層

| ファイル | 役割 |
|---|---|
| `App.xaml` | アプリ全体のテーマ。CentreCOM 風の青黄赤ストライプ、Button / TextBox / DataGrid / GroupBox / ProgressBar の Style、`BoolToVis`（標準 `BooleanToVisibilityConverter`）と `LicenseStatusToBg`（独自 `LicenseStatusToBackgroundConverter`）の登録 |
| `Views/MainWindow.xaml` (524 行) | フリート画面。Evidence フォルダ選択 / hostlist 読込 / Baseline PC 設定 / 検索バー / DataGrid / 納品データ出力 / ステータスバー（LINK/ACT 風 LED + ProgressBar） |
| `Views/MainWindow.xaml.cs` | `OnPcRowDoubleClick` で PC 行ダブルクリックから `PcDetailWindow` を生成、`OnSettingsClick` で `SettingsWindow` 起動 |
| `Views/PcDetailWindow.xaml` (2,081 行) | 1 PC = 1 ウィンドウ。各セクション（DomainStatus / LocalUsers / FW / Disk / Defender / Memory / PnP / Certificates ...）を `Visibility` バインドで動的表示 |
| `Views/PcDetailWindow.xaml.cs` | `SourceInitialized` で `WorkArea` 内に Width/Height をクランプ、`Loaded` で Top/Left を再クランプ（タイトルバー画面外防止） |
| `Views/SettingsWindow.xaml` (137 行) | 取得チェック CheckBox 群（MAC / BitLocker / Win/Office License 等）+ ベースラインカテゴリ ItemsControl |
| `Views/SettingsWindow.xaml.cs` | 閉じるボタンのみ。設定変更は即時反映（OK/Cancel パターン不採用） |

XAML テーマは Code-Behind ではなく `App.xaml` の `Application.Resources` に集約。Code-Behind は **イベントハンドラ（ダブルクリック / モーダル起動 / クランプ）に限定** され、ビジネスロジック・データ操作は記述しない（`CLAUDE.md` 「Code-Behind へのビジネスロジック記述は厳禁」）。

## ViewModels 層

| ViewModel | 主要責務 | 注入 Service |
|---|---|---|
| `MainWindowViewModel` | フリートの Discovery + Parse、`TargetPCs` (`ObservableCollection<PcEvidence>`) 管理、`SearchText` フィルタ、自動リロード `DispatcherTimer` (30 秒)、Caution 一括判定、Excel 出力ペイロード組み立て | `NestedEvidenceDiscoveryService` / `IEvidenceParserService` / `IEvidenceCollectorService` / `IExcelExportService` / `IHostlistService` / `IEvidenceVerificationService` / `IChecklistParserService` / `IBaselineService` / `IPcMemoService` |
| `PcDetailViewModel` | 1 PC のスナップショット保持（`Pc` / `VerificationReport` / `ChecklistResult` / `ExportHistory` / `BaselineReport` を起動時に固定）、メモ編集 (`SaveMemoCommand`)、auto_capture 画像ナビゲーション、§29/§24/§27/§28/§30 等の表示用算出プロパティ | `IPcMemoService` |
| `SettingsViewModel` | `EvidenceCheckOptions` 共有参照 + ベースラインカテゴリ `ObservableCollection<BaselineCategoryOption>` 管理。チェック ON/OFF 即時反映、`onBaselineCategoriesChanged` コールバックで `MainWindowViewModel` に再評価依頼 | `IBaselineService` |

`MainWindowViewModel` と `SettingsViewModel` は同一の `EvidenceCheckOptions` インスタンスを共有しており、設定ダイアログでチェックを変えると `EvidenceCheckOptions.PropertyChanged` 経由で `MainWindowViewModel` 側のリスナが発火し、全 PC を再評価する設計。

`PcDetailViewModel` は `MainWindowViewModel` に追従しないスナップショット保持型。同じ評価対象 PC を複数別窓で並列に開いて比較する用途を許す（`PcDetailWindow.xaml.cs` 側でクランプロジックがあるのもこの並列起動を想定）。

## Services 層

### 基幹サービス（4 件）

| Service | 役割 | 主な API |
|---|---|---|
| `NestedEvidenceDiscoveryService` (= `IEvidenceDiscoveryService`) | evidence ルート走査、`{timestamp}_{PCName}_{Serial}_evidence` フォルダ命名規則のパース、PC 単位集約、`manager_memo.json` 読み込み、`manifest.json` 連動読み込み | `Task<IReadOnlyList<PcEvidence>> DiscoverAsync(...)` |
| `ManifestReaderService` (= `IManifestReaderService`) | `manifest.json` の 5 段検証（存在 → JSON 妥当 → schemaVersion=1 → manifestType 一致 → デシリアライズ）、`SectionStatusConverter` で未知 status を `Failed` 正規化 | `EvidenceManifest Read(string pcInformationDirectoryPath)` |
| `EvidenceParserService` (= `IEvidenceParserService`) | manifest 駆動 dispatch、§01〜§31 の固有パース（自前実装 + 11 サブパーサに委譲）、SerialNumber Canonical/フォールバック判定、`EvidenceCheckOptions` を見た欠損警告生成 | `void PopulateDetails(PcEvidence pc)` / `void RevalidateEvidence(PcEvidence pc, EvidenceCheckOptions options)` + 各セクションの `Parse...` |
| 11 サブパーサ | §10 / §21 / §22 / §23 / §24 / §25 / §26 / §27 / §28 / §29 / §30 / §31 の専用パース | `Parse(string filePath)` / `Parse(string txtPath, string? csvPath)` 等 |

サブパーサ一覧（インターフェース 1 対 1 対応の実装）：

- `WindowsLicenseParserService` — §21 SoftwareLicensingProduct + Service + slmgr /dlv raw
- `OfficeLicenseParserService` — §22 C2R config + OSPP /dstatus + vNext CSV + INTERPRETATION
- `SecurityBaselineParserService` — §23 TPM / SecureBoot / VBS / LSA / BIOS（5 ブロック個別退避）
- `CertificatesParserService` — §25 4 ストア統合 10 列 CSV
- `BatteryReportParserService` — §26 powercfg /batteryreport HTML
- `HardwareIdentifiersParserService` — §31 ComputerSystem / CSP / BaseBoard / SystemEnclosure
- `MemoryInventoryParserService` — §29 MemorySlots + ArraySummary（2 ファイル）
- `EnvironmentVariablesParserService` — §27 Machine + User 混在 1 リスト
- `StartupItemsParserService` — §28 Registry Run + ScheduledTask 混在
- `PnpDevicesParserService` — §30 Win32_PnPEntity（200〜400 件規模）
- `GroupPolicyParserService` — §24 gpresult summary（HTML はパス保持のみ）

`EvidenceParserService` 内蔵の自前パース対象は §01 SystemInfo / §02 LocalUsers / §03 LocalGroups / §04 LocalGroupMembers / §05 DomainStatus + UserProfiles / §06 NetworkConfig / §07 Printers / §08 BitLocker / §8b Disks + Partitions / §09 MAC / §10 SerialNumber / §11 InstalledApps / §12 Firewall / §13 OptionalFeatures / §15 Power / §16 WiFi / §17 RestorePoints / §18 Defender / §19 WindowsUpdates。詳細 dispatch 表は [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md) §「セクション ID → サブパーサ → Model」を参照。

### 突合・比較サービス

| Service | 役割 |
|---|---|
| `HostlistService` | `hostlist.csv` を読み込み、`NewPCName` をキーとした `Dictionary` を保持。最大 10 台分のプリンタ列（`Printer1Name〜Printer10Port`）も DTO 化 |
| `EvidenceVerificationService` | `HostlistEntry` を期待値、PC の §01/§06/§07/§08 を実測値として 7+ 項目を `VerificationItem` 列に展開。BitLocker は `On (FullyEncrypted)` または `EncryptionInProgress` を許容、プリンタは `IP_xxx` プレフィックス正規化、余剰プリンタ検出は `EvidenceCheckOptions.CheckExtraPrinters` で ON/OFF |
| `ChecklistParserService` | チェックリスト HTML を `<table class="verify-table">` と `<div class="table-wrap">` の 2 表に分けて Regex 抽出（外部ライブラリ不使用）、`export_history.csv` を header-index lookup で安全パース |

### Baseline プラグインチェーン

`BaselineService` は薄いオーケストレータで、実装は `IBaselineComparator` プラグインに分離されている。新カテゴリ追加は **`IBaselineComparator` 実装を 1 つ作って DI 登録** するだけで完結する設計（v3.8.1 で 7 件、`InstalledAppsComparator` は `DesktopAppsComparator` と `StoreAppsComparator` に分割済み。共通比較ロジックは `InstalledAppsCompareLogic` static helper として共有）：

| Comparator | CategoryId | 比較対象 |
|---|---|---|
| `ExecutionSummaryComparator` | `ExecutionSummary` | `export_history.csv` の `ModuleName × Status` |
| `SystemInfoComparator` | `SystemInfo` | §01 `OsName / Version / Cpu / Memory` |
| `ChecklistComparator` | `Checklist` | チェックリスト HTML の `OverallStatus + VerifyItems` |
| `DesktopAppsComparator` | `DesktopApps` | §11 part 1: `11_DesktopApps.csv` の `Name × Version` |
| `StoreAppsComparator` | `StoreApps` | §11 part 2: `11_StoreApps.csv` の `Name × Version` |
| `LicenseComparator` | `License` | §21 Windows + §22 Office の license posture |
| `DomainStatusComparator` | `DomainStatus` | §05 `DomainStatus`（CurrentUser を除く） |

`BaselineService.LoadFromPc(PcEvidence)` で全 Comparator の `CacheBaseline` を呼び、`CompareAll(PcEvidence)` で `EnabledCategoryIds` に含まれる Comparator のみ順次走らせて `BaselineComparisonReport` の対応フィールドに書き込む。設定ダイアログでカテゴリを OFF にすると `SetEnabledCategoryIds` 経由で `CompareAll` から外される（キャッシュ自体は破棄しない）。

### 永続化・出力サービス

| Service | 役割 |
|---|---|
| `PcMemoService` | PC ルート直下の `manager_memo.json` を JSON Camel-case で R/W。fabriq 側の `evidence/` サブツリーには触らず、co-located で配置 |
| `EvidenceCollectorService` | `{outputRoot}/{YYYY_MM_DD_HHmmss}_fabriq_evi/{pcName}_{serial}_{date}/` を作り、`pc_information / auto_capture / bitlocker` をディレクトリ再帰コピー、`checklist.html / export_history.csv` を単独コピー、`manifest_sha256.txt` を生成（manifest 自身は除外） |
| `ExcelExportService` (2,916 行、ClosedXML) | 30+ シート構成のメイン台帳と、PC 個別詳細 `.xlsx` を `pc_details/` サブディレクトリに分割出力。メイン側 PC 名セルから個別ブックへハイパーリンク。32,767 文字超セルは `[TRUNCATED]` マーカーで安全に切り詰め |

## Models 層

集約ルートは `Models/PcEvidence.cs`（1 PC 1 インスタンス、`ObservableObject` 継承）。Discovery で生成され、Parser で各セクションのフィールドが充填されてから `MainWindowViewModel.TargetPCs` に投入される。

主要モデル分類（60+ 件）：

- **manifest 系**: `EvidenceManifest` / `EvidenceManifestSummary` / `ManifestSection` / `SectionStatus` / `UnknownSection`
- **入力期待値**: `HostlistEntry` / `PrinterExpectation`
- **基本情報**: `SystemInfoData` / `MacAddressEntry` / `NetworkInterfaceInfo` / `PrinterEntry` / `BitLockerEntry` / `BitLockerVolumeStatus` / `SerialSourceEntry` / `SerialNumberDetail` / `DomainStatusData`
- **Identity / Domain**: `LocalUserEntry` / `LocalGroupEntry` / `LocalGroupMemberEntry` / `UserProfileEntry`
- **インストール状態**: `InstalledAppEntry` / `OptionalFeatureEntry` / `WindowsUpdateEntry`
- **Storage**: `DiskEntry` / `PartitionEntry`
- **Security**: `FirewallProfileEntry` / `FirewallRuleEntry` / `DefenderStatusData` / `SecurityBaselineData` (+ `TpmInfo` / `VbsInfo` / `LsaProtectionInfo` / `BiosInfo`) / `CertificateEntry` / `RestorePointEntry`
- **Hardware**: `HardwareIdentifiersData` (+ `ComputerSystemInfo` / `ComputerSystemProductInfo` / `BaseBoardInfo` / `SystemEnclosureInfo`) / `MemorySlotEntry` / `MemoryArraySummaryEntry` / `BatteryReportData`
- **Inventory**: `EnvironmentVariableEntry` / `StartupItemEntry` / `PnpDeviceEntry`
- **Power / WiFi / GP**: `PowerSettingsData` / `WindowsLicenseData` / `OfficeLicenseData` (+ `OfficeClickToRunConfig` / `OfficeProductLicense` / `OfficeVnextLicenseEntry` / `OfficeLicenseEvaluation`) / `GroupPolicyReport`
- **Verification / Baseline**: `VerificationReport` / `VerificationItem` / `VerificationStatus` / `ChecklistResult` / `ChecklistVerifyItem` / `ChecklistModuleItem` / `ExportHistoryEntry` / `BaselineComparisonReport` / `BaselineComparisonItem` / `BaselineMatchStatus` / `SystemInfoComparisonResult` / `ChecklistComparisonResult` / `InstalledAppsComparisonResult` / `LicenseComparisonResult` / `DomainStatusComparisonResult` / `BaselineCategoryInfo` / `BaselineCategoryOption`
- **Options / Output**: `EvidenceCheckOptions`（`ObservableObject`）/ `ExcelExportOptions` / `ExcelDeliveryMetadata` / `PcMemo` / `SerialSourceValidity`

## Helpers 層

| Helper | 役割 |
|---|---|
| `Helpers/EvidenceConstants.cs` | ディレクトリ名・ファイル名・接頭辞・出力命名・既知セクション ID マップ・Excel 書式定数の単一ソース。31 セクション分の `KnownSections` ディクショナリと、`PcDetailsSubdirName="pc_details"` / `ExcelMaxColumnWidth=55` 等の出力規約を集約 |
| `Helpers/EvidencePathParser.cs` | log_uploader 親フォルダ名 `{yyyy_MM_dd_HHmmss}_{PCName}_{SerialNumber}_evidence` を `GeneratedRegex` で検証し、最後の `_` で `PCName / SerialNumber` を分割（PCName は `_` を含みうる、SerialNumber は含まない前提） |
| `Helpers/LicenseStatusToBackgroundConverter.cs` | DataGrid のライセンス状態テキストを薄緑（認証済）/ 薄赤（未認証 / 認証失敗）/ 薄黄（要確認）/ 薄グレー（未取得 / 未インストール）の `Brush` に変換 |

## Tests 層

xUnit + `TestData/` フィクスチャ駆動。ビルド時に `TestData/**` が `CopyToOutputDirectory="PreserveNewest"` で出力ディレクトリに複製され、`TestPaths.GetTestDataDirectory(...)` で解決される。

代表的なフィクスチャ：

- `TestData/Manifests/` — `valid_v1` / `valid_v1_6` / `unsupported_schema_v2` / `invalid_json` / `unknown_status`（schemaVersion 検証の正例・反例）
- `TestData/Licenses/` — `windows_license_kms_volume.txt` / `office_v15_subscription_provisioned.txt + .csv` / `office_oob_grace.txt` 等
- `TestData/SecurityBaseline/` — `full_modern_pc.txt` / `partial_failures.txt`（probe 個別退避の検証）
- `TestData/HardwareIdentifiers/` / `MemoryInventory/` / `EnvironmentVariables/` / `StartupItems/` / `PnpDevices/` / `GroupPolicy/` — §27〜§31 各セクションの正常 + 空 + 列順入替 + 失敗パターン
- `TestData/Certificates/sample_9rows.csv` / `Battery/laptop_real.html` / `Checklist/seven_columns.html` — 実ファイル形式の代表サンプル

## 関連ドキュメント

- 入力 evidence 構造と manifest dispatch: [fabriq_evidence_manager__architecture__02_evidence_input.md](fabriq_evidence_manager__architecture__02_evidence_input.md)
- プロジェクト概要: [fabriq_evidence_manager__overview__readme.md](fabriq_evidence_manager__overview__readme.md)
