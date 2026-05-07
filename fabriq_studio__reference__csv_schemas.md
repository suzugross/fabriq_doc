# fabriq_studio リファレンス: 小 CSV スキーマ集約

> **対象**: fabriq_studio / reference (CSV / DTO スキーマ)
> **対象バージョン**: commit `3897c6e`（取得元: `git -C E:\fabriq_studio rev-parse --short HEAD`）
> **ドキュメント更新日**: 2026-05-07

本書は fabriq_studio が **読み書きする CSV ファイル群** および **Service レイヤを跨ぐ Request/Result DSL** のうち、独立したリファレンスを建てるほどの分量がない小さなスキーマを 1 ファイルに集約したリファレンスである。

カバー範囲:

1. `module.csv`（モジュールマスタ — fabriq 本体が producer、studio が consumer）
2. `preset.csv`（モジュールごとの列値プリセット）
3. `looper_list.csv`（Script Looper 用ループ設定）
4. `kernel/csv/log_destinations.csv`（ログ出力先一覧）
5. `kernel/csv/workers.csv`（作業者マスタ）
6. `kernel/csv/categories.csv`（モジュール分類マスタ）
7. `HostListExportRequest` / `HostListExportResult`（端末一覧エクスポート DSL）

大型の CSV / JSON スキーマは別書（[fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md) / [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md) / [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md)）に分割しているのでそちらを参照。

---

## 共通の取り扱い（読み書きの規約）

### 文字コードと BOM

- **書き込み時**は **UTF-8 BOM** を付与する（`new UTF8Encoding(encoderShouldEmitUTF8Identifier: true)`）。これは PowerShell 5.1 がデフォルトで CSV を BOM 付き UTF-8 として読むため、fabriq 本体側との互換性を確保する目的。
- **読み取り時**は `Encoding.UTF8` + `detectEncodingFromByteOrderMarks: true` で BOM 有無のどちらも受容する。

### CSV ライブラリ

- すべて **CsvHelper 33.0.1** を使用。`CsvConfiguration(CultureInfo.InvariantCulture)` を基本とし、ヘッダ大小文字の許容や欠損列の許容はファイル種別ごとに微調整される。
- 改行は CsvHelper のデフォルト（**CRLF**、最終行末尾に改行付与）。

### 共通サービス層

- `kernel/csv/*.csv`（hostlist / workers / categories / log_destinations）は **`ICsvService`** 経由で読み書きされる。引数は **fabriq ルートからの相対パス**（`"kernel/csv/workers.csv"` のように `/` 区切り）を渡す。
- モジュール配下 CSV（`modules/{kind}/{module}/*.csv`）は **`IModuleService` / `IModulePresetService` / `LooperService`** が個別に File API + CsvHelper で読み書きする（こちらは絶対パス引数）。
- Service 層の役割分担とエラーハンドリングは [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md) を参照。

### producer / consumer の別

| ファイル | producer | consumer (studio 側) | 備考 |
|---|---|---|---|
| `module.csv` | fabriq 本体（手動編集 + 一部 studio 生成） | `ModuleService.GetAllModulesAsync` | Looper 系は studio が新規生成する |
| `preset.csv` | 開発者 / AI 生成（手動配置） | `ModulePresetService.LoadAsync` | 任意ファイル。無くても動く |
| `looper_list.csv` | studio (`LooperEditorViewModel`) | fabriq の `script_looper.ps1` | studio から書き込み専用、読み戻しは別経路 |
| `kernel/csv/log_destinations.csv` | studio (`BasicParamsViewModel`) | fabriq 本体ログ出力 | 双方向（studio で編集、fabriq で消費） |
| `kernel/csv/workers.csv` | studio | fabriq 本体（作業者選択 UI） | 双方向 |
| `kernel/csv/categories.csv` | studio (`ModuleEditViewModel`) | studio + fabriq | モジュール分類のマスタ |

---

## 1. module.csv — モジュールマスタ

### 配置

```
{fabriqRoot}/modules/standard/{module_dir}/module.csv
{fabriqRoot}/modules/extended/{module_dir}/module.csv
```

`standard` は fabriq 同梱の標準モジュール群、`extended` は現場拡張用。studio は両方を同じスキーマでスキャンする。

### モデル

[Models/ModuleMasterEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ModuleMasterEntry.cs)

