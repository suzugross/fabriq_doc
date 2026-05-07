# Models 索引（37 件）

> **対象**: fabriq_studio / reference
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`、2026-05-06）
> **ドキュメント更新日**: 2026-05-07

`FabriqStudio/Models/` 配下の **37 のデータモデル** の用途索引。各 Model の役割 + 主要フィールド + 使用箇所を表形式で網羅する。

各 Model は **CommunityToolkit.Mvvm の `[ObservableProperty]` Source Generator** + **CsvHelper の `[Name]` / `[Optional]` / `[Ignore]` 属性** + **System.Text.Json の `JsonPropertyName` / `JsonPropertyOrder`** を組み合わせて、**CSV / JSON 両方向のシリアライズ + WPF 双方向バインド + Dirty 検知** を実現している。

---

## カテゴリ別サマリ

| カテゴリ | 件数 | 代表 Model |
|---|---|---|
| Host | 3 | `HostEntry` / `PrinterInfo` / `HostListExportRequest` + Result |
| Profile | 2 | `ProfileEntry` / `ProfileScriptEntry` |
| Module | 2 | `ModuleMasterEntry` / `ModulePresetEntry` |
| Registry catalog | 1 | `RegistryTemplateEntry` |
| Worker / Category / Log | 3 | `WorkerEntry` / `CategoryItem` / `LogDestination` |
| Looper | 1 | `LooperEntry` |
| Printer detection | 3 | `PrinterDriverInfo` / `ArchiveExtractResult` / `DriverExportResult` |
| Backup | 1 | `FabriqBackupRequest` + Result |
| Update overlay | 3 | `OverlayRules` / `FabriqUpdatePlan` 系 / `FabriqUpdateResult` 系 |
| Utility | 2 | `SemVer` / `RowMoveRequest` |
| **Pianist** | **16** | 後述 |
| **合計** | **37** | |

---

## Host 系（3）

### `HostEntry`

`hostlist.csv` 1 行。ObservableObject で **43 ObservableProperty** + JSON ベースの `Clone` / `ContentEquals`。詳細: [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md)。

| 区分 | 列数 | 主なフィールド |
|---|---|---|
| 識別 | 3 | `AdminID` / `OldPCName` / `NewPCName` |
| 有線 LAN | 3 | `EthernetIP` / `EthernetSubnet` / `EthernetGateway` |
| 無線 LAN | 3 | `WifiIP` / `WifiSubnet` / `WifiGateway` |
| DNS | 4 | `DNS1` 〜 `DNS4` |
| BitLocker | 1 | `Pin` |
| プリンタ | 30 | `Printer1Name` / `Printer1Driver` / `Printer1Port` 〜 `Printer10*` |

### `PrinterInfo`

[Models/PrinterInfo.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PrinterInfo.cs)。`HostDetailView` の **「プリンタ」DataGrid 表示専用** に `HostEntry` のフラット 30 列を行単位に変換したモデル：

| フィールド | 型 |
|---|---|
| `Number` | int（1〜10） |
| `Name` / `Driver` / `Port` | string |

double-binding 対応 ObservableObject。`HostEntry` と `PrinterInfo` の同期は ViewModel 側責務。

### `HostListExportRequest` / `HostListExportResult`

[Models/HostListExportRequest.cs](file:///E:/fabriq_studio/FabriqStudio/Models/HostListExportRequest.cs) は **2 record を 1 ファイル**：

```csharp
public record HostListExportRequest(
    IReadOnlyList<HostEntry> Hosts,
    string ParentFolder,
    string Memo,
    bool Decrypt);

public record HostListExportResult(
    string ExportFolderPath,
    int HostCount,
    int DecryptedCells,
    int RemainingEncCells,
    IReadOnlyList<string> Errors);
```

`Decrypt` トグルで `ENC:` 値を平文化して export。詳細: [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md) §「hostlist 一括 Export」。

---

## Profile 系（2）

### `ProfileEntry`

[Models/ProfileEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ProfileEntry.cs)。`profiles/` 直下の 1 csv ファイルを表す軽量 class：

| フィールド | 用途 |
|---|---|
| `Name` | 拡張子なしファイル名（profile 名）|
| `FilePath` | フル絶対パス |

`ToString()` override で ComboBox 表示時に `Name` だけ出る。

### `ProfileScriptEntry`

[Models/ProfileScriptEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ProfileScriptEntry.cs)。`profiles/*.csv` の 1 行（fabriq の Profile CSV スキーマ準拠）：

| 列 | 型 | 必須 | CSV 順 |
|---|---|---|---|
| `Order` | int | 必須 | 1 |
| `ScriptPath` | string | 必須 | 2 |
| `Enabled` | string `"0"`/`"1"` | 必須 | 3 |
| `Description` | string | 必須 | 4 |
| `Segment` | string | `[Optional]` | 5 |
| `Note` | string | `[Optional]` | 6 |
| `ErrorMode` | string | `[Optional]` | 7 |
| `Group` | string | `[Optional]`（kernel 3.2.0+） | 8 |

`[Optional]` 属性は `CsvHelper.Configuration.Attributes` 由来で、列が CSV に無くても読み込み成功（旧形式互換）。

### 算出プロパティ

| プロパティ | 用途 |
|---|---|
| `IsEnabled` | `Enabled != "0"` の bool ラッパー（CheckBox 双方向バインド用、`[Ignore]`） |
| `IsSystemCommand` | `__RESTART__` 等のシステム組み込みコマンド判定（`__` で始まり / 終わる）|
| `DisplayName` | フレンドリー名（システムコマンドはそのまま、それ以外はファイル名の拡張子なし） |

詳細な Profile CSV 規約: [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md)、利用ガイド: [fabriq__usage__03_profile_execution_linear.md](fabriq__usage__03_profile_execution_linear.md)。

---

## Module 系（2）

### `ModuleMasterEntry`

[Models/ModuleMasterEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ModuleMasterEntry.cs)。`modules/{standard,extended}/<dir>/module.csv` 1 行 + ディレクトリスキャン由来メタ：

| フィールド | 由来 | CSV カラム |
|---|---|---|
| `MenuName` | csv | ○ |
| `Category` | csv | ○ |
| `Script` | csv | ○ |
| `Order` | csv（int） | ○ |
| `Enabled` | csv（string `"0"`/`"1"`） | ○ |
| `ModuleDir` | フォルダ名 | × `[Ignore]` |
| `Kind` | `"standard"` / `"extended"` | × `[Ignore]` |

`[Ignore]` は CsvHelper のヘッダーマッピング対象外。`IModuleService.GetAllModulesAsync` がディレクトリ走査時にこれらを後付けする。

### `ModulePresetEntry`

[Models/ModulePresetEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ModulePresetEntry.cs)。`preset.csv` の 1 行（**固定 3 列、UTF-8 BOM**）：

| 列 | 用途 |
|---|---|
| `Column` | 対象 CSV の列名（大文字小文字無視） |
| `Value` | セルに書き込まれる実値 |
| `Label` | 表示用ラベル（Phase 1 では UI 表示に使わない、将来予約） |

`IModulePresetService.LoadAsync` がこれを **`Dictionary<string, IReadOnlyList<string>>`**（列名 → Value 一覧）に整形する。詳細: [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md) §「preset.csv」。

---

## Registry catalog（1）

### `RegistryTemplateEntry`

[Models/RegistryTemplateEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/RegistryTemplateEntry.cs)。`catalog.json` の 1 entry。9 フィールド（id / category / title / hive / keyPath / keyName / type / value / description / tags）。

詳細: [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)。

---

## Worker / Category / Log 系（3）

### `WorkerEntry`

[Models/WorkerEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/WorkerEntry.cs)。`kernel/csv/workers.csv` の 1 行：

| フィールド | 用途 |
|---|---|
| `ID` | 作業者 ID（property name は `_iD`、生成プロパティ `ID`） |
| `Name` | 作業者名 |

`BasicParamsView` 編集対象。

### `CategoryItem`

[Models/CategoryItem.cs](file:///E:/fabriq_studio/FabriqStudio/Models/CategoryItem.cs)。`kernel/csv/categories.csv` の 1 行：

| フィールド | 用途 |
|---|---|
| `Category` | カテゴリ名（モジュールの Category 列と一致） |
| `Order` | int（表示順）|

軽量 class（`ObservableObject` ではない）。`BasicParamsViewModel` で読込・保存。

### `LogDestination`

[Models/LogDestination.cs](file:///E:/fabriq_studio/FabriqStudio/Models/LogDestination.cs)。`kernel/csv/log_destinations.csv` の 1 行：

| フィールド | 用途 |
|---|---|
| `Path` | UNC 共有 / ローカル / SFTP URL |
| `Type` | プロトコル種別 |
| `Enabled` | `"0"`/`"1"` 文字列保持（fabriq 慣例）|
| `AuthUser` / `AuthPass` | 認証情報（`AuthPass` は `ENC:` 暗号化対象）|
| `Description` | メモ |

`extended/log_uploader` モジュールが消費する。

---

## Looper（1）

### `LooperEntry`

[Models/LooperEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/LooperEntry.cs)。`looper_list.csv` の 1 行。**`INotifyDataErrorInfo` を実装** して WPF `Validation.Errors` 表示と連携：

| フィールド | 型 | 検証 |
|---|---|---|
| `Enabled` | int | `1=実行 / 0=スキップ` |
| `ScriptPath` | string | （検証なし） |
| `MaxRetry` | int | **>=1**（無限ループ防止） |
| `IntervalSec` | int | **>=0** |
| `Condition` | string | **`OnError` or `Always`** のみ |
| `Description` | string | |
| `Segment` | string | `[Optional]` |

定数：

```csharp
public const string ConditionOnError = "OnError";
public const string ConditionAlways  = "Always";
public static readonly IReadOnlyList<string> AllConditions = [ConditionOnError, ConditionAlways];
```

各検証は `partial void OnXxxChanged(value)` ハンドラで行い、`_errors` 辞書に蓄積。`HasErrors` / `GetErrors` を WPF が監視して赤枠表示。

詳細: [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md) §「looper_list.csv」。

---

## Printer detection（3）

### `PrinterDriverInfo`

[Models/PrinterDriverInfo.cs](file:///E:/fabriq_studio/FabriqStudio/Models/PrinterDriverInfo.cs)。INF 解析で抽出された 1 ドライバ（`sealed class` + `init` のみ）：

| フィールド | 用途 |
|---|---|
| `DriverName` | INF 内 `[*.NTamd64]` セクションの **モデル名文字列**（例: `"EPSON PX-S505 Series"`） |
| `InfFileName` | 短い名前（例: `E1WF1CAJ.INF`） |
| `InfFilePath` | 絶対パス |
| `FolderName` | スキャン起点直下のトップフォルダ名（例: `EPSON PX-S505 Series`） |
| `Architecture` | 現状 `NTamd64` 固定 |

fabriq の `printer_driver_list.csv` の `DriverName` 列にそのまま転記できる値。

### `ArchiveExtractResult`

[Models/ArchiveExtractResult.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ArchiveExtractResult.cs)。`IPrinterDriverDetectorService.ExtractArchivesAsync` の結果：

| フィールド | 用途 |
|---|---|
| `Extracted` | 新規展開件数 |
| `Skipped` | 同名フォルダ既存でスキップした件数（冪等）|
| `Failed` | 展開失敗件数（7z 未発見の .exe を含む）|
| `Messages` | ログ用 1 行メッセージの時系列 |

### `DriverExportResult`

[Models/DriverExportResult.cs](file:///E:/fabriq_studio/FabriqStudio/Models/DriverExportResult.cs)。`ExportToWorkspaceAsync` の結果：

| フィールド | 用途 |
|---|---|
| `Added` | 新規追加行数（0 or 1）|
| `Skipped` | DriverName 重複でスキップ（0 or 1）|
| `Error` | エラーメッセージ（null = 正常）|

レジストリ辞書の `Services.ExportResult` と同じ意味論。

---

## Backup（1）

### `FabriqBackupRequest` / `FabriqBackupResult`

[Models/FabriqBackupRequest.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqBackupRequest.cs) で 2 record を 1 ファイル：

```csharp
public record FabriqBackupRequest(
    string SourceRoot,
    string ParentFolder,
    string Memo);

public record FabriqBackupResult(
    string BackupFolderPath,
    int CopiedFileCount,
    int ExcludedFileCount,
    long TotalBytes,
    IReadOnlyList<string> Errors)
{
    public bool HasErrors => Errors.Count > 0;
}
```

`IFabriqBackupService.BackupAsync` 入出力。出力先は `<ParentFolder>/fabriq_backup_yyyyMMdd_HHmmss/`。

---

## Update overlay 系（3）

### `OverlayRules`

[Models/OverlayRules.cs](file:///E:/fabriq_studio/FabriqStudio/Models/OverlayRules.cs)。`dev/framework_overlay_rules.json` の C# 表現：

| トップレベルフィールド | 型 |
|---|---|
| `SchemaVersion` | int（現行 1）|
| `Description` | string? |
| `ExcludeDirsTopLevel` / `ExcludeDirsRecursive` | List\<string\>（除外パターン）|
| `ExcludeFilesKernelLevel` | List\<string\> |
| `ModuleCsvWhitelist` | List\<string\> |
| `Bundles.Kernel` | `KernelBundleDef`（VersionFile + IncludePaths）|
| `Bundles.Module` | `ModuleBundleDef`（PathPattern + VersionFilePattern + RequiresKernelFilePattern + TypeValues）|

詳細仕様は fabriq 公開契約 [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md)（KERNEL_API.md §9）。

### `FabriqUpdatePlan` / `BundleUpdateItem` / `UpdateAction`

[Models/FabriqUpdatePlan.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqUpdatePlan.cs) で 3 型を 1 ファイル：

#### `UpdateAction` enum

| 値 | 意味 |
|---|---|
| `Update` | template > target、UPDATE 対象（既定選択） |
| `New` | template あり / target 無し、lazy seed で新規（既定選択） |
| `SkipSame` | 両 version 同一、SKIP |
| `SkipTargetNewer` | template < target、SKIP（警告）|
| `SkipNoTemplate` | template に VERSION 無し、SKIP |
| `Preserve` | target にあり template に無いモジュール、site-custom として保持 |

#### `BundleUpdateItem`

| フィールド | 用途 |
|---|---|
| `BundleKey` | `kernel` or `modules/{std,ext}/<name>` |
| `DisplayName` / `GroupName` | UI 表示 |
| `TargetVersion` / `TemplateVersion` / `RequiresKernel` | `SemVer?` |
| `Action` | `UpdateAction` |
| `WarningMessage` | REQUIRES_KERNEL 不整合等の警告 |
| `IsSelected` / `IsBlocked` | ObservableProperty（チェックボックス状態） |
| `CanSelect` | `Action ∈ {Update / New / SkipSame / SkipTargetNewer}` のみ操作可 |

#### `FabriqUpdatePlan`

```csharp
public record FabriqUpdatePlan(
    string TemplateRoot,
    string TargetRoot,
    OverlayRules Rules,
    SemVer? TargetKernel,
    SemVer? TemplateKernel,
    IReadOnlyList<BundleUpdateItem> Bundles);
```

`IFabriqUpdateService.ComputePlanAsync` の戻り値。

### `FabriqUpdateRequest` / `BundleUpdateResult` / `FabriqUpdateResult` / `PreflightResult`

[Models/FabriqUpdateResult.cs](file:///E:/fabriq_studio/FabriqStudio/Models/FabriqUpdateResult.cs) で 4 record を 1 ファイル：

#### `FabriqUpdateRequest`

```csharp
public record FabriqUpdateRequest(
    FabriqUpdatePlan Plan,
    IReadOnlyList<BundleUpdateItem> SelectedBundles,
    string BackupZipPath,
    bool DryRun,
    string LogFilePath);
```

#### `BundleUpdateResult`

```csharp
public record BundleUpdateResult(
    string BundleKey,
    bool Success,
    int TouchedFileCount,
    int SkippedCount,
    IReadOnlyList<string> Errors,
    SemVer? FromVersion,
    SemVer? ToVersion,
    UpdateAction Action);
```

#### `FabriqUpdateResult`

```csharp
public record FabriqUpdateResult(
    IReadOnlyList<BundleUpdateResult> BundleResults,
    string BackupZipPath,
    string LogFilePath,
    bool DryRun,
    IReadOnlyList<string> SchemaWarnings)
{
    public int SuccessCount => ...;
    public int FailureCount => ...;
    public int SkippedCount => ...;
}
```

#### `PreflightResult`

```csharp
public record PreflightResult(
    IReadOnlyList<string> Errors,
    IReadOnlyList<string> RequiresKernelBlocks)
{
    public bool CanProceed => Errors.Count == 0 && RequiresKernelBlocks.Count == 0;
}
```

`IFabriqUpdateService.RunPreflight` の戻り値。fabriq 公開契約 §9.7 の安全チェック結果。

---

## Utility（2）

### `SemVer`

[Models/SemVer.cs](file:///E:/fabriq_studio/FabriqStudio/Models/SemVer.cs)。`readonly record struct` + `IComparable<SemVer>`：

```csharp
public readonly record struct SemVer(int Major, int Minor, int Patch) : IComparable<SemVer>
{
    public static bool TryParse(string? s, out SemVer result);
    public static SemVer? TryParseFile(string path);   // VERSION ファイル → SemVer
    public int CompareTo(SemVer other);
    public static bool operator > / < / >= / <=;
}
```

書式: `^(\d+)\.(\d+)\.(\d+)$` の 3 コンポーネント。**pre-release / build metadata は非対応**（現行 fabriq で未使用、`System.Version` は 4 コンポーネントなので独自パーサ）。

`KERNEL_VERSION` / モジュール `VERSION` / `REQUIRES_KERNEL` の比較に使用。

### `RowMoveRequest`

[Models/RowMoveRequest.cs](file:///E:/fabriq_studio/FabriqStudio/Models/RowMoveRequest.cs)。

```csharp
public sealed record RowMoveRequest(int SourceIndex, int TargetIndex);
```

DataGrid 行 D&D で `Helpers.DataGridRowDragDropBehavior` が構築 → ViewModel の `MoveRowCommand` に渡す。`ObservableCollection.Move(SourceIndex, TargetIndex)` にそのまま渡せるよう、Behavior 側で抜き取り後の位置に補正済み。

---

## Pianist 系（16）

[fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md) §「7. Pianist 16 Models 索引」で詳細説明。ここでは 1 行サマリのみ：

| Model | 役割 |
|---|---|
| `PianistProfileEntry` | プロファイルフォルダ 1 件（Name + FolderPath） |
| `PianistProfileMetadata` | `pianist.json` 6 フィールド（schema/label/description/target_app/default_phase/version） |
| `PianistProfileData` | プロファイル全データ集約 |
| `PianistStep` | `procedure.csv` 1 行（8 列、PhaseID/PhaseLabel/Color/StepNo/Action/Value/Wait/Note） |
| `PianistShortcut` | `shortcuts.csv` 1 行（5 列、Label/Type/Path/Args/Note）|
| `PianistValueTable` | `values.csv` 全体（VariableColumns + Rows + Star + WasLegacyFormat）|
| `PianistValueRow` | `values.csv` 1 行（NewPCName + Cells dictionary、`*` 行は IsStar） |
| `PianistInstructionFile` | `instructions/<PhaseID>.txt` パース結果（4 section + WasLegacyFormat） |
| `PianistSampleEntry` | `[Samples]` 1 行（File/Caption/Exists） |
| `PianistPhaseSummary` | Phase 一覧表示用集約（PhaseID/PhaseLabel/Color/StepCount） |
| `PianistKeyPreset` | SendKeys プリセット（Code + Description） |
| `PianistActionOption` | Action 列の選択肢（Code 英名 + Label 日本語） |
| `PianistListEntry` | `pianist_list.csv` 1 行（5 列、Enabled/ProfileName/Group/Description/Segment） |
| `PianistVariableSelection` | Variables sub-tab 編集状態（Name/IsIncluded/IsAutoDiscovered/SampleValue/IsOrphan） |
| `PianistValidationIssue` | 整合性チェック結果（Severity Error/Warning/Info + Category + Message + Source） |
| `PianistLegacyValueEntry` | 旧 long format values.csv 1 行（移行用、Phase 7） |

---

## Models 設計の共通パターン

### `[ObservableProperty]` Source Generator

Pianist / Host / Looper / Profile 系で多用。`private string _foo = ""` 宣言だけで public プロパティ + `INotifyPropertyChanged` 通知が自動生成。

```csharp
[ObservableProperty] private string _newPCName = "";

// 自動生成:
//   public string NewPCName {
//       get => _newPCName;
//       set { ...PropertyChanged 発火... }
//   }
```

### `[NotifyPropertyChangedFor]` で算出プロパティ通知

```csharp
[ObservableProperty]
[NotifyPropertyChangedFor(nameof(IsStar))]
private string _newPCName = "";

public bool IsStar => string.Equals(NewPCName, "*", StringComparison.Ordinal);
```

`NewPCName` の変更時に `IsStar` の通知も自動連動。

### `partial void OnXxxChanged(value)` 検証フック

`LooperEntry` の `OnMaxRetryChanged` / `OnIntervalSecChanged` / `OnConditionChanged` 等。Source Generator が生成する partial method を埋めて副作用を実装。

### CsvHelper 属性

| 属性 | 用途 |
|---|---|
| `[Name("foo")]` | CSV ヘッダー名指定（C# property 名と異なるとき） |
| `[Optional]` | CSV に列が無くても OK（旧形式互換） |
| `[Ignore]` | CSV マッピング対象外（in-memory 専用フィールド） |

`ProfileScriptEntry` で `[ObservableProperty] [property: Optional] private string _segment = "";` のように **Source Generator 経由で属性を生成プロパティに伝播** させるパターンが頻出（`property:` プレフィックス）。

### JSON 系属性

| 属性 | 用途 |
|---|---|
| `[JsonPropertyName("foo")]` | JSON キー名指定（snake_case 等） |
| `[JsonPropertyOrder(N)]` | キー出力順固定（diff 安定）|

`PianistProfileMetadata` / `RegistryTemplateEntry` / `OverlayRules` 等で多用。

### record 型の活用

Backup / HostListExport / FabriqUpdate 系の Request / Result が **record**。**immutable + 構造的等価** + `with` 式で書き換え。Service の入出力境界では record、UI 双方向バインドが必要なところは `ObservableObject` を使い分け。

---

## ファイル数 vs 型数

`Models/` 配下のファイル数は **37**、ただし型数は **40+**：

| ファイル | 含まれる型 |
|---|---|
| `HostListExportRequest.cs` | `HostListExportRequest` + `HostListExportResult` |
| `FabriqBackupRequest.cs` | `FabriqBackupRequest` + `FabriqBackupResult` |
| `FabriqUpdatePlan.cs` | `UpdateAction` enum + `BundleUpdateItem` + `FabriqUpdatePlan` |
| `FabriqUpdateResult.cs` | `FabriqUpdateRequest` + `BundleUpdateResult` + `FabriqUpdateResult` + `PreflightResult` |
| `OverlayRules.cs` | `OverlayRules` + nested `BundleDefs` + `KernelBundleDef` + `ModuleBundleDef` |

「Request + Result」「Plan + Result」のような **対になる型を同一ファイル**にまとめる規約。型単位で数えると 50 弱になる。

---

## 関連ドキュメント

- アーキテクチャ概要: [fabriq_studio__architecture__01_layers.md](fabriq_studio__architecture__01_layers.md)
- ワークスペース概念: [fabriq_studio__architecture__02_workspace.md](fabriq_studio__architecture__02_workspace.md)
- HostEntry の詳細スキーマ: [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md)
- Pianist 16 Models の DSL + ファイル構造詳細: [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md)
- Registry catalog: [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)
- Services 索引: [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md)
- 小さな CSV スキーマ集: [fabriq_studio__reference__csv_schemas.md](fabriq_studio__reference__csv_schemas.md)
- 暗号化（`ENC:` 値の規約）: [fabriq_studio__contracts__crypto_interop.md](fabriq_studio__contracts__crypto_interop.md)
- fabriq 本体側 Profile CSV 公開契約: [fabriq__contracts__profile_csv_schema.md](fabriq__contracts__profile_csv_schema.md)
- fabriq 本体側 Overlay 公開契約: [fabriq__contracts__overlay_contract.md](fabriq__contracts__overlay_contract.md)