```csharp
public class ModuleMasterEntry
{
    // CSV 上の 5 カラム（順序固定、ヘッダ名一致）
    public string MenuName { get; set; } = "";
    public string Category { get; set; } = "";
    public string Script   { get; set; } = "";
    public int    Order    { get; set; }
    public string Enabled  { get; set; } = "0";

    // [Ignore]: ディレクトリスキャン時に注入するメタデータ
    [Ignore] public string ModuleDir { get; set; } = "";   // フォルダ名
    [Ignore] public string Kind      { get; set; } = "";   // "standard" or "extended"
}
```

### カラム定義

| 列名 | 型 | 役割 | 例 / 制約 |
|---|---|---|---|
| `MenuName` | string | メニュー表示名 | `"Adobe Reader インストール"` |
| `Category` | string | カテゴリ名（`categories.csv` の `Category` と整合） | `"Apps"`, `"Scripts"` |
| `Script` | string | 同フォルダ内のスクリプト相対名 | `"main.ps1"`, `"script_looper.ps1"` |
| `Order` | int | カテゴリ内表示順 | 整数。Looper 自動生成時は **`90`** で固定 |
| `Enabled` | string `"0"`/`"1"` | 既定有効フラグ | **fabriq 慣例で文字列**（int ではない）。空欄は `"0"` |

`Enabled` を `string` 型としているのは fabriq 本体（PowerShell 側）が `"0"`/`"1"` 文字列で扱うためであり、`bool` への変換は studio 側で行わない。

### 注入される `[Ignore]` フィールドの意味

- `ModuleDir`: モジュールフォルダ名（`adobe_reader` 等）。CSV には書かれず、`ModuleService.GetAllModulesAsync` が `Path.GetFileName(moduleDir)` で注入。
- `Kind`: `"standard"` / `"extended"` のいずれか。同様にディレクトリスキャン時に注入。

### studio 側で生成される module.csv（Looper 用）

`LooperService.CreateLooperModuleAsync` は新規 Looper モジュール作成時に下記を **新規書き込み** する（[Services/LooperService.cs:101-109](file:///E:/fabriq_studio/FabriqStudio/Services/LooperService.cs)）:

```
MenuName,Category,Script,Order,Enabled
{sanitized_module_name},Scripts,script_looper.ps1,90,0
```

- ファイル名サニタイズは Windows 不正文字（`< > : " / \ | ? *`）を `_` に置換。
- Order は固定で `90`、Enabled は既定 `"0"`（無効状態）で生成。

---

## 2. preset.csv — モジュール列値プリセット

### 配置

```
{fabriqRoot}/modules/{kind}/{module_dir}/preset.csv   (任意)
```

存在しない／破損している場合は **空辞書を返してフォールバック**（fabriq 本体には一切影響しない graceful degradation）。

### モデル

[Models/ModulePresetEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/ModulePresetEntry.cs)

```csharp
public sealed class ModulePresetEntry
{
    public string Column { get; set; } = "";  // 対象 CSV の列名（大文字小文字非区別）
    public string Value  { get; set; } = "";  // セルに書き込まれる実値
    public string Label  { get; set; } = "";  // 表示用ラベル（Phase 1 では未使用、予約）
}
```

### カラム定義（固定 3 列）

| 列名 | 役割 |
|---|---|
| `Column` | 対象モジュール CSV の列名。**大文字小文字を区別しない**（`PrepareHeaderForMatch = args => args.Header.Trim()`）。 |
| `Value` | プリセットの実値。**空文字も有効な候補**（optional 列のクリア用に許容）。 |
| `Label` | 表示用ラベル。**Phase 1 では未使用**、将来の UI 改善のために予約された列。 |

### ロード仕様（[Services/ModulePresetService.cs](file:///E:/fabriq_studio/FabriqStudio/Services/ModulePresetService.cs)）

- ヘッダの空白は trim され、大文字小文字差は許容（AI 生成 CSV のブレに耐える設計）。
- `MissingFieldFound = null` / `HeaderValidated = null` で欠損列・余分列を許容。
- `Column` をキーに `Value` をリスト化。**同一 (Column, Value) の重複は初出を優先して排除**（CSV 書き間違い対策）。
- 戻り値は `IReadOnlyDictionary<string, IReadOnlyList<string>>` で大小非区別キー（`StringComparer.OrdinalIgnoreCase`）。

### 例

```csv
Column,Value,Label
TargetUser,"NT AUTHORITY\SYSTEM","システム"
TargetUser,"$env:USERNAME","現在ユーザー"
LogLevel,"INFO",""
LogLevel,"DEBUG",""
```

→ `{ "TargetUser": ["NT AUTHORITY\\SYSTEM", "$env:USERNAME"], "LogLevel": ["INFO", "DEBUG"] }`

---

## 3. looper_list.csv — Script Looper 用ループ設定

### 配置

```
{fabriqRoot}/modules/extended/{looper_module_name}/looper_list.csv
```

studio から **新規作成（書き込み）** する CSV。fabriq 本体の `script_looper.ps1`（同フォルダの `$PSScriptRoot/looper_list.csv` を読み込む）に渡る。

### モデル

[Models/LooperEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/LooperEntry.cs)

```csharp
public partial class LooperEntry : ObservableObject, INotifyDataErrorInfo
{
    public const string ConditionOnError = "OnError";
    public const string ConditionAlways  = "Always";
    public static readonly IReadOnlyList<string> AllConditions =
        [ConditionOnError, ConditionAlways];

    [ObservableProperty] private int    _enabled;
    [ObservableProperty] private string _scriptPath = "";
    [ObservableProperty] private int    _maxRetry   = 1;
    [ObservableProperty] private int    _intervalSec;
    [ObservableProperty] private string _condition  = "OnError";
    [ObservableProperty] private string _description = "";
    [ObservableProperty] [property: Optional] private string _segment = "";
}
```

### カラム定義（7 列固定順）

| 列名 | 型 | 役割 | 制約 |
|---|---|---|---|
| `Enabled` | int | 1=実行 / 0=スキップ | 0 または 1（数値） |
| `ScriptPath` | string | 対象スクリプトのパス | fabriq ルート相対パスまたは絶対パス |
| `MaxRetry` | int | 最大実行回数（無限ループ防止） | **1 以上**。違反時 `INotifyDataErrorInfo` で UI エラー |
| `IntervalSec` | int | リトライ間隔（秒） | **0 以上**。違反時同上 |
| `Condition` | string | リトライ条件 | **`"OnError"` または `"Always"` のみ**。違反時同上 |
| `Description` | string | 管理用説明文 | 自由記述。コンソール表示にも使用 |
| `Segment` | string | fabriq セグメント分離機能用 | 任意。`[Optional]` 付きで欠損列を許容 |

### バリデーション挙動

`LooperEntry` は `INotifyDataErrorInfo` を実装しており、以下の `partial void` ハンドラで値変更ごとにバリデーションを実行する:

| ハンドラ | 違反条件 | エラーメッセージ |
|---|---|---|
| `OnMaxRetryChanged(int value)` | `value < 1` | `"MaxRetry は 1 以上の値を指定してください。"` |
| `OnIntervalSecChanged(int value)` | `value < 0` | `"IntervalSec は 0 以上の値を指定してください。"` |
| `OnConditionChanged(string value)` | `value not in {"OnError","Always"}` | `"Condition は OnError または Always を指定してください。"` |

エラーは `Dictionary<string, List<string>>` に保持され、`ErrorsChanged` イベントと `HasErrors` プロパティ（OnPropertyChanged 通知）で WPF バインディングへ伝搬する。

### 書き込み仕様（[Services/LooperService.cs:48-57](file:///E:/fabriq_studio/FabriqStudio/Services/LooperService.cs)）

```csharp
using var writer = new StreamWriter(filePath, append: false,
    encoding: new UTF8Encoding(encoderShouldEmitUTF8Identifier: true));   // UTF-8 BOM
using var csv = new CsvWriter(writer, CultureInfo.InvariantCulture);
csv.WriteRecords(entries);
```

- ヘッダはフィールド宣言順（**Enabled, ScriptPath, MaxRetry, IntervalSec, Condition, Description, Segment**）が CSV 列順となる。CommunityToolkit.Mvvm の `[ObservableProperty]` で生成されるパブリックプロパティが対象。

### 関連テンプレート

`LooperService.CreateLooperModuleAsync` は新規 Looper モジュール作成時に以下を一括生成する:

```
{fabriqRoot}/modules/extended/{module_name}/
├── module.csv          (1 行: script_looper.ps1 を Order=90, Enabled=0 で登録)
├── script_looper.ps1   (アプリ同梱テンプレートからコピー)
├── Guide.txt           (アプリ同梱テンプレートからコピー)
└── looper_list.csv     (UI で編集中のエントリ群を書き込み)
```

---

## 4. kernel/csv/log_destinations.csv — ログ出力先一覧

### 配置

```
{fabriqRoot}/kernel/csv/log_destinations.csv
```

ICsvService 引数: `"kernel/csv/log_destinations.csv"` （`/` 区切り）

### モデル

[Models/LogDestination.cs](file:///E:/fabriq_studio/FabriqStudio/Models/LogDestination.cs)

```csharp
public partial class LogDestination : ObservableObject
{
    [ObservableProperty] private string _path        = "";
    [ObservableProperty] private string _type        = "";
    [ObservableProperty] private string _enabled     = "0";
    [ObservableProperty] private string _authUser    = "";
    [ObservableProperty] private string _authPass    = "";
    [ObservableProperty] private string _description = "";
}
```

### カラム定義

| 列名 | 役割 | 例 |
|---|---|---|
| `Path` | 出力先パス（UNC / ローカル） | `\\fileserver\logs\fabriq\` |
| `Type` | 出力種別（fabriq の慣例値） | `"local"`, `"smb"`, `"webdav"` |
| `Enabled` | 有効フラグ（**`"0"`/`"1"` 文字列**） | fabriq の `"0"`/`"1"` 慣例。bool 型ではない |
| `AuthUser` | 認証ユーザー名 | SMB / WebDAV 用 |
| `AuthPass` | 認証パスワード | **暗号化扱いではない**（hostlist のような ENC: 仕様は未適用） |
| `Description` | 説明文 | 管理上のメモ |

### studio 側 ViewModel 仕様

`BasicParamsViewModel` が `kernel/csv/log_destinations.csv` を `ReadAsync<LogDestination>` で読み込み、`LogDestinations` `ObservableCollection<LogDestination>` に流し込む。`SaveAsync` で `WriteAsync` 経由で書き戻す。

### セキュリティ上の注意

`AuthPass` は **平文保持**。studio は hostlist のような ENC: プレフィクスベースの暗号化を log_destinations へは適用しない（運用上、共有アカウント / 共有資源に対するパスワードであり個別端末固有でないため）。共有秘匿が必要な場合は fabriq 側のシークレット管理機構の利用を検討すること。

---

## 5. kernel/csv/workers.csv — 作業者マスタ

### 配置

```
{fabriqRoot}/kernel/csv/workers.csv
```

ICsvService 引数: `"kernel/csv/workers.csv"`

### モデル

[Models/WorkerEntry.cs](file:///E:/fabriq_studio/FabriqStudio/Models/WorkerEntry.cs)

```csharp
public partial class WorkerEntry : ObservableObject
{
    [ObservableProperty] private string _iD   = "";   // 公開プロパティ名は ID（[ObservableProperty] による生成）
    [ObservableProperty] private string _name = "";
}
```

### カラム定義

| 列名 | 役割 | 例 |
|---|---|---|
| `ID` | 作業者識別子（社員番号 / 任意 ID） | `"E12345"`, `"y_suzuki"` |
| `Name` | 表示名 | `"鈴木 由樹"` |

### 注記

バッキングフィールドが `_iD`（小文字 i + 大文字 D）になっているのは、`[ObservableProperty]` Source Generator が `_iD` → 公開プロパティ `ID` に変換する規約（先頭文字を大文字化）に従わせるため。CSV 列名は **`ID`** で固定。

`BasicParamsViewModel` が `Workers` `ObservableCollection<WorkerEntry>` に保持し、UI 上のリスト編集・追加・削除を行ってから一括 `WriteAsync` で書き戻す。

---

## 6. kernel/csv/categories.csv — モジュール分類マスタ

### 配置

```
{fabriqRoot}/kernel/csv/categories.csv
```

ICsvService 引数: `"kernel/csv/categories.csv"`

### モデル

[Models/CategoryItem.cs](file:///E:/fabriq_studio/FabriqStudio/Models/CategoryItem.cs)

```csharp
public class CategoryItem
{
    public string Category { get; set; } = "";
    public int    Order    { get; set; }
}
```

### カラム定義

| 列名 | 役割 | 例 |
|---|---|---|
| `Category` | カテゴリ名（`module.csv` の `Category` と参照整合） | `"Apps"`, `"Scripts"`, `"Network"` |
| `Order` | カテゴリ表示順 | 整数。小さい順に表示 |

### 利用箇所

- `ModuleEditViewModel`（[ViewModels/ModuleEditViewModel.cs:70](file:///E:/fabriq_studio/FabriqStudio/ViewModels/ModuleEditViewModel.cs)）が読み込み、左ペインのカテゴリ一覧として表示。
- `module.csv` の `Category` 列が categories.csv に存在しない場合の挙動は **studio 側で自動補完しない**（fabriq 本体 / 運用者の手動整合に委ねる）。

---

## 7. HostListExportRequest / HostListExportResult — 端末一覧エクスポート DSL

CSV ではなく、Service 層 (`IHostListExportService`) と ViewModel 層を結ぶ **Request / Result DTO**（C# `record` 型 immutable オブジェクト）。fabriq_studio が採用する「Service I/O 境界は record で表す」パターンの代表例。

### モデル

[Models/HostListExportRequest.cs](file:///E:/fabriq_studio/FabriqStudio/Models/HostListExportRequest.cs)

```csharp
public record HostListExportRequest(
    IReadOnlyList<HostEntry> Hosts,
    string ParentFolder,
    string Memo,
    bool   Decrypt);

public record HostListExportResult(
    string ExportFolderPath,
    int    HostCount,
    int    DecryptedCells,
    int    RemainingEncCells,
    IReadOnlyList<string> Errors)
{
    public bool HasErrors => Errors.Count > 0;
}
```

### Request 各引数の意味

| 引数 | 役割 |
|---|---|
| `Hosts` | エクスポート対象 `HostEntry` の **UI 側スナップショット**（保存前の編集中状態でもよい） |
| `ParentFolder` | ユーザーが選択した親フォルダ。**この下にタイムスタンプ付きサブフォルダ**（例: `hostlist_export_20260507_1234/`）が生成される |
| `Memo` | 同梱される `README.txt` に書き込まれるユーザーメモ（空文字可） |
| `Decrypt` | `true` の場合、`ENC:` プレフィクス付きセルを **復号してから CSV に書き出す**。詳細は [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md) を参照 |

### Result 各引数の意味

| 引数 | 役割 |
|---|---|
| `ExportFolderPath` | 実際に生成されたエクスポートフォルダの **絶対パス** |
| `HostCount` | 出力された端末数（`Hosts.Count` と通常一致） |
| `DecryptedCells` | 復号成功したセル数。**`Decrypt=false` の場合は 0** |
| `RemainingEncCells` | 出力 CSV に **残った `ENC:` セル数**（復号失敗 + Decrypt=false により残ったもの） |
| `Errors` | 復号失敗等の人間可読エラーメッセージリスト |
| `HasErrors` (派生) | `Errors.Count > 0` |

### 不変性保証

`record` の **位置記録パラメータ** で宣言されているため、コンストラクタ後の値書き換えは不可（`with` 式で派生インスタンス生成は可能）。Service が同一 Request を複数 ViewModel から並列受信しても安全。

### Errors リストの設計意図

`HasErrors` は派生プロパティとして公開されており、ViewModel 側は `Errors.Any()` ではなく `result.HasErrors` を判定に使う。エラーメッセージは UI 上の警告ダイアログに **そのまま表示できる日本語文** が入っている前提で、Service 層が翻訳責任を持つ。

---

## クロスリファレンス

| 関連ドキュメント | 関係 |
|---|---|
| [fabriq_studio__reference__services_catalog.md](fabriq_studio__reference__services_catalog.md) | 上記 CSV を読み書きする `ICsvService` / `IModuleService` / `IModulePresetService` / `LooperService` の I/O シグネチャ |
| [fabriq_studio__reference__models_catalog.md](fabriq_studio__reference__models_catalog.md) | 全モデルの索引（`HostEntry` 等の大型モデル含む） |
| [fabriq_studio__reference__hostlist_csv_schema.md](fabriq_studio__reference__hostlist_csv_schema.md) | hostlist.csv 43 列スキーマ（本書では HostListExportRequest/Result のみ扱う） |
| [fabriq_studio__reference__pianist_profile_schema.md](fabriq_studio__reference__pianist_profile_schema.md) | Pianist プロファイル（pianist.json + procedure.csv + values.csv 等） |
| [fabriq_studio__reference__registry_catalog.md](fabriq_studio__reference__registry_catalog.md) | レジストリ辞書 catalog.json + RegConfigRow（reg_config.csv） |
| [fabriq__contracts__module_result.md](fabriq__contracts__module_result.md) | module.csv の `Script` が返す ModuleResult の契約 |

---

## 変更履歴

- 2026-05-07 初版作成（commit `3897c6e` を対象に 7 スキーマを集約）
